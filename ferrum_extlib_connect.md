# Ferrum Extended Library — `connect` Module

**Module path:** `extlib::connect`
**Depends on:** `extlib::dns_secure`, `extlib::tls`, `stdlib::net`, `stdlib::async`

---

## 1. Overview and Rationale

The Ferrum standard library provides `net.TcpStream::connect(addr: SocketAddr)` and the
higher-level `TcpStream::connect_happy(host, port)`. These cover the common case, but
production-grade connection code requires a richer set of behaviors that bring in
dependencies outside the standard library:

| Problem | Root cause | `connect` module solution |
|---------|-----------|--------------------------|
| `SO_ERROR` miss | Nonblocking connect: writable event fires even when the OS-level connect failed | Always read and check `getsockopt(SO_ERROR)` after write-readiness |
| No happy eyeballs | Only one address family attempted | RFC 8305: parallel A + AAAA, IPv6-first, 250 ms IPv4 stagger |
| No SRV following | Application must manually resolve `_service._tcp.example.com` | `.srv_lookup(true)` resolves, priority/weight-selects, then connects |
| No HTTPS/SVCB integration | ALPN and Alt-Svc information requires an extra round trip | HTTPS record queried in parallel with A/AAAA; ALPN negotiated immediately |
| No DANE enforcement | TLS certificate validation uses only PKIX trust anchors | TLSA records retrieved over DNSSEC; DANE-EE (usage 3) bypasses PKIX |
| No QUIC surface | Raw TCP only | `ConnectTransport::Quic` with transparent TCP fallback |

### Why this is an extended library, not stdlib

The `connect` module depends on:

- `extlib::dns_secure` — DNSSEC-validating resolver required for DANE (TLSA records are
  meaningless without a validated chain)
- `extlib::tls` — TLS layer (`TlsConfig`, `TlsStream`, ALPN negotiation)

These carry significant code size and policy decisions (DNSSEC trust anchors, TLS cipher
suites) that do not belong in the standard library. The standard library provides the
raw primitives; this module composes them correctly.

---

## 2. `ConnectConfig` Builder

`ConnectConfig` is the single configuration point. Every option has a documented
default so callers only set what they care about.

```ferrum
use extlib::connect::{ConnectConfig, ConnectTransport, ConnectError, Connection}
use extlib::tls::TlsConfig
use stdlib::net::SocketAddr
use stdlib::time::Duration

type ConnectConfig
```

### Constructor

```ferrum
impl ConnectConfig {
    // Start a builder targeting a hostname and port.
    // Happy eyeballs, SO_ERROR checking, SRV lookup, HTTPS/SVCB, and DANE
    // are all enabled by default.  Override only what you need.
    fn new(): Self
```

### Target specification (mutually exclusive groups)

```ferrum
    // Connect to a hostname — DNS resolution happens at connect time.
    // This is the normal entry point; enables SRV and HTTPS/SVCB lookups.
    fn host(mut self, host: &str): Self

    // Port to connect to.  Required when using .host() unless SRV lookup
    // will supply it.  Ignored when .addr() is used.
    fn port(mut self, port: u16): Self

    // Connect directly to a known SocketAddr, bypassing DNS entirely.
    // Disables SRV lookup, HTTPS/SVCB lookup, and DANE (no hostname
    // to derive the TLSA owner name from).
    fn addr(mut self, addr: SocketAddr): Self
```

### Transport

```ferrum
    // Select the transport layer.  Default: ConnectTransport::Tcp.
    fn transport(mut self, t: ConnectTransport): Self
```

```ferrum
enum ConnectTransport {
    // Standard TCP.  Always available.
    Tcp,

    // QUIC (RFC 9000).  Requires an available QUIC implementation in the
    // process (loaded via extlib::quic or a platform-provided capability).
    // If no QUIC implementation is present at connect time the module falls
    // back to TCP + TLS and emits a warn!() log line.  The returned
    // Connection still satisfies AsyncRead + AsyncWrite; callers inspect
    // .transport_info() if they need to know which was used.
    Quic,
}
```

### TLS

```ferrum
    // Attach a TLS layer on top of the transport.
    // When omitted the connection is plaintext.
    // DANE enforcement requires TLS; setting .dane(true) without .tls(...)
    // is a configuration error caught at connect time.
    fn tls(mut self, config: TlsConfig): Self
```

### Timeout

```ferrum
    // Wall-clock deadline for the entire connect sequence including DNS,
    // happy eyeballs racing, and TLS handshake.  Default: 30 seconds.
    fn timeout(mut self, d: Duration): Self
```

### Happy eyeballs (RFC 8305)

```ferrum
    // Enable RFC 8305 happy eyeballs.  Default: true.
    //
    // When true: A and AAAA queries are sent in parallel.  The first IPv6
    // address is attempted immediately.  If no IPv6 connection is established
    // within 250 ms, IPv4 attempts begin.  The first TCP handshake to
    // complete wins; the losing socket is closed.
    //
    // Set false only when targeting a known-single-family address space
    // (e.g., a container network where IPv6 is absent) or for testing.
    fn happy_eyeballs(mut self, enabled: bool): Self
```

### DANE

```ferrum
    // Enable DANE enforcement via TLSA records.  Default: true.
    //
    // Requires: .tls(...) configured, .host(...) set (TLSA owner name is
    // derived from the hostname), and extlib::dns_secure providing a
    // DNSSEC-validating resolver.
    //
    // When a DNSSEC-validated TLSA record with usage DANE-EE (3) is present,
    // PKIX certificate path validation is bypassed; the TLSA record is the
    // sole trust anchor.  All other TLSA usages layer on top of PKIX.
    //
    // If the TLSA lookup returns NXDOMAIN or SERVFAIL the behavior depends
    // on the DanePolicy:
    //   - DanePolicy::Opportunistic: proceed without DANE (log at info)
    //   - DanePolicy::Required: return ConnectError::DaneValidationFailed
    // Default policy: DanePolicy::Opportunistic.
    fn dane(mut self, enabled: bool): Self
    fn dane_policy(mut self, policy: DanePolicy): Self
```

```ferrum
enum DanePolicy {
    // DANE is attempted; if TLSA records are unavailable the connection
    // proceeds using ordinary PKIX.
    Opportunistic,

    // TLSA records must be present and DNSSEC-validated.  If they are absent
    // or the chain is broken the connect returns an error.
    Required,
}
```

### SRV record following

```ferrum
    // Follow SRV records before connecting.  Default: true when .host() is set.
    //
    // Constructs the owner name as `_<service>._<proto>.<host>` using the
    // service and protocol labels set via .srv_service() and .srv_proto().
    // Selects a target using RFC 2782 priority/weight algorithm.
    // The selected SRV target hostname and port replace the configured
    // host/port for the actual connection attempt.
    //
    // If the SRV lookup returns NXDOMAIN the module falls back to a direct
    // connection to the original host/port.  Any other DNS error returns
    // ConnectError::DnsResolutionFailed.
    fn srv_lookup(mut self, enabled: bool): Self

    // The service label for SRV construction, e.g. "https", "imap", "xmpp".
    // Required when srv_lookup is true and no default can be inferred.
    fn srv_service(mut self, service: &str): Self

    // The protocol label for SRV construction.  Default: "tcp".
    fn srv_proto(mut self, proto: &str): Self
```

### HTTPS/SVCB records

```ferrum
    // Query HTTPS (type 65) / SVCB records in parallel with A/AAAA.
    // Default: true when .host() is set and TLS is configured.
    //
    // When an HTTPS record is found:
    //   - ALPN identifiers from the record's alpn SvcParam are offered
    //     during TLS negotiation, ahead of any ALPN from TlsConfig.
    //   - Alternative target addresses from ipv4hint / ipv6hint SvcParams
    //     are merged into the happy eyeballs candidate list.
    //   - An ech SvcParam causes Encrypted ClientHello to be attempted
    //     if the TLS implementation supports it.
    //
    // HTTPS records do not replace A/AAAA results; they are additive.
    // If no HTTPS record exists the connect proceeds normally.
    fn https_svcb(mut self, enabled: bool): Self
```

---

## 3. `connect` Function

```ferrum
// Establish a connection according to config.
//
// Effects: Async (suspends during DNS, TCP handshake, TLS handshake)
//          Net   (opens sockets, sends/receives packets)
//
// The function drives the full pipeline:
//   1. SRV lookup (if enabled)
//   2. Parallel A + AAAA + HTTPS/SVCB queries
//   3. Happy eyeballs racing across resolved candidates
//   4. SO_ERROR check on every candidate socket after write-readiness
//   5. TLS handshake (if TlsConfig provided)
//   6. DANE validation against TLSA records (if enabled)
//
// On success returns a Connection wrapping whichever transport won.
// On failure returns a ConnectError describing which step failed.
pub fn connect(config: ConnectConfig): Result[Connection, ConnectError] ! Async + Net
```

---

## 4. `Connection` Type

```ferrum
// An established, optionally-TLS-wrapped, optionally-QUIC connection.
// Implements AsyncRead + AsyncWrite regardless of the underlying transport.
pub type Connection
```

### Trait implementations

```ferrum
impl AsyncRead  for Connection
impl AsyncWrite for Connection
```

### Methods

```ferrum
impl Connection {
    // The remote address of the peer as seen by the OS.
    pub fn peer_addr(&self): SocketAddr

    // The local address the OS bound for this connection.
    pub fn local_addr(&self): SocketAddr

    // Metadata about the connection that was actually established.
    pub fn transport_info(&self): &TransportInfo

    // The DANE validation outcome, if DANE was attempted.
    // None when DANE was disabled or when .addr() was used (no hostname).
    pub fn dane_result(&self): Option[&DaneValidationResult]

    // Graceful shutdown: flushes and closes the write side.
    pub fn shutdown(&mut self): Result[(), ConnectError] ! Async + Net
}
```

### `TransportInfo`

```ferrum
pub type TransportInfo {
    // The transport that was actually used.
    pub transport:     TransportKind,

    // True when TLS was active on this connection.
    pub tls_active:    bool,

    // The TLS protocol version negotiated, e.g. TlsVersion::Tls13.
    // None when tls_active is false.
    pub tls_version:   Option[TlsVersion],

    // The ALPN protocol identifier negotiated during TLS handshake.
    // None when TLS was not used or ALPN was not offered/accepted.
    pub alpn:          Option[String],

    // The remote address family that won the happy eyeballs race.
    pub addr_family:   AddrFamily,

    // The peer address that was actually connected (may differ from the
    // hostname's A/AAAA result when SRV or HTTPS/SVCB redirected).
    pub connected_to:  SocketAddr,
}

pub enum TransportKind {
    Tcp,
    Quic,
    // TcpFallback is set when Quic was requested but not available.
    TcpFallback { reason: String },
}

pub enum AddrFamily { V4, V6 }
```

### `DaneValidationResult`

```ferrum
pub enum DaneValidationResult {
    // At least one TLSA record was present, DNSSEC-validated, and matched
    // the presented certificate or public key.
    Validated {
        // The TLSA record that matched.
        record: TlsaRecord,
        // When true, PKIX path validation was bypassed (DANE-EE usage 3).
        pkix_bypassed: bool,
    },

    // DANE was attempted but no TLSA records were found (NXDOMAIN).
    // Connection proceeded via PKIX under Opportunistic policy.
    NoRecords,

    // The TLSA lookup completed but the DNSSEC chain was broken or absent.
    // Treated as NoRecords under Opportunistic policy; fatal under Required.
    DnssecUnavailable { reason: String },
}
```

---

## 5. SO_ERROR Check

A nonblocking TCP connect reports readiness by making the socket writable. On most
operating systems a writable event can fire even when the underlying connect syscall
failed (e.g., `ECONNREFUSED`, `ENETUNREACH`). The error is deposited in the socket's
`SO_ERROR` option and is not visible until `getsockopt(SO_ERROR)` is called.

Code that tests only for writability without checking `SO_ERROR` silently proceeds on
a failed socket, then observes the failure only on the first subsequent read or write
— at which point the error appears to come from data transfer rather than connection
establishment, making diagnosis harder.

The `connect` module **always** calls `getsockopt(SO_ERROR)` immediately after the
write-readiness event fires for every candidate socket. A non-zero value is treated as
a connection failure for that candidate.

The error type for this condition is:

```ferrum
ConnectError::ConnectionFailed {
    addr:     SocketAddr,  // which candidate failed
    os_error: i32,         // raw errno value (ECONNREFUSED, ENETUNREACH, etc.)
}
```

When all candidates return `ConnectionFailed` the function returns
`ConnectError::AllAttemptsFailed` wrapping the per-address errors.

---

## 6. Happy Eyeballs — RFC 8305 Detail

Happy eyeballs (RFC 8305) addresses the problem that IPv6 connectivity is sometimes
broken silently: a host resolves both AAAA and A records, IPv6 is preferred, but the
IPv6 path is a black hole. A naive IPv6-first sequential strategy stalls until the
IPv6 connection times out before falling back to IPv4.

RFC 8305 requires racing the address families with a short stagger so the faster path
wins without waiting for the slower one to time out.

### Algorithm implemented by this module

```
1. Resolve A and AAAA records in parallel (two concurrent DNS queries).
   If HTTPS/SVCB records are enabled, that query runs in the same parallel group.

2. Build a candidate list.
   Ordering: AAAA addresses first, then A addresses, interleaved per RFC 8305 §4
   ("Sorting of Addresses").  ipv6hint / ipv4hint addresses from HTTPS records
   are merged into the appropriate family slot.

3. Attempt the first candidate (IPv6 if available).
   Set a 250 ms stagger timer.

4. If no connection is established before the stagger timer fires,
   begin the next candidate attempt in parallel.  Continue staggering
   at 250 ms intervals across remaining candidates.

5. The first candidate whose TCP handshake completes AND whose SO_ERROR check
   passes wins.  All other in-flight sockets are immediately closed.

6. Continue with TLS handshake and DANE validation on the winning socket.
```

The 250 ms stagger value is the RFC 8305 recommendation. It is not currently
configurable; if use cases arise for tuning it a `ConnectConfig::eyeballs_stagger`
field will be added.

### Scope structure

The racing is implemented with structured concurrency so no in-flight socket can
outlive the `connect` call:

```ferrum
// Conceptual implementation sketch — not the public API
fn connect_happy_eyeballs(
    candidates: Vec[SocketAddr],
    timeout:    Duration,
) : Result[RawSocket, ConnectError] ! Async + Net {
    scope s {
        let (winner_tx, winner_rx) = async_channel(1)

        for (i, addr) in candidates.iter().enumerate() {
            // Stagger: first attempt starts immediately, each subsequent
            // attempt waits 250 ms * i before starting.
            let stagger = Duration::from_millis(250 * i as u64)
            s.spawn(async {
                if stagger > Duration::ZERO {
                    sleep(stagger).await
                }
                match attempt_connect(addr).await {
                    Ok(sock) => { winner_tx.send(Ok((addr, sock))).await.ok() }
                    Err(e)   => { winner_tx.send(Err(e)).await.ok() }
                }
            })
        }

        // First success received wins; scope cancels remaining spawns on exit
        select {
            winner_rx -> result => result,
            timeout(timeout)   => Err(ConnectError::Timeout),
        }
    }
}
```

---

## 7. SRV Record Following

SRV records (RFC 2782) allow a service to advertise its actual hostname and port under
a well-known DNS name, enabling transparent relocation and load distribution.

When `srv_lookup(true)` is set (the default when a hostname is configured), the module
constructs the SRV owner name as:

```
_<service>._<proto>.<host>
```

For example, with `.host("example.com").srv_service("xmpp-client").srv_proto("tcp")`:

```
_xmpp-client._tcp.example.com
```

### Priority and weight selection (RFC 2782 §3)

```
1. Group records by priority.  Lower priority value = higher preference.

2. Within the lowest-priority group, perform weighted random selection:
   - Sum the weights of all records in the group.
   - Select a random number r in [0, sum).
   - Walk the records in order, subtracting each weight; select the first
     record where the running total >= r.

3. After connecting to the selected target, if it fails, remove it and
   repeat from step 2 within the same priority group.  Exhaust the group
   before moving to the next priority level.
```

The SRV target hostname and port become the effective `.host()` and `.port()` for
all subsequent steps (happy eyeballs, HTTPS/SVCB, DANE). The original hostname
is retained as the TLS SNI value and the TLSA owner name base.

If the SRV lookup returns NXDOMAIN or the record set is empty, the module silently
falls back to a direct connection to the original host/port. Any other DNS error
(SERVFAIL, timeout) returns `ConnectError::DnsResolutionFailed`.

---

## 8. HTTPS/SVCB Records

HTTPS records (RFC 9460, RR type 65) carry service parameters that allow a client to
negotiate HTTP/2 or HTTP/3 and reach alternative endpoints **without** an extra round
trip.

The module queries the HTTPS record for the target hostname in parallel with A and AAAA
queries. When a record is found:

### ALPN (`alpn` SvcParam)

The ALPN identifiers from the HTTPS record are prepended to any ALPN identifiers in the
`TlsConfig`. This allows the server to indicate `h3` support before the client has ever
spoken to it. The TLS handshake will therefore negotiate `h3` on the first connection
rather than requiring an `Alt-Svc` response followed by a reconnect.

```ferrum
// Effective ALPN list when TlsConfig specifies ["http/1.1"] and the
// HTTPS record advertises ["h3", "h2"]:
//   ["h3", "h2", "http/1.1"]
// The server negotiates h3 if it supports it; h2 next; http/1.1 last.
```

### Address hints (`ipv4hint`, `ipv6hint` SvcParams)

IP address hints from the HTTPS record are merged into the happy eyeballs candidate
list. They do not replace A/AAAA results; they supplement them. This allows the server
to pre-announce CDN edge addresses and avoid an A/AAAA lookup latency penalty on
subsequent connections.

### Encrypted ClientHello (`ech` SvcParam)

When an `ech` SvcParam is present and the configured TLS implementation supports ECH
(RFC 8744), the module attempts ECH on the first handshake. If ECH fails the module
retries without ECH (ECH retry logic per the RFC). The outcome is recorded in
`TransportInfo`.

### Port override

When the HTTPS record carries a `port` SvcParam it overrides the configured port for
the HTTPS endpoint. This is used by services that run HTTPS on a non-standard port.

---

## 9. QUIC-Aware API

`ConnectTransport::Quic` requests a QUIC (RFC 9000) connection instead of TCP. QUIC
provides 0-RTT connection establishment on repeat visits, built-in stream multiplexing,
and connection migration.

The module checks at connect time whether a QUIC implementation is available via the
process-local capability registry (populated by `extlib::quic` or a platform QUIC
library). If none is present:

1. The module logs a `warn!()` line identifying the hostname and the reason for fallback.
2. The connection proceeds as `ConnectTransport::Tcp` with TLS.
3. `TransportInfo::transport` is set to `TransportKind::TcpFallback { reason }`.

The returned `Connection` satisfies `AsyncRead + AsyncWrite` in both cases. Code that
does not inspect `transport_info()` is unaffected by whether QUIC was actually used.

QUIC provides its own TLS 1.3 handshake internally; the `TlsConfig` from
`ConnectConfig::tls()` is passed to the QUIC handshake rather than wrapping the
transport after the fact.

---

## 10. DANE Enforcement

DANE (RFC 6698, RFC 7671) allows a DNS operator to publish the certificate or public
key for a service in TLSA records protected by DNSSEC. This removes dependence on the
CA ecosystem for services whose DNS zone is DNSSEC-signed.

### Preconditions

- `.host()` must be set (the TLSA owner name is derived from the hostname).
- `.tls()` must be configured (DANE validates TLS certificates).
- `extlib::dns_secure` must be available (DNSSEC-validating resolver).

### TLSA owner name

The TLSA owner name is constructed from the hostname the TLS connection is made to
(after any SRV redirect, before HTTPS/SVCB port override):

```
_<port>._<proto>.<hostname>
```

For a connection to `mail.example.com` on TCP port 993:

```
_993._tcp.mail.example.com
```

### Validation logic

```
1. Query TLSA records for the owner name via extlib::dns_secure.
   The resolver must return the DNSSEC validation status alongside the records.

2. If the query returns NXDOMAIN:
   - Opportunistic policy: proceed with PKIX only.  Set DaneValidationResult::NoRecords.
   - Required policy: return ConnectError::DaneValidationFailed { reason: "no TLSA records" }.

3. If the DNSSEC chain is broken (BOGUS):
   - Opportunistic policy: treat as NoRecords.  Set DaneValidationResult::DnssecUnavailable.
   - Required policy: return ConnectError::DaneValidationFailed { reason: "DNSSEC chain broken" }.

4. For each TLSA record (DNSSEC-validated):
   a. Match the record against the server's presented certificate chain
      per RFC 6698 §2 (usage, selector, matching type).
   b. If usage == DomainIssued (3, DANE-EE):
      - A match here bypasses PKIX path validation entirely.
        The TLSA record is the sole trust anchor.
   c. All other usages (0–2) layer on top of PKIX; PKIX must also pass.

5. If no record matches: return ConnectError::DaneValidationFailed.

6. On success: set DaneValidationResult::Validated { record, pkix_bypassed }.
```

The TLSA lookup runs in parallel with the TLS handshake where possible. The
server's certificate chain is not available until the handshake completes, so
the actual matching step occurs after both the handshake and the TLSA query have
resolved.

---

## 11. Error Types

```ferrum
pub enum ConnectError {
    // The overall deadline was exceeded before a connection was established.
    Timeout,

    // DNS resolution failed for the target hostname or the SRV/TLSA owner name.
    DnsResolutionFailed {
        name:   String,
        reason: String,
    },

    // A specific candidate address was tried and the TCP connect failed.
    // The OS-level error is included for diagnostics.
    // This variant is wrapped inside AllAttemptsFailed when all candidates fail.
    ConnectionFailed {
        addr:     SocketAddr,
        os_error: i32,
    },

    // TLS handshake failed (certificate rejected, protocol mismatch, etc.).
    TlsError {
        addr:   SocketAddr,
        reason: String,
    },

    // DANE validation failed.  Returned under DanePolicy::Required or when
    // a TLSA record was found but the server certificate did not match.
    DaneValidationFailed {
        reason: String,
    },

    // DNS resolved successfully but the result set was empty (e.g., a zone
    // that exists but has no A, AAAA, or SRV records).
    NoAddressesFound {
        name: String,
    },

    // Every candidate address was tried and all failed.
    // The per-address errors are collected here for diagnostic logging.
    AllAttemptsFailed {
        errors: Vec[(SocketAddr, ConnectError)],
    },

    // ConnectConfig was logically invalid (e.g., dane(true) without tls(...)).
    InvalidConfig {
        reason: String,
    },
}

impl Display for ConnectError { ... }
impl Error   for ConnectError { ... }
```

---

## 12. Example Usage

### Connect to an HTTPS server with happy eyeballs and DANE

```ferrum
use extlib::connect::{ConnectConfig, ConnectTransport}
use extlib::tls::TlsConfig
use stdlib::time::Duration

async fn fetch_page(host: &str): Result[Vec[u8], ConnectError] ! Async + Net {
    let tls = TlsConfig::default_https()

    let config = ConnectConfig::new()
        .host(host)
        .port(443)
        .tls(tls)
        .transport(ConnectTransport::Quic)   // try QUIC, fall back to TCP+TLS
        .happy_eyeballs(true)                // default; shown for clarity
        .dane(true)                          // default; shown for clarity
        .https_svcb(true)                    // default; shown for clarity
        .timeout(Duration::from_secs(10))

    let mut conn = connect(config).await?

    let info = conn.transport_info()
    println("connected via {:?}, ALPN={:?}", info.transport, info.alpn)

    if let Some(dane) = conn.dane_result() {
        match dane {
            DaneValidationResult::Validated { pkix_bypassed, .. } =>
                println("DANE validated (PKIX bypassed: {})", pkix_bypassed),
            DaneValidationResult::NoRecords =>
                println("no TLSA records; PKIX only"),
            DaneValidationResult::DnssecUnavailable { reason } =>
                println("DNSSEC unavailable: {}", reason),
        }
    }

    let request = b"GET / HTTP/1.1\r\nHost: {host}\r\nConnection: close\r\n\r\n"
    conn.write_all(request).await?

    let mut body = Vec::new()
    conn.read_to_end(&mut body).await?
    Ok(body)
}
```

### Connect via SRV record (XMPP client example)

```ferrum
use extlib::connect::{ConnectConfig, DanePolicy}
use extlib::tls::TlsConfig

async fn xmpp_connect(domain: &str): Result[Connection, ConnectError] ! Async + Net {
    let tls = TlsConfig::builder()
        .alpn(&["xmpp-client"])
        .build()?

    let config = ConnectConfig::new()
        .host(domain)
        .srv_lookup(true)
        .srv_service("xmpp-client")
        .srv_proto("tcp")
        // SRV lookup constructs: _xmpp-client._tcp.<domain>
        // Selects a target by RFC 2782 priority/weight.
        // Falls back to direct connection on NXDOMAIN.
        .tls(tls)
        .dane(true)
        .dane_policy(DanePolicy::Required)   // XMPP operators commonly sign TLSA
        .timeout(Duration::from_secs(15))

    connect(config).await
}
```

### Plaintext TCP with happy eyeballs only (no TLS, no DANE)

```ferrum
async fn tcp_connect(host: &str, port: u16): Result[Connection, ConnectError] ! Async + Net {
    let config = ConnectConfig::new()
        .host(host)
        .port(port)
        .happy_eyeballs(true)
        .dane(false)
        .https_svcb(false)
        // No .tls() — plaintext connection

    connect(config).await
}
```

### Bypass DNS entirely — connect to a known address

```ferrum
use stdlib::net::SocketAddr

async fn raw_connect(addr: SocketAddr): Result[Connection, ConnectError] ! Async + Net {
    // .addr() bypasses DNS, SRV, HTTPS/SVCB, and DANE.
    // Happy eyeballs still applies if addr is an IPv6 address (single candidate,
    // so stagger logic is a no-op).
    let config = ConnectConfig::new()
        .addr(addr)
        .timeout(Duration::from_secs(5))

    connect(config).await
}
```

---

## 13. Dependencies

### Extended library dependencies

| Crate | Why required |
|-------|-------------|
| `extlib::dns_secure` | DNSSEC-validating resolver for DANE (TLSA lookup with chain validation), SRV lookup, and HTTPS/SVCB lookup.  The basic `net.Resolver` does not validate DNSSEC. |
| `extlib::tls` | `TlsConfig`, `TlsStream`, ALPN negotiation, ECH support.  TLS is an optional layer but DANE requires it. |
| `extlib::quic` | Optional.  Required only when `ConnectTransport::Quic` is used and a QUIC implementation is desired.  Absence causes transparent TCP fallback. |

### Standard library dependencies

| Module | Used for |
|--------|---------|
| `stdlib::net` | `TcpStream`, `SocketAddr`, `IpAddr`, `TlsaRecord`, raw socket primitives |
| `stdlib::async` | `Future`, `Runtime`, `scope`, `select`, async channels, `sleep`, `timeout` |
| `stdlib::time` | `Duration`, `Instant` |
| `stdlib::io` | `AsyncRead`, `AsyncWrite` traits |

### Effect requirements

All public functions in this module carry `! Async + Net`. The DNS queries, TCP
handshakes, and TLS handshake all require suspension (`Async`) and network access
(`Net`). There is no synchronous connect path; callers must provide an async runtime.

---

## 14. Design Notes and Rejected Alternatives

### Why a builder rather than a single function with many parameters?

A function like `connect(host, port, tls, timeout, happy_eyeballs, dane, ...)` would
require positional or named arguments for a dozen options, most of which have sensible
defaults. The builder pattern lets callers express intent clearly and the compiler
catches use of conflicting options (e.g., `.addr()` + `.srv_lookup(true)`) as
`InvalidConfig` at connect time rather than silently ignoring one.

### Why not make DANE default-off?

DANE enforcement is the correct behavior when DNSSEC is available. Making it
default-true with `DanePolicy::Opportunistic` means callers get DANE validation
silently when the zone is signed, and silent PKIX-only when it is not. The only
cost is a DNS query that would be made anyway in the typical HTTPS case. Callers
who have measured the latency impact and determined it is unacceptable can disable
it explicitly; the safe default is on.

### Why not expose the SO_ERROR check as a separate API?

`SO_ERROR` must be checked immediately after the writable event fires on the
connecting socket, within the same async task iteration that received the event.
Exposing it separately invites the exact bug it prevents: a caller who forgets to
call it. By inlining the check into the connection racing loop the module makes the
correct behavior automatic and the incorrect behavior impossible from the public API.

### Why is QUIC a best-effort fallback rather than an error when unavailable?

QUIC availability depends on whether the binary links `extlib::quic`, which is a
deployment decision outside the application code's control. Returning an error when
QUIC is unavailable would break deployments that legitimately do not include it.
The `warn!()` log line plus `TransportKind::TcpFallback` in `TransportInfo` gives
operators visibility without breaking connectivity.
