# Ferrum Extended Library — DTLS Module

**Module path:** `extlib.dtls` (provided by `lib_ccsp_dtls`)
**Part of:** Ferrum Extended Standard Library (extlib)

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [Dependencies](#2-dependencies)
3. [DtlsConfig](#3-dtlsconfig)
4. [DtlsServer](#4-dtlsserver)
5. [Cookie Exchange](#5-cookie-exchange)
6. [DtlsClient](#6-dtlsclient)
7. [DtlsSession](#7-dtlssession)
8. [DTLS 1.3 Epoch Management](#8-dtls-13-epoch-management)
9. [Handshake Retransmission and Fragmentation](#9-handshake-retransmission-and-fragmentation)
10. [Version Support Policy](#10-version-support-policy)
11. [Integration with net.dgram](#11-integration-with-netdgram)
12. [Integration with CoAP extlib](#12-integration-with-coap-extlib)
13. [Error Types](#13-error-types)
14. [Example Usage](#14-example-usage)

---

## 1. Overview and Rationale

### Why DTLS

TLS secures stream-oriented protocols (TCP). Many important protocols run over UDP — a connectionless, unreliable transport — and need the same confidentiality, integrity, and authentication guarantees. DTLS (Datagram Transport Layer Security) is TLS adapted for UDP.

**Protocols that mandate DTLS:**

| Protocol | Reference | DTLS role |
|---|---|---|
| CoAP secure (`coaps://`) | RFC 7252 | Mandatory transport security |
| RADIUS/DTLS | RFC 7360 | Replaces IPsec for RADIUS transport |
| TURN over DTLS | RFC 7065 | Relay traversal for WebRTC |
| SRTP key exchange | RFC 5764 (DTLS-SRTP) | Keying material for media encryption |
| QUIC (conceptual ancestor) | RFC 9000 | QUIC borrowed heavily from DTLS 1.3 design |

Any UDP protocol that needs confidentiality or peer authentication should use DTLS rather than rolling its own security layer. The alternatives — application-layer encryption, IPsec, or custom HMAC schemes — are all harder to get right.

### The Handshake-Over-Unreliable-Transport Problem

TCP guarantees ordered, reliable delivery, so a TLS handshake simply sends each flight and waits for the other side to respond. UDP provides neither ordering nor reliability. DTLS solves this with:

1. **Sequence numbers on handshake messages** — detect reordering and duplication
2. **Message fragmentation** — handshake messages can exceed a UDP datagram; DTLS fragments them
3. **Flight-based retransmission** — each side retransmits its last flight if no response arrives within a timeout
4. **Cookie exchange** — before performing the expensive asymmetric cryptography of a full handshake, the server verifies that the client IP address is reachable (amplification protection)

### Why extlib, Not stdlib

DTLS sits above the stdlib layering boundary for two reasons:

- It depends on `extlib.tls` for certificate validation, cipher suite definitions, and cryptographic handshake logic. The stdlib does not contain TLS.
- It depends on `crypto` primitives (HMAC-SHA256 for cookies, AEAD for record protection) from the stdlib `crypto` module, but the full DTLS state machine is substantial enough to belong in an extended library rather than core.

The split mirrors the stdlib's own design principle: `net` provides `UdpSocket`; DTLS is a protocol built on top of it.

---

## 2. Dependencies

```
extlib.dtls
    extlib.tls          — CipherSuite, CertChain, PrivateKey, CaStore,
                          TlsVersion, ServerName, ClientAuth
    std.crypto          — Hmac, Sha256, Aes128Gcm, Aes256Gcm,
                          ChaCha20Poly1305, SystemRng, ct_eq
    std.net             — UdpSocket, SocketAddr, IpAddr
    std.async           — Future, AsyncRead, AsyncWrite, Runtime
    std.time            — Duration, Instant
    std.alloc           — Vec, HashMap, Box
```

The `extlib.tls` module provides all certificate handling, cipher suite negotiation logic, and record layer primitives. `extlib.dtls` adapts them for datagram transport — it does not re-implement TLS internals.

---

## 3. DtlsConfig

`DtlsConfig` carries everything needed to initiate or accept DTLS connections. It is immutable after construction and cheaply clonable (it holds `Arc`-wrapped certificates).

```ferrum
type DtlsConfig {
    pub mode:             DtlsMode,
    pub cipher_suites:    Vec[CipherSuite],
    pub versions:         DtlsVersionSet,
    pub cookie_secret:    CookieSecret,
    pub session_timeout:  Duration,
    pub retransmit:       RetransmitConfig,
    pub epoch:            EpochConfig,
    pub max_dgram_size:   usize,
}

impl DtlsConfig {
    pub fn builder(): DtlsConfigBuilder
}
```

### DtlsMode

```ferrum
enum DtlsMode {
    Server {
        cert:        CertChain,
        key:         PrivateKey,
        client_auth: ClientAuth,
    },
    Client {
        server_name: Option[ServerName],   // None for anonymous/PSK-only
        ca_certs:    CaStore,
        client_cert: Option[(CertChain, PrivateKey)],
    },
}

enum ClientAuth {
    None,
    Optional,
    Required,
}
```

### Cipher Suites

DTLS 1.3 reuses TLS 1.3 cipher suites verbatim (the DTLS 1.3 record format changed but the cipher suite registry is shared). DTLS 1.2 supports only ECDHE-based suites; static RSA key exchange and DHE are excluded.

```ferrum
// DTLS 1.3 — same suites as TLS 1.3
//   CipherSuite.TLS_AES_128_GCM_SHA256
//   CipherSuite.TLS_AES_256_GCM_SHA384
//   CipherSuite.TLS_CHACHA20_POLY1305_SHA256

// DTLS 1.2 — ECDHE only (no static RSA, no plain DHE)
//   CipherSuite.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
//   CipherSuite.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
//   CipherSuite.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
//   CipherSuite.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
//   CipherSuite.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
//   CipherSuite.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256

// Convenience — default list prefers DTLS 1.3 suites, then DTLS 1.2 ECDHE
fn default_cipher_suites(): Vec[CipherSuite]
```

The default cipher suite list is ordered by preference: DTLS 1.3 suites first (smaller overhead, better AEAD), then DTLS 1.2 ECDHE suites for interoperability.

### Version Set

```ferrum
type DtlsVersionSet {
    pub dtls_12: bool,
    pub dtls_13: bool,
}

impl DtlsVersionSet {
    const DTLS_13_ONLY:     Self = DtlsVersionSet { dtls_12: false, dtls_13: true }
    const DTLS_12_AND_13:   Self = DtlsVersionSet { dtls_12: true,  dtls_13: true }
    // DTLS 1.0 and 1.1 are not present — see §10
}
```

### Cookie Secret

```ferrum
// Opaque wrapper — prevents accidental logging of the raw secret bytes
type CookieSecret([u8; 32])

impl CookieSecret {
    pub fn generate(): Result[Self, CryptoError] ! IO
        // Fills from SystemRng

    pub fn from_bytes(bytes: [u8; 32]): Self
        // For deterministic testing only; use generate() in production
}
```

The cookie secret is used by the server to compute `HMAC-SHA256(secret, client_addr)` for HelloVerifyRequest cookies. It should be rotated periodically; a new `CookieSecret` can be loaded into a running `DtlsServer` without dropping existing sessions.

### Retransmit Config

```ferrum
type RetransmitConfig {
    pub initial_timeout:  Duration,   // default: 1s (RFC 6347 §4.2.4)
    pub max_timeout:      Duration,   // default: 60s
    pub backoff_factor:   f32,        // default: 2.0 (exponential backoff)
    pub max_retransmits:  u8,         // default: 5; failure → Timeout error
}

impl RetransmitConfig {
    const DEFAULT: Self = RetransmitConfig {
        initial_timeout: Duration.from_secs(1),
        max_timeout:     Duration.from_secs(60),
        backoff_factor:  2.0,
        max_retransmits: 5,
    }
}
```

### DtlsConfigBuilder

```ferrum
type DtlsConfigBuilder {
    fn mode(&mut self, mode: DtlsMode): &mut Self
    fn cipher_suites(&mut self, suites: Vec[CipherSuite]): &mut Self
    fn versions(&mut self, versions: DtlsVersionSet): &mut Self
    fn cookie_secret(&mut self, secret: CookieSecret): &mut Self
    fn session_timeout(&mut self, timeout: Duration): &mut Self
    fn retransmit(&mut self, config: RetransmitConfig): &mut Self
    fn epoch(&mut self, config: EpochConfig): &mut Self
    fn max_dgram_size(&mut self, size: usize): &mut Self
    fn build(self): Result[DtlsConfig, DtlsError]
}
```

---

## 4. DtlsServer

`DtlsServer` binds a `UdpSocket` and manages a table of per-source DTLS sessions. UDP is connectionless, but DTLS state is keyed by `(local_addr, remote_addr)` pair. The server receives all datagrams on one socket and dispatches them by source address.

```ferrum
type DtlsServer

impl DtlsServer {
    pub fn bind(
        addr:   SocketAddr,
        config: DtlsConfig,
    ): Result[Self, DtlsError] ! Async + Net

    // Consume the server and iterate over completed handshakes.
    // Each yielded DtlsSession is a fully established connection.
    // Handshake failures are reported as Err variants; the iterator
    // continues — one bad client does not stop the server.
    pub fn accept(
        &self,
    ): impl AsyncIterator[Item = Result[DtlsSession, DtlsError]] ! Async + Net

    pub fn local_addr(&self): Result[SocketAddr, DtlsError]

    // Replace the cookie secret for future HelloVerifyRequests.
    // In-flight handshakes that already received a cookie keep using
    // the old secret until they complete or time out.
    pub fn rotate_cookie_secret(
        &self,
        new_secret: CookieSecret,
    ): Result[(), DtlsError]

    // Number of sessions currently in the connection table
    // (includes sessions in handshake and fully established sessions)
    pub fn session_count(&self): usize
}
```

### Connection Table

Internally, `DtlsServer` maintains a `HashMap[(local_addr, remote_addr), SessionState]`. When a datagram arrives:

1. Look up the source address in the table.
2. If found: route the record to the existing session's state machine.
3. If not found: this is a new source. Begin cookie exchange (see §5).

Sessions are removed from the table when:
- The session sends or receives a `close_notify` alert and `.close()` completes.
- The session has been idle for longer than `session_timeout`.
- The handshake exceeds `max_retransmits` without completing.

---

## 5. Cookie Exchange

Cookie exchange (RFC 6347 §4.2.1) is the mechanism DTLS servers use to prevent amplification attacks during the handshake. Without it, an attacker with a spoofed source IP could send a small ClientHello and cause the server to send a large Certificate message to a victim.

### Protocol

```
Client                          Server
------                          ------
ClientHello (no cookie)  →
                         ←      HelloVerifyRequest(cookie)
ClientHello(cookie)      →
                         ←      ServerHello, Certificate, ...
...full handshake...
```

The HelloVerifyRequest is small (roughly the same size as the ClientHello), limiting amplification to 1:1. A client that cannot receive the HelloVerifyRequest (because its source IP is spoofed) will never send the second ClientHello, so the server never commits memory for that handshake.

### Cookie Computation

```
cookie = HMAC-SHA256(
    key   = cookie_secret,
    input = client_ip_bytes || client_port_bytes || handshake_epoch_bytes
)
```

The handshake epoch is a server-side counter that increments on `rotate_cookie_secret()`. Cookies computed under old epochs are rejected, which provides forward-secrecy on cookie verification and bounds the validity window without requiring the server to store issued cookies.

Cookie validity is verified in constant time using `crypto.ct_eq` to prevent timing oracles.

### Transparency to Application Code

Cookie exchange is handled entirely within `DtlsServer`. The handler passed to `accept()` sees only fully-established `DtlsSession` values. A session only appears in the accept stream after:

1. The client has provided a valid cookie.
2. The full cryptographic handshake (ServerHello through Finished) has completed.
3. The record layer has switched to the negotiated cipher.

Application code never sees HelloVerifyRequest or pre-cookie ClientHello messages.

---

## 6. DtlsClient

`DtlsClient` wraps a `UdpSocket` and performs the client side of the DTLS handshake. The connect function returns a `DtlsSession` only after the full handshake completes, including receiving the server's Finished message.

```ferrum
type DtlsClient

impl DtlsClient {
    pub fn connect(
        addr:   SocketAddr,
        config: DtlsConfig,
    ): Result[DtlsSession, DtlsError] ! Async + Net
        // Binds an ephemeral local port, sends ClientHello,
        // responds to HelloVerifyRequest if received,
        // completes handshake, returns established session.

    // Use a pre-bound socket (e.g., when local port must be fixed)
    pub fn connect_with_socket(
        socket: UdpSocket,
        addr:   SocketAddr,
        config: DtlsConfig,
    ): Result[DtlsSession, DtlsError] ! Async + Net
}
```

### Automatic Handling

`DtlsClient::connect` handles transparently:

- **HelloVerifyRequest** — if the server sends one, the client retries its ClientHello with the provided cookie. No application action required.
- **Handshake message fragmentation** — if the server sends a Certificate or CertificateVerify that exceeds the path MTU, the DTLS record layer reassembles the fragments before presenting them to the handshake state machine.
- **Retransmission** — if no response arrives within `retransmit.initial_timeout`, the client retransmits the current flight with exponential backoff up to `retransmit.max_retransmits`. After that, returns `DtlsError.Timeout`.
- **Out-of-order delivery** — handshake messages carry sequence numbers; duplicates are discarded, reordered messages are buffered until the expected message arrives.

---

## 7. DtlsSession

`DtlsSession` represents a fully established DTLS session — both client-side (returned by `DtlsClient::connect`) and server-side (yielded by `DtlsServer::accept`). It implements the same `AsyncRead + AsyncWrite` traits as a TLS stream, making it substitutable in protocol implementations that abstract over the transport.

```ferrum
type DtlsSession

impl DtlsSession {
    // Peer's network address
    pub fn peer_addr(&self): SocketAddr

    // Our local address (useful when bound to 0.0.0.0)
    pub fn local_addr(&self): SocketAddr

    // Negotiated cipher suite
    pub fn cipher_suite(&self): CipherSuite

    // DTLS version negotiated (Dtls12 or Dtls13)
    pub fn version(&self): DtlsVersion

    // Server certificate chain presented during handshake.
    // Some on server sessions (we authenticated the server),
    // Some on client sessions when client_auth succeeded,
    // None if mutual auth was not used.
    pub fn peer_cert(&self): Option[CertChain]

    // Send a DTLS close_notify alert and wait for the peer's close_notify.
    // After this returns Ok(()), the session is unusable.
    // Records that arrive after sending close_notify but before receiving
    // the peer's close_notify are delivered normally.
    pub fn close(&mut self): Result[(), DtlsError] ! Async

    // Initiate a key update (DTLS 1.3 only — see §8)
    pub fn update_keys(&mut self): Result[(), DtlsError] ! Async
}

impl AsyncRead for DtlsSession {
    fn poll_read(
        &mut self,
        cx:  &mut Context,
        buf: &mut [u8],
    ): Poll[ReadResult]
    // Reads one decrypted application datagram.
    // Returns ReadResult.Eof when close_notify received.
    // The buf must be large enough to hold one DTLS record
    // (max_dgram_size - record header overhead); a short buf
    // returns DtlsError.RecordTooLarge.
}

impl AsyncWrite for DtlsSession {
    fn poll_write(
        &mut self,
        cx:  &mut Context,
        buf: &[u8],
    ): Poll[Result[usize, IoError]]
    // Encrypts and sends buf as one DTLS application record.
    // Rejects buf.len() > max_record_payload() with IoError.

    fn poll_flush(
        &mut self,
        cx: &mut Context,
    ): Poll[Result[(), IoError]]

    fn poll_shutdown(
        &mut self,
        cx: &mut Context,
    ): Poll[Result[(), IoError]]
    // Equivalent to close() — sends close_notify.
}

enum DtlsVersion {
    Dtls12,
    Dtls13,
}
```

### Datagram Semantics

DTLS preserves datagram message boundaries. One `write` call produces one DTLS record. One `read` call delivers one decrypted record — it does not aggregate multiple records or split them across reads. Applications that need streaming framing (e.g., RADIUS/DTLS) must handle their own length-prefixing at the application layer.

---

## 8. DTLS 1.3 Epoch Management

DTLS 1.3 (RFC 9147) introduces epoch-based key management that differs substantially from DTLS 1.2. Epochs index the key schedule: each epoch has its own traffic keys, and epoch numbers are embedded in the compact record header. When a key update occurs, the new epoch's keys are installed and used for all new records; records carrying the old epoch number are still decryptable using the retained prior-epoch keys.

This is necessary because UDP does not guarantee ordering. A key update might be acknowledged, but old records sent before the update can arrive after the update completes. Dropping them would violate application-level message delivery guarantees.

```ferrum
type EpochConfig {
    // How long to retain keys for a superseded epoch after a key update.
    // During this window, records tagged with the old epoch are decrypted
    // normally. After the window, old-epoch records return EpochError.
    // Default: 2 * maximum expected network RTT, or 30s if unknown.
    pub retention_window: Duration,

    // Maximum number of simultaneous epochs to retain.
    // Guards against an adversary triggering unbounded key accumulation
    // by sending synthetic KeyUpdate messages.
    // Default: 3 (current + 2 prior).
    pub max_retained_epochs: u8,
}

impl EpochConfig {
    const DEFAULT: Self = EpochConfig {
        retention_window:    Duration.from_secs(30),
        max_retained_epochs: 3,
    }
}
```

### Key Update Lifecycle

1. Either peer sends a `KeyUpdate` handshake message to signal intent to switch to a new epoch.
2. The sender immediately starts encrypting new records under the new epoch's keys.
3. The receiver installs the new epoch's derived keys and begins accepting records under the new epoch.
4. The receiver retains the prior epoch's keys for `retention_window`.
5. After `retention_window` elapses, the prior epoch's keys are zeroed from memory.
6. Records arriving after key eviction and tagged with the old epoch return `DtlsError.EpochError`.

Key material is zeroed on eviction using a memory-clearing operation that is not subject to compiler dead-store elimination (equivalent to `explicit_bzero` or `SecureZero` semantics).

Application code triggers key updates via `DtlsSession::update_keys()`. The runtime also triggers key updates automatically before the AEAD sequence number approaches the 2^64 limit.

---

## 9. Handshake Retransmission and Fragmentation

### Flight-Based Retransmission

The DTLS handshake is divided into flights — groups of messages sent together by one side. A side retransmits its last flight if it does not receive the first message of the next expected flight within the retransmission timeout.

```
Client                              Server
------                              ------
Flight 1: ClientHello           →
                                ←   Flight 2: HelloVerifyRequest
Flight 3: ClientHello(cookie)   →
                                ←   Flight 4: ServerHello, Certificate,
                                             CertificateRequest, ServerHelloDone
Flight 5: Certificate,          →
          ClientKeyExchange,
          CertificateVerify,
          ChangeCipherSpec,
          Finished
                                ←   Flight 6: ChangeCipherSpec, Finished
```

Each flight is retransmitted as a unit. The retransmission timer doubles on each retry (exponential backoff) up to `retransmit.max_timeout`. After `retransmit.max_retransmits` attempts, the handshake fails with `DtlsError.Timeout`.

### Message Fragmentation

Handshake messages — particularly `Certificate` — often exceed the network's path MTU. DTLS fragments them:

```ferrum
// Internal record header fields (not exposed to application code)
type HandshakeMsgHeader {
    msg_type:    u8,
    length:      u24,         // total message length before fragmentation
    msg_seq:     u16,         // per-message sequence number
    frag_offset: u24,         // byte offset of this fragment within message
    frag_length: u24,         // byte count of this fragment
}
```

The fragment reassembly buffer is bounded by `max_dgram_size * max_retained_fragments` (default: 16 fragments per message). If a handshake message cannot be reassembled within this bound, the handshake fails with `DtlsError.HandshakeFailed`.

Duplicate fragments (same `msg_seq`, same `frag_offset`) are silently discarded to handle network duplication.

### MTU Probing

`DtlsClient` and `DtlsServer` do not implement PMTU discovery automatically. Applications that know the path MTU should configure `max_dgram_size` in `DtlsConfig` to avoid fragmentation at the IP layer. The default `max_dgram_size` is 1200 bytes — a conservative value that fits within typical tunnel and VPN overhead without IP fragmentation.

---

## 10. Version Support Policy

| Version | RFC | Status |
|---|---|---|
| DTLS 1.0 | RFC 4347 | Not implemented. Deprecated by RFC 9147. |
| DTLS 1.1 | RFC 4347 errata | Not implemented. Same deprecation; no separate RFC was ever published. |
| DTLS 1.2 | RFC 6347 | Supported. Required for compatibility with deployed IoT and RADIUS infrastructure. |
| DTLS 1.3 | RFC 9147 | Supported. Preferred when both sides support it. |

DTLS 1.0 used MD5 and SHA-1 in its PRF and supported export-grade cipher suites. DTLS 1.1 was a version number correction within the same RFC and never saw independent deployment. Neither version is implementable without introducing broken cryptography, which violates Ferrum `crypto`'s stated policy of not exposing deprecated algorithms.

When a DTLS 1.2 client connects, `DtlsServer` will negotiate DTLS 1.2 if `config.versions.dtls_12` is `true`. If only `dtls_13` is enabled and a DTLS 1.2-only client connects, the handshake fails with a `protocol_version` alert, and `DtlsServer::accept` yields `DtlsError.HandshakeFailed`.

---

## 11. Integration with net.dgram

`DtlsServer` wraps `UdpSocket` from `std.net`. The socket is created internally by `DtlsServer::bind`, but can be accessed for platform-specific configuration (socket buffer sizes, `SO_REUSEPORT` on Linux, etc.) before the server starts accepting.

```ferrum
impl DtlsServer {
    // Access the underlying socket for platform socket options.
    // The socket is in non-blocking mode; do not call blocking send/recv on it.
    pub fn as_raw_socket(&self): &UdpSocket
}
```

The `net.dgram.DatagramSchema` (from `ferrum-stdlib-async-net.md §13`) provides amplification protection for non-DTLS datagram protocols. `DtlsServer` does not use `DatagramSchema` — it implements its own amplification protection via cookie exchange, which is the DTLS-specified mechanism and provides stronger guarantees than a generic response-size multiplier limit.

---

## 12. Integration with CoAP extlib

CoAP (`coap://`) runs over UDP without security. Secure CoAP (`coaps://`) mandates DTLS as the transport security layer (RFC 7252 §9). The CoAP extlib (`extlib.coap`) uses `extlib.dtls` automatically for `coaps://` URIs.

```ferrum
// CoAP client — coaps:// uses DtlsClient internally
fn coap_get(
    uri:    &Uri,
    config: &DtlsConfig,
): Result[CoapResponse, CoapError] ! Async + Net

// CoAP server — coaps:// listener uses DtlsServer internally
type CoapServer

impl CoapServer {
    fn bind_secure(
        addr:        SocketAddr,
        dtls_config: DtlsConfig,
        routes:      CoapRoutes,
    ): Result[Self, CoapError] ! Async + Net
}
```

Applications using only CoAP do not need to interact with `extlib.dtls` directly. They provide a `DtlsConfig` to the CoAP API and receive `CoapRequest` / `CoapResponse` values. The DTLS session lifecycle (handshake, epoch management, retransmission) is entirely managed by the CoAP layer.

---

## 13. Error Types

```ferrum
enum DtlsError {
    // Handshake did not complete. Inner string is the TLS alert description
    // or a protocol-level description ("unexpected message in state S").
    HandshakeFailed(String),

    // Certificate validation failed. Wraps the extlib.tls error.
    CertificateError(TlsCertError),

    // AEAD authentication tag verification failed on an incoming record.
    // The record is silently dropped — this variant is only reported on
    // DtlsSession::read when the failure is unambiguously non-transient
    // (e.g., a sequence of consecutive failures suggesting key mismatch).
    DecryptionFailed,

    // Application tried to read into a buffer smaller than the incoming
    // decrypted record, or the incoming record exceeds max_dgram_size.
    RecordTooLarge { record_bytes: usize, buf_bytes: usize },

    // Incoming record carries an epoch number for which keys have been
    // evicted (arrived after retention_window) or an epoch number that
    // has never been negotiated.
    EpochError { epoch: u16 },

    // Handshake or read/write timed out after max_retransmits.
    Timeout,

    // Received a ClientHello with a cookie that does not verify.
    // Logged and dropped — not propagated to application code under
    // normal operation; only appears if explicitly monitoring DtlsServer
    // diagnostics.
    InvalidCookie,

    // Version negotiation failed: no version in common.
    VersionMismatch,

    // Underlying UdpSocket or OS error.
    Io(IoError),
}

impl Display for DtlsError { ... }
impl Error   for DtlsError { ... }
```

`InvalidCookie` is included for completeness and diagnostic use. Under normal operation, `DtlsServer::accept` does not yield `InvalidCookie` — the server silently drops the ClientHello and waits for the client to retry. It is exposed in the diagnostic API (`DtlsServerMetrics`) for monitoring.

---

## 14. Example Usage

### DTLS Server (CoAP / General Purpose)

```ferrum
use extlib.dtls.{DtlsConfig, DtlsMode, DtlsServer, DtlsSession, CookieSecret,
                  DtlsVersionSet, ClientAuth}
use extlib.tls.{CertChain, PrivateKey, CaStore}
use std.net.SocketAddr
use std.async.AsyncReadExt
use std.async.AsyncWriteExt

fn load_server_config(): Result[DtlsConfig, DtlsError] ! IO {
    let cert = CertChain.from_pem_file("server.crt")?
    let key  = PrivateKey.from_pem_file("server.key")?
    let ca   = CaStore.from_pem_file("ca.crt")?

    DtlsConfig.builder()
        .mode(DtlsMode.Server {
            cert,
            key,
            client_auth: ClientAuth.Required,
        })
        .versions(DtlsVersionSet.DTLS_12_AND_13)
        .cookie_secret(CookieSecret.generate()?)
        .build()
}

async fn handle_session(mut session: DtlsSession): Result[(), DtlsError] ! Async + Net {
    let peer = session.peer_addr()
    let suite = session.cipher_suite()
    let cert  = session.peer_cert()

    // Log authenticated peer
    if let Some(chain) = cert {
        log.info("dtls: peer={peer} cipher={suite} subject={chain.leaf().subject()}")
    }

    let mut buf = [0u8; 1280]
    loop {
        let n = match session.read(&mut buf).await {
            Ok(0)    => break,            // close_notify received
            Ok(n)    => n,
            Err(e)   => return Err(e),
        }
        let payload = &buf[..n]
        // ... application protocol dispatch (e.g., CoAP) ...
        let response = process_coap(payload)?
        session.write_all(&response).await?
    }

    session.close().await?
    Ok(())
}

async fn run_server(): Result[(), DtlsError] ! Async + Net + IO {
    let config = load_server_config()?
    let addr   = SocketAddr.from_str("0.0.0.0:5684")?  // coaps:// standard port
    let server = DtlsServer.bind(addr, config)?

    let mut incoming = server.accept()
    while let Some(result) = incoming.next().await {
        match result {
            Ok(session) => {
                runtime.spawn(handle_session(session))
            }
            Err(e) => {
                log.warn("dtls: handshake error: {e}")
                // Continue accepting; one failed handshake does not stop the server
            }
        }
    }

    Ok(())
}
```

### DTLS Client (RADIUS/DTLS)

RADIUS/DTLS (RFC 7360) uses DTLS as a drop-in replacement for RADIUS over UDP. The DTLS session carries standard RADIUS PDUs. Both peers authenticate with certificates.

```ferrum
use extlib.dtls.{DtlsConfig, DtlsMode, DtlsClient, DtlsVersionSet}
use extlib.tls.{CaStore, CertChain, PrivateKey, ServerName}
use std.net.SocketAddr
use std.async.{AsyncReadExt, AsyncWriteExt}

async fn radius_dtls_request(
    server:  SocketAddr,
    request: &[u8],
): Result[Vec[u8], DtlsError] ! Async + Net + IO {
    let ca          = CaStore.from_pem_file("radius-ca.crt")?
    let client_cert = CertChain.from_pem_file("client.crt")?
    let client_key  = PrivateKey.from_pem_file("client.key")?

    let config = DtlsConfig.builder()
        .mode(DtlsMode.Client {
            server_name: Some(ServerName.from_str("radius.example.com")?),
            ca_certs:    ca,
            client_cert: Some((client_cert, client_key)),
        })
        .versions(DtlsVersionSet.DTLS_12_AND_13)
        .cookie_secret(CookieSecret.generate()?)
        .build()?

    // DtlsClient::connect handles HelloVerifyRequest and retransmission
    let mut session = DtlsClient.connect(server, config).await?

    session.write_all(request).await?

    let mut buf = Vec.with_capacity(4096)
    session.read_to_end(&mut buf).await?

    session.close().await?
    Ok(buf)
}
```

### DTLS 1.3 with Key Update

```ferrum
async fn long_running_session(
    mut session: DtlsSession,
): Result[(), DtlsError] ! Async + Net {
    let mut messages_sent: u64 = 0

    loop {
        let data = next_application_message().await
        session.write_all(&data).await?
        messages_sent += 1

        // Rotate keys every 100,000 messages for forward secrecy
        // (the runtime also rotates automatically before nonce exhaustion)
        if messages_sent % 100_000 == 0 && session.version() == DtlsVersion.Dtls13 {
            session.update_keys().await?
        }
    }
}
```

---

*End of extlib.dtls design document.*

*See also:*
- *[ferrum-stdlib-async-net.md](ferrum-stdlib-async-net.md) — UdpSocket, AsyncRead, AsyncWrite, net.dgram*
- *[ferrum-stdlib-crypto-testing.md](ferrum-stdlib-crypto-testing.md) — Hmac, Aead, SystemRng, ct_eq*
- *[ferrum-stdlib.md](ferrum-stdlib.md) — Standard library index*
