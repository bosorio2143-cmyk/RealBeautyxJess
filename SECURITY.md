# RealBeautyXJess PWA — Security Guide

## Zero-Exposure API Key Policy

**Rule: No API keys, tokens, or secrets ever go in `index.html` or any client-side file.**

This app is a pure front-end PWA. All sensitive operations (email notifications, form submissions, analytics) must be routed through a back-end proxy or serverless function.

---

## Current Security Layers (Built Into index.html)

| Layer | What It Does |
|---|---|
| `sanitize()` | Strips all HTML tags, `<script>`, event handlers from every user input |
| `validateVaultCode()` | Enforces A-Z/0-9 only, 12-char max, constant-time comparison |
| `validateProcDate()` | Rejects future dates, NaN, injections, dates before 2015 |
| `validateBookingFields()` | Sanitizes name, phone, email, notes before any DOM rendering |
| DOM-only rendering | All user data rendered via `textContent` — never `innerHTML` |
| Zero-Knowledge Diary | Healing state saved only to `localStorage` on user's device |

---

## For Future Email Notifications (e.g. Booking Confirmations)

### Option A — Formspree (Simplest, No Back-End Needed)

1. Sign up at [formspree.io](https://formspree.io)
2. Create a form → copy your endpoint (e.g. `https://formspree.io/f/xaqlbnwd`)
3. In `index.html`, update `submitBk()` to POST to Formspree:

```javascript
// Inside submitBk() after validation passes:
const fields = validateBookingFields();
if (!fields) return;

fetch('https://formspree.io/f/YOUR_ENDPOINT_ID', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name: fields.name,
    phone: fields.phone,
    email: fields.email,
    notes: fields.notes,
    service: document.getElementById('sumS').textContent,
    date: document.getElementById('sumD').textContent,
  })
})
.then(r => r.ok ? showConfirmation() : showToast('Submission failed. Please text Jess directly.'))
.catch(() => showToast('Network error. Please text Jess directly.'));
```

> **Note:** Formspree endpoint IDs are semi-public (rate-limited, spam-protected). Do NOT put a full private API key here.

---

### Option B — GitHub Actions + Secrets (If You Add a Back-End)

If you ever add a serverless function (Netlify Functions, Vercel Edge, Cloudflare Workers):

1. Go to your GitHub repo → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Add secrets like:
   - `SENDGRID_API_KEY`
   - `FORMSPREE_KEY`
   - `NOTION_API_KEY` (if logging bookings)

4. Reference them in your workflow file (`.github/workflows/deploy.yml`):

```yaml
- name: Deploy with secrets
  env:
    SENDGRID_API_KEY: ${{ secrets.SENDGRID_API_KEY }}
  run: node functions/send-confirmation.js
```

> **Critical:** Secrets injected via GitHub Actions are **never** exposed in logs, build output, or to client-side JavaScript. They only exist server-side.

---

### Option C — Environment Variables (Netlify / Vercel)

If hosting on Netlify or Vercel instead of GitHub Pages:

**Netlify:**
1. Go to Site → **Site configuration** → **Environment variables**
2. Add your key (e.g. `SENDGRID_API_KEY = sk-xxxx`)
3. Access it in a Netlify Function:
```javascript
// netlify/functions/send-booking.js
exports.handler = async (event) => {
  const key = process.env.SENDGRID_API_KEY; // never exposed to browser
  // ... send email logic
};
```

**Vercel:**
1. Go to Project → **Settings** → **Environment Variables**
2. Same pattern — never accessible client-side.

---

## Vault Code Management

The vault access code (`JESS2026`) is stored directly in `index.html`.

**For a public repo:** This is acceptable for a client-facing portal because:
- The code protects convenience features (PDF downloads), not financial data
- It can be rotated easily by updating one line
- Real secrets (payment, PII) should NEVER be in a front-end file

**To rotate the vault code:**
1. Open `index.html`
2. Find: `const VAULT_CODE = 'JESS2026';`
3. Change to your new code (A-Z, 0-9, max 12 chars)
4. Commit and push — GitHub Pages will redeploy in ~30 seconds

---

## What NEVER Goes In index.html

| Type | Why |
|---|---|
| SendGrid / Mailgun API keys | Exposed to anyone who views source |
| Stripe publishable key | Use Stripe.js hosted fields instead |
| Database connection strings | No databases in client-side code |
| Admin passwords | Never in any front-end file |
| Private JWT secrets | Server-side only |

---

## localStorage Privacy Note

The Healing Diary stores the user's procedure date and procedure type in `localStorage` under the key `rbxj_diary_state`. This data:

- **Never leaves the device** — no server calls, no analytics, no logs
- **Is not in the GitHub repository** — client data is never committed
- **Can be cleared** by the user at any time via browser settings
- **Survives page refreshes** so users don't re-enter their date every visit
- **Is validated on read** — corrupt or injected data is silently discarded
