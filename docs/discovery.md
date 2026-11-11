# Discovery

Devices can be found three ways. All of them only *locate* a peer; authentication always happens in the
[protocol handshake](protocol.md#5-handshake-every-connection-including-lanes) and the certificate is pinned
to the fingerprint learned during discovery.

## 1. mDNS / DNS-SD (primary)

| Property | Value |
|---|---|
| Service type | `_omniferry._tcp.local.` |
| Instance name | the device `id` (`fp[:16]`) |
| Port | the TLS port (48655 by default) |
| Addresses | every private IPv4 address of the device, primary route first |

TXT records:

| key | value |
|---|---|
| `id` | `fp[:16]` |
| `fp` | full 64-character fingerprint |
| `name` | display name (max 60 characters shown) |
| `os` | `windows` or `android` |
| `v` | protocol version, `1` |

**Validation rule:** ignore any record where `fp` is not 64 characters or `fp[:16]` differs from the instance name.
Ignore your own `id`. A `Removed` event removes the peer.

Dual-stack IPv6 is not used in v1. Clients try all advertised addresses **in parallel** and keep the first that
completes the handshake, because machines with VPN or virtual adapters often advertise unreachable addresses first.

## 2. Bluetooth LE (fallback when multicast is blocked)

Roles: the **phone is the BLE peripheral**, the **PC is the central** (Windows cannot easily advertise a GATT
service; Android can).

| Item | Value |
|---|---|
| Service UUID (advertised) | `6f6d6e69-0000-4665-7272-79000000f3e1` |
| Info characteristic (read) | `6f6d6e69-0000-4665-7272-79000000f3e2` |

The characteristic holds UTF-8 JSON, at most 512 bytes:

```json
{"fp":"<64 hex>","name":"Pixel 8","port":48655,"ips":["192.168.1.20"]}
```

The central scans for the service UUID, reads the characteristic, builds a peer
(`source = "ble"`) and then connects over TCP/TLS as usual. The scanner re-reads a device at most every 20 s and
backs off for 30 s when no adapter is available. BLE carries **no file data and no secrets**.

Note that Windows -> phone discovery over BLE is not possible in v1; once the PC has found the phone, the
phone sees the PC as an ordinary incoming connection.

## 3. Manual

The user types `host` or `host:port`. The app opens a TLS connection, completes the handshake (without a pinned
fingerprint) and records the peer with `source = "manual"`. The fingerprint is then shown in the pairing prompt.

## Peer lifecycle

* mDNS peers appear/disappear with service events; BLE and manual peers stay until the app exits.
* A peer's address list is advisory: every send re-resolves from the current peer table and falls back to the
  hint stored with a queued transfer, so queued sends survive a restart.
