# RegCapture: how to use it, and how to add it to your web app

Written for: operators who use the app, and developers who connect a website to it.

## 1. What it is

A small Windows app that talks to the fingerprint reader on the PC, plus a **local service** that lets a website ask for a
scan. Your website keeps its own screens; it only calls the app when it needs a fingerprint.

```
Website in the browser  --(http(s) to 127.0.0.1:12345)-->  RegCapture app  -->  reader
```

Works with most USB readers: DigitalPersona, SecuGen, ZKTeco, Futronic, Suprema, Lumidigm, Hikvision, Morpho, Nitgen, and any
reader with a Windows Biometric Framework driver. Which ones work on a given PC depends on its driver being installed.

## 2. Install (each PC that has a reader)

1. Run `RegCaptureSetup.msi`. The installer lists the reader drivers it can detect or install; tick the ones you need.
2. Plug in the reader. Open **RegCapture** (or let the website start it; see section 4).
3. In the window, choose your reader in the **Reader** list. The dot beside it turns green when the reader is ready.
   If a reader is plugged in but unusable (no driver), a banner offers **Install driver...**.

The app registers the `regcapture://` link for the current Windows user so websites can start it.

**Which readers appear in the list?** Only readers whose software works. When RegCapture starts it loads each reader's SDK and checks that it
answers (no reader needs to be plugged in for this). A brand whose SDK is missing or fails is greyed out with the reason, for example "The ZKTeco
SDK could not start (error -1). Install the ZKTeco driver from the Drivers page."

**More scanner brands:** open **Drivers** and choose **Neurotechnology scanner support** > *Download and install*. It downloads the free
Neurotechnology SDK (about 350 MB, checked against a known checksum, installed for your Windows user with no administrator rights) and adds
DERMALOG, Aratek, Mantra, Startek, Tatvik, Integrated Biometrics, CrossMatch, Jenetric, Biometrika, GreenBit, HF Security, id3, IMD and
NextBiometrics.

**Network readers (access-control terminals):** open **Network readers**, enter the terminal's address, port, user name and password. The
**Hikvision terminal** then appears in the Reader list. (The Hikvision network SDK is included with the app.)

**Skipped a driver, or got a new reader later?** Open **Drivers** (top right of the app window, or Start menu > *RegCapture - Reader
drivers*). Each supported brand has **Download and install**: after you confirm, the app uses the driver shipped with it, or looks for
the driver on Windows Update (Microsoft-signed, and only for that brand's hardware ID), and installs it. Windows asks for administrator
permission once. If nothing is found automatically, **Browse for file...** installs a driver package you downloaded, and **Vendor page**
opens the manufacturer's download page. A banner with the same *Download and install* button also appears when the app sees a reader that
Windows cannot use. In setup, ticking a driver is your consent for Setup to fetch it the same way if it is not already inside the installer.

## 2b. Linux and macOS (preview)

The Linux and macOS packages contain **RegCapture.Service**: the same local API, browser page (`http://127.0.0.1:12345/ui`), fingerprint matching
against your stored templates (`match`, and `validate` with `candidates`) and relay agent as on Windows, running as a background service.
**Scanning with a reader is not available yet**, because RegCapture has no reader drivers for these systems: the Reader list says so.

| System | Install |
|---|---|
| Ubuntu / Debian | `sudo apt install ./regcapture_<version>_amd64.deb` (use `arm64` on ARM). Log out and in, or run `systemctl --user start regcapture`. |
| Other Linux | Unpack the `.tar.gz`, run `./install.sh` (no root; needs systemd for the automatic start), remove with `./uninstall.sh`. |
| macOS | Open the `.pkg` (right-click > Open the first time: it is not signed or notarized yet). It starts the service at login and registers the `regcapture://` link. |

A background service has no window to ask permission, so tell it which websites may use it:
`regcapture approve https://your-site.example` (macOS: `/usr/local/regcapture/RegCapture.Service approve ...`), then restart it
(`systemctl --user restart regcapture`). Settings file: `RegCapture.Service config` prints its path.

## 3. Using the app window

| Do this | How |
|---|---|
| Choose your scanner once | In the **Reader** list pick your scanner and tick **Default**. RegCapture then selects it by itself every time it starts, so you never have to choose it before scanning. With no default it uses the one you used last, or the only known reader that is plugged in. |
| Pick a finger | Click it on the hands, or use the **Finger** drop-down. Both stay in sync. Green ✓ = already captured. |
| Capture | Mode **Capture**, then **Capture**. Place the finger on the reader when prompted. |
| Capture every finger | **Capture all** goes through the fingers not yet captured. |
| Check someone | Mode **Verify**, then **Verify**. Tick *Compare only against the selected finger* for a 1:1 check; otherwise it searches all captured fingers. |
| Fix a mistake | **Clear finger** / **Clear all**. **Cancel** stops a scan in progress. |
| Save results | **Export...** writes images, templates and a manifest to a folder. |

The same scan engine serves the window and the website, so only one scan runs at a time. A second request is told "busy".

If a website is waiting for a fingerprint, a blue banner says so; the next successful capture in the window is sent to it.

## 4. The three launch links (the link decides which UI appears)

| Link | Result |
|---|---|
| `regcapture://start` | Starts the app **minimised**, no window pops up. Use this from websites. |
| `regcapture://open` | Shows the **desktop window**. |
| `regcapture://web` | Shows a small **browser popup page** (served by the app at `http://127.0.0.1:12345/ui`) with the same hand picker. |
| `regcapture://drivers` | Shows the desktop window with the **Reader drivers** page open (the Start menu shortcut uses it). |

Your website normally never needs these: the SDK starts the app with `start` if it is not running.

## 5. Adding it to your web app

The web side is plain JavaScript and does not depend on your backend stack (PHP, .NET, Java, Python, Node, Ruby...).
Everything is in the `web-sdk/` folder.

### 5.1 Fastest: one HTML element

```html
<script src="/js/fingerprint-client.js"></script>
<script src="/js/fingerprint-capture.js"></script>

<form method="post" action="/patients/42/fingerprint">
  <fingerprint-capture finger="RightIndex" name="template" preview></fingerprint-capture>
  <button>Save</button>
</form>
```

After a scan the hidden field `template` holds the base64 data and `template_finger` holds the finger name, so a normal form POST
works in any server-side stack. Add `finger-select` to let the user pick the finger. The element also fires `fp-captured`,
`fp-error` and `fp-status` events.

### 5.2 In code (any framework)

```js
import FingerprintClient from './fingerprint-client.mjs';     // or <script src=".../fingerprint-client.js"> -> window.FingerprintClient

const fp = new FingerprintClient();

try {
  const r = await fp.capture('RightIndex');                   // user places the finger
  // r = { status: "success", template: "<base64>", fingerprintData, width, height, dpi, quality, ... }
  await fetch('/api/patients/42/fingerprint', {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ finger: 'RightIndex', template: r.template }),
  });
} catch (e) {
  showMessage(e.message + ' ' + e.hint);                      // e.code is stable, e.hint is user-friendly text
}
```

Other calls:

| Call | Purpose |
|---|---|
| `fp.capture(finger, { signal, onPhase })` | Scan one finger. Aborting `signal` cancels the scan on the PC. |
| `fp.verify({ finger? })` | Scan a probe and compare with fingers captured in the app this session; returns `matched`, `score`, `matchedFinger`. |
| `fp.validateAgainst(candidates)` | Scan a probe and compare with **your own stored fingerprints** (1:1 or 1:N). See section 5.6. |
| `fp.match(probe, candidates)` | Compare a stored fingerprint with a list, no scan (1:N). See section 5.6. |
| `fp.receive({ timeoutSeconds, finger })` | Wait for a fingerprint the operator captures **in the app** and return it. |
| `fp.captureWithUi(finger, { ui })` | Show the app's capture UI and return what is captured there (section 5.5). |
| `fp.status()`, `fp.devices()` | Reader and state information. |
| `fp.diagnose()` | Probe every address and report what the browser blocked (for support screens). |
| `fp.openApp()` / `fp.openApp('web')` | Show the desktop window / the popup page. |

Finger names: `RightThumb`, `RightIndex`, `RightMiddle`, `RightRing`, `RightLittle`, and the same with `Left`.

### 5.3 Payload reference (what to send, what to expect)

All calls are JSON over `POST http://127.0.0.1:12345/api/fingerprint/` (or `GET .../status`, `.../devices`). The SDK builds these for
you; use this section to read the reply, to call the HTTP API yourself, or to write your server-side model.
Fields that do not apply are **left out** of the JSON (never `null`). Field names are camelCase.

**Requests**

| Action | Request body | Notes |
|---|---|---|
| Capture | `{ "action": "capture", "finger": "RightIndex" }` | `finger` required. Optional `"device": "SecuGen"` selects a reader. |
| Verify | `{ "action": "validate" }` or `{ "action": "validate", "finger": "RightIndex" }` | With `finger`: 1:1 against that finger. Without: 1:N against all captured fingers. (`"verify"` is accepted as an alias.) |
| Receive | `{ "action": "receive", "finger": "RightIndex", "timeoutSeconds": 90 }` | Waits for a capture made in the app's own UI. `finger` optional (pre-selects it in the UI); `timeoutSeconds` 5 to 120, default 90. |
| Cancel | `{ "action": "cancel" }` | Stops the running scan. |

Finger values: `RightThumb`, `RightIndex`, `RightMiddle`, `RightRing`, `RightLittle`, `LeftThumb`, `LeftIndex`, `LeftMiddle`, `LeftRing`, `LeftLittle`.

**Capture success** (HTTP 200). This is the payload a web app should expect:

```json
{
  "status": "success",
  "code": "ok",
  "message": "Captured RightIndex.",
  "action": "capture",
  "finger": "RightIndex",
  "template": "<base64>",
  "fingerprintData": "<base64, same value as template>",
  "width": 256, "height": 360, "dpi": 500, "quality": 78,
  "nativeTemplate": "<base64 vendor template, only if the reader provides one>",
  "nativeFormat": "ZKTeco",
  "capturedAt": "2026-11-11T14:03:22.514Z"
}
```

Only `status` and `template` are guaranteed to be there on success; treat everything else as optional.

| Field | Type | Meaning |
|---|---|---|
| `status` | `"success"` / `"error"` | Check this first. |
| `code` | string | `"ok"` on success, otherwise the error code (below). |
| `template` | base64 string | The fingerprint data. Same bytes as `fingerprintData`. For readers that give an image it is a **raw 8-bit grayscale image, row by row, no header** (length = `width` x `height`); for readers that give only a vendor template it is that template. |
| `width`, `height`, `dpi` | integers | Image size and resolution; present only when the data is an image. Use them to decode it (`FingerprintClient.toDataUrl(result)` does this in the browser). |
| `quality` | integer 0 to 100 | Reader-reported quality, when the reader reports one. |
| `nativeTemplate`, `nativeFormat` | base64 string, name | Vendor-format template (e.g. `Nitgen`, `Lumidigm`, `Suprema`, `ZKTeco`, `Futronic`, `DigitalPersona`) when the reader provides it. |
| `capturedAt` | ISO 8601 UTC | When the scan was taken. |

**Verify success** (HTTP 200, both for a match and for "no match"):

```json
{
  "status": "success", "code": "ok", "action": "validate",
  "message": "Match: RightIndex (score 183)",
  "matched": true, "score": 183, "matchedFinger": "RightIndex",
  "template": "<base64 of the probe scan>", "width": 256, "height": 360, "dpi": 500
}
```

A non-match is **not** an error: `"matched": false`, no `matchedFinger`, and `message` says "No match". Always test `matched`, not
just `status`. `validate` without `candidates` compares the new scan with fingers **captured in the app during the current session** (held in the
app's memory). To compare against fingerprints stored in your own database, send them as `candidates` (section 5.6).

**Receive success** is exactly the capture or verify payload above, taken from whatever was captured in the app's UI.

**Status** (`GET status`): `{ "status":"success","code":"ok","device":"SecuGen","busy":false,"phase":"Idle","waiting":0,"enrolledFingers":["RightIndex"] }`
(`phase`: `Idle`, `OpeningDevice`, `WaitingForFinger`, `Processing`; `waiting` and `waitingFinger` show a website waiting for the app's UI).

**Devices** (`GET devices`): `{ "status":"success","code":"ok","device":"SecuGen","devices":[ { "name":"SecuGen","vendor":"SecuGen","available":true }, { "name":"Nitgen","vendor":"Nitgen","available":false,"reason":"SDK not installed" } ] }`

**Errors** (non-2xx): `{ "status": "error", "code": "timeout", "message": "No finger was placed in time." }`

| HTTP | `code` | Meaning |
|---|---|---|
| 400 | `invalid_json`, `unknown_action`, `invalid_finger`, `invalid_request` | Bad request. `invalid_request` also covers verifying when nothing has been captured yet. |
| 401 / 403 | `unauthorized` / `origin_not_allowed` | Website not approved on this PC (or missing API key for non-browser callers). |
| 408 | `timeout` | No finger placed in time (capture), or nothing captured in the app in time (receive). |
| 409 | `busy`, `duplicate`, `cancelled` | Another scan is running; this print is already captured as another finger; the scan was cancelled. |
| 413 | `body_too_large` | Request too big. |
| 422 | `processing_failed` | The scan could not be processed (poor image). Ask for a retry. |
| 502 / 503 | `device_error` / `no_device` | Reader error / no reader available. |

With the SDK, a non-2xx reply is thrown as an error object: `e.code` is the `code` above, `e.message` the message, `e.hint` a
user-friendly instruction, and `e.details` the raw reply body.

Relay replies use the same payloads; offline workstations answer `503` with `code: "agent_offline"`.

### 5.4 Without the SDK (plain HTTP)

```js
const r = await fetch('http://127.0.0.1:12345/api/fingerprint/', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ action: 'capture', finger: 'RightIndex' }),
});
const data = await r.json();                 // { status, template, ... }
```

Use the SDK in production: it works around browser rules, starts the app, asks for approval and gives readable errors.

### 5.5 No UI of your own? Use the app's UI

If your web app does not want to build a capture screen, let it use ours. Call one method from a click handler:

```js
const r = await fp.captureWithUi('RightIndex');      // a small capture window opens with the hands; the finger is pre-selected
// resolves with { status: "success", template: "<base64>", ... } as soon as a finger is captured there
```

Or with no JavaScript at all, add `ui="web"` to the element: `<fingerprint-capture ui="web" finger="RightIndex" name="template">`.

| `ui` value | What the person sees |
|---|---|
| `web` (default) | The app opens its small capture page as a popup window (`regcapture://web`). |
| `popup` | Your page opens that same page with `window.open` (needs the click; works even if the link handler is blocked). |
| `desktop` | The app's desktop window comes up (`regcapture://open`). |

How it works: the call asks the app to wait for a capture ("receive", with the finger you want), then shows the UI. The UI sees
a website is waiting, pre-selects that finger and says so. When the person captures, the result goes back to your page and the
popup closes itself. Cancelling (`AbortController`) or a timeout (default 120 s) ends the wait. It needs the app on the same PC as the
browser; it is not available through the relay.

### 5.6 Check a fingerprint against your own list (1:1 and 1:N)

If your system keeps its own fingerprints (for example one or more per patient), the app can do the comparison for you.

**Step 1: store the compact `matchTemplate`.** Every capture reply now includes `matchTemplate` (a small base64 string, about 1 to 2 KB).
Save it with the person's record instead of the raw image.

**Step 2: ask the app to compare.** Two ways:

```js
// A) Scan now, compare with a list you supply (1:1 with one item, 1:N with several, up to 500)
const r = await fp.validateAgainst([
  { id: 'patient-3-right-index', matchTemplate: '<stored>' },
  { id: 'patient-9-right-index', matchTemplate: '<stored>' },
], { threshold: 40 });                       // threshold is optional
// r = { status:"success", matched:true, score:183, matchedId:"patient-9-right-index", template, matchTemplate, ... }

// B) Compare a fingerprint you already have with a list, no scanning and no reader
const m = await fp.match({ matchTemplate: '<probe>' }, candidates);
```

Over plain HTTP the same calls are `POST /api/fingerprint/` with `{"action":"validate","candidates":[...]}` and
`{"action":"match","probe":{...},"candidates":[...]}`.

| Field | Meaning |
|---|---|
| `candidates[].id` | Your identifier (up to 128 characters). Returned as `matchedId` when that candidate is the best match. |
| `candidates[].matchTemplate` | The compact template from an earlier capture (recommended). |
| `candidates[].template` + `width`, `height`, `dpi` | Alternative when you stored the raw image instead (max 512 KB each). Send exactly one of the two forms per item. |
| `probe` | `match` only: the fingerprint to look for, same shape as a candidate. |
| `threshold` | Optional minimum score, 1 to 1000. Default about 40. A higher number is stricter. |

Reply: `matched` (true/false), `score` (the best score), `matchedId` (the winning candidate). **No match is a normal reply (HTTP 200,
`matched:false`), not an error.** Errors are 400 with `invalid_candidate` (bad or missing template, duplicate id), `too_many_candidates`
(over 500), `invalid_threshold`, or `invalid_request`. The comparison uses the same engine and score scale for every reader brand, so a
template captured on one brand can be matched against a scan from another. Readers that return no image (template-only readers) cannot
be used for this.

## 6. Making it work in production (not just on `localhost`)

Browsers treat a call from a public website to software on the visitor's PC differently from a developer's `localhost:3000`.
The SDK handles this, but the following still has to be set up.

1. **Approve the website on each PC.** First time a site calls the app, the person at the PC gets a prompt: **Allow** / **Don't allow**.
   To avoid the prompt, an administrator adds the site to `Api.AllowedOrigins` in the app settings, e.g.
   `["https://hmis.example.org", "https://*.hospital.org"]` (there is no allow-everything option).
2. **Browser permission.** Chrome and Edge may show a one-time "access apps on your device" prompt for HTTPS sites. The user
   must click Allow. Administrators can pre-allow it by policy (`Set-BrowserLocalAccessPolicy.ps1`, in `RegCapture-AdminTools.zip` on the release page).
3. **Plain http sites.** A site served over `http://` on the public internet is blocked by Chrome and Edge from reaching the PC.
   Serve the site over `https://`. Sites on a private network address (for example `http://10.x.x.x`) are not blocked.
4. **Optional HTTPS to the app.** In the app, *Website access and local API* > **Enable secure browser access** creates a
   certificate for that PC only and asks Windows to trust it (Windows prompts once). Needed mainly for Safari; Chrome, Edge and
   Firefox can use the plain-http address from an https page.
5. **Browser and reader on different machines** (Citrix, remote desktop, cloud-hosted browser): use the relay,
   see [RELAY.md](RELAY.md). Your backend issues a short-lived ticket, the app on the workstation connects out to the relay,
   and the SDK is created with `transport: 'relay'` (or `'auto'` to try the local app first).

Go-live checklist: site on https · site added to `AllowedOrigins` (or users told to click Allow) · app installed and reader driver
present on each PC · tested in the browsers you support · relay deployed only if you have remote-desktop users.

## 7. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| "Could not reach the app" / `app_not_running` | App not running or blocked. Start it; the SDK tries `regcapture://start`. Run `fp.diagnose()` for details. |
| Works on `localhost`, fails when deployed | Browser blocking (see section 6, points 2 and 3). `fp.diagnose()` shows which. |
| `origin_not_allowed` / "not approved" | Website not approved on this PC. Click Allow in the prompt, or add it to `AllowedOrigins`. |
| `no_device` | No reader selected or driver missing. Check the **Reader** list; use **Check readers** / **Install driver...**. |
| `busy` | Another scan is running (window or another page). Wait or cancel. |
| `timeout` | No finger placed within the time limit (`Session.CaptureTimeoutSeconds`). |
| `duplicate` | The same fingerprint is already captured as a different finger. |

Logs: `%LocalAppData%\GenericFingerPrint\logs`. Per-user settings: `%AppData%\GenericFingerPrint\settings.json`.

## 8. What has and has not been verified

Verified by automated tests and real-browser runs: the API and its security rules, website approval, the browser behaviour in
headless Edge, the JavaScript SDK, the relay (using a fake reader), the launch links, and the desktop and popup UIs by
screenshot. **Not verified:** scanning with real reader hardware for most vendors, the Windows certificate-trust prompt,
Safari and Firefox, behaviour behind a specific reverse proxy for the relay, and the web component inside a real application page.
Test with your actual reader and browsers before go-live.

## More detail

* [BROWSER-INTEGRATION.md](BROWSER-INTEGRATION.md): why browsers block calls, security model, administrator policies.
* [RELAY.md](RELAY.md): remote-desktop and cloud setup, ticket examples for several languages.
* [../README.md](../README.md): build, project layout, hardware status, installer details.
