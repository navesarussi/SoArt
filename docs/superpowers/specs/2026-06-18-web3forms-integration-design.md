# Web3Forms Integration — Design Spec

**Date:** 2026-06-18  
**Status:** Approved  

---

## Summary

Replace the custom serverless registration function (`api/register.js`) with a direct client-side submission to [Web3Forms](https://web3forms.com). The form POST goes directly from the browser to `https://api.web3forms.com/submit`. Web3Forms delivers a notification email to the admin inbox and exposes submissions in a dashboard. No backend code, no secrets, no environment variables needed.

---

## Motivation

The current `api/register.js` required two external services with secrets (`RESEND_API_KEY` for email, `GITHUB_TOKEN` + `GITHUB_REPO` for storage). Neither was configured in production, causing all real form submissions to return HTTP 500 and lose data. Web3Forms eliminates both services and all secrets.

---

## Architecture

```
Browser → FormData POST → https://api.web3forms.com/submit
                                      ↓
                          Email → michal.foux97@gmail.com
                          Dashboard → app.web3forms.com/dashboard
```

The `api/track.js` analytics endpoint remains unchanged.

---

## Access Key

**Public key:** `34aabf78-bfb6-4514-967c-9edcb7467ab9`  
Web3Forms access keys are intentionally public and are safe to embed in client-side HTML. Security is enforced server-side by Web3Forms (domain allowlist + spam filtering).

---

## Changes

### `index.html` — form hidden fields

Replace the current hidden fields and honeypot with Web3Forms equivalents:

**Remove:**
- `<input name="_subject" ...>` → rename to `subject`
- `<input name="_template" ...>` → rename to `template`
- `<input name="_captcha" ...>` → remove (not a Web3Forms field)
- `<input name="_honey" type="text" ...>` → replace with `botcheck` checkbox

**Add / rename:**
```html
<input type="hidden" name="access_key" value="34aabf78-bfb6-4514-967c-9edcb7467ab9">
<input type="hidden" name="subject" value="New SoArt Movement registration">
<input type="hidden" name="from_name" value="SoArt Movement">
<input type="hidden" name="template" value="table">
<input type="checkbox" name="botcheck" class="honeypot-field" tabindex="-1" aria-hidden="true">
```

Update `form action`:
```html
<form ... action="https://api.web3forms.com/submit" method="POST">
```

### `script.js` — submit handler

Replace the `fetch("/api/register", ...)` block with a Web3Forms submission. Key differences:

| Before | After |
|---|---|
| Builds a custom JSON object | Uses `FormData` directly from the form |
| POSTs JSON to `/api/register` | POSTs FormData to `https://api.web3forms.com/submit` |
| Checks `result.ok` | Checks `data.success` |
| Sets `Content-Type: application/json` | Sets `Accept: application/json` (lets FormData set its own Content-Type) |

Fields appended programmatically (not in the form HTML):
- `language` — current UI language (`en` / `he` / `ar` / `es`)
- `source_path` — `window.location.pathname`

The localized sending / success / error messages and the `form.reset()` on success remain unchanged.

### `api/register.js` — deleted

This file is removed. Web3Forms replaces its functionality. No new serverless function is needed.

### `.env.example` — updated

Remove `RESEND_API_KEY`, `FORM_TO_EMAIL`, `FORM_FROM_EMAIL`, `GITHUB_TOKEN`, `GITHUB_REPO`, `GITHUB_STORAGE_BRANCH`, `GITHUB_STORAGE_PATH`. Update to reflect that no environment variables are required for the registration flow.

---

## Data Stored per Submission

Web3Forms receives and stores:

| Field | Source |
|---|---|
| `fullName` | form input |
| `email` | form input |
| `country` | form input |
| `role` | form select |
| `message` | form textarea |
| `language` | appended by JS |
| `source_path` | appended by JS |

---

## Spam Protection

Web3Forms uses the `botcheck` honeypot checkbox (hidden from real users; bots may fill it in). Web3Forms also applies its own server-side spam scoring. The previous custom `_honey` text-field honeypot in `api/register.js` is no longer needed.

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| `data.success === false` | Throw error → show localized `submitError` message |
| Network failure / fetch throws | Catch block → show localized `submitError` message |
| Bot submission (botcheck filled) | Web3Forms rejects silently server-side |

---

## Files Not Touched

- `styles.css` — responsive title fix from the previous session is already in place
- `.gitignore` — `data/` exclusion from the previous session is already in place
- `api/track.js` — page-view analytics, unchanged
- `about.html`, `approach.html`, all assets

---

## Out of Scope

- Confirmation / autoresponder email to the registrant (Web3Forms Pro feature; not included)
- Google Sheets sync (Web3Forms Pro feature; not included)
- Localized admin notification email (Web3Forms sends a fixed-format table; language is visible as a field value)
