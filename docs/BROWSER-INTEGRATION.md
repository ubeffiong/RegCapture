# Using the fingerprint app from a website

> Browser and reader on different machines (Citrix/RDP/cloud)? See [RELAY.md](RELAY.md). Any web stack: use `web-sdk/` (`<fingerprint-capture>` element, ES module, or script tag); the capture reply is `{ "status": "success", "template": "<base64>", ... }`.

Your setup is sound and is the standard pattern: **website in the browser -> desktop app on the visitor's own PC**. This page
explains why it works, why it fails in production, what this project does about each cause, and what is verified.


## The three launch links (the link decides which UI appears)

| Link | What happens |
|---|---|
| `regcapture://start` | Starts the app **minimised** and never takes focus. This is what the web SDK uses: your own page is the UI. Also the default for a bare `regcapture://`. |
| `regcapture://open` | Shows the **desktop window** (hands + capture), raising it if the app is already running. |
| `regcapture://web` | Shows the app's **small web page** (`http://127.0.0.1:12345/ui`) as a browser popup (Edge/Chrome app window, else the default browser). Same hand picker and finger dropdown, no desktop window. |

From JavaScript: `fp.openApp()` (desktop) or `fp.openApp('web')`. Normal capture needs none of them.


## Operator captures in the app, your page receives it

If the person prefers the app's own window (or its hands UI) for scanning, your page can wait for the result instead of starting the scan:

```js
const r = await fp.receive({ timeoutSeconds: 90 });   // resolves with { status:"success", template, ... } when a finger is captured in the app
```

(`POST /api/fingerprint/ {"action":"receive","timeoutSeconds":90}` over HTTP; works through the relay too.) The desktop window shows
"A website is waiting for a fingerprint" while a page is waiting; the next successful capture or verify made there is delivered to
the waiting pages. Nothing is ever pushed: a page must ask, with the same authorization as a capture (approved website or API key),
and with nobody waiting a desktop capture stays in the app. A request that times out returns `code: "timeout"` (HTTP 408).

## Your questions, answered

**When a user opens my site over the internet, does `localhost` mean *their* PC?**
Yes. The request is made by JavaScript running *in the visitor's browser*, so `127.0.0.1` / `localhost` is the visitor's own
computer. Your web server never sees it. That is exactly why the pattern works - and why it breaks for remote-desktop users
(see "When this pattern cannot work").

**Can the web app call the desktop app installed on the client PC?**
Yes, subject to the browser rules below. The web app talks to the app's loopback service
(`http://127.0.0.1:12345` and, if enabled, `https://127.0.0.1:12346`).

**Why did it work in development and fail in production?**
In development the page is `http://localhost:3000`: the same machine, so none of the rules apply. In production the page is a
*different, remote* website, so the browser applies cross-site and local-network rules. In order of how often they bite:

| # | Cause | Applies to | Fixed by |
|---|---|---|---|
| 1 | **The app did not allow your site's origin (CORS).** A shared API key cannot fix this: any secret in page JavaScript is public. | all browsers | Origin allow-list + user approval ("pairing"), built in |
| 2 | **Chrome/Edge "Local Network Access" (v142+).** A public site reaching `127.0.0.1` needs the user's permission. `https://` sites get a prompt; **public `http://` sites are blocked** (measured, below). | Chrome, Edge | Serve your site over HTTPS; the SDK waits for the prompt and explains denial; for http sites use the policy script |
| 3 | App not running / port taken / blocked by security software | all | SDK starts the app via `regcapture://`; clear diagnosis |
| 4 | `localhost` resolves to IPv6 `::1` but the app listened on IPv4 only | all | The service listens on both |
| 5 | Your site's Content-Security-Policy forbids the connection | all | Add the addresses to `connect-src` (below) |
| 6 | Mixed content (`https` page -> `http` local address) | **Safari only** | Local HTTPS option |

> **Correction to a common claim:** "HTTPS blocks HTTP localhost (mixed content) - 90% of your issue." That is **not true for
> Chrome, Edge or Firefox**: they treat `http://127.0.0.1` and `http://localhost` as secure origins and allow the call from an
> `https://` page. Only Safari blocks it. The failure you see in production is almost always cause 1 or 2.

**Is the HTTPS option (A) the right call?**
It is supported, and it is the more conservative choice (no dependence on browser exceptions for plain http). It is **optional**
because on Windows' mainstream browsers plain `http://127.0.0.1` already works from HTTPS pages, and installing a trusted
certificate on every PC has a cost. The SDK tries HTTPS first from HTTPS pages and falls back to HTTP, so both work.
You do not need a relay server (option B) for the normal case.

## How it works

```
 Visitor's PC                                                            Your server
 +---------------------------------------------------------+            +----------+
 |  Browser                                                |            |          |
 |   page from https://hmis.example.org  <------------------------------|  web app |
 |     |  fingerprint-client.js                            |            +----------+
 |     |  GET  /hello      (public, CORS-readable)         |
 |     |  POST /pair       (user approves in a desktop prompt)
 |     v  POST / {capture}                                 |
 |  RegCapture (desktop app)                      |
 |     Kestrel on 127.0.0.1 + ::1 : 12345 (http) / 12346 (https)
 |     checks: Host is loopback -> Origin allowed? -> action|
 |     -> FingerprintSession -> reader                     |
 +---------------------------------------------------------+
```

### Security model

| Threat | Defence |
|---|---|
| Any website uses the reader | Only origins listed by an administrator or **approved by the person at the PC** (a prompt only they can click; "Don't allow" is the default) |
| Page forges its identity | The browser sets `Origin`; page JavaScript cannot change it. Opaque (`null`), non-http and malformed origins are refused |
| DNS rebinding (attacker's domain re-pointed to 127.0.0.1) | `Host` header must be `localhost`, `127.0.0.1` or `[::1]` on the right port, else `421` |
| "Allow everything" misconfiguration | No wildcard-all. `https://*.example.org` matches subdomains only (not the apex, not `evilexample.org`) |
| Prompt spam | One prompt at a time; 3 refusals => 10 minutes' silence; the page can't see data until approved |
| Stolen API key | Browsers don't use it. Non-browser callers (no `Origin`) still need `X-Api-Key`; a valid key does **not** bypass the origin check |
| TLS key theft | Per-machine key, **non-exportable**, in the user's own store; certificate is not a CA and valid only for loopback names + server-auth; no private key ships in the installer |
| Scan left running after the tab closes | The scan is bound to the HTTP request and is cancelled when the browser aborts |

## Deployment scenarios

| Your site | Chrome / Edge | Firefox | What to do |
|---|---|---|---|
| **Cloud, `https://`** | Works after a **one-time browser permission prompt** ("allow access to apps on your device"). The SDK waits for it and, if denied, tells the user where to re-enable it. | Works | Use the SDK. Add your origin to `AllowedOrigins` to skip the app-side prompt, or let users approve it once |
| **Facility server, `https://` (internal CA)** | Same as cloud | Same | Recommended for local deployments: put a certificate on the facility server |
| **Facility server, `http://` on a private address** (192.168.x, 10.x, 172.16-31.x) | **Not blocked in Chrome/Edge 157** (measured); Chrome's documentation says non-secure pages are denied, so this may change | Works | Works today; plan to move to HTTPS. Optionally allow-list the origin with `Set-BrowserLocalAccessPolicy.ps1` (in `RegCapture-AdminTools.zip` on the release page) |
| **Public `http://`** (cloud over plain HTTP) | **Blocked** (measured: request fails, permission "denied") | Works | Move the site to HTTPS, or allow-list the origin by browser policy (script below). There is no client-side workaround |
| `http://localhost:3000` (development) | Works | Works | - |
| Safari / WebKit | Blocks `https` page -> `http` loopback (mixed content) | - | Enable "secure browser access" in the app so HTTPS is available. *(not tested here)* |

## Integrating your web app

1. Copy `web-sdk/fingerprint-client.js` (and optionally `fingerprint-client.d.ts`) into your project. No dependencies.
2. Allow the connections in your Content-Security-Policy (if you set one):
   `connect-src 'self' http://127.0.0.1:12345 https://127.0.0.1:12346 http://localhost:12345 https://localhost:12346;`
3. If you embed the page in an **iframe**, the top-level page must send
   `Permissions-Policy: local-network=(self "https://embedded.example"), loopback-network=(self "https://embedded.example")`
   and the iframe needs `allow="local-network; loopback-network"` (Chrome's rule for cross-origin frames).
4. Use it:

```js
import FingerprintClient from './fingerprint-client.js';

const fp = new FingerprintClient({
  onStatus: (code, msg) => showBanner(msg),          // e.g. "Waiting for you to allow access in the browser prompt"
});

try {
  await fp.connect();                                // finds the app, handles the browser prompt and pairing
  const r = await fp.capture('RightIndex', {
    onPhase: (p) => setPhase(p),                     // "WaitingForFinger" | "Processing" | ...
    signal: abortController.signal,                  // aborting cancels the scan on the PC
  });
  save(r.fingerprintData /* base64 8-bit image */, r.width, r.height, r.quality);
} catch (e) {
  showError(e.hint || e.message, e.code);            // plain-language hint for every failure
}
```

Errors always carry a stable `code` (`app_not_running`, `permission_denied`, `insecure_page`, `pairing_denied`, `busy`,
`duplicate`, `timeout`, `no_device`, ...) and a user-facing `hint`. `await fp.diagnose()` returns a report of every endpoint
tried, the browser's permission state and the page's security context - attach it to support tickets.

### Administrators: pre-approving sites (no prompt on the PC)

`%AppData%\GenericFingerPrint\settings.json` (or ship it with your deployment tooling):

```json
{ "Api": { "AllowedOrigins": [ "https://hmis.hospital.org", "https://*.hospital.org" ], "PairingEnabled": false } }
```

`PairingEnabled: false` makes the list exclusive. For Chrome/Edge's own permission prompt, allow-list the site by policy
(preview, then apply from an elevated prompt):

```powershell
.\Set-BrowserLocalAccessPolicy.ps1 -Origins https://hmis.hospital.org          # preview only
.\Set-BrowserLocalAccessPolicy.ps1 -Origins https://hmis.hospital.org -Apply   # write
.\Set-BrowserLocalAccessPolicy.ps1 -Origins https://hmis.hospital.org -Remove -Apply
```

### Enabling HTTPS on a PC

In the app: *Local API and website access -> Enable secure browser access*. The app creates a certificate for that Windows user
only and asks Windows to trust it (a standard Windows confirmation - the app cannot do this silently). **Turn off** removes it.
`tools/Remove-LocalHttpsCertificate.ps1` removes it for a user without the app. Firefox uses its own certificate store and keeps
using the plain http address, which it allows.

## Troubleshooting

| What the user/developer sees | Likely cause | Fix |
|---|---|---|
| `app_not_running` right after connect | App closed, port in use, security software | Start the app; check *Local API* status in the window; `fp.diagnose()` |
| `permission_denied` | The user clicked "Block" on the browser prompt | Site settings -> "Local network access"/"Apps on device" -> Allow, reload |
| `insecure_page` | Public `http://` site in Chrome/Edge | Serve the site over HTTPS or allow-list it by policy |
| `pairing_denied` / `pairing_timeout` | User clicked "Don't allow" / nobody at the PC | Ask them to allow; revoke/re-add under *Approved websites* |
| `pairing_disabled` | Administrator disabled prompts | Add the origin to `AllowedOrigins` |
| `certificate_untrusted` | HTTPS enabled but not trusted for this Windows user | Click *Enable secure browser access* and accept the Windows prompt (or just use http) |
| Works on one browser only | Safari vs Chrome rules; Firefox has its own cert store | See the matrix above |
| Works for one Windows user, not another | Certificate and approvals are per user | Enable/approve per user |

## What was verified

Verified on this machine (Windows, Chrome 157 / Edge 157):

* **Real-browser end-to-end** (headless Edge, automated test `RealBrowserTests`): cross-origin discovery, preflighted pairing,
  device list, a real capture, abort-cancels-the-scan, HTTPS with the real per-machine certificate, and the diagnosis when the app is down.
* **Full product flow**: a real Edge page -> the real desktop app -> its real approval dialog (clicked "Allow" through UI Automation)
  -> API calls succeed -> the approval is saved.
* **Local Network Access behaviour** (Chrome 157 and Edge 157, headed and headless): a private-LAN `http://` page *is not blocked*;
  a **public `http://` page is blocked**; a public `https://` page shows the permission prompt (state `prompt`; requests wait for the answer).
* 200+ automated tests (183 .NET + 23 JavaScript): origin policy, DNS-rebinding defence, CORS/preflight, pairing and abuse limits, IPv6, HTTPS, request-abort
  cancellation, certificate properties (non-CA, loopback SANs, serverAuth only, non-exportable key, renewal), the JavaScript SDK
  (23 tests incl. every diagnosis branch) and the admin scripts.

**Not verified** (be aware before relying on them):

* The Windows "trust this certificate" confirmation and trusting it in the Root store (needs an interactive click, and changes the
  machine's trust store, so it was not run here); TLS itself was verified with a pinned handshake and in a real browser using
  an SPKI exception.
* Clicking **Allow** on Chrome's own permission prompt (the *prompt state* and waiting behaviour were observed, not the allowed path).
* Safari, and Firefox's own local-network permission behaviour (only Chromium browsers were available).
* The browser-policy names in `Set-BrowserLocalAccessPolicy.ps1` follow Google's and Okta's documentation; confirm in `chrome://policy`.
* Public (non-LAN) addresses were *simulated* with Chromium's `--ip-address-space-overrides` test switch.

## When this pattern cannot work

* **Remote desktop / Citrix / VDI where the browser runs on the server** and the scanner is on the client: `localhost` is then the
  *server*, not the scanner's PC. Use a relay (the desktop app connects *out* to your server over WebSocket and receives capture
  commands) or device redirection. A relay needs a component on your server, so it is not built here; the app's `FingerprintSession`
  is transport-independent and a relay client can sit beside the local service. Ask if you need it.
* **Two Windows users on one PC at the same time** (terminal server): each session's app wants the same port.
* Environments where browser policy forbids all local-network access and cannot be changed.
