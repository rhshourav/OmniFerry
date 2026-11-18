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
| Web gateway guessing | Random token, constant-time compare, 10 wrong tokens/60 s per IP (the right token is never throttled), instant revoke |
| Unwanted inbound connections | Windows Firewall rules that allow OmniFerry only from private/local addresses (below) |

## Windows Firewall rules

Windows drops unsolicited inbound connections, and its own first-run prompt is unreliable (*Cancel* creates a block
rule that beats any allow rule; *Private only* leaves the app unreachable on the many Wi-Fi networks Windows files as
Public). So the installer, or the **Allow OmniFerry** button on the Share page, runs `OmniFerry.exe --firewall-setup`
elevated, which creates:

| Rule | Allows |
|---|---|
| `OmniFerry LAN` | inbound for `OmniFerry.exe`, every network profile |
| `OmniFerry LAN ports` | inbound TCP 48655 and 48656 (and any custom ports from Settings) |

Both rules accept connections **only from** `LocalSubnet`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`,
`100.64.0.0/10` (carrier-grade NAT, Tailscale) and `169.254.0.0/16`. A stranger on a public network, or anything on
the internet, cannot reach the app, even though the rule covers the Public profile. Setup first removes every older
rule for the program, including the old installer's `OmniFerry` rule and any *block* rule Windows made itself.
`--firewall-remove` (run by the uninstaller) deletes them all. Both write `logs\firewall.log`. The rules live in the
Windows Firewall policy store, managed with `netsh advfirewall`: no registry values are written by hand. The installer
records only a marker (`HKCU\Software\OmniFerry\FirewallRules`) so the uninstaller knows whether to ask for permission.

### Marking a network Private

*Check connection* can run `OmniFerry.exe --network-private "<adapter>"` elevated, which calls
`Set-NetConnectionProfile -InterfaceAlias '<adapter>' -NetworkCategory Private`. It only runs when you click **Make this
network private**, and only for the one adapter shown. The adapter name is checked against a strict pattern (no quotes,
`$`, backticks, `;`, `&`, `|`, `<`, `>` or line breaks) before it is placed in the command. Private is Microsoft's
recommended setting for a home network, but it relaxes network discovery and file sharing defaults for that network:
do not do it on a network you do not trust. It is not needed unless the Public profile is set to block all incoming
connections.

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
* **Devices on your own LAN can reach the transfer port** (the firewall rule allows private addresses on every
  profile). They still need the TLS handshake, and an unpaired sender triggers an approval prompt, but on a shared
  network without client isolation anyone can see that the port is open.
* **Web gateway is HTTP.** Anyone on the LAN who sees the QR code or URL can use it until you rotate the link or
  switch it off.
* **A paired device can push clipboard content**, including content you did not copy deliberately. Unpair devices you no longer use.
* **Binaries can be reverse-engineered.** Do not rely on secrecy of anything shipped inside the app.

## Reporting a vulnerability

Use the repository's **Security -> Report a vulnerability** (private advisory). Please do not open a public issue
for security problems. See `SECURITY.md` in the public repository.
