# Ferrum Standard Library — async, net, http

**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)

---

## 9. async — Event Loop and Async Runtime

### 9.1 Design Philosophy

Ferrum does not have colored functions (`async fn` vs `fn`). Instead:

- `scope` provides structured concurrency for blocking + async code.
- The async runtime is a capability, not a global.
- IO operations on non-blocking handles return `WouldBlock` which the runtime handles.
- The programmer chooses the concurrency model per scope; the language does not impose one.

```ferrum
// Blocking style — simplest, always works
fn handle(conn: TcpStream): Result[Response] ! Net {
    let req = Request.parse(&conn)?
    let resp = process(req)?
    resp.write(&conn)?
    Ok(resp)
}

// Async style — multiplexed over many connections
fn handle_async(conn: TcpStream): Result[Response] ! Net + Async {
    let req = Request.parse_async(&conn).await?
    let resp = process(req).await?
    resp.write_async(&conn).await?
    Ok(resp)
}

// The difference is in the IO types used, not the function color.
// TcpStream implements both blocking and async IO.
```

### 9.2 The Runtime

The runtime manages the thread pool that powers both `scope.spawn()` and
async task execution. It is a capability, not a global singleton.

**Thread model:** The runtime owns a pool of worker threads. Tasks spawned
via `scope.spawn()` or `runtime.spawn()` execute on this pool. Raw OS
threads are not exposed in portable code — see `sys.posix.pthread` and
`sys.windows.thread` for platform-specific thread primitives when needed.

```ferrum
// Runtime is a capability — not a global singleton
type Runtime  given [A: Allocator]

impl Runtime {
    fn new(): Result[Self, RuntimeError]  given [A: Allocator]
    fn builder(): RuntimeBuilder

    fn block_on[T](&self, f: impl Future[Output=T]): T  ! IO + Async
        // Run an async computation to completion, blocking the current thread

    fn spawn[T: Send + 'static](&self, f: impl Future[Output=T]): JoinHandle[T]  ! Async
        // Spawn a task on the runtime's thread pool

    fn spawn_blocking[T: Send + 'static](&self, f: fn(): T): JoinHandle[T]  ! Async
        // Run a blocking operation on the blocking thread pool

    fn enter(&self): RuntimeGuard
        // Set this runtime as the ambient runtime for the current thread
}

struct RuntimeBuilder {
    fn worker_threads(&mut self, n: usize): &mut Self
    fn blocking_threads(&mut self, n: usize): &mut Self
    fn thread_stack_size(&mut self, size: usize): &mut Self
    fn thread_name(&mut self, name: impl Into[String]): &mut Self
    fn on_thread_start(&mut self, f: fn()): &mut Self
    fn on_thread_stop(&mut self, f: fn()): &mut Self
    fn build(&self): Result[Runtime, RuntimeError]
}
```

### 9.3 Futures

```ferrum
trait Future {
    type Output

    fn poll(&mut self, cx: &mut Context): Poll[Self.Output]
}

enum Poll[T] {
    Ready(T),
    Pending,
}

// Context carries the waker
struct Context { ... }
struct Waker   { ... }

// fn with ! Async desugars to a state machine implementing Future
// await desugars to poll + yield-if-pending
```

### 9.4 Async IO Traits

```ferrum
trait AsyncRead {
    fn poll_read(
        &mut self,
        cx: &mut Context,
        buf: &mut [u8],
    ): Poll[ReadResult]
}

trait AsyncWrite {
    fn poll_write(
        &mut self,
        cx: &mut Context,
        buf: &[u8],
    ): Poll[Result[usize, IoError]]

    fn poll_flush(
        &mut self,
        cx: &mut Context,
    ): Poll[Result[(), IoError]]

    fn poll_shutdown(
        &mut self,
        cx: &mut Context,
    ): Poll[Result[(), IoError]]
}

trait AsyncSeek {
    fn poll_seek(
        &mut self,
        cx: &mut Context,
        pos: SeekFrom,
    ): Poll[Result[u64, IoError]]
}

trait AsyncBufRead: AsyncRead {
    fn poll_fill_buf(
        &mut self,
        cx: &mut Context,
    ): Poll[Result[&[u8], IoError]]

    fn consume(&mut self, amt: usize)
}

// Extension traits add .await-friendly methods
// NOTE: No read_to_string — async IO deals in bytes.
// Use AsyncTextReader for text with explicit encoding.
trait AsyncReadExt: AsyncRead {
    fn read(&mut self, buf: &mut [u8]): impl Future[Output=ReadResult]
    fn read_exact(&mut self, buf: &mut [u8]): impl Future[Output=Result[(), IoError]]
    fn read_to_end(&mut self, buf: &mut Vec[u8]): impl Future[Output=Result[usize, IoError]]
}

trait AsyncWriteExt: AsyncWrite {
    fn write(&mut self, buf: &[u8]): impl Future[Output=Result[usize, IoError]]
    fn write_all(&mut self, buf: &[u8]): impl Future[Output=Result[(), IoError]]
    fn flush(&mut self): impl Future[Output=Result[(), IoError]]
    fn shutdown(&mut self): impl Future[Output=Result[(), IoError]]
}
```

### 9.5 Timers and Scheduling

```ferrum
// Sleep
fn sleep(duration: Duration): impl Future[Output=()] ! Async

// Timeout — wraps any future
fn timeout[T](duration: Duration, f: impl Future[Output=T])
    : impl Future[Output=Result[T, Elapsed]]  ! Async

// Interval — fires at regular intervals
fn interval(period: Duration): Interval ! Async
struct Interval {
    fn tick(&mut self): impl Future[Output=Instant]
    fn reset(&mut self)
    fn reset_at(&mut self, deadline: Instant)
}

// Select over multiple futures
// (select expression or explicit select combinator)
fn select2[A, B](
    f1: impl Future[Output=A],
    f2: impl Future[Output=B],
): impl Future[Output=Either[A, B]]

// Select over a dynamic set
fn select_all[T](
    futures: Vec[impl Future[Output=T]],
): impl Future[Output=(T, usize, Vec[impl Future[Output=T]])]
```

### 9.6 Async Channels

```ferrum
// Async equivalents of sync channels
fn async_channel[T](capacity: usize): (AsyncSender[T], AsyncReceiver[T])
fn async_broadcast[T: Clone](capacity: usize): (BroadcastSender[T], BroadcastReceiver[T])

struct AsyncSender[T] {
    fn send(&self, val: T): impl Future[Output=Result[(), SendError[T]]]
    fn try_send(&self, val: T): Result[(), TrySendError[T]]
    fn closed(&self): impl Future[Output=()]  // resolves when receiver dropped
    fn is_closed(&self): bool
    fn len(&self): usize
    fn capacity(&self): Option[usize]
}

struct AsyncReceiver[T] {
    fn recv(&mut self): impl Future[Output=Option[T]]
    fn try_recv(&mut self): Result[T, TryRecvError]
    fn len(&self): usize
    fn is_empty(&self): bool
}
```

### 9.7 Task Utilities

```ferrum
// Yield the current task — allows other tasks to run
fn yield_now(): impl Future[Output=()]

// Join — wait for multiple futures concurrently
fn join2[A, B](a: impl Future[Output=A], b: impl Future[Output=B])
    : impl Future[Output=(A, B)]

fn join_all[T, I: IntoIterator[Item=impl Future[Output=T]]](futures: I)
    : impl Future[Output=Vec[T]]

// Try join — short-circuits on first error
fn try_join2[A, B, E](
    a: impl Future[Output=Result[A, E]],
    b: impl Future[Output=Result[B, E]],
): impl Future[Output=Result[(A, B), E]]

// Spawn and forget — still bounded by runtime lifetime
fn spawn_and_forget(f: impl Future[Output=()] + Send + 'static): () ! Async
```

---

## 10. net — Networking

### 10.1 IP Addresses and Sockets

```ferrum
// Addresses — strongly typed, no stringly-typed mistakes
enum IpAddr {
    V4(Ipv4Addr),
    V6(Ipv6Addr),
}

type Ipv4Addr([u8; 4])
type Ipv6Addr([u8; 16])

impl Ipv4Addr {
    fn new(a: u8, b: u8, c: u8, d: u8): Self
    const LOCALHOST:   Self = Ipv4Addr([127, 0, 0, 1])
    const UNSPECIFIED: Self = Ipv4Addr([0, 0, 0, 0])
    const BROADCAST:   Self = Ipv4Addr([255, 255, 255, 255])
    fn is_private(&self): bool
    fn is_loopback(&self): bool
    fn is_multicast(&self): bool
    fn to_ipv6_mapped(&self): Ipv6Addr
    fn octets(&self): [u8; 4]
}

struct SocketAddr {
    ip:   IpAddr,
    port: Port,
}

impl SocketAddr {
    fn new(ip: IpAddr, port: Port): Self
    fn from_str(s: &str): Result[Self, AddrParseError]  // "127.0.0.1:8080"
}

// Port is a constrained integer (see language reference §3.3)
type Port = u16 where value > 0
```

### 10.2 TCP

```ferrum
type TcpListener

impl TcpListener {
    fn bind(addr: SocketAddr): Result[Self, IoError] ! Net
    fn bind_all(addrs: &[SocketAddr]): Result[Self, IoError] ! Net  // try each
    fn accept(&self): Result[(TcpStream, SocketAddr), IoError] ! Net
        // Blocks until a connection arrives
    fn incoming(&self): Incoming  // iterator over accepted connections
    fn local_addr(&self): Result[SocketAddr, IoError]
    fn set_ttl(&self, ttl: u32): Result[(), IoError] ! Net
    fn ttl(&self): Result[u32, IoError]
    fn set_only_v6(&self, only_v6: bool): Result[(), IoError] ! Net
    fn only_v6(&self): Result[bool, IoError]

    // Async
    fn accept_async(&self): impl Future[Output=Result[(TcpStream, SocketAddr), IoError]]
}

type TcpStream

impl TcpStream {
    fn connect(addr: SocketAddr): Result[Self, IoError] ! Net
    fn connect_timeout(addr: SocketAddr, timeout: Duration): Result[Self, IoError] ! Net

    // Happy Eyeballs (RFC 8305) — the right way to connect
    //
    // Resolves hostname, tries IPv6 first with 250ms stagger, races IPv4
    // in parallel if IPv6 hasn't connected. First to complete wins.
    // Always checks SO_ERROR after nonblocking connect becomes writable.
    //
    // This is what connect() should have been. Use this for hostnames.
    fn connect_happy(host: &str, port: Port): Result[Self, IoError] ! Net
    fn connect_happy_timeout(host: &str, port: Port, timeout: Duration): Result[Self, IoError] ! Net

    // With DANE — validates server cert against TLSA records if DNSSEC validates
    fn connect_dane(host: &str, port: Port, tls: &TlsConfig): Result[TlsStream, IoError] ! Net
    fn peer_addr(&self): Result[SocketAddr, IoError]
    fn local_addr(&self): Result[SocketAddr, IoError]
    fn shutdown(&self, how: Shutdown): Result[(), IoError] ! Net
    fn set_nodelay(&self, nodelay: bool): Result[(), IoError] ! Net
    fn nodelay(&self): Result[bool, IoError]
    fn set_keepalive(&self, keepalive: Option[Duration]): Result[(), IoError] ! Net
    fn set_read_timeout(&self, dur: Option[Duration]): Result[(), IoError] ! Net
    fn set_write_timeout(&self, dur: Option[Duration]): Result[(), IoError] ! Net
    fn try_clone(&self): Result[TcpStream, IoError] ! Net
    fn peek(&self, buf: &mut [u8]): Result[usize, IoError] ! Net
}

impl Read for TcpStream
impl Write for TcpStream

enum Shutdown { Read, Write, Both }
```

### 10.3 UDP

```ferrum
type UdpSocket

impl UdpSocket {
    fn bind(addr: SocketAddr): Result[Self, IoError] ! Net
    fn connect(&self, addr: SocketAddr): Result[(), IoError] ! Net
        // "connect" for UDP just sets the default destination

    fn send(&self, buf: &[u8]): Result[usize, IoError] ! Net
        // requires connect() called first
    fn recv(&self, buf: &mut [u8]): Result[usize, IoError] ! Net

    fn send_to(&self, buf: &[u8], addr: SocketAddr): Result[usize, IoError] ! Net
    fn recv_from(&self, buf: &mut [u8]): Result[(usize, SocketAddr), IoError] ! Net
    fn peek_from(&self, buf: &mut [u8]): Result[(usize, SocketAddr), IoError] ! Net

    fn local_addr(&self): Result[SocketAddr, IoError]
    fn peer_addr(&self): Result[SocketAddr, IoError]
    fn set_broadcast(&self, on: bool): Result[(), IoError] ! Net
    fn join_multicast_v4(&self, multicast: Ipv4Addr, interface: Ipv4Addr): Result[(), IoError] ! Net
    fn leave_multicast_v4(&self, multicast: Ipv4Addr, interface: Ipv4Addr): Result[(), IoError] ! Net
    fn set_multicast_ttl_v4(&self, ttl: u32): Result[(), IoError] ! Net
    fn set_ttl(&self, ttl: u32): Result[(), IoError] ! Net
    fn set_read_timeout(&self, dur: Option[Duration]): Result[(), IoError] ! Net
    fn set_write_timeout(&self, dur: Option[Duration]): Result[(), IoError] ! Net
    fn try_clone(&self): Result[UdpSocket, IoError] ! Net
}
```

### 10.4 DNS

```ferrum
// Resolver — not a global, a capability
type Resolver  given [A: Allocator]

impl Resolver {
    fn system(): Result[Self, ResolverError] ! Net  // use system DNS
    fn with_servers(servers: &[SocketAddr]): Result[Self, ResolverError] ! Net
    fn with_config(config: ResolverConfig): Result[Self, ResolverError]

    fn lookup_ip(&self, host: &str): Result[LookupIp, ResolveError] ! Net
    fn lookup_host(&self, host: &str): Result[LookupHost, ResolveError] ! Net
    fn reverse_lookup(&self, ip: IpAddr): Result[ReverseLookup, ResolveError] ! Net
    fn lookup_mx(&self, host: &str): Result[LookupMx, ResolveError] ! Net
    fn lookup_txt(&self, host: &str): Result[LookupTxt, ResolveError] ! Net
    fn lookup_srv(&self, service: &str, proto: &str, name: &str): Result[LookupSrv, ResolveError] ! Net
    fn lookup_tlsa(&self, port: u16, proto: &str, name: &str): Result[LookupTlsa, ResolveError] ! Net

    // Async versions
    fn lookup_ip_async(&self, host: &str): impl Future[Output=Result[LookupIp, ResolveError]]
}

struct ResolverConfig {
    servers:     Vec[SocketAddr],
    transport:   DnsTransport,          // UDP, TCP, DoT, DoH
    dnssec:      DnssecMode,            // Off, Validate, Require
    search:      Vec[String],
    ndots:       u8,
    timeout:     Duration,
    attempts:    u8,
}

enum DnsTransport {
    Udp,                                // Default, falls back to TCP on truncation
    Tcp,                                // TCP only
    DoT(TlsConfig),                     // DNS-over-TLS (port 853)
    DoH { url: Uri, tls: TlsConfig },   // DNS-over-HTTPS
}

enum DnssecMode {
    Off,       // No validation (legacy networks)
    Validate,  // Validate when possible, accept unsigned
    Require,   // Reject responses that fail validation
}

struct LookupIp {
    fn iter(&self): impl Iterator[Item=IpAddr]
    fn query_name(&self): &Name
    fn ttl(&self): Duration
    fn dnssec_validated(&self): bool    // True if DNSSEC chain validated
}

struct LookupTlsa {
    fn iter(&self): impl Iterator[Item=TlsaRecord]
    fn dnssec_validated(&self): bool
}

// DANE — certificate validation via DNS
struct TlsaRecord {
    usage:        TlsaUsage,            // CA, EE, Trust Anchor, Domain Issued
    selector:     TlsaSelector,         // Full cert or SubjectPublicKeyInfo
    matching:     TlsaMatching,         // Exact, SHA-256, SHA-512
    data:         Vec[u8],
}

enum TlsaUsage {
    CaConstraint,       // PKIX-TA (0): CA must be in path
    ServiceCert,        // PKIX-EE (1): EE cert must match
    TrustAnchor,        // DANE-TA (2): TA for this service only
    DomainIssued,       // DANE-EE (3): EE cert, no PKIX required
}
```

---

## 11. http — HTTP Stack

### 11.1 Philosophy

HTTP is a first-class type. Not a pile of socket calls. Not a stringly-typed mess.
Go's `net/http` is the target experience quality. No external crate required for basic HTTP.

### 11.2 Types

```ferrum
// HTTP versions
enum Version { Http10, Http11, Http2, Http3 }

// Methods — typed, not strings
enum Method {
    Get, Post, Put, Delete, Head, Options, Patch, Trace, Connect,
    Extension(String),  // for non-standard methods
}

impl Method {
    fn as_str(&self): &str
    fn is_safe(&self): bool        // GET HEAD OPTIONS TRACE
    fn is_idempotent(&self): bool  // GET HEAD PUT DELETE OPTIONS TRACE
}

// Status codes — typed with named constants
type StatusCode(u16)
    where value >= 100 and value <= 999

impl StatusCode {
    const OK:                    Self = StatusCode(200)
    const CREATED:               Self = StatusCode(201)
    const ACCEPTED:              Self = StatusCode(202)
    const NO_CONTENT:            Self = StatusCode(204)
    const MOVED_PERMANENTLY:     Self = StatusCode(301)
    const FOUND:                 Self = StatusCode(302)
    const NOT_MODIFIED:          Self = StatusCode(304)
    const TEMPORARY_REDIRECT:    Self = StatusCode(307)
    const PERMANENT_REDIRECT:    Self = StatusCode(308)
    const BAD_REQUEST:           Self = StatusCode(400)
    const UNAUTHORIZED:          Self = StatusCode(401)
    const FORBIDDEN:             Self = StatusCode(403)
    const NOT_FOUND:             Self = StatusCode(404)
    const METHOD_NOT_ALLOWED:    Self = StatusCode(405)
    const CONFLICT:              Self = StatusCode(409)
    const GONE:                  Self = StatusCode(410)
    const UNPROCESSABLE_ENTITY:  Self = StatusCode(422)
    const TOO_MANY_REQUESTS:     Self = StatusCode(429)
    const INTERNAL_SERVER_ERROR: Self = StatusCode(500)
    const NOT_IMPLEMENTED:       Self = StatusCode(501)
    const BAD_GATEWAY:           Self = StatusCode(502)
    const SERVICE_UNAVAILABLE:   Self = StatusCode(503)
    const GATEWAY_TIMEOUT:       Self = StatusCode(504)

    fn is_informational(&self): bool  // 1xx
    fn is_success(&self): bool        // 2xx
    fn is_redirection(&self): bool    // 3xx
    fn is_client_error(&self): bool   // 4xx
    fn is_server_error(&self): bool   // 5xx
    fn canonical_reason(&self): Option[&str]
}

// Headers — case-insensitive name, Vec[String] values
type HeaderName    // case-insensitive, interned
type HeaderValue   // bytes, validated to be ASCII or opaque
type HeaderMap     // multi-value map

impl HeaderMap {
    fn new(): Self
    fn insert(&mut self, name: HeaderName, value: HeaderValue)
    fn append(&mut self, name: HeaderName, value: HeaderValue)
    fn get(&self, name: &HeaderName): Option[&HeaderValue]
    fn get_all(&self, name: &HeaderName): GetAll  // iterator over all values
    fn remove(&mut self, name: &HeaderName): Option[HeaderValue]
    fn contains_key(&self, name: &HeaderName): bool
    fn iter(&self): impl Iterator[Item=(&HeaderName, &HeaderValue)]
    fn len(&self): usize
    fn is_empty(&self): bool
}

// Well-known header names as constants
mod header {
    const ACCEPT:           HeaderName = HeaderName.from_static("accept")
    const ACCEPT_ENCODING:  HeaderName = ...
    const AUTHORIZATION:    HeaderName = ...
    const CACHE_CONTROL:    HeaderName = ...
    const CONTENT_LENGTH:   HeaderName = ...
    const CONTENT_TYPE:     HeaderName = ...
    const COOKIE:           HeaderName = ...
    const HOST:             HeaderName = ...
    const LOCATION:         HeaderName = ...
    const SET_COOKIE:       HeaderName = ...
    const USER_AGENT:       HeaderName = ...
    // ... etc.
}

// URI/URL — typed, validated on construction
type Uri  given [A: Allocator]

impl Uri {
    fn parse(s: &str): Result[Self, UriError]
    fn scheme(&self): Option[&str]
    fn authority(&self): Option[&str]
    fn host(&self): Option[&str]
    fn port(&self): Option[Port]
    fn path(&self): &str
    fn query(&self): Option[&str]
    fn fragment(&self): Option[&str]
    fn query_pairs(&self): QueryPairs     // iterator over (key, value)
    fn with_path(&self, path: &str): Result[Uri, UriError]
    fn with_query(&self, query: &str): Result[Uri, UriError]
}
```

### 11.3 Request and Response

```ferrum
struct Request  given [A: Allocator] {
    pub method:  Method,
    pub uri:     Uri,
    pub version: Version,
    pub headers: HeaderMap,
    pub body:    Body,
}

impl Request {
    fn builder(): RequestBuilder
    fn method(&self): &Method
    fn uri(&self): &Uri
    fn headers(&self): &HeaderMap
    fn body(&self): &Body
    fn body_mut(&mut self): &mut Body
    fn into_body(self): Body
}

struct RequestBuilder {
    fn method(self, method: Method): Self
    fn uri(self, uri: impl TryInto[Uri]): Self
    fn header(self, name: HeaderName, value: impl IntoHeaderValue): Self
    fn body(self, body: impl Into[Body]): Result[Request, HttpError]
    fn json[T: Serialize](self, body: &T): Result[Request, HttpError]
    fn form[T: Serialize](self, body: &T): Result[Request, HttpError]
    fn multipart(self, form: Multipart): Result[Request, HttpError]
}

struct Response  given [A: Allocator] {
    pub status:  StatusCode,
    pub version: Version,
    pub headers: HeaderMap,
    pub body:    Body,
}

impl Response {
    fn builder(): ResponseBuilder
    fn status(&self): StatusCode
    fn headers(&self): &HeaderMap

    fn bytes(&mut self): impl Future[Output=Result[Bytes, HttpError]]  ! Net
    fn text(&mut self): impl Future[Output=Result[String, HttpError]]   ! Net
    fn json[T: Deserialize](&mut self): impl Future[Output=Result[T, HttpError]]  ! Net
    fn error_for_status(self): Result[Self, HttpError]  // Err if 4xx/5xx
    fn error_for_status_ref(&self): Result[&Self, HttpError]
}

// Body — streaming, not eagerly-loaded
enum Body {
    Empty,
    Bytes(Bytes),
    Stream(Box[dyn Stream[Item=Result[Bytes, IoError]] + Send]),
}

impl Body {
    fn empty(): Self
    fn from_bytes(bytes: impl Into[Bytes]): Self
    fn from_stream(s: impl Stream[Item=Result[Bytes, IoError]] + Send + 'static): Self
    fn len(&self): Option[u64]      // known only for Bytes
    fn is_empty(&self): bool
}
```

### 11.4 HTTP Client

```ferrum
type Client  given [A: Allocator]

impl Client {
    fn new(): Self
    fn builder(): ClientBuilder

    // Primary interface
    fn get(&self, url: impl IntoUrl): RequestBuilder     ! Net
    fn post(&self, url: impl IntoUrl): RequestBuilder    ! Net
    fn put(&self, url: impl IntoUrl): RequestBuilder     ! Net
    fn delete(&self, url: impl IntoUrl): RequestBuilder  ! Net
    fn patch(&self, url: impl IntoUrl): RequestBuilder   ! Net
    fn head(&self, url: impl IntoUrl): RequestBuilder    ! Net
    fn request(&self, method: Method, url: impl IntoUrl): RequestBuilder  ! Net
    fn execute(&self, req: Request): impl Future[Output=Result[Response, HttpError]]  ! Net

    // Convenient shorthand for common patterns
    fn get_json[T: Deserialize](&self, url: impl IntoUrl)
        : impl Future[Output=Result[T, HttpError]]  ! Net
    fn post_json[T: Serialize, R: Deserialize](&self, url: impl IntoUrl, body: &T)
        : impl Future[Output=Result[R, HttpError]]  ! Net
}

struct ClientBuilder {
    fn timeout(&mut self, timeout: Duration): &mut Self
    fn connect_timeout(&mut self, timeout: Duration): &mut Self
    fn user_agent(&mut self, val: impl IntoHeaderValue): &mut Self
    fn default_headers(&mut self, headers: HeaderMap): &mut Self
    fn redirect(&mut self, policy: RedirectPolicy): &mut Self
    fn cookie_store(&mut self, enable: bool): &mut Self
    fn cookie_provider(&mut self, store: Arc[dyn CookieStore]): &mut Self
    fn tcp_nodelay(&mut self, enable: bool): &mut Self
    fn tcp_keepalive(&mut self, val: Duration): &mut Self
    fn connection_verbose(&mut self, enable: bool): &mut Self
    fn pool_idle_timeout(&mut self, val: Duration): &mut Self
    fn pool_max_idle_per_host(&mut self, max: usize): &mut Self
    fn http1_only(&mut self): &mut Self
    fn http2_prior_knowledge(&mut self): &mut Self
    fn tls(... ): &mut Self              // TLS config — see crypto
    fn build(&self): Result[Client, HttpError]
}

enum RedirectPolicy {
    None,
    Limited(usize),
    Custom(fn(&Request, &Response): bool),
}
```

### 11.5 HTTP Server

```ferrum
type Server  given [A: Allocator]

impl Server {
    fn bind(addr: SocketAddr): ServerBuilder ! Net
}

struct ServerBuilder {
    fn handler(self, handler: impl Handler): Self
    fn timeout(self, timeout: Duration): Self
    fn max_connections(self, n: usize): Self
    fn workers(self, n: usize): Self
    fn serve(self): impl Future[Output=Result[(), HttpError]]  ! Net + Async

    // Graceful shutdown
    fn serve_with_shutdown(
        self,
        signal: impl Future[Output=()],
    ): impl Future[Output=Result[(), HttpError]]  ! Net + Async
}

// Handler trait — the core abstraction
trait Handler: Clone + Send + 'static {
    fn handle(&self, req: Request): impl Future[Output=Response]  ! Net
}

// Blanket impl for closures
impl[F, Fut] Handler for F
where
    F: Fn(Request): Fut + Clone + Send + 'static,
    Fut: Future[Output=Response] + Send,
{ ... }

// Router — path-based dispatch
type Router  given [A: Allocator]

impl Router {
    fn new(): Self
    fn get(self, path: &str, handler: impl Handler): Self
    fn post(self, path: &str, handler: impl Handler): Self
    fn put(self, path: &str, handler: impl Handler): Self
    fn delete(self, path: &str, handler: impl Handler): Self
    fn patch(self, path: &str, handler: impl Handler): Self
    fn any(self, path: &str, handler: impl Handler): Self
    fn nest(self, prefix: &str, router: Router): Self
    fn fallback(self, handler: impl Handler): Self
    fn layer(self, middleware: impl Layer): Self
        // middleware wraps the entire router
}

// Path parameters
// Path patterns: "/users/{id}" — typed extraction
trait FromRequest: Sized {
    fn from_request(req: &Request, params: &PathParams): Result[Self, HttpError]
}

// Extractors
type PathParams   // extracted from URL pattern
type QueryParams  // from ?key=value
type JsonBody[T]  // deserializes JSON body
type FormBody[T]  // deserializes form body
type TypedHeader[T: Header]  // typed header extraction

// Middleware
trait Layer: Clone {
    type Service: Handler
    fn layer(&self, inner: impl Handler): Self.Service
}

// Built-in middleware
fn trace_layer(): impl Layer           // request/response logging
fn compression_layer(): impl Layer     // gzip/brotli response compression
fn timeout_layer(d: Duration): impl Layer
fn cors_layer(config: CorsConfig): impl Layer
fn rate_limit_layer(n: u32, per: Duration): impl Layer
fn auth_layer(validator: impl AuthValidator): impl Layer

// Response builder helpers
fn ok(body: impl Into[Body]): Response
fn created(location: Uri, body: impl Into[Body]): Response
fn no_content(): Response
fn bad_request(msg: impl Into[String]): Response
fn unauthorized(): Response
fn forbidden(): Response
fn not_found(): Response
fn internal_server_error(msg: impl Into[String]): Response
fn json_response[T: Serialize](status: StatusCode, body: &T): Result[Response, HttpError]
```

### 11.6 WebSocket

```ferrum
// Upgrade from HTTP handler
fn websocket_upgrade(req: Request, handler: fn(WebSocket): impl Future[Output=()])
    : Response  ! Net

struct WebSocket {
    fn recv(&mut self): impl Future[Output=Option[Result[Message, WsError]]]
    fn send(&mut self, msg: Message): impl Future[Output=Result[(), WsError]]
    fn close(&mut self, code: Option[CloseCode]): impl Future[Output=Result[(), WsError]]
    fn split(self): (WsSender, WsReceiver)
}

enum Message {
    Text(String),
    Binary(Vec[u8]),
    Ping(Vec[u8]),
    Pong(Vec[u8]),
    Close(Option[CloseFrame]),
}
```

---

## 9.8 async.poll — Readiness Polling

A unified polling abstraction for waiting on multiple async sources.

```ferrum
// Pollable — anything that can be polled for readiness
// This is an abstract capability, not a file descriptor
trait Pollable {
    /// Returns true if ready now (non-blocking check).
    fn ready(&self): bool

    /// Blocks until ready. Returns immediately if already ready.
    fn block(&self) ! Async

    /// Returns a token for use with poll_list.
    fn subscribe(&self): PollToken
}

// PollToken — opaque handle for polling
// Represents a "subscription" to a pollable event
struct PollToken {
    id: u64,
    pollable: &dyn Pollable,
}

/// Polls multiple pollables, returning indices of ready ones.
/// Blocks until at least one is ready.
fn poll_list(pollables: &[&dyn Pollable]): Vec[usize] ! Async
    requires !pollables.is_empty()
    ensures result.len() >= 1
    ensures result.iter().all(|i| *i < pollables.len())

/// Polls with timeout. Returns empty vec if timeout expires.
fn poll_list_timeout(
    pollables: &[&dyn Pollable],
    timeout: Duration,
): Vec[usize] ! Async
    requires !pollables.is_empty()

/// Non-blocking poll. Returns indices of currently ready pollables.
fn poll_list_ready(pollables: &[&dyn Pollable]): Vec[usize]

// Standard pollable implementations

// Timer pollable
struct TimerPollable {
    deadline: Instant,
}

impl TimerPollable {
    fn after(duration: Duration): Self {
        TimerPollable { deadline: Instant.now() + duration }
    }

    fn at(instant: Instant): Self {
        TimerPollable { deadline: instant }
    }
}

impl Pollable for TimerPollable {
    fn ready(&self): bool {
        Instant.now() >= self.deadline
    }

    fn block(&self) ! Async {
        // Platform-specific sleep
        sleep_until(self.deadline)
    }

    fn subscribe(&self): PollToken {
        // Return token for this timer
    }
}

// IO pollable wrapper
struct IoPollable[T: AsRawFd] {
    inner: T,
    interest: Interest,
}

enum Interest {
    Read,
    Write,
    ReadWrite,
}

impl[T: AsRawFd] IoPollable[T] {
    fn readable(inner: T): Self {
        IoPollable { inner, interest: Interest.Read }
    }

    fn writable(inner: T): Self {
        IoPollable { inner, interest: Interest.Write }
    }
}

impl[T: AsRawFd] Pollable for IoPollable[T] {
    fn ready(&self): bool {
        // Non-blocking check via poll(2) with zero timeout
    }

    fn block(&self) ! Async {
        // Wait for fd to be ready
    }

    fn subscribe(&self): PollToken
}

// Channel pollable
impl[T] Pollable for Receiver[T] {
    fn ready(&self): bool {
        !self.is_empty()
    }

    fn block(&self) ! Async {
        // Wait for message
    }

    fn subscribe(&self): PollToken
}

impl[T] Pollable for Sender[T] {
    fn ready(&self): bool {
        !self.is_full()
    }

    fn block(&self) ! Async {
        // Wait for capacity
    }

    fn subscribe(&self): PollToken
}
```

### 9.8.1 Poll Usage Examples

```ferrum
// Select first ready from multiple sources
fn handle_events(
    socket: &TcpStream,
    timer: &TimerPollable,
    shutdown: &Receiver[()],
) ! Async + IO {
    loop {
        let pollables: [&dyn Pollable; 3] = [
            &IoPollable.readable(socket),
            timer,
            shutdown,
        ]

        let ready = poll_list(&pollables)

        for idx in ready {
            match idx {
                0 => {
                    // Socket is readable
                    let data = socket.read()?
                    process(data)
                }
                1 => {
                    // Timer fired
                    send_heartbeat()
                    timer.reset()
                }
                2 => {
                    // Shutdown signal
                    return Ok(())
                }
                _ => unreachable!(),
            }
        }
    }
}

// Timeout wrapper
fn read_with_timeout(
    reader: &mut impl Read,
    buf: &mut [u8],
    timeout: Duration,
): Result[usize, TimeoutError] ! Async + IO {
    let io_poll = IoPollable.readable(reader)
    let timer = TimerPollable.after(timeout)

    let ready = poll_list(&[&io_poll, &timer])

    if ready.contains(&0) {
        Ok(reader.read(buf)?)
    } else {
        Err(TimeoutError)
    }
}

// Fan-in from multiple channels
fn merge_channels[T](
    receivers: &[Receiver[T]],
): impl Stream[Item = T] ! Async {
    stream! {
        let pollables: Vec[&dyn Pollable] = receivers.iter()
            .map(|r| r as &dyn Pollable)
            .collect()

        loop {
            if pollables.iter().all(|p| p.is_closed()) {
                break
            }

            let ready = poll_list(&pollables)

            for idx in ready {
                if let Ok(item) = receivers[idx].try_recv() {
                    yield item
                }
            }
        }
    }
}
```

### 9.8.2 Platform Mapping

```ferrum
// The poll model maps to platform primitives:
//
// | Platform | Primitive       |
// |----------|-----------------|
// | Linux    | epoll           |
// | macOS    | kqueue          |
// | Windows  | IOCP            |
// | WASI     | poll_list       |
//
// Key differences from raw epoll/kqueue:
// 1. Type-safe tokens (not raw integers)
// 2. Composable with any Pollable, not just fds
// 3. Works uniformly across platforms
// 4. Integrates with async/await

// Integration with async runtime
impl[T] Future for PollFuture[T] {
    type Output = T

    fn poll(self: Pin[&mut Self], cx: &mut Context): Poll[Self.Output] {
        if self.pollable.ready() {
            Poll.Ready(self.get_result())
        } else {
            // Register waker with runtime's reactor
            cx.reactor().register(self.pollable.subscribe(), cx.waker())
            Poll.Pending
        }
    }
}
```

---

## 12. net.lineproto — Line Protocol Framework

Declarative state machine engine for line-based protocols (SMTP, IMAP, POP3, FTP, Redis, memcached, IRC). The application defines handlers, not buffers. The library reads lines, parses verbs, enforces state, and dispatches.

This eliminates the buffer overflow vulnerability class by construction — the application never touches raw buffers.

```ferrum
// Protocol schema — declarative, not imperative
struct LineProtoSchema {
    commands:     Vec[CommandDef],
    states:       Vec[StateDef],
    max_line_len: usize,            // hard limit, default 8192
    line_ending:  LineEnding,       // Crlf, Lf, CrlfOrLf
    pipelining:   bool,             // allow multiple commands before response
    starttls:     Option[&str],     // command that triggers TLS upgrade
}

struct CommandDef {
    verb:           &str,           // e.g., "HELO", "MAIL", "RCPT"
    arg_type:       ArgType,        // None, Single, List, Remainder
    allowed_states: StateSet,       // which states allow this command
    handler:        fn(ctx: &mut ProtoContext, args: &str): ProtoResult,
}

enum ArgType { None, Single, List, Remainder }

struct StateDef {
    id:       u32,
    name:     &str,
    timeout:  Duration,
    greeting: Option[&str],         // sent on entering this state
    on_error: &str,                 // response on error in this state
}

// The protocol engine
struct LineProto[S: Send] {
    schema:  &LineProtoSchema,
    state:   u32,
    user:    S,                     // user-defined session state
}

impl[S: Send] LineProto[S] {
    fn new(schema: &LineProtoSchema, initial_state: u32, user: S): Self
    fn run(&mut self, stream: &mut impl BufReadWrite): Result[(), ProtoError] ! Net
    fn run_async(&mut self, stream: &mut impl AsyncBufReadWrite): impl Future[...] ! Net
}

// Example: minimal SMTP
const SMTP_SCHEMA: LineProtoSchema = LineProtoSchema {
    commands: [
        CommandDef { verb: "HELO", arg_type: Single, allowed_states: [0], handler: smtp_helo },
        CommandDef { verb: "EHLO", arg_type: Single, allowed_states: [0], handler: smtp_ehlo },
        CommandDef { verb: "MAIL", arg_type: Remainder, allowed_states: [1], handler: smtp_mail },
        CommandDef { verb: "RCPT", arg_type: Remainder, allowed_states: [2], handler: smtp_rcpt },
        CommandDef { verb: "DATA", arg_type: None, allowed_states: [2, 3], handler: smtp_data },
        CommandDef { verb: "QUIT", arg_type: None, allowed_states: [0, 1, 2, 3], handler: smtp_quit },
    ],
    states: [
        StateDef { id: 0, name: "init", timeout: 30s, greeting: Some("220 ready"), on_error: "500 error" },
        StateDef { id: 1, name: "greeted", timeout: 300s, greeting: None, on_error: "500 error" },
        // ...
    ],
    max_line_len: 1000,
    line_ending: LineEnding.CrlfOrLf,
    pipelining: true,
    starttls: Some("STARTTLS"),
}
```

---

## 13. net.dgram — Datagram Framework

Framework for datagram protocols with built-in amplification protection. Prevents DNS/NTP/SSDP amplification attack patterns through the API surface.

```ferrum
struct DatagramSchema {
    handlers:              Vec[DatagramHandler],
    max_dgram_size:        usize,          // drop oversized, default 65535
    max_response_multiple: f32,            // response <= N * request, default 1.0
    rate_limit_per_source: RateLimit,
    rate_limit_global:     RateLimit,
    source_validation:     SourceValidation,
}

struct DatagramHandler {
    opcode:       u8,                      // or range, or matcher
    handler:      fn(ctx: &mut DgramContext, req: &[u8]): Result[Vec[u8], DgramError],
}

// Amplification protection is enforced before calling handler
// If response exceeds max_response_multiple * request.len(), it is:
// 1. Truncated (if truncatable protocol)
// 2. Dropped with WARN log (if not truncatable)
// The handler cannot bypass this — it's enforced at the framework level

struct DgramServer {
    schema: &DatagramSchema,
    socket: UdpSocket,
}

impl DgramServer {
    fn bind(addr: SocketAddr, schema: &DatagramSchema): Result[Self, IoError] ! Net
    fn serve(&self): Result[never, IoError] ! Net + Async
}

// Client side: correlation, retransmission, timeout
struct DgramClient {
    socket:     UdpSocket,
    timeout:    Duration,
    max_retry:  u8,
    backoff:    BackoffStrategy,
}

impl DgramClient {
    fn request(&self, addr: SocketAddr, req: &[u8]): Result[Vec[u8], DgramError] ! Net
    fn request_async(&self, addr: SocketAddr, req: &[u8]): impl Future[...] ! Net
}

enum BackoffStrategy {
    Fixed(Duration),
    Exponential { initial: Duration, max: Duration, factor: f32 },
}

// Rate limiting
struct RateLimit {
    requests: u32,
    per:      Duration,
}

enum SourceValidation {
    None,                          // trust all sources
    Cookie,                        // require cookie exchange (like DTLS)
    KnownOnly(HashSet[IpAddr]),    // whitelist
}
```

---

## 14. data.cbor — CBOR Binary Serialization

Concise Binary Object Representation (RFC 8949). Binary serialization format used by COSE, CWT, FIDO2/WebAuthn, CoAP. Self-describing like JSON but binary. No schema compiler or code generation required.

```ferrum
// DOM style — parse to value tree
fn parse(bytes: &[u8]): Result[CborValue, CborError]
fn to_bytes(value: &CborValue): Vec[u8]

enum CborValue {
    Unsigned(u64),
    Negative(i64),
    Bytes(Vec[u8]),
    Text(String),
    Array(Vec[CborValue]),
    Map(Vec[(CborValue, CborValue)]),
    Tag(u64, Box[CborValue]),
    Float(f64),
    Bool(bool),
    Null,
    Undefined,
}

// Streaming encoder
struct CborEncoder[W: Write] {
    fn new(writer: W): Self
    fn write_u64(&mut self, v: u64): Result[(), IoError]
    fn write_i64(&mut self, v: i64): Result[(), IoError]
    fn write_bytes(&mut self, v: &[u8]): Result[(), IoError]
    fn write_text(&mut self, v: &str): Result[(), IoError]
    fn write_array_header(&mut self, len: usize): Result[(), IoError]
    fn write_map_header(&mut self, len: usize): Result[(), IoError]
    fn write_tag(&mut self, tag: u64): Result[(), IoError]
    fn write_bool(&mut self, v: bool): Result[(), IoError]
    fn write_null(&mut self): Result[(), IoError]
    fn write_float(&mut self, v: f64): Result[(), IoError]
}

// Serde integration
fn from_slice[T: Deserialize](bytes: &[u8]): Result[T, CborError]
fn to_vec[T: Serialize](value: &T): Result[Vec[u8], CborError]

// CBOR sequences (RFC 8742)
fn parse_sequence(bytes: &[u8]): Result[Vec[CborValue], CborError]
```

---

*End of async, net, http modules — see [ferrum-stdlib.md](ferrum-stdlib.md) for index.*
