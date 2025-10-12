# KeyVault – KV Checker

Secure Xbox 360 KeyVault (KV) checker with a same-origin PHP proxy. No raw KV retention. Detailed UI feedback. Privacy-first design.

<p align="center">
  <a href="https://xbox360kvchecker.com/" target="_blank" rel="noopener noreferrer">
    Visit KeyVault – KV Checker
  </a>
</p>

---

## Contents
- [What it does](#what-it-does)
- [Why it is safe](#why-it-is-safe)
- [How it works](#how-it-works)
- [UI walkthrough](#ui-walkthrough)
- [Security model](#security-model)
- [Privacy model](#privacy-model)
- [Content Security Policy](#content-security-policy)
- [Data flow](#data-flow)
- [API contract](#api-contract)
- [Performance and limits](#performance-and-limits)
- [Self-hosting](#self-hosting)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [Roadmap](#roadmap)
- [Community and use policy](#community-and-use-policy)
- [Add images to this README](#add-images-to-this-readme)
- [License](#license)

---

![KV Checker UI](https://xbox360kvchecker.com/social/kvchecker-1200x630.png.png)

## What it does
- Accepts a `KV.bin` file (≤ 256 KiB).
- Computes SHA-1 client-side for quick recognition. Never uploads the hash value as a secret; it is for display.
- Sends the raw KV to a **same-origin** PHP proxy at `/kv-check.php`.
- Proxy attaches server-only headers (API key, optional HMAC) and calls the local KV API.
- Displays classification: `Unbanned` or `Banned`, plus latency and counters.
- Shows a **Walkthrough & Live Raw** panel during checks with a sanitized JSON response.
- Animates staged progress: **Upload → Auth → AP1 → AP2 → TGS → Decrypt → Done**.
- Status chip switches to **“Status: KV Checked”** after any completed check.
- **Clear** resets all on-page state.

---

## Why it is safe
- **No raw KV retention**: Uploads are streamed, processed, discarded. Temporary OS files rotate per server defaults.
- **Server secrets never in the browser**: API key/HMAC headers are added only by the proxy.
- **Tight CSP** and **no embeds** reduce exfiltration vectors.
- **No browser storage** of KV material: no cookies, no LocalStorage, no IndexedDB.
- **Minimal counters** on the server (e.g., first check flag, last-checked timestamp) keyed by safe identifiers, not the raw KV.

---

## How it works
1. **File intake**  
   - User drops or selects `KV.bin`. Size is validated (≤ 262,144 bytes).
2. **Local derivations**  
   - Browser computes SHA-1 of the full file to display a prefix.  
   - Console ID extracted from bytes at offset `0x09CA` (5 bytes).
3. **Progress rings**  
   - UI animates ring stages while the request is in flight.
4. **Same-origin POST**  
   - `multipart/form-data` to `POST /kv-check.php` on the **same domain**.
5. **Proxy authentication**  
   - Proxy injects `Authorization`/HMAC headers. Browser never sees them.
6. **Backend checks**  
   - Performs Xbox flow: Auth, AP1/AP2, TGS, Decrypt. Returns JSON.
7. **UI mapping**  
   - Status, latency, counters, and console ID are revealed with masked animation.
8. **Live Raw**  
   - JSON is sanitized in-browser before display. Potential sensitive fields like `kdcNonce` and `kvHashPrefix` are dropped.

---

## UI walkthrough
- **Header chips**  
  - API connectivity probe. Status chip shows `Awaiting File`, `Checking…`, or `KV Checked`.
- **Upload section**  
  - Drag-and-drop zone plus Select button. Selected file name and size shown.
- **Stages**  
  - Centered rings with arrows. Each stage flips `✓` on success or `✕` on failure.
- **Stats grid**  
  - Status, Console ID, KV SHA-1 prefix, Response Time (ms), Checks (API), Last Checked.
- **Walkthrough & Live Raw**  
  - Opens automatically while checking. Closes on **Clear**.  
  - Live JSON response (sanitized) with `aria-live` for assistive tech.

---

## Security model
- **Threats considered**
  - Exfiltration via third-party scripts or frames.  
  - Token/header leakage to the browser.  
  - KV persistence on disk or in logs.
- **Mitigations**
  - `default-src 'self'` CSP, `frame-ancestors 'none'`, `form-action 'self'`.  
  - Proxy-only headers for backend auth and signing; never rendered client-side.  
  - Logging omits raw KV. Metrics bounded and rotated.  
  - HTTPS required. Mixed content blocked/upgraded.
- **Out of scope**
  - Compromised user machines or malicious browser extensions.  
  - Mods that alter how KVs are created.  
  - Upstream service changes by Microsoft.

---

## Privacy model
- **No raw KV storage** beyond transient upload handling.  
- **No third-party analytics**.  
- **No local persistence** in the browser.  
- **Optional counters**: first check, last-checked display string, check counts. These are keyed by safe identifiers.

---

## Content Security Policy
The page ships with a strict CSP. Example:
```html
<meta http-equiv="Content-Security-Policy"
  content="
  default-src 'self';
  connect-src 'self';
  img-src 'self' data: https://xbox360kvchecker.com;
  font-src 'self';
  script-src 'self' 'unsafe-inline';
  style-src 'self' 'unsafe-inline';
  object-src 'none';
  base-uri 'self';
  frame-ancestors 'none';
  form-action 'self';
  upgrade-insecure-requests">
