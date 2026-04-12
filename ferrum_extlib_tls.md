# Ferrum Extended Library — TLS Module

**Module path:** `extlib.tls` (provided by `lib_ccsp_tls`)
**Part of:** Ferrum Extended Standard Library (extlib)

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [Dependencies](#2-dependencies)
3. [Cipher Suites](#3-cipher-suites)
4. [TLS Client](#4-tls-client)
5. [TLS Server](#5-tls-server)
6. [TlsStream](#6-tlsstream)
7. [SNI Requirement](#7-sni-requirement)
8. [Heartbleed Structural Fix](#8-heartbleed-structural-fix)
9. [QUIC Transport API](#9-quic-transport-api)
10. [Post-Quantum Key Exchange](#10-post-quantum-key-exchange)
11. [Certificate Store](#11-certificate-store)
12. [Error Types](#12-error-types)
13. [Example Usage](#13-example-usage)
14. [Dependencies Reference](#14-dependencies-reference)

---

## 1. Overview and Rationale

### Why TLS Is an Extended Library Module

The Ferrum standard library (`std`) covers the primitives every program needs: memory allocation, async I/O, TCP/UDP sockets, and low-level cryptographic building blocks (AEAD ciphers, hash functions, HMAC, key derivation). A full TLS stack sits above this baseline for several reasons:

**Binary size and embedded targets.** A TLS implementation — certificate parsing, X.509 chain validation, ECDH key agreement, the full handshake state machine — adds substantial code to any binary that links it. An embedded controller managing a CAN bus, a bootloader verifying a firmware signature, or a bare-metal sensor node running over a proprietary radio protocol does not need TLS. Placing TLS in stdlib would impose its cost on every Ferrum program. The extended library is opt-in.

**Dependency cycle risk.** The standard library's DNS resolver is used by many programs, including those that eventually make TLS connections. A TLS module in stdlib that needs hostname validation would create a circular dependency with the DNS module. The extlib boundary breaks this cycle: `extlib.tls` depends on `std.net` and `std.crypto`; `extlib.dns_secure` depends on `extlib.tls` for encrypted transport without any reverse dependency.

**Policy decisions belong at the boundary.** TLS is not a neutral transport — it embeds security policy. Which versions to support, which cipher suites to allow, whether to require SNI, what to do with renegotiation — these are engineering decisions with correctness implications. The stdlib is not the right place to enshrine them for all callers. The extlib module makes these decisions explicitly and documents them here.

### Explicit Security Policy

This module implements TLS 1.2 and TLS 1.3 only. The policy decisions below are not configuration options — they are structural properties of the implementation:

**TLS 1.0 and 1.1 are absent, not disabled.** There is no configuration knob to enable them. The handshake state machine does not contain code paths for TLS 1.0 or 1.1. A `ClientHello` that offers only those versions will receive a `protocol_version` alert; the connection will not be established. This is intentional: TLS 1.0 uses MD5 and SHA-1 in its PRF; TLS 1.1 is a minor revision of the same RFC (RFC 4346) that never addressed the underlying cryptographic weaknesses. Neither version can be made secure. The correct response to a deployment that requires them is to fix the server, not to provide a compatibility shim.

**SNI is required, not optional.** A `TlsClientConfig` cannot be built without a server name (see §7). Connections without SNI are structurally impossible through the public API.

**No renegotiation.** TLS 1.2 renegotiation is the mechanism behind CVE-2009-3555 and a persistent source of protocol confusion attacks. This implementation does not initiate renegotiation and does not respond to incoming renegotiation requests. An incoming `ClientHello` on an established TLS 1.2 session triggers a `no_renegotiation` warning alert followed by connection closure.

**CBC cipher suites are absent.** CBC-mode TLS has been the source of BEAST (CVE-2011-3389), Lucky Thirteen (CVE-2013-0169), POODLE (CVE-2014-3566), and related timing-oracle attacks. All cipher suites in this module use AEAD construction (GCM or ChaCha20-Poly1305). CBC suites are not in the negotiation list; they cannot be selected.

**RC4, DES, 3DES, export ciphers, NULL, and anonymous DH are absent.** These are never negotiated for the same reason: they are broken.

---

## 2. Dependencies

```
extlib.tls
    std.crypto          — Aes256Gcm, Aes128Gcm, ChaCha20Poly1305,
                          Sha256, Sha384, HkdfSha256, HkdfSha384,
                          P256, P384, X25519, Ed25519, SystemRng, ct_eq
    std.net             — TcpStream, SocketAddr
    std.async           — Future, AsyncRead, AsyncWrite, Runtime
    std.io              — IoError, ReadResult
    std.alloc           — Vec, Arc, Box
    extlib.cert         — Certificate, CertChain, PrivateKey,
                          CertificateVerifier, OcspResponse
    extlib.pki          — X509ChainValidator, CertStore, RevocationPolicy
```

`extlib.cert` handles certificate parsing and DER/PEM encoding. `extlib.pki` handles X.509 chain validation, revocation, and trust anchor management. `extlib.tls` consumes both; it does not re-implement certificate handling.

---

## 3. Cipher Suites

### 3.1 TLS 1.3 Cipher Suites

TLS 1.3 cipher suites are negotiated automatically by the implementation. Callers cannot configure or restrict this list. The negotiation order below is the library's preference; the server selects from this list in their own preference order, and the library accepts whichever intersection is reached.

| Preference | Suite | Notes |
|---|---|---|
| 1 | `TLS_AES_256_GCM_SHA384` | 256-bit AES-GCM, HKDF-SHA-384. Preferred for PQ-resistant symmetric strength. |
| 2 | `TLS_CHACHA20_POLY1305_SHA256` | Software-friendly. Preferred when AES hardware acceleration is absent. |
| 3 | `TLS_AES_128_GCM_SHA256` | 128-bit AES-GCM, HKDF-SHA-256. Accepted for interoperability. |

`TLS_AES_128_CCM_SHA256` and `TLS_AES_128_CCM_8_SHA256` are not included. CCM is correct but uncommon in TLS deployments and adds implementation surface for minimal benefit.

The implementation detects at runtime whether AES-NI is available. When present, AES-GCM suites are preferred; when absent, ChaCha20-Poly1305 is promoted to first preference. This decision is transparent to application code.

### 3.2 TLS 1.2 Cipher Suites

TLS 1.2 is supported for servers that cannot yet deploy TLS 1.3. Only ECDHE-based AEAD suites are offered.

| Suite | Key Exchange | Auth | Cipher | MAC |
|---|---|---|---|---|
| `ECDHE-ECDSA-AES256-GCM-SHA384` | ECDHE | ECDSA | AES-256-GCM | SHA-384 |
| `ECDHE-RSA-AES256-GCM-SHA384` | ECDHE | RSA | AES-256-GCM | SHA-384 |
| `ECDHE-ECDSA-CHACHA20-POLY1305` | ECDHE | ECDSA | ChaCha20-Poly1305 | — |
| `ECDHE-RSA-CHACHA20-POLY1305` | ECDHE | RSA | ChaCha20-Poly1305 | — |

The 256-GCM suites are offered first. As with TLS 1.3, AES-NI detection affects runtime preference between AES-GCM and ChaCha20-Poly1305.

### 3.3 Absent Cipher Suite Categories

The following categories are not in the negotiation list and cannot be enabled:

| Category | Examples | Reason absent |
|---|---|---|
| CBC | `ECDHE-RSA-AES256-SHA384`, `AES128-SHA` | Timing oracle attacks (Lucky Thirteen, POODLE) |
| RC4 | `RC4-MD5`, `RC4-SHA` | Stream cipher broken; RFC 7465 prohibits it |
| DES / 3DES | `DES-CBC3-SHA` | DES is 56-bit; 3DES has Sweet32 (CVE-2016-2183) |
| RSA key exchange | `RSA-AES128-GCM-SHA256` | No forward secrecy; server private key decrypts past sessions |
| Anonymous DH | `ADH-AES256-GCM-SHA384` | No peer authentication |
| Export ciphers | `EXP-RC4-MD5` | 40-bit key; export-regime relics |
| NULL | `NULL-MD5` | Provides authentication only; no confidentiality |

### 3.4 Cipher Suite Type

```ferrum
enum CipherSuite {
    // TLS 1.3
    TLS_AES_256_GCM_SHA384,
    TLS_CHACHA20_POLY1305_SHA256,
    TLS_AES_128_GCM_SHA256,

    // TLS 1.2
    ECDHE_ECDSA_AES256_GCM_SHA384,
    ECDHE_RSA_AES256_GCM_SHA384,
    ECDHE_ECDSA_CHACHA20_POLY1305,
    ECDHE_RSA_CHACHA20_POLY1305,
}

impl Display for CipherSuite { ... }
```

### 3.5 TLS Version Type

```ferrum
enum TlsVersion {
    Tls12,
    Tls13,
}

impl Display for TlsVersion { ... }
```

---

## 4. TLS Client

### 4.1 TlsClientConfig

`TlsClientConfig` is constructed through a typestate builder that enforces SNI at the type level (see §7 for the full typestate explanation). The completed config is immutable and `Clone`-able at `Arc` cost.

```ferrum
// Completed, immutable client configuration.
// Constructed via TlsClientConfig::builder().
type TlsClientConfig
```

#### Builder Methods

```ferrum
impl TlsClientConfigBuilder<NoServerName> {
    // Start a new builder. server_name is required before .build() is available.
    pub fn new(): Self

    // Required: set the server name for SNI and certificate validation.
    // Consuming self, returns a builder in the HasServerName state where
    // .build() becomes available.
    pub fn server_name(self, name: &str): Result[TlsClientConfigBuilder<HasServerName>, TlsError]
}

impl TlsClientConfigBuilder<HasServerName> {
    // Trust anchor store. Default: CertStore::system().
    pub fn ca_certs(mut self, store: CertStore): Self

    // Present a client certificate during mutual TLS.
    // cert is the chain from leaf to intermediate (not including root).
    // key is the corresponding private key.
    pub fn client_cert(mut self, cert: Arc[Certificate], key: PrivateKey): Self

    // ALPN protocol identifiers, in preference order.
    // e.g. ["h2", "http/1.1"] for an HTTP client.
    pub fn alpn_protocols(mut self, protocols: Vec[&str]): Self

    // Minimum TLS version to accept. Default: TlsVersion::Tls12.
    // Setting Tls13 refuses TLS 1.2 connections.
    pub fn min_version(mut self, version: TlsVersion): Self

    // Enable X25519Kyber768 hybrid post-quantum key exchange.
    // Default: false. Requires feature = "post-quantum".
    // When enabled, the hybrid is offered; fallback to X25519 when
    // the server does not support the hybrid group.
    pub fn post_quantum(mut self, enabled: bool): Self

    // Enable TLS session ticket resumption. Default: true.
    // Tickets allow a resumed handshake with fewer round trips.
    // Disable for deployments where session tracking is a concern.
    pub fn session_tickets(mut self, enabled: bool): Self

    // Finalize the configuration.
    pub fn build(self): Result[TlsClientConfig, TlsError]
}
```

### 4.2 TlsConnector

`TlsConnector` wraps a `TlsClientConfig` and provides the connection upgrade method. It is cheaply cloneable — the internal config is `Arc`-wrapped.

```ferrum
type TlsConnector

impl TlsConnector {
    pub fn new(config: TlsClientConfig): Self

    // Upgrade an existing TCP stream to a TLS stream.
    //
    // server_name overrides the SNI hostname for this specific connection,
    // which is useful when the same TlsConnector is reused across hosts
    // (e.g., in a connection pool). When None, uses the server_name from
    // TlsClientConfig.
    //
    // Performs the full TLS handshake before returning.
    // Effects: Async (suspends during handshake), Net (sends/receives)
    pub fn connect(
        &self,
        stream: TcpStream,
        server_name: &str,
    ): Result[TlsStream, TlsError] ! Async + Net
}
```

### 4.3 Convenience Function

```ferrum
// One-shot TLS client connection without constructing a TlsConnector.
// Equivalent to TlsConnector::new(config).connect(stream, server_name).
pub fn tls_connect(
    stream:      TcpStream,
    server_name: &str,
    config:      TlsClientConfig,
): Result[TlsStream, TlsError] ! Async + Net
```

---

## 5. TLS Server

### 5.1 TlsServerConfig

```ferrum
// Completed, immutable server configuration.
// Constructed via TlsServerConfig::builder().
type TlsServerConfig
```

#### Builder Methods

```ferrum
impl TlsServerConfigBuilder {
    pub fn new(): Self

    // Certificate chain from leaf to intermediate (not including trust anchor).
    // Required before .build() is available.
    pub fn cert_chain(mut self, chain: Vec[Arc[Certificate]]): Self

    // Private key corresponding to the leaf certificate's public key.
    // Required before .build() is available.
    pub fn private_key(mut self, key: PrivateKey): Self

    // Client authentication policy. Default: ClientAuth::None.
    pub fn client_auth(mut self, auth: ClientAuth): Self

    // ALPN protocol identifiers, in preference order.
    // The server selects the first client-offered identifier that appears
    // in this list, or rejects the connection if there is no intersection.
    // When empty, ALPN is not negotiated (accepted for any client ALPN offer).
    pub fn alpn_protocols(mut self, protocols: Vec[&str]): Self

    // Enable TLS session ticket resumption. Default: true.
    pub fn session_tickets(mut self, enabled: bool): Self

    // Provide a pre-computed OCSP staple to send to clients.
    // The bytes must be a valid DER-encoded OCSPResponse (RFC 6960).
    // The staple is not validated by the library; callers are responsible
    // for obtaining a fresh staple from the CA's OCSP responder.
    pub fn ocsp_staple(mut self, staple: Vec[u8]): Self

    // SNI callback for virtual hosting. See §5.3.
    // When set, the callback is invoked with the SNI hostname after
    // ClientHello is received. The returned config overrides self for
    // that connection. Return None to use self (default config).
    pub fn sni_resolver(
        mut self,
        f: fn(&str) -> Option[TlsServerConfig],
    ): Self

    pub fn build(self): Result[TlsServerConfig, TlsError]
}
```

### 5.2 ClientAuth

```ferrum
enum ClientAuth {
    // No client certificate requested. Default.
    None,

    // Client certificate is requested but not required.
    // If the client presents one, it is validated against ca_certs.
    // HandshakeData::peer_cert_chain is Some on success, None if the
    // client declined.
    Optional { ca_certs: CertStore },

    // Client certificate is required. Handshake fails with
    // TlsError::NoCertificatePresented if the client does not send one.
    // All presented certificates are validated against ca_certs.
    Required { ca_certs: CertStore },
}
```

### 5.3 TlsAcceptor

```ferrum
type TlsAcceptor

impl TlsAcceptor {
    pub fn new(config: TlsServerConfig): Self

    // Upgrade an accepted TCP stream to a TLS stream.
    //
    // The SNI hostname from the ClientHello is extracted and passed to the
    // sni_resolver callback (if configured) to select per-vhost config.
    // The handshake proceeds with the selected config.
    //
    // Returns TlsStream only after the full handshake completes.
    pub fn accept(
        &self,
        stream: TcpStream,
    ): Result[TlsStream, TlsError] ! Async + Net
}
```

### 5.4 SNI-Based Virtual Hosting

The `.sni_resolver()` callback enables a single `TlsAcceptor` to serve multiple virtual hosts, each with its own certificate and configuration:

```ferrum
// Each vhost gets its own TlsServerConfig with its own cert/key.
// The resolver is called after ClientHello is parsed, before the
// Certificate message is sent, so the correct cert is presented.
fn make_sni_resolver(
    vhosts: HashMap[String, TlsServerConfig],
): fn(&str) -> Option[TlsServerConfig] {
    move |hostname| {
        vhosts.get(hostname).cloned()
    }
}
```

If the callback returns `None`, the acceptor uses its own default `TlsServerConfig`. If the client sends no SNI extension — which this library treats as a protocol error at the client end; see §7 — the callback receives an empty string and typically returns `None` (using the default certificate, or optionally refusing the connection).

---

## 6. TlsStream

`TlsStream` is the result of a completed TLS handshake — either client-side (from `TlsConnector::connect`) or server-side (from `TlsAcceptor::accept`).

### 6.1 Trait Implementations

```ferrum
// TlsStream implements the same async I/O traits as TcpStream.
// Application code that abstracts over AsyncRead + AsyncWrite works
// identically whether the underlying transport is TLS or plaintext.
impl AsyncRead  for TlsStream
impl AsyncWrite for TlsStream
```

Both traits operate on the application data layer. Encryption and decryption are invisible to the caller: `write` encrypts and transmits; `read` receives and decrypts. The caller never sees TLS record boundaries.

### 6.2 Methods

```ferrum
type TlsStream

impl TlsStream {
    // Handshake metadata. Available immediately after the stream is created.
    pub fn handshake_data(&self): &HandshakeData

    // Graceful closure: sends a close_notify alert, then waits to receive
    // the peer's close_notify before returning.
    //
    // After this returns Ok(()), the stream is unusable. Subsequent reads
    // return ReadResult::Eof; subsequent writes return TlsError::Io.
    //
    // If the peer closes the underlying TCP connection without sending
    // close_notify, this returns Ok(()) after a short timeout to avoid
    // indefinite blocking. The absence of close_notify is noted in
    // TlsError context but is not treated as a fatal error in most
    // applications (HTTP/1.1 connection teardown, for example, commonly
    // omits close_notify).
    pub fn close(&mut self): Result[(), TlsError] ! Async

    // Access the underlying TcpStream after graceful closure.
    // Only available after close() returns Ok(()).
    // Useful when the same TCP connection will be reused for another
    // protocol after TLS (HTTP Upgrade pattern).
    pub fn into_inner(self): TcpStream
}
```

### 6.3 HandshakeData

```ferrum
type HandshakeData {
    // TLS version that was negotiated.
    pub protocol_version: TlsVersion,

    // Cipher suite that was selected.
    pub cipher_suite: CipherSuite,

    // ALPN protocol identifier agreed during handshake.
    // None if ALPN was not negotiated.
    pub alpn_negotiated: Option[String],

    // Certificate chain presented by the peer.
    // On client streams: Some with the server's chain.
    // On server streams with ClientAuth::Required or Optional (when
    //   client presented a cert): Some with the client's chain.
    // On server streams with ClientAuth::None: None.
    pub peer_cert_chain: Option[Vec[Arc[Certificate]]],

    // SNI hostname from the ClientHello.
    // Always Some on server streams.
    // None on client streams (SNI was sent, not received).
    pub sni_hostname: Option[String],

    // Whether this connection resumed a previous session via a session ticket.
    pub session_resumed: bool,

    // Whether X25519Kyber768 post-quantum hybrid key exchange was used.
    // false when the "post-quantum" feature is disabled or when the peer
    // did not support the hybrid group.
    pub post_quantum_used: bool,
}
```

### 6.4 No Renegotiation

If an established TLS 1.2 connection receives a renegotiation `ClientHello`, the implementation sends a `no_renegotiation` warning alert and closes the connection. Application code does not need to handle this: the next `read` or `write` call returns `TlsError::ProtocolViolation { reason: "renegotiation not supported" }`.

TLS 1.3 does not define renegotiation. Received `KeyUpdate` messages are processed transparently (they update traffic keys; the application sees no interruption).

---

## 7. SNI Requirement

### 7.1 Rationale

Server Name Indication (RFC 6066) allows a client to identify which virtual host it intends to connect to, so the server can present the correct certificate before the handshake completes. A TLS connection without SNI is ambiguous: the server must guess which certificate to present, often presenting the wrong one for multi-vhost deployments, and the client cannot validate the hostname it intended to reach.

Anonymous TLS — TLS without SNI — provides confidentiality and integrity against passive observers but provides no authentication guarantee. An attacker that can intercept TCP connections can substitute any valid certificate from any trusted CA. This is not a theoretical concern: it is the attack model for hotel captive portals, corporate inspection proxies, and national-scale interception infrastructure.

Requiring SNI eliminates the ambiguity on both sides and is mandatory for correct operation with any modern TLS server deployment. The cost is zero: SNI has been supported by every major TLS implementation since 2009.

### 7.2 Typestate Enforcement

The builder for `TlsClientConfig` uses a typestate pattern to make it a compile error (not a runtime error) to build a config without a server name:

```ferrum
// Phantom types marking builder state
type NoServerName
type HasServerName

// Builder is generic over its state
type TlsClientConfigBuilder[State]

impl TlsClientConfigBuilder<NoServerName> {
    pub fn new(): Self { ... }

    // Calling .server_name() transitions to HasServerName.
    // The return type changes — this is not the same type as before.
    pub fn server_name(
        self,
        name: &str,
    ): Result[TlsClientConfigBuilder<HasServerName>, TlsError]
}

// .build() is only available on the HasServerName state.
// Code that calls TlsClientConfigBuilder::new().build() does not compile:
// the method does not exist on TlsClientConfigBuilder<NoServerName>.
impl TlsClientConfigBuilder<HasServerName> {
    pub fn build(self): Result[TlsClientConfig, TlsError]

    // All other builder methods are available on HasServerName only,
    // to ensure they are not called before server_name is set.
    pub fn ca_certs(mut self, store: CertStore): Self
    pub fn client_cert(mut self, cert: Arc[Certificate], key: PrivateKey): Self
    pub fn alpn_protocols(mut self, protocols: Vec[&str]): Self
    pub fn min_version(mut self, version: TlsVersion): Self
    pub fn post_quantum(mut self, enabled: bool): Self
    pub fn session_tickets(mut self, enabled: bool): Self
}
```

The error is structural, not just diagnostic:

```ferrum
// This does not compile — .build() does not exist on NoServerName state:
let config = TlsClientConfigBuilder::new()
    .ca_certs(CertStore::system()?)
    .build()?  // error: no method 'build' on TlsClientConfigBuilder<NoServerName>

// This compiles:
let config = TlsClientConfigBuilder::new()
    .server_name("api.example.com")?
    .ca_certs(CertStore::system()?)
    .build()?
```

### 7.3 Encrypted Client Hello

Encrypted Client Hello (ECH, RFC draft) hides the SNI and other ClientHello fields from on-path observers by encrypting them under a public key published in the server's HTTPS/SVCB DNS record. ECH is negotiated transparently: when the server advertises ECH support and the implementation encounters an ECH configuration (typically provided via `extlib.connect`'s HTTPS record integration), ECH is used automatically. No API change is required. `HandshakeData` does not expose an ECH field; `extlib.connect` records the ECH outcome in `TransportInfo` for diagnostic purposes.

---

## 8. Heartbleed Structural Fix

The Heartbleed vulnerability (CVE-2014-0160) was a read beyond the end of a heap buffer in OpenSSL's implementation of the TLS heartbeat extension (RFC 6520). The root cause was not the heartbeat extension itself but a structural property of OpenSSL's implementation: the handshake layer held a pointer into the record layer's buffer, and a malformed heartbeat length field could direct that pointer beyond the record boundary, leaking adjacent heap contents.

This module does not implement the heartbeat extension (RFC 6520). Heartbeat was primarily used for DTLS path liveness; `extlib.dtls` also omits it. But the structural property that enabled Heartbleed — a handshake-layer pointer into a record-layer buffer — is corrected by design:

**The record layer and handshake layer have separate, independently bounded buffers.** The record layer reads from the network into its own buffer, validates the record header length field against the physical bytes received, decrypts in place, and delivers a bounded plaintext slice to the handshake layer. The handshake layer maintains its own buffer into which it copies data from the record layer slice. There is no pointer from the handshake layer into the record layer's buffer: the only connection is a copy through a bounded slice operation.

**Record layer buffer limit:** 16,384 bytes (the TLS maximum record size, RFC 8446 §5.1). Records larger than this are rejected with `TlsError::RecordTooLarge` before decryption; no memory is allocated for the payload.

**Handshake layer buffer limit:** Handshake messages are bounded by the same 16,384-byte limit. Handshake messages that claim a length exceeding this bound are rejected with `TlsError::ProtocolViolation`.

These are implementation properties guaranteed by the CCSP library's code structure. They are documented here so security-conscious reviewers auditing the library can locate and verify the relevant invariant without reading the full implementation.

---

## 9. QUIC Transport API

QUIC (RFC 9000) embeds TLS 1.3 for its handshake and key schedule, but uses its own framing and record protection rather than TLS record layer framing. A QUIC implementation needs access to the TLS handshake state machine and the derived traffic keys, but not to TLS record layer encoding.

`extlib.tls` exposes this as `TlsQuicClient` and `TlsQuicServer`, which provide the TLS machinery that QUIC needs without the record layer that QUIC does not use.

### 9.1 TlsQuicClient

```ferrum
type TlsQuicClient

impl TlsQuicClient {
    pub fn new(config: TlsClientConfig): Self

    // Run the TLS 1.3 handshake over QUIC's crypto transport.
    //
    // quic_transport provides the functions used to send and receive
    // TLS handshake messages over QUIC CRYPTO frames. The handshake
    // messages are delivered by QUIC at the appropriate encryption level
    // (Initial, Handshake, Application).
    //
    // On success, returns QuicCryptoState containing the traffic keys
    // for each encryption level.
    pub fn quic_handshake(
        &self,
        server_name: &str,
        quic_transport: QuicTransportCallbacks,
    ): Result[QuicCryptoState, TlsError] ! Async
}
```

### 9.2 TlsQuicServer

```ferrum
type TlsQuicServer

impl TlsQuicServer {
    pub fn new(config: TlsServerConfig): Self

    pub fn quic_handshake(
        &self,
        quic_transport: QuicTransportCallbacks,
    ): Result[QuicCryptoState, TlsError] ! Async
}
```

### 9.3 QuicTransportCallbacks

QUIC provides its own reliable delivery of crypto data. The TLS handshake uses these callbacks to exchange data with QUIC rather than writing to a TLS record layer socket:

```ferrum
// Callbacks provided by the QUIC implementation to the TLS handshake engine.
type QuicTransportCallbacks {
    // Called by TLS to send handshake data at a given encryption level.
    // QUIC wraps this in a CRYPTO frame and sends it.
    pub send_crypto: fn(level: EncryptionLevel, data: &[u8]): Result[(), TlsError],

    // Called by TLS to read handshake data. QUIC delivers CRYPTO frames
    // from the peer by calling this.
    pub recv_crypto: fn(level: EncryptionLevel, data: &[u8]): Result[(), TlsError],

    // Called by TLS when traffic secrets are derived for a new level.
    // QUIC uses these secrets with HKDF-Expand-Label to derive the packet
    // protection keys (header protection, packet number encryption, AEAD key/IV).
    pub set_write_secret: fn(level: EncryptionLevel, secret: &TrafficSecret): Result[(), TlsError],
    pub set_read_secret:  fn(level: EncryptionLevel, secret: &TrafficSecret): Result[(), TlsError],

    // Called when the handshake completes at each level so QUIC can
    // discard Initial/Handshake keys.
    pub handshake_confirmed: fn(level: EncryptionLevel),
}

enum EncryptionLevel {
    Initial,
    EarlyData,
    Handshake,
    Application,
}
```

### 9.4 QuicCryptoState

```ferrum
// Traffic key material for all encryption levels, derived by the TLS handshake.
// The QUIC implementation uses these to construct AEAD contexts for packet protection.
type QuicCryptoState {
    // Seals (encrypts + authenticates) a QUIC packet payload.
    // level: the encryption level of the packet.
    // packet_number: QUIC packet number, used as the AEAD nonce.
    // header: the QUIC packet header, used as additional authenticated data.
    // payload: the plaintext payload.
    pub seal: fn(
        level:         EncryptionLevel,
        packet_number: u64,
        header:        &[u8],
        payload:       &[u8],
    ) -> Result[Vec[u8], TlsError],

    // Opens (decrypts + verifies) a QUIC packet payload.
    pub open: fn(
        level:         EncryptionLevel,
        packet_number: u64,
        header:        &[u8],
        ciphertext:    &[u8],
    ) -> Result[Vec[u8], TlsError],

    // Handshake data available after the handshake completes.
    pub handshake_data: HandshakeData,
}
```

The QUIC record layer uses its own framing (RFC 9000 §12). `QuicCryptoState.seal` and `.open` provide packet protection only; QUIC frame parsing, flow control, and stream multiplexing are the responsibility of the QUIC implementation. This module is not a QUIC implementation — it is the TLS substrate that a QUIC implementation links against.

---

## 10. Post-Quantum Key Exchange

**Feature gate:** `post-quantum`

### 10.1 X25519Kyber768 Hybrid

Post-quantum cryptography addresses the threat that a sufficiently capable quantum computer could break the elliptic-curve and finite-field assumptions underlying current TLS key exchange. ML-KEM (formerly Kyber), standardized as FIPS 203 in August 2024, provides key encapsulation with conjectured resistance to quantum attacks.

This module supports `X25519Kyber768` hybrid key exchange, combining classical X25519 with ML-KEM-768 (Kyber768). The hybrid approach is critical: if the ML-KEM assumption turns out to be incorrect — due to a classical or quantum attack on the lattice structure — the classical X25519 component still provides its usual security. The session key is the HKDF combination of both shared secrets; an attacker must break both to learn the session key.

The key exchange group identifier is `X25519Kyber768Draft00` (IANA group 0x6399, per the IETF draft and Chrome/BoringSSL deployment). This identifier may change as the standard finalizes; the library will track the assigned codepoint.

### 10.2 Configuration

```ferrum
// On TlsClientConfigBuilder<HasServerName>:
pub fn post_quantum(mut self, enabled: bool): Self
    // Default: false
    // When true: X25519Kyber768 is offered as the preferred key exchange group,
    //            with X25519 as fallback.
    // When false: only classical key exchange groups are offered.
```

Server-side post-quantum support is automatic when the feature is compiled in: the acceptor negotiates X25519Kyber768 if the client offers it, regardless of any server-side configuration. There is no server-side opt-out; a server that is compiled with `feature = "post-quantum"` will always accept the hybrid group from a client that offers it.

### 10.3 Negotiation Behavior

```
Client offers:   [X25519Kyber768, X25519, P-256]
Server supports: [X25519Kyber768, X25519]
Result:          X25519Kyber768 negotiated
HandshakeData:   post_quantum_used = true

Client offers:   [X25519Kyber768, X25519, P-256]
Server supports: [X25519, P-256]            (no PQ support)
Result:          X25519 negotiated (fallback)
HandshakeData:   post_quantum_used = false

Client offers:   [X25519, P-256]            (post_quantum: false)
Server supports: [X25519Kyber768, X25519]
Result:          X25519 negotiated
HandshakeData:   post_quantum_used = false
```

The fallback is automatic and transparent. The handshake proceeds normally; only `HandshakeData::post_quantum_used` records which path was taken.

### 10.4 FIPS 203 Reference

ML-KEM-768 is the middle security level of the three ML-KEM variants (ML-KEM-512, ML-KEM-768, ML-KEM-1024). ML-KEM-768 targets approximately NIST security category 3 (roughly comparable to AES-192). This is appropriate for TLS session key establishment combined with TLS 1.3's AES-256-GCM cipher suite.

The implementation uses the CCSP library's ML-KEM implementation, which is independently validated against the FIPS 203 Known Answer Tests.

---

## 11. Certificate Store

`CertStore` manages the set of trust anchors (CA certificates) used to validate peer certificate chains.

```ferrum
type CertStore

impl CertStore {
    // Load the operating system's default trust store.
    // On Linux: /etc/ssl/certs/ or the distro trust bundle.
    // On macOS: Security framework root certificates.
    // On Windows: Certificate Store (ROOT store).
    // Returns an error if the platform trust store is unavailable or empty.
    pub fn system(): Result[Self, TlsError] ! IO

    // Empty trust store. No certificates are trusted.
    // Useful as a starting point for building a custom store, or for
    // deployments that use CertStore::with_custom_validator exclusively.
    pub fn empty(): Self

    // Parse PEM-encoded certificates and use them as trust anchors.
    // pem may contain multiple certificates concatenated.
    // Returns an error if no valid certificates are found.
    pub fn from_pem(pem: &str): Result[Self, TlsError]

    // Add a single certificate to the store.
    pub fn add(&mut self, cert: Arc[Certificate]): Result[(), TlsError]

    // Merge another store's contents into this one.
    pub fn add_store(&mut self, other: &CertStore)

    // Attach a custom validation function.
    // The function receives the leaf certificate of each chain presented
    // by a peer. If it returns false, the certificate is rejected even
    // if the chain would otherwise pass PKIX validation.
    //
    // Use cases:
    //   - Certificate pinning: compare the certificate's public key
    //     fingerprint against a stored pin
    //   - DANE-EE (usage 3): bypass PKIX, accept only when the certificate
    //     matches a DNSSEC-validated TLSA record
    //   - SPKI pinning for internal services with a known key
    //
    // The validator is called after chain validation succeeds. Returning
    // false produces TlsError::CertificateVerification.
    pub fn with_custom_validator(
        mut self,
        f: fn(&Certificate) -> bool,
    ): Self

    // Number of trust anchors in the store.
    pub fn len(&self): usize
}
```

### Certificate and PrivateKey Types

These types are defined in `extlib.cert` and re-exported from `extlib.tls` for convenience:

```ferrum
// Re-exported from extlib.cert
pub use extlib.cert.Certificate
pub use extlib.cert.PrivateKey

impl Certificate {
    pub fn from_pem(pem: &str): Result[Arc[Self], TlsError]
    pub fn from_der(der: &[u8]): Result[Arc[Self], TlsError]

    pub fn subject(&self): &str
    pub fn issuer(&self): &str
    pub fn not_before(&self): DateTime
    pub fn not_after(&self): DateTime
    pub fn public_key_fingerprint_sha256(&self): [u8; 32]
    pub fn serial_number(&self): &[u8]
}

impl PrivateKey {
    // Load from PEM (PKCS#8 or SEC1 format).
    pub fn from_pem(pem: &str): Result[Self, TlsError]

    // Load from DER-encoded PKCS#8.
    pub fn from_der(der: &[u8]): Result[Self, TlsError]
}
```

---

## 12. Error Types

```ferrum
enum TlsError {
    // The TLS handshake did not complete. reason is a human-readable
    // description of the failure point (e.g., "certificate chain validation
    // failed", "no common cipher suite", "server sent unexpected message").
    HandshakeFailed { reason: String },

    // Certificate chain validation failed. error describes the specific
    // failure (expired, untrusted root, hostname mismatch, revoked, etc.).
    CertificateVerification { error: String },

    // A TLS alert was received from the peer. See RFC 8446 §6.
    AlertReceived { alert: TlsAlert },

    // An incoming TLS record claimed a payload length exceeding 16,384 bytes.
    // The record was rejected before decryption.
    RecordTooLarge { claimed_bytes: usize },

    // The peer sent a message that violates the TLS protocol state machine
    // or message structure. Includes renegotiation attempts.
    ProtocolViolation { reason: String },

    // Wraps an underlying I/O error from the TCP stream.
    Io(IoError),

    // The server required a client certificate and the client did not
    // present one. Also used server-side when ClientAuth::Required
    // is set and no certificate was received.
    NoCertificatePresented,

    // The server_name provided to the builder failed validation
    // (e.g., empty string, invalid hostname syntax).
    InvalidServerName { name: String },

    // session_tickets or OCSP staple configuration error.
    ConfigError { reason: String },
}

impl Display for TlsError { ... }
impl Error   for TlsError { ... }

// TLS alert codes (RFC 8446 §6.2)
enum TlsAlert {
    CloseNotify,
    UnexpectedMessage,
    BadRecordMac,
    RecordOverflow,
    HandshakeFailure,
    BadCertificate,
    UnsupportedCertificate,
    CertificateRevoked,
    CertificateExpired,
    CertificateUnknown,
    IllegalParameter,
    UnknownCa,
    AccessDenied,
    DecodeError,
    DecryptError,
    ProtocolVersion,
    InsufficientSecurity,
    InternalError,
    InappropriateFallback,
    UserCanceled,
    NoRenegotiation,
    MissingExtension,
    UnsupportedExtension,
    CertificateUnobtainable,
    UnrecognizedName,
    BadCertificateStatusResponse,
    UnknownPskIdentity,
    CertificateRequired,
    NoCertificate,
    Other(u8),
}

impl Display for TlsAlert { ... }
```

---

## 13. Example Usage

### 13.1 TLS Client — HTTPS Request

```ferrum
use extlib.tls.{TlsClientConfigBuilder, TlsConnector, CertStore}
use std.net.TcpStream
use std.async.{AsyncReadExt, AsyncWriteExt}

fn https_get(host: &str, path: &str): Result[Vec[u8], TlsError] ! Async + Net + IO {
    // Build config — .server_name() is required; the builder enforces this
    // at the type level, so forgetting it is a compile error.
    let config = TlsClientConfigBuilder::new()
        .server_name(host)?
        .ca_certs(CertStore::system()?)
        .alpn_protocols(vec!["h2", "http/1.1"])
        .build()?

    let connector = TlsConnector::new(config)

    // Connect TCP, then upgrade to TLS
    let tcp = TcpStream::connect_host(host, 443).await?
    let mut tls = connector.connect(tcp, host).await?

    // Inspect handshake results
    let info = tls.handshake_data()
    log.debug(
        "tls: version={} cipher={} alpn={:?} resumed={}",
        info.protocol_version,
        info.cipher_suite,
        info.alpn_negotiated,
        info.session_resumed,
    )

    // Send HTTP request
    let request = format!(
        "GET {path} HTTP/1.1\r\nHost: {host}\r\nConnection: close\r\n\r\n"
    )
    tls.write_all(request.as_bytes()).await?

    // Read response
    let mut body = Vec::new()
    tls.read_to_end(&mut body).await?

    tls.close().await?
    Ok(body)
}
```

### 13.2 TLS Server — Mutual Authentication

```ferrum
use extlib.tls.{TlsServerConfigBuilder, TlsAcceptor, ClientAuth, CertStore}
use extlib.cert.{Certificate, PrivateKey}
use std.net.{TcpListener, TcpStream}
use std.async.{AsyncReadExt, AsyncWriteExt}
use std.alloc.Arc

fn run_mtls_server(): Result[(), TlsError] ! Async + Net + IO {
    let cert_pem = std.fs::read_to_string("server.crt")?
    let key_pem  = std.fs::read_to_string("server.key")?
    let ca_pem   = std.fs::read_to_string("client-ca.crt")?

    let leaf = Certificate::from_pem(&cert_pem)?
    let key  = PrivateKey::from_pem(&key_pem)?
    let ca   = CertStore::from_pem(&ca_pem)?

    let server_config = TlsServerConfigBuilder::new()
        .cert_chain(vec![leaf])
        .private_key(key)
        .client_auth(ClientAuth::Required { ca_certs: ca })
        .alpn_protocols(vec!["my-protocol/1"])
        .build()?

    let acceptor = TlsAcceptor::new(server_config)
    let listener = TcpListener::bind("0.0.0.0:8443").await?

    loop {
        let (tcp, peer_addr) = listener.accept().await?
        let acceptor = acceptor.clone()

        runtime.spawn({
            match acceptor.accept(tcp).await {
                Err(e) => {
                    log.warn("tls: handshake from {peer_addr} failed: {e}")
                }
                Ok(mut tls) => {
                    let info = tls.handshake_data()
                    let client_cert = info.peer_cert_chain.as_ref().unwrap()
                    log.info(
                        "tls: client={} cipher={} subject={}",
                        peer_addr,
                        info.cipher_suite,
                        client_cert[0].subject(),
                    )
                    handle_connection(tls).await.ok()
                }
            }
        })
    }
}
```

### 13.3 SNI-Based Virtual Hosting

```ferrum
use extlib.tls.{TlsServerConfigBuilder, TlsAcceptor, CertStore}
use extlib.cert.{Certificate, PrivateKey}
use std.alloc.Arc
use std.collections.HashMap

fn run_vhost_server(): Result[(), TlsError] ! Async + Net + IO {
    // Build per-vhost configs
    let mut vhosts: HashMap[String, TlsServerConfig] = HashMap::new()

    for (hostname, cert_file, key_file) in [
        ("api.example.com",   "api.crt",   "api.key"),
        ("www.example.com",   "www.crt",   "www.key"),
        ("admin.example.com", "admin.crt", "admin.key"),
    ] {
        let cert = Certificate::from_pem(&std.fs::read_to_string(cert_file)?)?
        let key  = PrivateKey::from_pem(&std.fs::read_to_string(key_file)?)?
        let cfg  = TlsServerConfigBuilder::new()
            .cert_chain(vec![cert])
            .private_key(key)
            .build()?
        vhosts.insert(hostname.to_string(), cfg)
    }

    // Default config for unrecognized SNI
    let default_cert = Certificate::from_pem(&std.fs::read_to_string("default.crt")?)?
    let default_key  = PrivateKey::from_pem(&std.fs::read_to_string("default.key")?)?
    let default_cfg  = TlsServerConfigBuilder::new()
        .cert_chain(vec![default_cert])
        .private_key(default_key)
        .build()?

    let vhosts = Arc::new(vhosts)

    let server_config = TlsServerConfigBuilder::new()
        .cert_chain(vec![default_cert.clone()])
        .private_key(default_key.clone())
        .sni_resolver({
            let vhosts = Arc::clone(&vhosts)
            move |hostname| { vhosts.get(hostname).cloned() }
        })
        .build()?

    let acceptor = TlsAcceptor::new(server_config)
    let listener = TcpListener::bind("0.0.0.0:443").await?

    loop {
        let (tcp, _) = listener.accept().await?
        let acceptor = acceptor.clone()
        runtime.spawn({
            if let Ok(tls) = acceptor.accept(tcp).await {
                let sni = tls.handshake_data().sni_hostname.as_deref().unwrap_or("")
                log.debug("serving vhost: {sni}")
                handle_connection(tls).await.ok()
            }
        })
    }
}
```

### 13.4 Post-Quantum Client

```ferrum
// Requires: feature = "post-quantum" in Cargo.ferrum
use extlib.tls.{TlsClientConfigBuilder, TlsConnector, CertStore}
use std.net.TcpStream

fn pq_connect(host: &str): Result[TlsStream, TlsError] ! Async + Net + IO {
    let config = TlsClientConfigBuilder::new()
        .server_name(host)?
        .ca_certs(CertStore::system()?)
        .post_quantum(true)      // offer X25519Kyber768; fall back to X25519
        .min_version(TlsVersion::Tls13)  // PQ benefit is TLS 1.3 only
        .build()?

    let connector = TlsConnector::new(config)
    let tcp = TcpStream::connect_host(host, 443).await?
    let tls = connector.connect(tcp, host).await?

    let info = tls.handshake_data()
    if info.post_quantum_used {
        log.info("pq: X25519Kyber768 key exchange negotiated with {host}")
    } else {
        log.info("pq: server does not support hybrid; using X25519 fallback")
    }

    Ok(tls)
}
```

---

## 14. Dependencies Reference

### Extended Library Dependencies

| Module | Role |
|---|---|
| `extlib.cert` | Certificate parsing (DER, PEM), `Certificate` and `PrivateKey` types, OCSP response parsing |
| `extlib.pki` | X.509 chain validation, revocation checking, `CertStore` trust anchor management |

### Standard Library Dependencies

| Module | Role |
|---|---|
| `std.crypto` | AEAD ciphers (AES-GCM, ChaCha20-Poly1305), SHA-256/384, HKDF, X25519, P-256, P-384, Ed25519, system RNG, constant-time comparison |
| `std.net` | `TcpStream` — the underlying byte transport for TLS records |
| `std.async` | `Future`, `AsyncRead`, `AsyncWrite` traits; async runtime for handshake suspension |
| `std.io` | `IoError`, `ReadResult` |
| `std.alloc` | `Vec`, `Arc`, `Box` |

### Effect Requirements

All public async functions carry `! Async + Net`. The TLS handshake requires network I/O (exchanging ClientHello, ServerHello, Certificate, Finished messages) and suspends during round trips. `CertStore::system()` carries `! IO` to access the platform certificate store.

Synchronous certificate parsing (`Certificate::from_pem`, `CertStore::from_pem`) carries no effects — parsing is pure computation over in-memory bytes.

### Feature Flags

| Feature | Effect when enabled |
|---|---|
| `post-quantum` | Compiles in ML-KEM-768 support; enables `TlsClientConfigBuilder::post_quantum()` and server-side negotiation of X25519Kyber768 |

---

*End of extlib.tls design document.*

*See also:*
- *[ferrum_extlib_dtls.md](ferrum_extlib_dtls.md) — DTLS (TLS over UDP); depends on extlib.tls*
- *[ferrum_extlib_connect.md](ferrum_extlib_connect.md) — High-level connection establishment with happy eyeballs, DANE, and HTTPS/SVCB; uses extlib.tls for TLS*
- *[ferrum_extlib_dns_secure.md](ferrum_extlib_dns_secure.md) — DNSSEC-validating resolver with DoT/DoH; uses extlib.tls for encrypted DNS transport*
- *[ferrum-stdlib-async-net.md](ferrum-stdlib-async-net.md) — TcpStream, AsyncRead, AsyncWrite, net primitives*
- *[ferrum-stdlib-crypto-testing.md](ferrum-stdlib-crypto-testing.md) — AEAD, HKDF, SystemRng primitives used by the TLS implementation*
