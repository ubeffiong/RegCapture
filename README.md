# RegCapture

RegCapture lets a website capture and check fingerprints with a USB reader plugged into the visitor's PC. It is a small Windows app
plus a local service. The website keeps its own screens (or uses RegCapture's), and calls RegCapture only when it needs a fingerprint.

```
Website in the browser  ->  RegCapture on the same PC  ->  USB fingerprint reader
```

This repository publishes the **releases** and the **guides**. It does not contain the source code.

## Download

Get the latest version from **[Releases](../../releases/latest)**:

| File | What it is |
|---|---|
| `RegCaptureSetup.exe` | One-file setup. Recommended for most people. |
| `RegCaptureSetup.msi` | Windows Installer package for Intune, group policy and silent installs. |
| `regcapture-web-sdk-<version>.zip` | The JavaScript SDK for websites (script, ES module, `<fingerprint-capture>` element, TypeScript types, demo page). |
| `RegCapture-Relay-win-x64.zip` | The optional relay server, for remote-desktop / Citrix / cloud setups. |
| `RegCapture-AdminTools.zip` | Administrator scripts (browser policy for the local app, certificate removal). |
| `SHA256SUMS.txt` | Checksums of the files above. |

Each release is a version tag (`v1.0.1`, ...). Check a download with `Get-FileHash <file> -Algorithm SHA256` against `SHA256SUMS.txt`.

## Install

1. Run **`RegCaptureSetup.exe`**. The setup lists the reader drivers it can install; drivers for readers it detects on your PC are pre-selected.
2. Plug in the reader and open **RegCapture**. Pick your reader at the top; a green dot means it is ready.

* **Windows only** (x86/x64). The installers are not code-signed yet, so SmartScreen may show "unknown publisher": choose *More info > Run anyway*.
* **Missed a driver?** Open **Drivers** in the app (or Start menu > *RegCapture - Reader drivers*) and choose *Download and install* any time.
* **Setup error?** Run `msiexec /i RegCaptureSetup.msi /l*v setup.log` and attach `setup.log` to an issue.

Supported readers: SecuGen, ZKTeco, Suprema, Futronic, DigitalPersona, Lumidigm, Morpho, Nitgen, DERMALOG, Hikvision, and any reader with a
Windows Biometric Framework driver. The installers include drivers for SecuGen, ZKTeco and Suprema; the **Drivers** page gets the others.

## Use the app

| Do this | How |
|---|---|
| Pick a finger | Click it on the hands, or use the **Finger** list. A green tick means captured. |
| Capture | Mode **Capture**, then **Capture**; place the finger on the reader. |
| Check someone | Mode **Verify**, then **Verify**. |
| Save | **Export...** writes images, templates and a manifest to a folder. |

A website can start the app with these links: `regcapture://start` (minimised), `regcapture://open` (window), `regcapture://web` (small
browser popup), `regcapture://drivers` (reader drivers page).

## Add it to your website (short version)

```html
<script src="fingerprint-client.js"></script>
<script>
  const fp = new FingerprintClient();
  const r = await fp.capture('RightIndex');          // { status: "success", template: "<base64>", matchTemplate, ... }
</script>
```

* No screen of your own? `fp.captureWithUi('RightIndex')` shows RegCapture's screen and returns the result.
* No matching of your own? `fp.validateAgainst(storedList)` scans a finger and checks it against your stored fingerprints (1:1 or 1:N, up to
  500). `fp.match(probe, storedList)` does the same without scanning.
* Drop-in element: `<fingerprint-capture finger="RightIndex" name="template"></fingerprint-capture>` fills a hidden form field.

## Guides

| Guide | What it covers |
|---|---|
| [docs/USER-GUIDE.md](docs/USER-GUIDE.md) | Install, use, launch links, **web integration**, payload reference, matching against your own list, troubleshooting |
| [docs/BROWSER-INTEGRATION.md](docs/BROWSER-INTEGRATION.md) | Why browsers block calls to the local app, the security model, administrator policies |
| [docs/RELAY.md](docs/RELAY.md) | Remote desktop / Citrix / cloud setup, ticket examples for several languages |

## Help

Open an [issue](../../issues) with the RegCapture version, your Windows version, the reader brand and what you saw (a screenshot or log helps).

## License

RegCapture is licensed under the **GNU General Public License v3.0** (see [LICENSE](LICENSE)). Reader vendors' drivers and SDK libraries
inside the installers remain under their own licenses.
