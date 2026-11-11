# OmniFerry documentation

| Document | What it covers | Audience |
|---|---|---|
| [protocol.md](protocol.md) | The wire protocol v1: framing, handshake, messages, limits. **The contract every client implements.** | Implementers |
| [discovery.md](discovery.md) | mDNS, Bluetooth LE and manual discovery | Implementers |
| [security.md](security.md) | Identity, trust model, threat model, known limitations | Everyone |
| [web-gateway.md](web-gateway.md) | The QR/browser fallback HTTP API | Implementers |
| [transfer-engine.md](transfer-engine.md) | Chunking, lanes, resume, persistence, cleanup, health score | Contributors |
| [architecture.md](architecture.md) | Components, threads, data on disk, ports | Contributors |
| [android-guide.md](android-guide.md) | Mapping the protocol to Android APIs, platform limits | Android developers |

Maintainer-only process docs live in `internal/` and are **not** published to the public repository.
