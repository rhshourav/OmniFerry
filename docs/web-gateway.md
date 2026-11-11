# Web gateway (no-app access)

A small HTTP server lets a phone without OmniFerry upload to, and download from, the PC using only a browser.
The PC shows a QR code containing `http://<lan-ip>:48656/?t=<token>`.

## Authentication

* Every request must carry the query parameter `t=<token>`; the token is 9 random bytes, URL-safe base64
  (`secrets.token_urlsafe(9)`), compared in constant time.
* Wrong or missing token: `403`. More than **10 failures from one IP in 60 s**: `429`.
* **New link** rotates the token immediately (old links stop working). Turning the gateway off stops the listener.
* The page and all API calls are plain **HTTP**: browsers do not accept self-signed HTTPS from a QR scan without
  warnings. Only enable it on networks you trust. See [security.md](security.md).

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | the single-page mobile UI |
| GET | `/api/state` | `{"host":"PC name","files":[{"id","name","size"}]}` |
| GET | `/f/{id}` | download a shared file (HTTP Range supported; `Content-Disposition: attachment`) |
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
