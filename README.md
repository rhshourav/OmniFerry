# OmniFerry

**Fast, zero-setup file and clipboard sharing between Windows and Android, over your own network.**

**Website: [rhshourav.github.io/OmniFerry](https://rhshourav.github.io/OmniFerry/)**

[![Latest release](https://img.shields.io/github/v/release/rhshourav/OmniFerry?label=release)](https://github.com/rhshourav/OmniFerry/releases)

Pick files, pick a device, done. No accounts, no cloud upload, no cables. Devices on the same Wi-Fi find each other
automatically and move data directly between them.

## Highlights

- **Zero setup**: nearby devices appear on their own (mDNS), with Bluetooth and "connect by IP" as fallbacks
- **Fast**: several parallel encrypted connections per file to use your whole Wi-Fi link
- **Doesn't lose your place**: pause, resume and auto-retry; a dropped connection or restarted app continues from the exact chunk
- **1-to-many**: send to several devices at once; folders keep their structure
- **Clipboard sync**: copy on one device, paste on another (text and images) with paired devices
- **No app on the other end?** Show a QR code; the phone opens a web page to upload and download in the browser
- **Private by design**: data goes straight between your devices over TLS; unknown devices must be approved first
- **Looks the part**: dark and light modes, pastel accents, smooth spring animations, a live chunk map and speed graph for every transfer

## Platforms

| Platform | Status |
|---|---|
| Windows 10 / 11 (x64) | Available: see [Releases](https://github.com/rhshourav/OmniFerry/releases) |
| Android | In development |

## Install (Windows)

1. Download `OmniFerry-<version>-win-x64.zip` from the [latest release](https://github.com/rhshourav/OmniFerry/releases/latest) and extract it.
2. Run `OmniFerry\OmniFerry.exe`. When Windows Firewall asks, allow **Private networks**.
3. Verify the download if you like: compare the SHA-256 of the zip with `SHA256SUMS.txt` (`Get-FileHash <zip>`).

Winget (`winget install rhshourav.OmniFerry`) will work once the package is accepted into the Windows Package Manager repository.

## Using it

- **Send**: open *Devices*, drop files or a folder on a device card (or click **Send**). Select several cards to send to all of them.
- **Receive**: unknown devices ask first; choose *Always trust* for your own devices. Files land in `Downloads\OmniFerry`.
- **Pair** a device to sync the clipboard and skip prompts.
- **Phone without the app**: open *Share by QR* and scan the code.

## Troubleshooting

| Problem | Try |
|---|---|
| A device doesn't show up | Same Wi-Fi? Guest or "AP isolation" networks block discovery. Use **Connect by IP** on the Devices page, or turn on Bluetooth discovery in Settings |
| "Can't connect" after install | Allow OmniFerry on **Private networks** in Windows Firewall (Windows Security -> Firewall -> Allow an app) |
| Transfer is slow | Wi-Fi is the limit; try 5 GHz, or raise *Parallel connections* in Settings |
| Something crashed | Attach `%APPDATA%\OmniFerry\logs\omniferry.log` to an issue |

## Documentation

| | |
|---|---|
| [Protocol](docs/protocol.md) | the wire format both apps speak |
| [Discovery](docs/discovery.md) | mDNS, Bluetooth, manual |
| [Security](docs/security.md) | how devices are authenticated, and the known limits |
| [Web gateway](docs/web-gateway.md) | the QR/browser fallback |
| [Transfer engine](docs/transfer-engine.md) | chunking, resume, cleanup |
| [Architecture](docs/architecture.md) | how the Windows app is built |
| [Android guide](docs/android-guide.md) | protocol on Android, platform limits |

## Security and privacy

OmniFerry has no server: your files and clipboard travel directly between your devices on your network, encrypted with TLS.
See [SECURITY.md](SECURITY.md) to report a vulnerability and [PRIVACY.md](PRIVACY.md) for exactly what the app does with data.

## Feedback

Bug reports and ideas: [open an issue](https://github.com/rhshourav/OmniFerry/issues/new/choose). This repository hosts releases,
documentation and the issue tracker; the source code is not published here. See [CHANGELOG.md](CHANGELOG.md) for what changed.

## License

Free to use. All rights reserved by the author; see [LICENSE](LICENSE).

Made by [rhshourav](https://github.com/rhshourav).
