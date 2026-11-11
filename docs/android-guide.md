# Android implementation guide

How to implement [protocol.md](protocol.md) on Android (Kotlin, Jetpack Compose) and where Android differs from Windows.
Code is illustrative; verify on API 26-35.

## 1. Identity and TLS

Generate a P-256 key in the Android Keystore; Keystore creates a self-signed certificate for it:

```kotlin
val kpg = KeyPairGenerator.getInstance(KeyProperties.KEY_ALGORITHM_EC, "AndroidKeyStore")
kpg.initialize(
    KeyGenParameterSpec.Builder("omniferry-identity", KeyProperties.PURPOSE_SIGN)
        .setAlgorithmParameterSpec(ECGenParameterSpec("secp256r1"))
        .setDigests(KeyProperties.DIGEST_SHA256, KeyProperties.DIGEST_SHA384, KeyProperties.DIGEST_NONE)
        .setCertificateSubject(X500Principal("CN=OmniFerry Device"))
        .setCertificateNotBefore(Date())
        .setCertificateNotAfter(Date(System.currentTimeMillis() + 3650L * 86_400_000))
        .build()
)
kpg.generateKeyPair()
```

Fingerprint, signature and CRC (all must match the Windows reference byte for byte):

```kotlin
fun fingerprint(der: ByteArray) =
    MessageDigest.getInstance("SHA-256").digest(der).joinToString("") { "%02x".format(it) }

fun sign(key: PrivateKey, nonce: ByteArray, serverFp: String): ByteArray =
    Signature.getInstance("SHA256withECDSA").run {
        initSign(key); update(nonce); update(serverFp.toByteArray(Charsets.US_ASCII)); sign()
    }

fun crc32(buf: ByteArray, off: Int, len: Int): Long = CRC32().apply { update(buf, off, len) }.value // u32
```

Framing with `DataOutputStream`/`DataInputStream` (both big-endian): `writeByte(type); writeInt(length)`.
Read lengths as unsigned and enforce the [limits](protocol.md#4-framing) before allocating.

TLS: build an `SSLContext` whose `KeyManager` serves the Keystore key, and a client `TrustManager` that accepts any
chain **and then pins** the leaf's SHA-256 against the expected `fp` (do the pin check in your handshake code, not by
disabling checks silently). Use `SSLServerSocket` for the listener. Run sockets on `Dispatchers.IO` with one coroutine
per lane.

## 2. Discovery

* **mDNS**: `NsdManager` with service type `_omniferry._tcp.`, TXT attributes `id`, `fp`, `name`, `os=android`, `v=1`.
  `NsdManager` resolves one service at a time; queue resolves. Apply the same self-certifying check as the PC.
* **BLE**: advertise service UUID `6f6d6e69-0000-4665-7272-79000000f3e1` with `BluetoothLeAdvertiser` and expose the
  info characteristic from a `BluetoothGattServer` ([discovery.md](discovery.md)). Android 12+ needs the runtime
  permissions `BLUETOOTH_ADVERTISE` and `BLUETOOTH_CONNECT`.

## 3. Background transfers

* Run transfers in a **foreground service** with a persistent notification showing progress and pause/cancel actions.
* Android 14+ requires a declared foreground-service type (`dataSync`) and the `FOREGROUND_SERVICE_DATA_SYNC` permission.
  **Android 15 limits `dataSync` services to roughly 6 hours per 24 hours**: implement `onTimeout()`, persist state,
  and resume cleanly (the resume protocol was designed for this).
* Hold a `WifiManager.WifiLock` (high-perf) during transfers; acquire a `MulticastLock` if you use a raw mDNS library.
* Persist the queue in Room/DataStore the same way the PC persists `queue.json`.

## 4. Storage

* Receive into app-private `filesDir/partial/<peer>-<fid>.part` (random access via `RandomAccessFile`).
* On completion, publish to `Downloads/OmniFerry` with `MediaStore.Downloads` (API 29+), using `IS_PENDING = 1` while
  copying, so a half-written file is never visible. Pick the final name with the never-overwrite rule.
* Reading files to send: accept `content://` URIs from the picker/share sheet; open with `ContentResolver`, and record
  `size` and `lastModified` at queue time for the source-changed check.

## 5. Clipboard: the big platform difference

Since **Android 10 an app can read the clipboard only while it is in the foreground** (or is the default keyboard).
A background service cannot watch the clipboard. Writing it is allowed, and Android 12+ shows a "pasted from" toast.
Design accordingly:

| Direction | Approach |
|---|---|
| PC -> phone | Receive `clip` and call `setPrimaryClip` from the service. Works in the background. |
| Phone -> PC | A deliberate action: a **Quick Settings tile**, a **notification action**, or the **share sheet** that opens a transparent activity, reads the clipboard in the foreground, sends it, and finishes. Optionally sync automatically while the app is open. |

Do not recommend accessibility-service or log-reading workarounds; they are fragile and rejected by Play policy.

## 6. Interop test plan

Run each case phone -> PC and PC -> phone, then repeat while killing and restarting either app mid-transfer:

0 B, 1 B, exactly one chunk, one chunk + 1 B, 100 MB, several GB, 1-to-3 devices, a folder with nested names,
a name containing `..\`, `C:`, and `CON.txt`, a corrupted chunk (flip a byte in a debug build), toggling Wi-Fi mid-transfer,
pairing and clipboard text/image both ways, and an incorrect pinned fingerprint (must refuse).
