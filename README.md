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
| `regcapture_<version>_amd64.deb`, `regcapture_<version>_arm64.deb` | Ubuntu / Debian package (Linux, preview: see *Linux and macOS* below). |
| `RegCapture-<version>-linux-x64.tar.gz`, `-linux-arm64.tar.gz` | Any Linux distribution with systemd; installs for your user, no root (`./install.sh`). |
| `RegCapture-<version>-macos-arm64.pkg`, `-macos-x64.pkg` | macOS installer (Apple Silicon / Intel). |
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

## Linux and macOS (preview)

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
