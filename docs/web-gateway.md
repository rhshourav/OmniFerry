# Web gateway (no-app access)

A small HTTP server lets a phone without OmniFerry upload to, and download from, the PC using only a browser.
The PC shows a QR code containing `http://<lan-ip>:48656/?t=<token>`.

## One code per network

The server listens on every interface, so a phone can reach it through whichever network it shares with the PC.
The **Share by QR** page therefore shows one code per network the PC is on (Wi-Fi, Ethernet, a second Ethernet
adapter, VPN...), each with the adapter's name and address. Switch between them with the arrows beside the code
(or the Left/Right keys, the mouse wheel, or the dots under the code). The list refreshes every 4 s while the page
is open, so plugging in a cable or connecting a VPN adds its code without restarting, and the code you were on stays
selected.

Order: physical networks first (the one carrying the default route first), then VPN, then virtual adapters
(WSL, Hyper-V, VMware, Docker). A full-tunnel VPN owns the default route but a phone is rarely on it, so VPN never
outranks Wi-Fi or Ethernet. Adapter type is guessed from the adapter's name and driver description
(`util.classify_adapter`). The token is the same for every code; **New link** rotates it for all of them.

## What the PC shares

On the **Share by QR** tab, anything you drop **anywhere in the window** (or choose with the box: *Files...* or
*Folder...*) is shared with browsers, and is never sent to a device from that tab. Files are served as they are. A
**folder** is one entry in the list and downloads as a single ZIP built while it is sent: no temporary file, nothing
held in memory, stored uncompressed (photos and videos do not shrink, and the PC should not burn a core for a phone).
Symbolic links inside a shared folder are skipped, so a link cannot hand out what it points at. The folder is read when
the phone downloads it, so changes made after it was added are included; files deleted in the meantime are left out. A
folder's size and file count in the list are a snapshot from when it was added. Dropping something that is already
shared does not add it twice. Dropping works even while the gateway is off (the list is kept; phones see it once the
gateway is on).

## Authentication

* Every request must carry the query parameter `t=<token>`; the token is 9 random bytes, URL-safe base64
  (`secrets.token_urlsafe(9)`), compared in constant time.
* Wrong or missing token: `403`. More than **10 wrong tokens from one IP in 60 s**: `429` for further wrong tokens.
  A request with the **right** token is never throttled, so a phone can reload its own link as often as it likes.
* Browsers ask for `/favicon.ico`, `/apple-touch-icon.png`, `/apple-touch-icon-precomposed.png` and `/robots.txt`
  without a token on every page load. These get a plain `404` and never count as failures.
* **New link** rotates the token immediately (old links stop working). Turning the gateway off stops the listener.
* The page and all API calls are plain **HTTP**: browsers do not accept self-signed HTTPS from a QR scan without
  warnings. Only enable it on networks you trust. See [security.md](security.md).

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | the single-page mobile UI |
| GET | `/api/state` | `{"host":"PC name","files":[{"id","name","size","dir","count"}]}` (`dir` is true for a folder; `count` is its number of files) |
| GET | `/f/{id}` | download a shared file (HTTP Range supported; `Content-Disposition: attachment`), or a shared folder as one streamed ZIP |
| POST | `/api/upload` | `multipart/form-data`, field `file` (repeat for several). Streamed to disk. Returns `{"ok":true,"saved":[...]}` |
| POST | `/api/text` | JSON `{"text":"..."}` (max 200,000 chars). Delivered to the PC clipboard history |

All responses carry `Cache-Control: no-store` and `X-Content-Type-Options: nosniff`.

## Upload behaviour

* Files are written to `<download folder>\Web Uploads\` as `name.omniferry.tmp`, then atomically renamed;
  an interrupted upload deletes its temp file. Abandoned `*.tmp` files older than 1 h are purged at startup.
* The browser-supplied file name is sanitised like any peer-supplied name and never overwrites an existing file.
* Uploads appear in the Transfers page with live progress (the chunk map is derived from bytes received).
* Uploads are **not resumable** in v1 (a single streaming POST). Use the app for very large files.

## Browser UI

Responsive, light/dark via `prefers-color-scheme`, pastel accent. Uploads run two at a time with per-file progress
(XHR), the file list polls `/api/state` every 3 s, and an expired token shows "Link expired. Scan the QR code again."

## Troubleshooting: the phone cannot open the link

**Start with *Check connection*** on the Share page. It tests the network whose code is on screen and lists, problems
first, what it found and what it could not see:

* does OmniFerry answer on exactly that address (a server problem, not a network one)?
* has a phone ever opened the link through that address, or through a *different* one? The server records the address
  each phone came in through; the code's icon gets a green dot once a phone has used it, so you can see which network
  really works. "A phone reached Ethernet but not Wi-Fi" proves the phone and PC can talk and narrows the cause.
* Windows' view of the network: Public / Private / Domain, and whether that profile has *Block all incoming
  connections* on, which overrides every allow rule. A hardened **Public** profile is the usual reason Wi-Fi fails while
  Ethernet (a Private network) works; the dialog offers **Make this network private** (one UAC prompt, only when you
  click it; OmniFerry never changes the network type on its own).
* OmniFerry's own firewall rules, and any other firewall product that is active.
* whether Wi-Fi and Ethernet are on the same subnet (see 2b below).

**Copy report** puts the whole result on the clipboard for a bug report.


The app checks itself at start-up (it requests its own page on this PC) and shows an error toast if the gateway is
not answering. If it is answering but the phone still cannot reach it, the usual causes are, in order:

1. **Windows Firewall.** Without a rule, Windows silently drops connections from other devices. The Share page shows
   an **Allow OmniFerry** button when the rules are missing or in conflict (for example after clicking *Cancel* on
   Windows' own first-run prompt, which creates a *block* rule that beats any allow rule). One click, one UAC prompt,
   and OmniFerry creates its rules (see [security.md](security.md#windows-firewall-rules)). The installer does the same
   during installation. Silent installs (winget, scripts) never show a prompt, so use the button once.
2. **The phone is on a different network** than the code you scanned. When the PC is on several networks (Wi-Fi,
   Ethernet, VPN), use the arrows beside the code and scan the one for the network the phone is connected to.
2b. **Ethernet works, Wi-Fi does not, both on the same router.** The phone reaches the PC's wired address through the
   router, but the router (often in *guest network* or *AP / client isolation* mode) stops Wi-Fi devices from reaching
   each other, which is what the PC's Wi-Fi address needs. Nothing on the PC can change that: use the Ethernet code, or
   turn off AP isolation / client isolation in the router.
3. **Client isolation / guest Wi-Fi.** Some routers stop devices on the same Wi-Fi from talking to each other. Nothing
   on the PC can fix this; use a different network or turn the router's *AP isolation* off.
4. **Phone ignores the Wi-Fi.** Android's *Switch to mobile data automatically* (or similar) can send the browser over
   mobile data when the Wi-Fi has no internet; the PC's address is unreachable from there. Turn it off for this network.
5. **Windows blocks all incoming connections.** If *Block all incoming connections* is ticked for the network profile
   (Windows Security -> Firewall & network protection), no allow rule can help. OmniFerry never changes that setting.

