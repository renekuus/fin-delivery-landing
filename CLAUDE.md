# CLAUDE.md — fin-delivery-landing

Auto-loaded context for agents, and a quick reference for humans. This repo is the **public
marketing site** at `https://www.fin-delivery.com`, including the two public apply forms that
feed the EcoBite backend.

> **This is NOT the monorepo.** The backend, dashboard and mobile apps live in a separate
> private repo (`~/Ecobite`, `renekuus/Ecobite`) with its own `CLAUDE.md` and a very different
> deploy runbook. Nothing here shares its tooling: no pnpm, no build, no framework, no CI.

---

## 1. What this is

Plain static HTML. Every page is **one self-contained file** with its CSS in a `<style>` block
and its JS in a `<script>` block at the bottom. No bundler, no npm, no dependencies except two
Google Fonts links. Open a file in a browser and it works.

That is a deliberate choice, not an accident: the site must stay editable and deployable by
anyone with a text editor and `git pull`. **Do not introduce a build step, a framework or a
package.json** without an explicit decision to change the deploy model too.

```
index.html                          Homepage (FI/EN, animated hero)
partner/index.html                  Merchant pitch page      → links to hakemus/
partner/hakemus/index.html          MERCHANT APPLY FORM      → POSTs to the API
become-a-courier/index.html         Courier pitch page       → links to hakemus/
become-a-courier/hakemus/index.html COURIER APPLY FORM       → POSTs to the API
privacy/index.html                  Privacy policy (single language)
robots.txt · sitemap.xml · og-image.png
```

`.gitignore` is one line: `*.bak*`. Editing on the server produces `index.html.bak` files —
they are ignored so a `git pull` never trips over them.

Both `hakemus/` pages and both pitch pages carry `<meta name="robots" content="noindex,
nofollow">`; only the homepage is in `sitemap.xml`.

## 2. Deploy — git pull, nothing else

The server (`13.60.125.111`) serves `/var/www/landing` with nginx, and **that directory is a
git checkout of this repo** (owned by `ubuntu`).

```bash
# local
git push origin main
# on the server
cd /var/www/landing && git pull
```

That is the whole deploy. **No `pnpm install`, no build, no `pm2 restart`, no migration** —
nginx serves the files from disk, so the change is live the moment the pull finishes. Hard-
refresh to beat the browser cache.

⚠️ **This is NOT the monorepo's deploy.** The backend has a strict `&&`-chained build/restart
runbook (its `CLAUDE.md` §10). Never run that here, and never run this there.

## 3. The two apply forms — the API contract

Both forms talk to the production API at **`https://api.fin-delivery.com`** (hard-coded as
`const API` at the top of each script; there is no environment switching).

| Form | Endpoint | Success |
|---|---|---|
| `partner/hakemus/` | `POST /api/v1/register/merchant` | `201` → swap to `#doneScreen` |
| `become-a-courier/hakemus/` | `POST /api/v1/register/courier` | `201` → swap to `#doneScreen` |
| both, per file | `POST /api/v1/register/upload?type=<slot>` | `201 {key}` → stashed in `keys[slot]` |

**Uploads happen BEFORE submit.** Each `<input type="file" data-upload="<slot>">` uploads on
`change`, the returned `key` is kept in the in-memory `keys` object, and submit sends
`<slot>_key: keys[slot]`. A failed upload just clears that key — the applicant can retry. The
10 MB cap is checked client-side for a fast message; the server enforces it for real.

Upload slots — merchant: `elintarvikeilmoitus`, `omavalvonta`, `alcohol_license`.
Courier: `id_document`, `work_permit`, `tax_card`, `drivers_license`, `insurance`.

CORS: the API allowlists `https://www.fin-delivery.com` and `https://fin-delivery.com`
explicitly. A new hostname for this site needs a matching change in the API's CORS list.

## 4. Merchant category — a CROSS-REPO contract

`partner/hakemus/` submits a `<select>` whose **values are DB-facing** and whose labels are
translated:

| `value` (sent) | FI label | EN label |
|---|---|---|
| `QSR` | QSR | QSR |
| `Restaurant` | Ravintola | Restaurant |
| `Store` | Ruokakauppa | Grocery store |
| `Darkstore` | Darkstore | Darkstore |
| `Other` | Muu | Other |

**These five values must stay in sync with the API's accepted set** (`MERCHANT_CATEGORY_OPTIONS`
in the monorepo's `@ecobit/shared`). The API rejects anything else with `category_not_chosen`,
so a typo here breaks every merchant application silently until someone tries to apply.

Two things that are easy to get wrong:

- **`grossi` is gone.** It was the grocery group and was removed from the database enum
  entirely (monorepo migration 068). Grocery is **`Store`** now. Never reintroduce `grossi`.
- **`Store` must not become a dark store.** The API maps the dropdown value to a
  `merchant_group` by exact lookup, `Store → store`. An older free-text path matched the word
  "store" to `darkstore`, which would price and merchandise a grocery shop as a dark store.
  This is why the field is a fixed `<select>` and not free text.
- **Casing is forgiven, but do not rely on it.** This form sends `Other` (capital O) while the
  API's canonical value is lowercase `other`; the API matches case-insensitively and stores its
  own canonical casing, so it round-trips. Verified against production. Still, if you touch
  either side, match the API's canonical spelling rather than leaning on that leniency.

## 5. Inline field errors — the `details` contract

Both forms paint the exact failing input red and print a bilingual sentence under it. This
depends on an API response shape, so it is a **contract, not an implementation detail**:

```jsonc
// 400 (validation_failed) or 409 (conflict)
{ "error": { "code": "validation_failed", "status": 400, "message": "…",
  "details": {
    "email":    { "code": "invalid_email", "fi": "Tarkista sähköposti…", "en": "Please check…" },
    "y_tunnus": { "code": "required",      "fi": "Ole hyvä ja täytä…",  "en": "Please fill in…" }
  } } }
```

**THE KEY IS THE INPUT'S `name` ATTRIBUTE.** `applyServerErrors()` looks the field up with
`document.querySelector('[name="' + key + '"]')` and walks up to its `.field` / `.check` /
`.seg` wrapper. A `details` key with no matching input renders nothing inline — the code
detects that (no field matched → `first` is null) and falls back to the red banner, so nothing
is ever silently swallowed. Renaming an input `name` therefore breaks the highlight for that
field; keep them identical to the API's field names.

`applyServerErrors()` also accepts the **legacy** shape (`field: ["message"]`) and plain
strings, so an older API cannot break the page — it just shows one language.

Flow on submit:

1. `clearAllErr()` wipes previous errors.
2. Client-side checks run first (required, email regex, phone ≥7 digits, y-tunnus
   `\d{7}-\d`, age ≥18 on the courier form, conditional permit/licence rules). First bad field
   is scrolled into view; **nothing is sent**.
3. Only if that passes does the request go out. A `201` swaps in the thank-you screen.
4. On a non-201, `error.details` drives the inline errors; otherwise `error.code` picks a
   form-level banner (`conflict` → duplicate, `rate_limited` → too many attempts,
   `validation_failed`/`invalid_upload` → generic, anything else → generic).
5. Editing any input clears that field's error (`input`/`change` listener).

Client-side checks are **UX only** — deliberately looser than the server (e.g. IBAN and
henkilötunnus are not validated here at all; the API rejects them with `invalid_iban` /
`invalid_personal_id` and the form paints them red). Never assume a value reached the API
because the form accepted it.

## 6. FI/EN — TWO different patterns, know which page you are on

**Pattern A — translation object (`T`).** Used by `partner/index.html`,
`become-a-courier/index.html` and **both apply forms**.

```html
<h1 data-i="title">Ryhdy kumppaniksi</h1>
```
```js
const T = { fi: { title:'Ryhdy kumppaniksi', … }, en: { title:'Become a partner', … } };
function setLang(l){ …document.querySelectorAll('[data-i]').forEach(el => { … el.textContent = T[l][el.dataset.i] }) }
```

- Every visible string needs a `data-i` key present in **both** `T.fi` and `T.en`. A key
  missing from one language leaves the other language's text on screen — silent, not an error.
- `textContent` is used, so a `data-i` element **cannot contain markup**. Strings with a link
  (terms/privacy checkboxes) keep the link in the HTML and translate only the surrounding
  `<label>`, which is why some `T` values read slightly differently from the HTML.
- The forms additionally hold error copy in `ERRT` (client-side messages) and in each rendered
  error's `dataset.fi` / `dataset.en` (server messages), so `renderErrLang()` re-renders
  visible errors on a language switch. **New error copy goes in `ERRT`, both languages.**
- These pages default to **FI on every load** (`setLang('fi')` at the end of the script) and do
  **not** persist the choice.

**Pattern B — duplicated markup toggled by CSS.** Used by `index.html` only:
`[data-fi]` / `[data-en]` elements are shown/hidden, and the choice **is** persisted in
`localStorage`. There is no `T` object there.

Consequence worth knowing: a visitor who picks EN on the homepage lands on a form in FI. That
is current behaviour, not a bug report — change it deliberately if you change it.

`privacy/index.html` has no i18n at all.

## 7. Conventions

- **Edit the file, not a copy.** No partials, no includes — the header/footer markup is
  duplicated per page on purpose. Changing the nav means editing each file.
- Keep the CSS custom properties (`--forest`, `--green`, `--mint`, …) rather than hard-coding
  colours; they are redefined per file but with the same names and values.
- The apply forms use `novalidate` on the `<form>` so the browser's own bubbles never fire —
  all validation feedback is the inline red state. Do not remove it.
- `TC_VERSION` (currently `'2026-07'`) is sent as `tc_version` with every application and is
  what the backend records as the accepted terms version. Bump it when the terms change.
- Test a form change by actually submitting it against the live API — a `400` creates nothing,
  so a deliberately-invalid submit is a safe way to check the error rendering.
