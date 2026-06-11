# Wiring the signup field to Buttondown

**Date:** 2026-06-10
**Status:** Approved (design)
**Scope:** `index.html` only — single static page on GitHub Pages.

## Problem

The landing page's beta signup form (`index.html`, the `#signup` block) validates
the email client-side and then shows a "You're on the list" message, but the
address goes **nowhere** — the submit handler is explicitly a client-side stub
(`// signup (client-side only)`). We want submitted emails to actually reach a
place we can broadcast the beta launch from.

## Decision

Send signups to **Buttondown** (a newsletter service), so the captured addresses
land in a real subscriber list we can broadcast from when the beta ships — no
export/import step later.

- **Username:** `anya-ai`
- **Endpoint:** `POST https://buttondown.com/api/emails/embed-subscribe/anya-ai`
- **Opt-in:** confirmation/welcome email is **off** in Buttondown, so addresses
  are added immediately and the success copy can say "You're on the list."

Hosting stays on **GitHub Pages** — no migration, no backend, no serverless
function.

## Why this is safe on a public static page

Buttondown's `embed-subscribe` endpoint is **public** and keyed only to the
public username — it requires **no API key**. The only Buttondown-specific value
baked into the HTML is the username `anya-ai`, which is already public (it's the
`buttondown.com/anya-ai` URL). No secret is exposed by committing this.

## Design

### Markup (progressive enhancement)

Make the form natively functional so it works even if JS fails, then enhance:

- Set on the `<form>`: `action="https://buttondown.com/api/emails/embed-subscribe/anya-ai"`
  and `method="post"`.
- Add `name="email"` to the email `<input>` (it currently has only `id`, so a
  native POST would send nothing).
- Add a hidden field `<input type="hidden" name="embed" value="1" />`.

With JS disabled, submitting does a real cross-page POST and Buttondown shows its
own confirmation page. With JS enabled, we intercept and keep the inline UX.

### Behavior (enhanced path)

The existing submit handler is extended (not replaced):

1. Keep the current client-side email regex validation — bad input is blocked
   before any network request and the field border turns crimson, as today.
2. On valid input: `preventDefault`, then fire the subscription in the background
   with `fetch(action, { method:'POST', mode:'no-cors', body: <form-encoded email + embed=1> })`.
   - `mode:'no-cors'` is required for a cross-origin POST from a static page; the
     body uses `URLSearchParams`/`FormData` (a "simple" content type) so there is
     no CORS preflight and the request goes through.
3. The `no-cors` response is opaque (we cannot read status), so success is shown
   **optimistically** — reveal the already-styled inline success (`#signup.done`).
4. A genuine **network failure still rejects the fetch**, so `.catch()` shows a
   gentle retry state (re-enable the button, show an error hint) rather than a
   false success.
5. While the request is in flight, disable the submit button and guard against
   double-submit.

### Copy

Keep the existing success line: **"You're on the list. Don't make her wait."**
(confirmation email is off, so this is accurate).

## Components / data flow

```
[ visitor types email ]
        |
        v
[ index.html form ] --(client regex)--> invalid -> crimson border, focus, stop
        | valid
        v
  fetch POST (no-cors, form-encoded: email, embed=1)
        |
        v
[ buttondown.com/api/emails/embed-subscribe/anya-ai ]
        |
        +-- resolves (opaque) ----> show inline success (#signup.done)
        +-- network reject -------> show retry hint, re-enable button
```

There is no state owned by this page beyond the in-flight flag; the subscriber
list lives entirely in Buttondown.

## Out of scope

- The pre-existing `your-domain.com` placeholder in the `twitter:image` meta tag
  (unrelated to this change — left as-is).
- Any hosting migration to Cloudflare (explicitly not happening).
- Honeypot/extra spam protection — Buttondown handles basic abuse; can revisit if
  spam signups appear.

## Verification

The page is static, so verification is manual via a local server + browser:

1. **Invalid email** (`foo`, empty) → blocked, crimson border, no request fired.
2. **Valid email** → exactly one POST to the embed-subscribe URL with
   `email` and `embed=1` in the body (checked in the network panel); inline
   success appears; no page navigation.
3. **Forced network failure** (offline / blocked host) → retry hint shown, button
   re-enabled, no false success.
4. **No-JS fallback** → with JS disabled, submit performs a native POST and
   Buttondown's own confirmation page loads.
5. **End-to-end** → a test address actually appears as **subscribed** (not
   pending) in the Buttondown dashboard. If it lands as pending despite the
   welcome/confirmation toggle being off, flag it and revisit the success copy.
