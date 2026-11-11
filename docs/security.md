# Security

## Goals

1. Content in transit between two OmniFerry devices cannot be read or modified by other devices on the network.
2. A device cannot pretend to be a device you already trust.
3. Nothing is written to your disk, and no clipboard is changed, without your consent or an earlier pairing.
4. A malicious peer cannot write outside the download folder or exhaust memory with oversized messages.

## Identity and trust

* Each install generates a P-256 key and a self-signed certificate (valid 10 years) on first run.
  `fp = SHA-256(cert DER)`.
* **Client authenticates server** by TLS: the certificate fingerprint must equal the one learned from mDNS/BLE/pairing.
  A mismatch aborts with "device fingerprint changed" and is never retried.
* **Server authenticates client** at the application layer: a random nonce signed together with the server's
  fingerprint ([protocol.md](protocol.md#5-handshake-every-connection-including-lanes)).
  Binding the server fingerprint prevents relaying one server's challenge to another.
* **Pairing** stores `{id, name, fp}`. A stored id with a different fingerprint is treated as **untrusted**.
* Unpaired senders trigger a prompt (accept/decline, optional "always trust"). Paired senders are auto-accepted
  only if the user left *Accept from paired devices automatically* on.

## Hardening already in place

| Risk | Mitigation |
|---|---|
| Path traversal / reserved names in file names | Segment-wise sanitisation, never overwrite, resolved under the download folder |
| Memory exhaustion | 64 KiB cap on pre-auth JSON, 8 MiB after; chunk cap 16 MiB; limits checked before allocation |
| Data corruption | Per-chunk CRC-32 with NACK/resend |
| Disk filling | Reject when free space < size + 64 MiB |
| Stuck/dead peers | Handshake timeouts, 60 s lane idle timeout, TCP keepalive |
| Approval spam | One prompt per batch; prompts time out and count as "Declined" |
| Clipboard leaks | Skipped when the source marks content as excluded (`ExcludeClipboardContentFromMonitorProcessing`); accepted only from paired peers; size caps |
| Web gateway guessing | Random token, constant-time compare, 10 failures/60 s per IP, instant revoke |

## Known limitations (be honest with users)

* **Transport confidentiality only.** TLS protects data on the wire, but a device you approve receives the file
  in the clear. There is no end-to-end encryption across relays because there are no relays.
* **mDNS is public on the LAN.** Anyone on the Wi-Fi sees your device name, OS and fingerprint while the app runs.
  Choose a neutral device name on shared networks.
* **Private key at rest.** On Windows the key is a plain PEM in `%APPDATA%\OmniFerry`, protected only by the user
  profile ACL. Planned: DPAPI protection. Android should keep its key in the Android Keystore
  ([android-guide.md](android-guide.md)).
* **First contact is trust-on-first-use** unless the fingerprint is compared out of band. The approval prompt shows
  a device id fragment; a verification code (short authentication string) is a planned addition.
* **Web gateway is HTTP.** Anyone on the LAN who sees the QR code or URL can use it until you rotate the link or
  switch it off.
* **A paired device can push clipboard content**, including content you did not copy deliberately. Unpair devices you no longer use.
* **Binaries can be reverse-engineered.** Do not rely on secrecy of anything shipped inside the app.

## Reporting a vulnerability

Use the repository's **Security -> Report a vulnerability** (private advisory). Please do not open a public issue
for security problems. See `SECURITY.md` in the public repository.
