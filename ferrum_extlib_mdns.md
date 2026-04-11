# Ferrum Extended Library — mDNS / DNS-SD

**Module:** `extlib.mdns`
**RFC references:** RFC 6762 (Multicast DNS), RFC 6763 (DNS-Based Service Discovery)
**Status:** Extended library — not part of stdlib core

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [RFC References](#2-rfc-references)
3. [MdnsResponder — Core Type](#3-mdnsresponder--core-type)
4. [Name Announcement and Service Registration](#4-name-announcement-and-service-registration)
5. [Service Discovery — DNS-SD](#5-service-discovery--dns-sd)
6. [Host Resolution](#6-host-resolution)
7. [Conflict Resolution](#7-conflict-resolution)
8. [Goodbye Packets and Deregistration](#8-goodbye-packets-and-deregistration)
9. [Multicast Group Management](#9-multicast-group-management)
10. [Integration with ferrum_extlib_dns_secure](#10-integration-with-ferrum_extlib_dns_secure)
11. [Error Types](#11-error-types)
12. [Example Usage](#12-example-usage)
13. [Dependencies](#13-dependencies)

---

## 1. Overview and Rationale

Multicast DNS (mDNS) enables hostname resolution and service discovery on a local network
segment without a central DNS server. A device can announce its own name and services,
and other devices can discover them, using only IP multicast. No configuration, no
administrator, no DHCP-assigned DNS — it just works on any LAN or directly-connected
network.

DNS-Based Service Discovery (DNS-SD) builds on mDNS to publish structured service
records: a printer, an SSH server, an IoT sensor cluster, or a development toolchain.
Clients browse for services by type and receive live updates as services appear and
disappear.

**Why this belongs in an extended library, not the stdlib:**

- Not all programs need local network service discovery. A CLI tool, a batch processor,
  a cryptographic library — none of these touch mDNS. Pulling socket-joining and
  multicast management into the standard library imposes cost on programs that will
  never use it.
- Platform coverage varies. mDNS requires multicast UDP on a network interface.
  Embedded targets without a network stack, WASI sandboxes, and some high-security
  environments cannot support it at all. An extended library can declare its platform
  requirements explicitly.
- The protocol involves persistent background state (a running responder, joined
  multicast groups, timers for probe and announce cycles). This is qualitatively
  different from a one-shot name lookup and does not fit the stdlib's model of
  stateless utilities.

**Primary use cases:**

- IoT and embedded devices announcing their presence on a local network
- Development tooling (printers, development servers, debuggers) auto-discovered
  without manual configuration
- Home automation hubs discovering sensors and actuators
- Local peer-to-peer applications that do not require a broker

---

## 2. RFC References

| RFC | Title | Role in this module |
|-----|-------|---------------------|
| RFC 6762 | Multicast DNS | Core protocol: packet format, probe/announce cycles, conflict resolution, TTL, goodbye records, caching |
| RFC 6763 | DNS-Based Service Discovery | Service type naming, PTR/SRV/TXT record structure, browsing, subtypes |
| RFC 4795 | Link-Local Multicast Name Resolution (LLMNR) | Informative only — a Microsoft alternative, not implemented here |
| RFC 8305 | Happy Eyeballs v2 | Informs dual-stack address selection during resolution |

**Key protocol points from RFC 6762:**

- mDNS uses UDP port 5353 on multicast addresses `224.0.0.251` (IPv4) and `FF02::FB`
  (IPv6).
- Records in the `.local` domain are resolved via mDNS, not forwarded to upstream DNS.
- TTL of zero signals record withdrawal (goodbye packet).
- Responses may be sent unicast when the query includes the QU (unicast-response) bit
  and the responding host's address is known to the querier.
- The probe phase (RFC 6762 §8) must complete before a name is announced, to detect
  conflicts on the network.

**Key protocol points from RFC 6763:**

- Service types follow the form `_service._proto.local` (e.g., `_http._tcp.local`).
- A PTR record under the service type points to the instance name.
- An SRV record under the instance name gives the host and port.
- A TXT record under the instance name carries key=value metadata.
- Subtypes allow fine-grained browsing: `_printer._sub._ipp._tcp.local`.

---

## 3. MdnsResponder — Core Type

The `MdnsResponder` is the central object. It owns the multicast sockets, runs the
probe/announce state machines, responds to incoming queries, and manages the registry
of announced names and services.

```ferrum
use extlib.mdns {
    MdnsResponder, MdnsConfig, MdnsError,
    ConflictPolicy, NetworkInterface,
}

/// The mDNS responder. Owns all multicast sockets and runs the
/// RFC 6762 state machine for all registered names and services.
///
/// Drop to shut down: sends goodbye packets for all registrations,
/// leaves multicast groups, and closes sockets.
type MdnsResponder

impl MdnsResponder {
    /// Create and start a responder.
    ///
    /// Binds to 224.0.0.251:5353 (IPv4) and [FF02::FB]:5353 (IPv6)
    /// on each interface in `config.interfaces`. Joins the mDNS
    /// multicast group on each interface. Spawns background tasks
    /// for packet receive/transmit loops.
    pub fn new(config: MdnsConfig): Result[Self, MdnsError] ! Async + Net
}

impl Drop for MdnsResponder {
    fn drop(&mut self) {
        // Send goodbye packets (TTL=0) for all live registrations.
        // Leave multicast groups. Close sockets.
        // Background tasks are cancelled via their runtime handles.
    }
}
```

### 3.1 MdnsConfig

```ferrum
/// Configuration for an MdnsResponder.
type MdnsConfig {
    /// Network interfaces to bind and join multicast groups on.
    /// Pass an empty Vec to bind on all available interfaces.
    interfaces: Vec[NetworkInterface],

    /// The local hostname this responder will own.
    /// Announced as `<responder_name>.local` via an A/AAAA record.
    /// Must not include `.local` — the module appends it.
    /// Example: "my-device"
    responder_name: String,

    /// How to handle a naming conflict discovered during probing.
    conflict_resolution: ConflictPolicy,

    /// How long to wait for probe responses before announcing.
    /// RFC 6762 §8 requires at least 250 ms per probe interval.
    /// Default: 250 ms.
    probe_interval: Duration,

    /// Number of probe transmissions before considering a name safe.
    /// RFC 6762 §8 requires at least 3 probes.
    /// Default: 3.
    probe_count: u8,

    /// Number of times to repeat the initial announcement.
    /// RFC 6762 §8 recommends repeating the announcement 2-8 times.
    /// Default: 3.
    announce_count: u8,
}

impl MdnsConfig {
    /// Sensible defaults for a single-interface desktop or embedded device.
    pub fn default(responder_name: &str): Self
}

/// A network interface to operate on.
/// Obtain from `net.interface.list()` or `net.interface.by_name()`.
type NetworkInterface  // re-exported from net stdlib
```

### 3.2 ConflictPolicy

```ferrum
/// What to do when the probe phase discovers another host has claimed
/// the same name (RFC 6762 §9).
enum ConflictPolicy {
    /// Append a numeric suffix and retry. "my-device" becomes
    /// "my-device-2", then "my-device-3", up to the retry limit.
    RenameWithSuffix { max_retries: u8 },

    /// Return `MdnsError.ConflictUnresolvable` from the operation
    /// that triggered the conflict. The caller must handle it.
    Fail,
}
```

---

## 4. Name Announcement and Service Registration

### 4.1 Host Name Announcement

```ferrum
impl MdnsResponder {
    /// Announce `hostname.local` mapping to the given addresses.
    ///
    /// Runs the RFC 6762 §8 probe cycle for `hostname.local`, then
    /// sends the initial announcement. After this call returns, the
    /// responder will answer A/AAAA queries for `hostname.local`
    /// for the lifetime of the responder.
    ///
    /// `hostname` must not contain `.` or `.local`.
    /// `addrs` must not be empty.
    ///
    /// If `config.conflict_resolution` is `RenameWithSuffix`, the
    /// actual name claimed (after suffix addition) is returned.
    pub fn announce_host(
        &mut self,
        hostname: &str,
        addrs: &[IpAddr],
    ): Result[String, MdnsError] ! Async + Net

    /// Remove a host announcement. Sends a goodbye packet (TTL=0)
    /// for `hostname.local` and removes the record from the registry.
    pub fn withdraw_host(&mut self, hostname: &str): Result[(), MdnsError] ! Async + Net
}
```

### 4.2 Service Registration

```ferrum
impl MdnsResponder {
    /// Register a DNS-SD service instance.
    ///
    /// Creates PTR, SRV, and TXT records per RFC 6763:
    ///   PTR  _service._tcp.local  ->  "Instance Name._service._tcp.local"
    ///   SRV  "Instance Name._service._tcp.local"  ->  host:port
    ///   TXT  "Instance Name._service._tcp.local"  ->  key=value pairs
    ///
    /// Also registers any subtypes:
    ///   PTR  _subtype._sub._service._tcp.local  ->  (same instance)
    ///
    /// Runs a probe cycle for the instance name before announcing.
    /// Returns a `ServiceHandle` that keeps the registration alive.
    /// Drop the handle (or call `deregister`) to send goodbye packets.
    pub fn register_service(
        &mut self,
        service: ServiceRecord,
    ): Result[ServiceHandle, MdnsError] ! Async + Net
}

/// Describes a service to register.
type ServiceRecord {
    /// Service type in DNS-SD form: "_http._tcp" or "_printer._tcp".
    /// Must not include ".local" — the module appends it.
    service_type: String,

    /// Human-readable instance name. May contain spaces and Unicode.
    /// Example: "My HP LaserJet"
    instance_name: String,

    /// Port the service listens on.
    port: u16,

    /// TXT record key=value metadata (RFC 6763 §6).
    txt: TxtRecord,

    /// Optional DNS-SD subtypes for fine-grained browsing.
    /// Example: ["_color", "_duplex"]
    /// Each becomes a PTR record under `_subtype._sub._service._proto.local`.
    subtypes: Vec[String],
}

impl ServiceRecord {
    pub fn new(service_type: &str, instance_name: &str, port: u16): Self
    pub fn with_txt(self, txt: TxtRecord): Self
    pub fn with_subtype(self, subtype: &str): Self
}

/// A live service registration. The service remains announced for as
/// long as this handle is alive. Drop to send goodbye packets.
type ServiceHandle {
    /// The full DNS instance name as announced on the network.
    /// Example: "My HP LaserJet._printer._tcp.local"
    pub instance_fqdn: String,

    /// The service type as announced.
    /// Example: "_printer._tcp.local"
    pub service_type_fqdn: String,
}

impl ServiceHandle {
    /// Explicitly deregister this service. Sends goodbye packets
    /// immediately. Equivalent to dropping the handle, but returns
    /// an error if the goodbye transmission fails.
    pub fn deregister(self): Result[(), MdnsError] ! Async + Net

    /// Update the TXT record for this service without re-probing.
    /// The new TXT record is announced immediately.
    pub fn update_txt(&mut self, txt: TxtRecord): Result[(), MdnsError] ! Async + Net
}

impl Drop for ServiceHandle {
    fn drop(&mut self) {
        // Best-effort goodbye packet transmission.
        // Errors are logged but not propagated — callers that need
        // reliable deregistration should call deregister() explicitly.
    }
}
```

### 4.3 TxtRecord

```ferrum
/// A DNS-SD TXT record (RFC 6763 §6).
///
/// Carries key=value pairs that describe service attributes.
/// Keys are case-insensitive ASCII. Values are arbitrary bytes.
/// The wire encoding is a sequence of length-prefixed strings:
///   "key=value" or "key" (boolean present/absent flag).
///
/// Maximum wire size: 8900 bytes (one DNS message worth).
type TxtRecord {
    entries: Vec[TxtEntry],
}

enum TxtEntry {
    /// key=value pair. Value is arbitrary bytes (may not be UTF-8).
    KeyValue { key: String, value: Vec[u8] },
    /// Boolean flag — key is present, no value.
    Flag(String),
}

impl TxtRecord {
    pub fn new(): Self
    pub fn empty(): Self    // single empty string on wire, per RFC 6763 §6.1

    /// Insert a key=value pair.
    /// Returns Err if the key is invalid (empty, or contains '=').
    pub fn insert(&mut self, key: &str, value: impl Into[Vec[u8]]): Result[(), MdnsError]

    /// Insert a boolean flag (key present, no value).
    pub fn insert_flag(&mut self, key: &str): Result[(), MdnsError]

    /// Look up a value by key (case-insensitive).
    pub fn get(&self, key: &str): Option[&[u8]]

    /// True if the flag key is present (regardless of value).
    pub fn has(&self, key: &str): bool

    /// Iterate all entries.
    pub fn iter(&self): impl Iterator[Item = &TxtEntry]

    /// Encode to wire format. Returns Err if total size exceeds limit.
    pub fn to_wire(&self): Result[Vec[u8], MdnsError]

    /// Decode from wire format.
    pub fn from_wire(bytes: &[u8]): Result[Self, MdnsError]
}
```

---

## 5. Service Discovery — DNS-SD

Service discovery is separate from the responder. A program can browse for services
without running its own responder. Browse functions take a `BrowseConfig` that specifies
which interface(s) to listen on.

```ferrum
use extlib.mdns { browse, BrowseConfig, ServiceBrowser, ServiceEvent,
                  DiscoveredService, ResolvedService }

/// Configuration for browsing/resolving (no announcement).
type BrowseConfig {
    /// Interfaces to listen on. Empty = all interfaces.
    interfaces: Vec[NetworkInterface],
    /// How long to wait for initial responses before the first item
    /// appears in the stream. Does not bound the stream's lifetime.
    initial_wait: Duration,
}

impl BrowseConfig {
    pub fn default(): Self
}
```

### 5.1 Browsing for Services

```ferrum
/// Browse for services of a given type.
///
/// `service_type` is in the form `_http._tcp` or `_printer._tcp`.
/// Does not include `.local`.
///
/// Returns an async stream that yields `ServiceEvent` items as
/// services appear, disappear, or are resolved. The stream runs
/// until dropped.
///
/// Internally sends PTR queries to the mDNS multicast group and
/// listens for responses, keeping a local cache. Cache entries
/// are expired when their TTL elapses or a goodbye packet arrives.
pub fn browse(
    service_type: &str,
    config: BrowseConfig,
): ServiceBrowser ! Async + Net

/// An async stream of service events. Implements `Stream`.
/// Drop to stop listening and leave multicast groups.
type ServiceBrowser

impl Stream for ServiceBrowser {
    type Item = ServiceEvent
}

impl ServiceBrowser {
    /// Force resolution of a discovered service, blocking until
    /// SRV and TXT records are obtained or the timeout elapses.
    pub fn resolve(
        &mut self,
        instance: &DiscoveredService,
        timeout: Duration,
    ): impl Future[Output = Result[ResolvedService, MdnsError]] ! Async + Net
}
```

### 5.2 ServiceEvent

```ferrum
/// Events produced by a ServiceBrowser stream.
enum ServiceEvent {
    /// A new instance was discovered (PTR record received).
    /// The instance name and service type are known; host/port/TXT
    /// have not yet been fetched.
    Added(DiscoveredService),

    /// An instance was withdrawn (goodbye packet received or TTL expired).
    Removed(DiscoveredService),

    /// An instance's SRV/TXT records have been fetched.
    /// This event is emitted after an explicit `resolve()` call or
    /// after the browser auto-resolves following an `Added` event
    /// (configurable via `BrowseConfig.auto_resolve`).
    Resolved(ResolvedService),
}

/// An mDNS service instance whose existence is known but whose
/// host/port/TXT records have not yet been fetched.
type DiscoveredService {
    /// DNS-SD instance name. Example: "My HP LaserJet"
    pub instance_name: String,
    /// Service type. Example: "_printer._tcp"
    pub service_type: String,
    /// Full FQDN. Example: "My HP LaserJet._printer._tcp.local"
    pub instance_fqdn: String,
    /// The interface this was discovered on.
    pub interface: NetworkInterface,
}

/// A fully resolved mDNS service instance.
type ResolvedService {
    /// The discovered service record this was resolved from.
    pub discovered: DiscoveredService,
    /// Hostname of the host providing the service. Example: "printserver.local"
    pub host: String,
    /// Port the service listens on.
    pub port: u16,
    /// All addresses the host announced. May include both IPv4 and IPv6.
    pub addrs: Vec[IpAddr],
    /// TXT record metadata.
    pub txt: TxtRecord,
    /// Time remaining before re-resolution is needed (from SRV record TTL).
    pub ttl: Duration,
}
```

### 5.3 Standalone Resolution

```ferrum
/// Resolve a specific service instance without browsing.
///
/// Sends a targeted query for the SRV and TXT records of the named
/// instance, then resolves the host's A/AAAA records via mDNS.
/// Returns the fully resolved service or an error if resolution
/// does not complete within `timeout`.
pub fn resolve_service(
    instance_fqdn: &str,
    timeout: Duration,
    config: BrowseConfig,
): Result[ResolvedService, MdnsError] ! Async + Net
```

---

## 6. Host Resolution

### 6.1 Resolving .local Hostnames

```ferrum
/// Resolve a `.local` hostname to one or more IP addresses via mDNS.
///
/// `hostname` must end in `.local`. Sends an mDNS query for A and
/// AAAA records and collects all responses received within `timeout`.
///
/// Returns at least one address on success. Returns
/// `MdnsError.Timeout` if no response arrives within `timeout`.
///
/// By default, sends the query with the QU (unicast-response) bit set
/// on the first attempt, falling back to multicast if no response
/// arrives within 400 ms. This matches the RFC 6762 §5.4 recommendation.
pub fn resolve_host(
    hostname: &str,
    timeout: Duration,
    config: BrowseConfig,
): Result[Vec[IpAddr], MdnsError] ! Async + Net
```

### 6.2 Timeout Handling

The `resolve_host` function uses a two-phase strategy matching RFC 6762 §5.4:

1. **Unicast phase** (first 400 ms): sends a query with the QU bit set. If the
   destination has recently seen a multicast response from this host, it replies
   unicast, reducing multicast traffic.
2. **Multicast phase**: if no unicast response arrives, resends as a standard
   multicast query. Collects all responses until `timeout` elapses.

Callers requiring only the first address can use:

```ferrum
pub fn resolve_host_first(
    hostname: &str,
    timeout: Duration,
    config: BrowseConfig,
): Result[IpAddr, MdnsError] ! Async + Net
```

This returns as soon as any single address is received, which is appropriate for
latency-sensitive applications such as connecting to a known device.

---

## 7. Conflict Resolution

RFC 6762 §9 defines the probe-then-announce cycle that prevents two hosts from
claiming the same name.

**Probe phase:**

Before announcing a name, the responder sends three probe queries at 250 ms intervals,
asking whether any host is already using the name. If a response arrives during the
probe window, a conflict is detected.

**Conflict handling:**

The behavior on conflict is controlled by `MdnsConfig.conflict_resolution`:

- `ConflictPolicy.RenameWithSuffix`: a numeric suffix is appended and the probe
  restarts with the new name. "my-device" → "my-device-2" → "my-device-3", up to
  `max_retries` attempts.
- `ConflictPolicy.Fail`: the operation returns `MdnsError.ConflictUnresolvable`
  immediately. The caller decides how to proceed.

**Simultaneous probe tiebreaking (RFC 6762 §8.2):**

If two hosts probe for the same name simultaneously, the probe packets carry the
records the host intends to announce. The tiebreaker compares the records
lexicographically. The host with the lexicographically later records wins and
continues probing; the loser backs off and retries (if `RenameWithSuffix`) or fails.

```ferrum
// Conflict policy is specified at responder construction time.
// Individual registration calls inherit it.

let config = MdnsConfig {
    responder_name: "my-sensor".to_string(),
    conflict_resolution: ConflictPolicy.RenameWithSuffix { max_retries: 5 },
    ..MdnsConfig.default("my-sensor")
}

let mut responder = MdnsResponder.new(config).await?

// If "my-sensor.local" is taken, the responder will try
// "my-sensor-2.local", "my-sensor-3.local", etc.
// The returned String is the name actually claimed.
let claimed_name = responder.announce_host("my-sensor", &[my_ip]).await?
println("Announced as: {claimed_name}.local")
```

---

## 8. Goodbye Packets and Deregistration

RFC 6762 §11 specifies that a host SHOULD send a goodbye packet — a DNS response
with TTL of zero — when it withdraws a record. This allows other hosts to expire the
record immediately rather than waiting for TTL expiry.

**Automatic goodbye on drop:**

```ferrum
{
    let handle = responder.register_service(my_service).await?
    // ... service is live ...
} // handle drops here — goodbye packets sent automatically
```

`Drop` for `ServiceHandle` sends goodbye packets on a best-effort basis. If the
goodbye transmission fails (e.g., the network interface was removed), the error is
logged but not propagated, because `Drop` cannot return errors.

**Explicit deregistration:**

```ferrum
// Prefer explicit deregister when you need to know it succeeded.
handle.deregister().await?
```

`deregister` consumes the handle and returns `Result[(), MdnsError]`. After this
call, the responder no longer answers queries for the service.

**Host goodbye:**

```ferrum
responder.withdraw_host("my-sensor").await?
```

This sends a goodbye (TTL=0) for the A/AAAA records of `my-sensor.local`.

**Responder shutdown:**

Dropping the `MdnsResponder` itself sends goodbye packets for all live host
announcements and service registrations, leaves all multicast groups, and shuts down
the background tasks. As with `ServiceHandle`, errors during shutdown are logged but
not propagated.

---

## 9. Multicast Group Management

The responder joins `224.0.0.251` (IPv4) and `FF02::FB` (IPv6) on each configured
interface using `IP_ADD_MEMBERSHIP` and `IPV6_ADD_MEMBERSHIP`. These are joined at
construction time and left when the responder is dropped.

**Source-specific multicast (SSM):**

On platforms that support `IP_ADD_SOURCE_MEMBERSHIP` (Linux, FreeBSD, macOS), the
responder can optionally use SSM to reduce processing of off-link mDNS traffic when
running in a multi-segment environment. SSM is disabled by default; enable via
`MdnsConfig.use_source_specific_multicast`.

**Interface binding:**

Each interface gets its own socket pair (one for IPv4, one for IPv6). On platforms
where `SO_REUSEPORT` is available, multiple processes can run mDNS responders on the
same host (e.g., a system daemon and a user application). The module sets
`IP_MULTICAST_LOOP` so that the responder receives its own transmissions (required
for correct conflict detection).

**Interface change detection:**

When `MdnsConfig.interfaces` is empty (listen on all interfaces), the responder
monitors interface addition and removal events via the platform network event system.
New interfaces are joined automatically. Removed interfaces trigger withdrawal of
any host or service records that were announced only on that interface.

---

## 10. Integration with ferrum_extlib_dns_secure

When the `ferrum_extlib_dns_secure` module's `SecureResolver` is configured with
`mdns_enabled: true`, queries for `.local` names are automatically routed to the
mDNS subsystem rather than forwarded to upstream DNS servers.

```ferrum
use extlib.dns_secure { SecureResolver, SecureResolverConfig, MdnsBackend }
use extlib.mdns { MdnsResponder, MdnsConfig, BrowseConfig }

// Build an mDNS-capable secure resolver.
let resolver = SecureResolver.new(SecureResolverConfig {
    dnssec: DnssecMode.Require,
    transport: DnsTransport.DoT(my_tls_config),
    mdns: Some(MdnsBackend {
        // Provide a browse config. The SecureResolver spawns
        // an internal responder/listener for .local resolution.
        browse_config: BrowseConfig.default(),
        // If true, the SecureResolver also answers .local queries
        // using the responder's own registry (useful when both
        // announcing and resolving on the same host).
        use_local_registry: true,
    }),
    ..SecureResolverConfig.default()
}).await?

// Now .local names are resolved via mDNS; everything else uses DoT + DNSSEC.
let addrs = resolver.lookup_ip("printserver.local").await?
let web   = resolver.lookup_ip("example.com").await?   // goes to upstream
```

**Routing rules:**

- Any query whose QNAME ends in `.local` (case-insensitive) is sent to the mDNS
  backend. DNSSEC validation is not applied to mDNS responses — the local-network
  trust model applies instead.
- All other queries go to the configured upstream resolvers with DNSSEC validation
  as configured.
- If the mDNS backend returns `MdnsError.Timeout`, the `SecureResolver` returns
  `ResolveError.NxDomain` (the name does not exist on the local network).

---

## 11. Error Types

```ferrum
/// Errors produced by the mDNS module.
enum MdnsError {
    /// An underlying socket operation failed.
    SocketError(IoError),

    /// A name conflict was detected during probing and could not be
    /// resolved automatically (ConflictPolicy.Fail, or RenameWithSuffix
    /// exhausted its retry limit).
    ConflictUnresolvable {
        /// The name that could not be claimed.
        attempted_name: String,
        /// How many suffixes were tried (0 if policy is Fail).
        retries_attempted: u8,
    },

    /// A resolution or browse operation did not complete within
    /// the specified timeout.
    Timeout { duration: Duration },

    /// The provided hostname or service type was malformed.
    /// Includes a description of which rule was violated.
    InvalidName { name: String, reason: String },

    /// An operation on a specific network interface failed.
    InterfaceError { interface: String, cause: IoError },

    /// A TXT record key was invalid (empty string, or contains '=').
    InvalidTxtKey(String),

    /// The TXT record exceeded the maximum wire size.
    TxtRecordTooLarge { size: usize, limit: usize },

    /// The wire-format packet was malformed and could not be parsed.
    MalformedPacket(String),
}

impl Display for MdnsError { ... }
impl Error for MdnsError { ... }
```

---

## 12. Example Usage

### 12.1 Registering an HTTP Service

```ferrum
use extlib.mdns { MdnsResponder, MdnsConfig, ServiceRecord, TxtRecord, ConflictPolicy }
use net { IpAddr, Ipv4Addr }

async fn run_server(port: u16) ! Async + Net + IO {
    // Build a TXT record with HTTP metadata.
    let mut txt = TxtRecord.new()
    txt.insert("path", "/api/v1").unwrap()
    txt.insert("version", "2.4").unwrap()
    txt.insert_flag("tls").unwrap()

    // Construct the service record.
    let svc = ServiceRecord.new("_http._tcp", "My API Server", port)
        .with_txt(txt)
        .with_subtype("_api")

    // Start the responder. It will announce "api-server.local" and
    // the _http._tcp service.
    let config = MdnsConfig.default("api-server")
    let mut responder = MdnsResponder.new(config).await?

    // Announce our address. Return value is the actual name claimed
    // (may differ if conflict resolution appended a suffix).
    let my_addr = IpAddr.V4(Ipv4Addr.new(192, 168, 1, 42))
    let name = responder.announce_host("api-server", &[my_addr]).await?
    println("Announced as {name}.local")

    // Register the service. Keep the handle alive for the lifetime
    // of the server.
    let _handle = responder.register_service(svc).await?
    println("Service registered as: {_handle.instance_fqdn}")

    // Run the server. When this function returns, _handle and responder
    // are both dropped, sending goodbye packets automatically.
    my_http_server_loop(port).await
}
```

### 12.2 Browsing for Printers

```ferrum
use extlib.mdns { browse, BrowseConfig, ServiceEvent }

async fn find_printers() ! Async + Net + IO {
    let config = BrowseConfig.default()
    let mut browser = browse("_ipp._tcp", config)

    // Collect events for 5 seconds.
    let deadline = Instant.now() + Duration.from_secs(5)

    loop {
        match timeout(deadline - Instant.now(), browser.next()).await {
            Ok(Some(event)) => {
                match event {
                    ServiceEvent.Added(svc) => {
                        println("Found printer: {svc.instance_name}")
                        // Resolve it to get host/port/TXT.
                        match browser.resolve(&svc, Duration.from_secs(2)).await {
                            Ok(resolved) => {
                                println("  host: {resolved.host}:{resolved.port}")
                                println("  addrs: {resolved.addrs:?}")
                                if let Some(pdl) = resolved.txt.get("pdl") {
                                    println("  PDL: {pdl:?}")
                                }
                            }
                            Err(e) => println("  (could not resolve: {e})")
                        }
                    }
                    ServiceEvent.Removed(svc) => {
                        println("Printer gone: {svc.instance_name}")
                    }
                    ServiceEvent.Resolved(_) => {}
                }
            }
            Ok(None) => break,       // browser stream ended
            Err(_elapsed) => break,  // 5 second window elapsed
        }
    }
}
```

### 12.3 Resolving a .local Host

```ferrum
use extlib.mdns { resolve_host, BrowseConfig }

async fn connect_to_device(hostname: &str) ! Async + Net {
    // hostname should end in ".local"
    let addrs = resolve_host(
        hostname,
        Duration.from_secs(3),
        BrowseConfig.default(),
    ).await?

    println("Resolved {hostname} to:")
    for addr in &addrs {
        println("  {addr}")
    }

    // Connect to the first address returned.
    let socket_addr = SocketAddr.new(addrs[0], 8080)
    let conn = TcpStream.connect(socket_addr)?
    // ... use conn ...
    Ok(())
}
```

### 12.4 Conflict Resolution Example

```ferrum
use extlib.mdns { MdnsResponder, MdnsConfig, ConflictPolicy, MdnsError }

async fn start_with_conflict_handling() ! Async + Net + IO {
    let config = MdnsConfig {
        responder_name: "my-device".to_string(),
        conflict_resolution: ConflictPolicy.RenameWithSuffix { max_retries: 10 },
        ..MdnsConfig.default("my-device")
    }

    let mut responder = MdnsResponder.new(config).await?

    match responder.announce_host("my-device", &[my_addr]).await {
        Ok(name) => println("Running as {name}.local"),
        Err(MdnsError.ConflictUnresolvable { attempted_name, retries_attempted }) => {
            eprintln("Could not claim {attempted_name} after {retries_attempted} retries")
            return Err(MdnsError.ConflictUnresolvable { attempted_name, retries_attempted })
        }
        Err(e) => return Err(e),
    }
}
```

---

## 13. Dependencies

### Required

| Dependency | Role |
|------------|------|
| `net` (Ferrum stdlib) | `UdpSocket`, `IpAddr`, `SocketAddr`, `NetworkInterface`, multicast socket options |
| `async` (Ferrum stdlib) | `Runtime`, `Future`, `Stream`, timers (`sleep`, `timeout`, `interval`) |
| `collections` (Ferrum stdlib) | `Vec`, `HashMap` for record registry and TXT entries |
| `time` (Ferrum stdlib) | `Duration`, `Instant` for TTL tracking and probe timers |

### Optional

| Dependency | Role |
|------------|------|
| `extlib.dns_secure` | `.local` routing integration in `SecureResolver` (§10). Without this, mDNS and the secure resolver operate independently. |

### Platform Requirements

mDNS requires:

- UDP multicast send and receive (`IP_MULTICAST_IF`, `IP_ADD_MEMBERSHIP`,
  `IPV6_ADD_MEMBERSHIP`).
- `SO_REUSEPORT` or `SO_REUSEADDR` to share port 5353 with other responders.
- Loopback of multicast (`IP_MULTICAST_LOOP`).

**Supported platforms:** Linux, macOS, FreeBSD, OpenBSD, Windows (via WSA multicast).

**Unsupported:** WASI (no multicast), bare-metal targets without a network stack.
On unsupported platforms, `MdnsResponder.new` returns `MdnsError.InterfaceError`
explaining the missing capability.

---

*See also:*
- *[ferrum-stdlib-async-net.md](ferrum-stdlib-async-net.md) — `UdpSocket`, `Resolver`, `async` runtime*
- *[ferrum-introduction-to-effects.md](ferrum-introduction-to-effects.md) — `! Async + Net` effect notation*
- *RFC 6762 — Multicast DNS*
- *RFC 6763 — DNS-Based Service Discovery*
