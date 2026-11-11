# OmniFerry wire protocol v1

Status: stable for v1. Reference implementation: the Windows app (`windows/src/omniferry/protocol.py`,
`transfer.py`). All integers are **big-endian**. All strings are UTF-8. Key words MUST/SHOULD follow RFC 2119.

## 1. Transport

* TCP, default port **48655**. A device MAY use another port; it advertises the real one (see [discovery.md](discovery.md)).
* **TLS 1.2 or newer** on every connection. Certificates are self-signed ECDSA P-256. There is no CA and no
  hostname check; identity is established by fingerprint (section 3).
* Clients SHOULD set `TCP_NODELAY` and enlarge socket buffers (the reference uses 4 MiB) and enable TCP keepalive
  (15 s idle, 5 s interval) so dead peers are noticed.
* Every logical conversation uses its own connection: a *control* connection (`offer`, `clip`, `pair`) and,
  for file data, N additional *lane* connections.

## 2. Versioning

The server announces `"v": 1` in its challenge. A client MUST abort on a version it does not support.
Receivers MUST ignore unknown JSON fields (additive changes keep `v` at 1). An unknown request type `t`
MUST be answered with `{"t":"error","reason":"unknown request"}` and the connection closed.

## 3. Identity

* `fp` = lowercase hex of SHA-256 over the certificate's **DER** bytes (64 characters).
* `id` = `fp[:16]`. Used as a label and as part of storage keys only.
* **Trust decisions MUST compare the full `fp`**, never `id`.

## 4. Framing

```
+---------+-----------------+---------------------+
| type u8 | length u32 (BE) | payload (length B)  |
+---------+-----------------+---------------------+
```

| type | name  | payload | max payload |
|---|---|---|---|
| 1 | JSON  | one UTF-8 JSON object | 64 KiB before the handshake completes, 8 MiB after |
| 2 | CHUNK | `index u32` `crc32 u32` `data` | 16 MiB + 8 |
| 3 | ACK   | `index u32` | 4 |
| 4 | NACK  | `index u32` (CRC failed: resend) | 4 |

`crc32` is the standard zlib/IEEE CRC-32 of `data`, as an unsigned 32-bit integer.
A frame that exceeds its limit, has an unknown type, or is not valid JSON where JSON is required is a
protocol error: close the connection.

Worked example. `ACK` for chunk 7: `03 00 00 00 04 00 00 00 07`.
JSON `{"t":"ok"}` (10 bytes): `01 00 00 00 0A 7B 22 74 22 3A 22 6F 6B 22 7D`.

## 5. Handshake (every connection, including lanes)

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: TCP + TLS handshake
    Note over C: compute server_fp from the TLS leaf certificate;<br/>abort if it differs from the pinned fp
    S->>C: {"t":"challenge","v":1,"nonce":b64(16 random bytes),"name","os"}
    C->>S: {"t":"hello","name","os","cert":b64(DER),"sig":b64(signature)}
    Note over S: verify sig with the public key in cert;<br/>peer fp = SHA-256(cert DER)
    S->>C: {"t":"welcome"}
    C->>S: exactly one request: offer | lane | clip | pair
```

* `sig` = ECDSA-SHA256 over `nonce_bytes || ASCII(server_fp_hex)`, DER-encoded (the format Java's
  `SHA256withECDSA` and Python's `cryptography` both produce). Binding the server fingerprint into the signed
  data stops a malicious server from relaying a client's signature to a different server.
* Base64 is standard alphabet with padding.
* On a bad signature the server sends `{"t":"error","reason":"bad signature"}` and closes.
* Timeout: 10 s for each handshake message.

## 6. Messages

### 6.1 File transfer

**`offer`** (client -> server, control connection)

| field | type | notes |
|---|---|---|
| `t` | string | `"offer"` |
| `tid` | hex string, 8-64 chars | transfer id, chosen by the sender; also used by lanes |
| `fid` | hex string, 8-64 chars | stable file id used for resume: `sha1(name\|size\|mtime_ms)[:24]` |
| `name` | string | file name; `/` separates folders. The receiver sanitises it (section 8) |
| `size` | integer | bytes, 0 to 2^50 |
| `mtime` | number | seconds since epoch; the receiver MAY apply it to the saved file |
| `chunk` | integer | chunk size in bytes, 4 KiB to 16 MiB |
| `lanes` | integer | lanes the sender would like (1-8) |
| `batch` | string | groups files sent together so the user is asked once |
| `count`, `total` | integers | files in the batch and their combined size (for the prompt) |

Number of chunks = `ceil(size / chunk)`; every chunk is `chunk` bytes except the last. A zero-byte file has zero chunks.

**`accept`** (server -> client): `{"t":"accept","have":[[start,end),...],"lanes":N}`
`have` lists chunk index ranges the receiver already holds (half-open). `lanes` is the maximum the sender may open (1-8).

**`reject`**: `{"t":"reject","reason":"..."}`. The reason is informational. The reference implementation uses
`Declined` and `Not enough disk space` (free space must exceed `size + 64 MiB`).

**`lane`** (client -> server, new connection after the handshake): `{"t":"lane","tid":"..."}`
Server replies `{"t":"ok"}`, or `{"t":"error","reason":"unknown transfer"}` if `tid` is not an active session
of that same authenticated peer. After `ok` the client streams CHUNK frames.

For each CHUNK the server replies ACK (stored, or already held) or NACK (CRC mismatch). A chunk whose index is out
of range or whose length is wrong is a protocol error and closes the lane.

**`done`** (server -> client, control connection): sent when every chunk is stored. The client may then close everything.

**`cancel`**: either side, on the control connection. The receiver deletes its partial data.

**`error`**: `{"t":"error","reason":"..."}` from the receiver on the control connection (for example a disk write failure). Fatal; the sender does not retry.

### 6.2 Clipboard

`{"t":"clip","kind":"text"|"image","data":"...","ts":1731234567.0}` -> `{"t":"ok"}`
* `text`: the string, up to 512 KiB. `image`: base64 PNG, up to 5 MiB decoded.
* Accepted only from **paired** peers and only if the user has clipboard sync on; otherwise
  `{"t":"error","reason":"not paired"}`.
* Receivers MUST NOT re-broadcast content they just applied (loop prevention).

### 6.3 Pairing

`{"t":"pair"}` -> the receiving user is prompted -> `{"t":"ok"}` or `{"t":"reject","reason":"Declined"}`.
On `ok` both sides store `{id, name, fp}` as trusted.

## 7. Behaviour requirements

**Sender**

1. Connect and handshake, verifying the server fingerprint. Send `offer`; wait up to **75 s** for `accept`/`reject`.
2. Queue every chunk index not in `have`. Open `min(accept.lanes, offer.lanes, missing)` lanes (at least 1).
3. Per lane keep at most **8 unacknowledged chunks**. Mark a chunk delivered only on ACK. On NACK, resend.
4. If a lane breaks, requeue its unacknowledged chunks and reconnect with backoff
   (`min(0.25 * 2^fails, 4)` s). Give up after more than 6 consecutive fruitless reconnects.
5. Finish when `done` arrives on the control connection.
6. If the whole transfer fails with a network error, retry from step 1 (up to 5 times, `min(2^attempt, 15)` s
   apart). The new `accept.have` makes the retry resume exactly where it stopped.
7. Before starting, check the source file still has the size and mtime recorded at queue time.

**Receiver**

1. Validate every `offer` field (ranges above). Ask the user unless the peer is paired and auto-accept is on.
   Prompts time out after ~70 s as a rejection.
2. Write each chunk at offset `index * chunk` into a temporary file; mark it only after the write succeeded.
   Persist the received ranges regularly (reference: every 2 s) so a crash loses at most a little work.
3. Verify CRC before storing. Ignore duplicate chunks but still ACK them.
4. If a new `offer` arrives for the same peer and `fid` while an old session exists, the new one **supersedes** it
   (the sender reconnected before the old socket timed out).
5. Close a lane that sends nothing for 60 s.
6. When complete: move the file into the download folder (never overwrite: add ` (1)`, ` (2)`...), apply
   `mtime`, delete the persisted state, send `done`.
7. If the control connection ends before completion, keep the partial data and its ranges ("interrupted") so a
   later `offer` can resume. Purge stale partials (reference: 7 days).

## 8. Names and paths

Receivers MUST treat `name` as hostile. The reference algorithm: split on `/` and `\`, drop empty, `.` and `..`
segments, replace `< > : " | ? *` and control characters with `_`, strip trailing dots/spaces, prefix reserved
Windows device names (`CON`, `PRN`, `AUX`, `NUL`, `COM1-9`, `LPT1-9`) with `_`, cap each segment at 150
characters, and join under the download folder. An empty result becomes `file`.

## 9. Limits and timers (reference values)

| Item | Value |
|---|---|
| Max file size | 2^50 bytes |
| Chunk size | 4 KiB - 16 MiB (default 1 MiB for files >= 8 MiB, else 256 KiB) |
| Lanes | 1-8 (default 4; 1 when the file is at most two chunks) |
| In-flight chunks per lane | 8 |
| Handshake message timeout | 10 s |
| Approval wait (sender) | 75 s |
| Receiver lane idle timeout | 60 s |
| Concurrent transfers (sender) | 3 |

## 10. Conformance checklist

- [ ] TLS server and client with a persistent self-signed P-256 identity
- [ ] Fingerprint pinning on the client, signed challenge on the server
- [ ] Framing limits enforced before allocating buffers
- [ ] CRC-32 verified; NACK on mismatch; duplicate chunks ACKed
- [ ] `have` ranges honoured and produced; resume works after killing either app
- [ ] Hostile file names sanitised; no overwrite of existing files
- [ ] Clipboard accepted only from paired peers; no echo loops
- [ ] Interop with the Windows reference in both directions, with files of 0 B, 1 B, exactly one chunk, one chunk + 1 B, and several GB
