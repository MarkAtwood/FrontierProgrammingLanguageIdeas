# Ferrum Extended Library — CoAP

**Module:** `extlib.ccsp.coap`
**RFC basis:** RFC 7252 (CoAP), RFC 7959 (Block-Wise Transfer), RFC 7641 (Observe), RFC 7390 (Multicast)
**Roadmap status:** Post-1.0 (designed now, implemented after stdlib stabilizes)
**Dependencies:** `extlib.ccsp.dtls`, `extlib.ccsp.dns_secure`, `std.net`, `std.async`, `std.data.cbor`

---

## 1. Overview and Rationale

CoAP (Constrained Application Protocol, RFC 7252) is REST over UDP. It was designed for the same environments where C lives: microcontrollers, sensor nodes, actuators, and edge devices with kilobytes of RAM and milliwatts of power budget. Where HTTP/TCP is too heavy, CoAP provides GET, PUT, POST, DELETE semantics with a binary wire format, confirmable delivery, and optional DTLS security — all in a fraction of the overhead.

CoAP matters to Ferrum for two reasons.

**IoT and embedded.** Ferrum targets the same constrained-device space as C. A language that can write an RTOS kernel but cannot speak to a ZigBee sensor node over CoAP has a gap. The standard library covers TCP and HTTP; CoAP fills the UDP-native REST gap below that.

**Matter protocol.** Matter (formerly Project CHIP) is the dominant smart-home interoperability standard. It uses CoAP over UDP with CASE/PASE security sessions. Any Ferrum program that wants to communicate with a Matter device needs CoAP. The integration note in §11 describes how CASE/PASE sits above the DTLS layer this module provides.

### Why Extended Library, Not Stdlib

CoAP depends on DTLS for its secure transport mode (`coaps://`), which in turn depends on certificate management and a DTLS handshake engine. Those live in `extlib.ccsp.dtls`. CoAP's URI resolution may require DNSSEC-validated lookups, which live in `extlib.ccsp.dns_secure`. Pulling these dependencies into the standard library would require every Ferrum target — including no-alloc embedded targets with no TLS stack — to carry crypto and DNS machinery. The extended library layer exists precisely for high-value modules with non-trivial dependency trees.

### Roadmap

This module is designed now so that the interface contracts are stable before the first implementation. It will ship after:

- `extlib.ccsp.dtls` reaches 1.0
- `extlib.ccsp.dns_secure` reaches 1.0
- The Ferrum async runtime is stable

The types defined here are final for purposes of user code. Implementors may add methods; no existing method signature will change after 1.0.

---

## 2. URI Handling

CoAP uses two URI schemes: `coap://` for plain UDP transport and `coaps://` for DTLS-secured transport. The `CoapUri` type is strongly typed — you cannot accidentally send a `coaps://` URI to a plain UDP socket or vice versa.

```ferrum
// The scheme determines the transport; they are not interchangeable
enum CoapScheme {
    Coap,   // coap:// — plain UDP, port 5683 by default
    Coaps,  // coaps:// — DTLS, port 5684 by default
}

struct CoapUri {
    scheme:  CoapScheme,
    host:    String,
    port:    u16,
    path:    Vec[String],    // each path segment, percent-decoded
    query:   Vec[(String, String)],  // key=value pairs
}

impl CoapUri {
    // Parse "coap://host:port/path?key=val"
    fn parse(s: &str): Result[Self, CoapUriError]

    // Build programmatically
    fn new(scheme: CoapScheme, host: impl Into[String]): Self
    fn with_port(self, port: u16): Self
    fn with_path_segment(self, segment: impl Into[String]): Self
    fn with_query_param(self, key: impl Into[String], val: impl Into[String]): Self

    fn scheme(&self): CoapScheme
    fn host(&self): &str
    fn port(&self): u16
    fn path_str(&self): String        // rejoins segments with '/'
    fn to_socket_addr(&self): Result[SocketAddr, CoapError] ! Net
        // Resolves host via dns_secure if not a literal IP address.
        // Validates DNSSEC when the dns_secure extlib is configured
        // for Require mode.

    fn default_port(scheme: CoapScheme): u16
        // Coap => 5683, Coaps => 5684
}

impl Display for CoapUri { ... }
```

---

## 3. Client

### 3.1 Configuration and Transport

```ferrum
// Transport selection — coap:// and coaps:// are statically distinct
enum CoapTransport {
    Udp(SocketAddr),        // plain UDP; use for coap:// URIs only
    Dtls(DtlsConfig),       // DTLS from extlib.ccsp.dtls; use for coaps:// URIs
}

struct CoapClientConfig {
    pub transport:          CoapTransport,
    pub timeout:            Duration,          // per-request deadline; default 30s
    pub max_retransmit:     u8,                // RFC 7252 §4.8; default 4
    pub ack_timeout:        Duration,          // initial retransmit interval; default 2s
    pub ack_random_factor:  f32,               // jitter multiplier; default 1.5
    pub max_payload_size:   usize,             // for block-wise auto-splitting; default 1024
}

impl CoapClientConfig {
    fn default_udp(addr: SocketAddr): Self
    fn default_dtls(config: DtlsConfig): Self
}
```

### 3.2 Client Type

```ferrum
type CoapClient

impl CoapClient {
    fn new(config: CoapClientConfig): Result[Self, CoapError] ! Async
        // Binds the local UDP socket (or initiates DTLS handshake).
        // Does not send any CoAP traffic.

    // REST methods — each builds a Confirmable message, retransmits per §6,
    // and returns the first valid Acknowledgement or RST

    fn get(
        &self,
        uri: &CoapUri,
    ): Result[CoapResponse, CoapError] ! Async + Net

    fn put(
        &self,
        uri: &CoapUri,
        payload: &[u8],
        content_format: ContentFormat,
    ): Result[CoapResponse, CoapError] ! Async + Net

    fn post(
        &self,
        uri: &CoapUri,
        payload: &[u8],
        content_format: ContentFormat,
    ): Result[CoapResponse, CoapError] ! Async + Net

    fn delete(
        &self,
        uri: &CoapUri,
    ): Result[CoapResponse, CoapError] ! Async + Net

    // Non-Confirmable variants — fire-and-forget, no retransmission
    fn get_non(
        &self,
        uri: &CoapUri,
    ): Result[CoapResponse, CoapError] ! Async + Net

    fn put_non(
        &self,
        uri: &CoapUri,
        payload: &[u8],
        content_format: ContentFormat,
    ): Result[CoapResponse, CoapError] ! Async + Net

    // Observe (RFC 7641) — returns an async stream of notifications
    fn observe(
        &self,
        uri: &CoapUri,
    ): Result[CoapObserver, CoapError] ! Async + Net

    // Multicast — see §9
    fn multicast(
        &self,
        group: IpAddr,
        port: u16,
        req: CoapRequest,
        collect_for: Duration,
    ): Result[Vec[CoapResponse], CoapError] ! Async + Net
}

struct CoapResponse {
    pub code:    CoapCode,
    pub token:   Vec[u8],
    pub options: Vec[CoapOption],
    pub payload: Vec[u8],       // reassembled if block-wise
}

impl CoapResponse {
    fn is_success(&self): bool
        // true for 2.xx codes
    fn content_format(&self): Option[ContentFormat]
    fn etag(&self): Option[&[u8]>]
}
```

---

## 4. Server

```ferrum
struct CoapServerConfig {
    pub transport:        CoapTransport,
    pub max_payload_size: usize,   // largest payload accepted in a single message; default 1024
    pub max_block_size:   usize,   // maximum reassembled body via block-wise; default 64 KiB
    pub timeout:          Duration,
}

type CoapServer

impl CoapServer {
    fn bind(
        addr: SocketAddr,
        config: CoapServerConfig,
    ): Result[Self, CoapError] ! Async + Net
        // Binds the UDP socket. Does not start the event loop.

    // Register a handler for an exact path prefix.
    // Later registrations do not override earlier ones; the first match wins.
    fn route(
        &mut self,
        path: &str,
        handler: impl CoapHandler,
    ): &mut Self

    // Register a resource that supports Observe (RFC 7641).
    // The server tracks subscribers and calls notify() on the resource
    // when the application calls CoapServer.notify_resource().
    fn observable_route(
        &mut self,
        path: &str,
        resource: impl ObservableResource,
    ): &mut Self

    // Start the event loop. Runs until error or cancellation.
    // Never returns Ok(()); only returns Err on a fatal socket error.
    fn run(&mut self): Result[(), CoapError] ! Async + Net

    // Push a notification to all registered observers of path.
    // Intended to be called from application code while run() is executing
    // on a separate task.
    fn notify_resource(&self, path: &str): Result[(), CoapError] ! Async + Net
}

// Handler trait — implemented by anything that responds to CoAP requests
trait CoapHandler: Send + Sync + 'static {
    fn handle(
        &self,
        req: CoapRequest,
    ): impl Future[Output=CoapResponse] ! Async
}

// Blanket impl — closures work as handlers
impl[F, Fut] CoapHandler for F
where
    F: Fn(CoapRequest): Fut + Send + Sync + 'static,
    Fut: Future[Output=CoapResponse] + Send,
{ ... }

struct CoapRequest {
    pub code:           CoapCode,
    pub token:          Vec[u8],
    pub options:        Vec[CoapOption],
    pub payload:        Vec[u8],        // reassembled if block-wise
    pub peer:           SocketAddr,
}

impl CoapRequest {
    fn uri_path(&self): String
        // Reconstructs the path from Uri-Path options
    fn uri_query_pairs(&self): impl Iterator[Item=(&str, &str)]
    fn content_format(&self): Option[ContentFormat]
    fn accept(&self): Option[ContentFormat]
    fn is_confirmable(&self): bool
}
```

---

## 5. Messages

```ferrum
struct CoapMessage {
    pub type_:      MessageType,
    pub code:       CoapCode,
    pub message_id: u16,
    pub token:      Vec[u8],    // 0–8 bytes per RFC 7252 §3
    pub options:    Vec[CoapOption],
    pub payload:    Vec[u8],
}

impl CoapMessage {
    fn encode(&self): Result[Vec[u8], CoapError]
        // Serializes to wire format (4-byte fixed header + options + payload)
    fn decode(bytes: &[u8]): Result[Self, CoapError]
}

enum MessageType {
    Confirmable,      // CON — expects ACK or RST
    NonConfirmable,   // NON — no acknowledgement expected
    Acknowledgement,  // ACK — reply to a CON
    Reset,            // RST — unprocessable CON or NON
}

// CoAP codes: one byte, split 3/5 as class.detail
// Request codes occupy class 0; response codes occupy classes 2, 4, 5
enum CoapCode {
    // Requests (class 0)
    Empty,      // 0.00 — used in bare ACK and RST
    Get,        // 0.01
    Post,       // 0.02
    Put,        // 0.03
    Delete,     // 0.04

    // Success responses (class 2)
    Created,            // 2.01
    Deleted,            // 2.02
    Valid,              // 2.03
    Changed,            // 2.04
    Content,            // 2.05
    Continue,           // 2.31 (block-wise)

    // Client error responses (class 4)
    BadRequest,         // 4.00
    Unauthorized,       // 4.01
    BadOption,          // 4.02
    Forbidden,          // 4.03
    NotFound,           // 4.04
    MethodNotAllowed,   // 4.05
    NotAcceptable,      // 4.06
    RequestEntityIncomplete,  // 4.08 (block-wise)
    Conflict,           // 4.09
    PreconditionFailed, // 4.12
    RequestEntityTooLarge,    // 4.13
    UnsupportedContentFormat, // 4.15

    // Server error responses (class 5)
    InternalServerError,  // 5.00
    NotImplemented,       // 5.01
    BadGateway,           // 5.02
    ServiceUnavailable,   // 5.03
    GatewayTimeout,       // 5.04
    ProxyingNotSupported, // 5.05
}

impl CoapCode {
    fn class(&self): u8         // 0, 2, 4, or 5
    fn detail(&self): u8
    fn is_request(&self): bool
    fn is_success(&self): bool
    fn is_client_error(&self): bool
    fn is_server_error(&self): bool
    fn as_byte(&self): u8       // encodes as (class << 5) | detail
    fn from_byte(b: u8): Result[Self, CoapError]
}

// Option numbers from IANA CoAP Option Registry
enum CoapOption {
    IfMatch(Vec[u8]),           //  1 — precondition on ETag
    UriHost(String),            //  3
    ETag(Vec[u8]),              //  4 — up to 8 bytes
    IfNoneMatch,                //  5 — presence is the condition
    Observe(u32),               //  6 — 0=register, 1=deregister, or sequence number
    UriPort(u16),               //  7
    LocationPath(String),       //  8
    UriPath(String),            // 11 — one per path segment
    ContentFormat(u16),         // 12
    MaxAge(u32),                // 14 — seconds; default 60
    UriQuery(String),           // 15 — one per key=value pair
    HopLimit(u8),               // 16 — proxy hop limit
    Accept(u16),                // 17 — preferred content format
    LocationQuery(String),      // 20
    Block2(BlockOption),        // 23 — response block
    Block1(BlockOption),        // 27 — request block
    Size2(u32),                 // 28 — total response body size
    ProxyUri(String),           // 35
    ProxyScheme(String),        // 39
    Size1(u32),                 // 60 — total request body size
}

// Block option encoding: NUM | M | SZX (three fields packed into 1–3 bytes)
struct BlockOption {
    pub num:  u32,    // block number (zero-based)
    pub more: bool,   // M bit — more blocks follow
    pub szx:  u8,     // size exponent: actual size = 2^(szx+4), range 16–1024
}

impl BlockOption {
    fn block_size(&self): usize     // 2^(szx+4)
    fn byte_offset(&self): usize    // num * block_size()
}
```

---

## 6. Confirmable Messages and Retransmission

The CoAP reliability mechanism is specified in RFC 7252 §4. This library implements it transparently: callers invoke `client.get()` and see either a response or a `CoapError.Timeout`.

### Retransmission Schedule

When a Confirmable message is sent, the library starts a retransmission timer. The initial timeout is `ack_timeout * rand(1.0, ack_random_factor)`. On each timeout without an ACK, the timeout doubles. After `max_retransmit` attempts without a response, the message is abandoned and `CoapError.Timeout` is returned.

With the RFC defaults (`ack_timeout = 2s`, `ack_random_factor = 1.5`, `max_retransmit = 4`), the total transmission window before giving up is at most 247 seconds.

```
Attempt 1:  send  CON   msg_id=0x1234
            wait  [2.0s, 3.0s) (random jitter)
Attempt 2:  send  CON   msg_id=0x1234  (same message_id)
            wait  [4.0s, 6.0s)
Attempt 3:  send  CON   msg_id=0x1234
            wait  [8.0s, 12.0s)
Attempt 4:  send  CON   msg_id=0x1234
            wait  [16.0s, 24.0s)
Attempt 5:  send  CON   msg_id=0x1234
            -> no ACK: return Err(CoapError.Timeout)
```

### Deduplication

The server maintains a deduplication window indexed by `(peer_addr, message_id, token)`. If an identical CON is received while a response is being prepared, the server holds off until the handler completes and then sends the same ACK to both. If a CON is received after the ACK was already sent and the response is still cached, the cached response is retransmitted immediately.

The deduplication window is bounded: entries expire after `EXCHANGE_LIFETIME` (RFC 7252 §4.8.2, default 247 seconds).

---

## 7. Block-Wise Transfer (RFC 7959)

Block-wise transfer allows CoAP to carry payloads larger than a single UDP datagram. The maximum CoAP message size fits in a UDP packet without IP fragmentation (typically ~1024 bytes on constrained links); block-wise splits larger bodies across multiple exchanges.

This library implements block-wise transfer transparently. Application code — both handlers and client callers — sees complete payloads.

### Client Side

When `client.put()` or `client.post()` is called with a payload larger than `CoapClientConfig.max_payload_size`, the client automatically:

1. Sends the first block as `Block1: NUM=0, M=1, SZX=<negotiated>`.
2. Waits for a `2.31 Continue` response.
3. Sends subsequent blocks, incrementing NUM.
4. Sends the final block with `M=0`.
5. Returns the response from the final `2.04 Changed` or `2.01 Created`.

When a `get()` response carries `Block2`, the client automatically fetches all subsequent blocks and reassembles them before returning.

### Server Side

The server reassembles Block1 request bodies before invoking the handler. The handler receives a `CoapRequest` with the complete `payload` field populated. The handler does not see individual blocks.

For Block2 responses, if the handler returns a `CoapResponse` whose payload exceeds `CoapServerConfig.max_payload_size`, the server automatically fragments it and serves subsequent blocks in response to GET requests carrying `Block2: NUM=N`.

### Size Options

When initiating a block-wise transfer, the library includes `Size1` (for requests) or `Size2` (for responses) options carrying the total body length, allowing the peer to pre-allocate buffers.

---

## 8. Observe (RFC 7641)

Observe extends CoAP with a publish/subscribe mechanism. A client registers interest in a resource by sending GET with `Observe: 0`. The server sends notifications as the resource changes. The client deregisters with GET with `Observe: 1`.

### Client API

```ferrum
// Returned by CoapClient.observe()
type CoapObserver

impl CoapObserver {
    // Receive the next notification. Returns None when the observation ends
    // (server sent RST, max-age expired, or cancel_observe() was called).
    fn next(&mut self): impl Future[Output=Option[CoapObservation]] ! Async + Net

    // Send GET with Observe=1 to deregister, then close the stream.
    fn cancel_observe(&mut self): Result[(), CoapError] ! Async + Net
}

// AsyncIterator impl — works with `while let` loops
impl AsyncIterator for CoapObserver {
    type Item = CoapObservation
}

struct CoapObservation {
    pub sequence: u32,          // Observe option value — monotonically increasing
    pub code:     CoapCode,     // typically 2.05 Content
    pub options:  Vec[CoapOption],
    pub payload:  Vec[u8],
}

impl CoapObservation {
    fn content_format(&self): Option[ContentFormat]
    fn max_age(&self): Option[Duration>]
}
```

Example client usage:

```ferrum
let mut observer = client.observe(&temperature_uri).await?

while let Some(obs) = observer.next().await {
    let temp = parse_temperature(&obs.payload)?
    println("temperature: {temp:.1}°C")
    if temp > 80.0 {
        alert("overheat").await?
        observer.cancel_observe().await?
        break
    }
}
```

### Server API

```ferrum
// Implemented by resources that support observation
trait ObservableResource: Send + Sync + 'static {
    // Called for every GET (including the initial registration GET).
    // Returns the current representation of the resource.
    fn handle(&self, req: CoapRequest): impl Future[Output=CoapResponse] ! Async

    // Called when the server needs the current state to notify observers.
    // Typically simpler than handle() — no routing, just the payload.
    fn current_state(&self): impl Future[Output=CoapResponse] ! Async
}

// The server manages the observer list for each registered ObservableResource.
// Application code triggers notifications:
server.notify_resource("/sensors/temperature").await?
// This calls current_state() on the registered resource and sends
// the result as a CON notification to each registered observer.
// Observers that fail to ACK after max_retransmit attempts are removed.
```

---

## 9. Multicast (RFC 7390)

CoAP multicast allows a single request to reach all devices on a multicast group simultaneously. Multicast requests are always Non-Confirmable: confirmable multicast is explicitly prohibited by RFC 7252 §8.1 because ACK implosion would overwhelm the sender.

```ferrum
// Send a Non-Confirmable request to a multicast group.
// Collects all responses received within collect_for.
// The returned Vec may be empty (no devices responded) or contain
// responses from multiple devices; each response includes the sender's address
// in the CoapResponse (see below).
fn multicast(
    &self,
    group:       IpAddr,
    port:        u16,
    req:         CoapRequest,
    collect_for: Duration,
): Result[Vec[CoapResponse], CoapError] ! Async + Net

// Multicast responses include the sender address
struct CoapResponse {
    // ... (fields from §3) ...
    pub peer: Option[SocketAddr],   // Some for multicast responses, None for unicast
}
```

The UDP socket must have `set_multicast_ttl_v4` or equivalent configured before calling `multicast()`. The `CoapClientConfig` may include a `multicast_interface` field (an interface IP address or index) to select the outgoing interface.

Well-known CoAP multicast addresses (from RFC 7252 §12.8):

- `224.0.1.187` — All CoAP Nodes (IPv4)
- `FF0X::FD` — All CoAP Nodes (IPv6, scope X)

---

## 10. Content Formats

```ferrum
// IANA CoAP Content-Formats Registry (subset)
enum ContentFormat {
    TextPlain,          //     0 — text/plain; charset=utf-8
    LinkFormat,         //    40 — application/link-format
    Xml,                //    41 — application/xml
    OctetStream,        //    42 — application/octet-stream
    Exi,                //    47 — application/exi
    Json,               //    50 — application/json
    JsonPatch,          //    51 — application/json-patch+json
    MergePatch,         //    52 — application/merge-patch+json
    Cbor,               //    60 — application/cbor
    CoseEncrypt0,       //    16 — application/cose; cose-type="cose-encrypt0"
    CoseMac0,           //    17 — application/cose; cose-type="cose-mac0"
    CoseSign1,          //    18 — application/cose; cose-type="cose-sign1"
    CoseEncrypt,        //    96
    CoseMac,            //    97
    CoseSign,           //    98
    CoseKey,            //   101 — application/cose-key
    CoseKeySet,         //   102 — application/cose-key-set
    SenmlJson,          //   110 — application/senml+json (sensor measurements)
    SenmlCbor,          //   112 — application/senml+cbor
    Custom(u16),        // any other IANA-assigned or private value
}

impl ContentFormat {
    fn as_u16(&self): u16
    fn from_u16(n: u16): Self
    fn media_type(&self): &str
}
```

The `CoseEncrypt0`, `CoseMac0`, and `CoseSign1` variants are relevant to Matter: Matter uses COSE-encoded payloads for application-layer message protection on top of the DTLS transport.

---

## 11. Matter Integration

Matter (formerly Project CHIP) uses CoAP over UDP as its application-layer protocol. The security architecture has two layers that this library is aware of but does not implement:

**DTLS transport security.** `coaps://` URIs use DTLS 1.2 or 1.3 with the PSK or certificate cipher suites required by Matter. The `extlib.ccsp.dtls` module provides the DTLS engine. `CoapTransport.Dtls(DtlsConfig)` carries the session parameters.

**CASE/PASE session layer.** Matter's Certificate Authenticated Session Establishment (CASE) and Password Authenticated Session Establishment (PASE) run above DTLS. They establish a symmetric session key that is then used to encrypt individual CoAP messages with COSE (`ContentFormat.CoseEncrypt0`). This library does not parse CASE/PASE handshakes. A Matter implementation would:

1. Establish the CASE/PASE session (out of scope — handled by the Matter SDK or a dedicated extlib).
2. Construct CoAP messages with COSE-encrypted payloads using `std.data.cbor` and `extlib.ccsp.cose`.
3. Send them via `CoapClient` with `CoapTransport.Dtls(...)`.

The CoAP message framing, retransmission, block-wise transfer, and observe mechanics work identically regardless of whether the payload is plain CBOR or COSE-wrapped. Matter implementors use this module for CoAP mechanics and layer COSE on top.

---

## 12. Error Types

```ferrum
enum CoapError {
    // Transport errors
    TransmissionError(IoError),
        // Underlying UDP or DTLS send/receive failed

    Timeout,
        // No ACK received after max_retransmit attempts (§6)

    // Message errors
    MessageTooLarge { actual: usize, limit: usize },
        // Encoded message exceeds datagram size limit

    MalformedMessage(String),
        // Received message failed to decode

    UnexpectedCode { expected: CoapCode, got: CoapCode },
        // Response code does not match the request

    // Block-wise errors (RFC 7959)
    BlockTransferFailed { block_num: u32, reason: String },
        // A block exchange failed mid-transfer

    BlockSizeChanged,
        // Server changed SZX between blocks (prohibited by RFC 7959 §2.5)

    // Observe errors (RFC 7641)
    ObserveError(String),
        // Observation registration failed or was rejected by server

    ObserveSequenceGap { last: u32, got: u32 },
        // Sequence numbers jumped backward; observation may be stale

    // DTLS errors
    DtlsError(DtlsError),
        // Wrapped from extlib.ccsp.dtls — handshake, record, or alert errors

    // Server errors
    HandlerPanic(String),
        // A CoapHandler returned via panic propagation

    // DNS errors
    DnsError(DnsError),
        // Host resolution failed during URI-to-SocketAddr conversion

    // Multicast errors
    MulticastNotSupported,
        // Platform does not support multicast on this socket type

    ConfirmableMulticastProhibited,
        // Caller attempted to send a Confirmable multicast (RFC 7252 §8.1)
}
```

---

## 13. Example Usage

### Temperature Sensor Server with Observable Resource

A constrained device exposes a temperature sensor over CoAP. Clients can poll it or register as observers.

```ferrum
use extlib.ccsp.coap.{
    CoapServer, CoapServerConfig, CoapTransport, CoapRequest, CoapResponse,
    CoapCode, ContentFormat, ObservableResource,
}
use std.net.{SocketAddr, IpAddr, Ipv4Addr}
use std.async.Runtime
use std.time.Duration
use std.sync.Arc
use std.sync.Mutex

struct TemperatureSensor {
    // In real code: reads from hardware ADC
    current_celsius: Mutex[f32],
}

impl TemperatureSensor {
    fn new(): Self {
        Self { current_celsius: Mutex.new(22.0) }
    }

    fn set_temperature(&self, t: f32) {
        *self.current_celsius.lock() = t
    }

    fn read_temperature(&self): f32 {
        *self.current_celsius.lock()
    }
}

impl ObservableResource for TemperatureSensor {
    fn handle(
        &self,
        req: CoapRequest,
    ): impl Future[Output=CoapResponse] ! Async {
        {
            let temp = self.read_temperature()
            let payload = format!("{:.1}", temp).into_bytes()
            CoapResponse {
                code: CoapCode.Content,
                options: vec![
                    CoapOption.ContentFormat(ContentFormat.TextPlain.as_u16()),
                    CoapOption.MaxAge(30),
                ],
                payload,
                token: req.token.clone(),
            }
        }
    }

    fn current_state(
        &self,
    ): impl Future[Output=CoapResponse] ! Async {
        {
            let temp = self.read_temperature()
            let payload = format!("{:.1}", temp).into_bytes()
            CoapResponse {
                code: CoapCode.Content,
                options: vec![
                    CoapOption.ContentFormat(ContentFormat.TextPlain.as_u16()),
                    CoapOption.MaxAge(30),
                ],
                payload,
                token: vec![],
            }
        }
    }
}

fn run_sensor_server(): Result[(), CoapError] ! Async + Net {
    let sensor = Arc.new(TemperatureSensor.new())
    let sensor_notify = sensor.clone()

    let addr: SocketAddr = SocketAddr.new(
        IpAddr.V4(Ipv4Addr.UNSPECIFIED),
        5683,
    )
    let config = CoapServerConfig.default_udp(addr)

    let mut server = CoapServer.bind(addr, config).await?

    server.observable_route("/sensors/temperature", sensor)

    scope s {
        // Simulate hardware sensor readings on a separate task
        let server_ref = &server
        s.spawn({
            let mut interval = std.async.interval(Duration.from_secs(5))
            loop {
                interval.tick().await
                let new_temp = read_adc_temperature()
                sensor_notify.set_temperature(new_temp)
                server_ref.notify_resource("/sensors/temperature").await.ok()
            }
        })

        server.run().await?
    }

    Ok(())
}

fn main() ! IO + Async {
    let runtime = Runtime.new()?
    runtime.block_on(run_sensor_server()).unwrap()
}
```

### Client Polling a Matter Device

A gateway queries a Matter device's OnOff cluster and toggles it.

```ferrum
use extlib.ccsp.coap.{CoapClient, CoapClientConfig, CoapTransport, CoapUri, CoapCode, ContentFormat}
use extlib.ccsp.dtls.DtlsConfig

fn toggle_matter_light(dtls_config: DtlsConfig): Result[(), CoapError] ! Async + Net {
    let config = CoapClientConfig.default_dtls(dtls_config)
    let client = CoapClient.new(config).await?

    // Read current OnOff state (Matter cluster 0x0006, attribute 0x0000)
    // Matter URIs use coaps:// over DTLS
    let state_uri = CoapUri.parse("coaps://light-1.local/clusters/6/attributes/0")?

    let response = client.get(&state_uri).await?

    if !response.is_success() {
        return Err(CoapError.UnexpectedCode {
            expected: CoapCode.Content,
            got: response.code,
        })
    }

    // Parse the CBOR-encoded attribute value (extlib.ccsp.cose would unwrap
    // COSE encryption first in a real Matter implementation)
    let is_on: bool = std.data.cbor.from_slice(&response.payload)?

    // Toggle via PUT
    let toggle_uri = CoapUri.parse("coaps://light-1.local/clusters/6/commands/2")?
    let toggle_payload = std.data.cbor.to_vec(&())?  // empty command payload

    let put_response = client.put(
        &toggle_uri,
        &toggle_payload,
        ContentFormat.Cbor,
    ).await?

    if !put_response.is_success() {
        return Err(CoapError.UnexpectedCode {
            expected: CoapCode.Changed,
            got: put_response.code,
        })
    }

    println("light was {}, now toggled", if is_on { "on" } else { "off" })
    Ok(())
}
```

### Observing Temperature with Timeout

```ferrum
fn monitor_temperature(
    client: &CoapClient,
    uri: &CoapUri,
    duration: Duration,
): Result[Vec[f32], CoapError] ! Async + Net {
    let mut observer = client.observe(uri).await?
    let deadline = std.time.Instant.now() + duration
    let mut readings = Vec.new()

    loop {
        let remaining = deadline.saturating_duration_since(std.time.Instant.now())
        if remaining.is_zero() {
            break
        }

        let result = select {
            observer.next() -> obs => obs,
            std.async.timeout(remaining) -> _ => None,
        }

        match result {
            Some(obs) => {
                let temp: f32 = std.str.from_utf8(&obs.payload)?.parse()?
                readings.push(temp)
            },
            None => break,
        }
    }

    observer.cancel_observe().await?
    Ok(readings)
}
```

---

## 14. Dependencies

| Dependency | Role |
|---|---|
| `extlib.ccsp.dtls` | DTLS 1.2/1.3 engine for `coaps://` transport; `DtlsConfig` and `DtlsError` types |
| `extlib.ccsp.dns_secure` | DNSSEC-validated DNS resolution used by `CoapUri.to_socket_addr()` |
| `std.net` | `UdpSocket`, `SocketAddr`, `IpAddr`, `Ipv4Addr`, `Ipv6Addr`, `IoError` |
| `std.async` | `Runtime`, `Future`, `AsyncIterator`, `scope`, `select`, `interval`, `timeout` |
| `std.data.cbor` | CBOR encoding and decoding for `ContentFormat.Cbor` payloads and Matter integration |
| `std.sync` | `Arc`, `Mutex` — shared state in observable resources and multi-task server loops |
| `std.time` | `Duration`, `Instant` — timeout and retransmission scheduling |

### Feature Flags

```
coap-dtls    = ["extlib.ccsp.dtls"]   // enables CoapTransport.Dtls; required for coaps://
coap-dns     = ["extlib.ccsp.dns_secure"]   // enables DNSSEC validation in URI resolution
coap-cbor    = ["std.data.cbor"]      // enables CBOR convenience methods on CoapResponse
coap-observe = []                     // always included; no extra deps
coap-block   = []                     // always included; no extra deps
```

A constrained build that omits DTLS and DNS can still speak plain `coap://` by depending only on `std.net` and `std.async`.

---

*Part of the CCSP (Constrained and Cryptographic Systems Protocol) extended library suite.*
*See also: `extlib.ccsp.dtls`, `extlib.ccsp.dns_secure`, `extlib.ccsp.cose`*
