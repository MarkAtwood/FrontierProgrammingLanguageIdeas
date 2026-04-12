# Ferrum Extended Library — Secure DNS Resolver

**Module path:** `ferrum_extlib_dns_secure`
**Part of:** Ferrum Extended Standard Library

---

## 1. Overview and Rationale

### What the stdlib resolver provides

The Ferrum standard library's `net.Resolver` covers the common case: forward
and reverse lookups, MX records for mail routing, TXT records, basic SRV and
TLSA access. Its transport is UDP with TCP fallback on truncation. It is
sufficient for application code that delegates security to the OS stub resolver
or a local recursive resolver.

What it deliberately omits:

- **DNS-over-TLS (DoT, RFC 7858):** Encrypts queries to the nameserver.
  Without it, queries are visible to any on-path observer and susceptible to
  injection attacks.
- **DNS-over-HTTPS (DoH, RFC 8484):** Carries DNS over HTTPS, enabling
  resolver use through HTTP infrastructure and further obscuring query content.
- **DNSSEC validation (RFC 4033–4035):** Cryptographically authenticates DNS
  responses. Without it, a resolver cannot distinguish a legitimate answer from
  a forged one, even over a secure transport.
- **DANE/TLSA enforcement (RFC 6698):** Binds TLS certificates to DNS names
  using DNSSEC-authenticated TLSA records. Allows clients to verify server
  certificates without depending solely on the WebPKI.
- **Extended record types:** SVCB (RFC 9460), HTTPS (a SVCB subtype), NAPTR
  (RFC 3403), CAA (RFC 8659), DS, RRSIG, NSEC, NSEC3, DNSKEY — all required
  for DNSSEC chains, service discovery, and modern protocol negotiation.

### Why this is an extended library, not stdlib

The stdlib resolver has no external dependencies beyond the OS socket API.
Secure DNS requires:

- A TLS stack (`ferrum_extlib_tls`) for DoT and DNSSEC signature verification.
- An HTTP client (`ferrum_extlib_http`) for DoH transport.
- Cryptographic primitives for DNSSEC chain validation (RSA, ECDSA, Ed25519
  over SHA-256 and SHA-384).

Pulling these into stdlib would make the stdlib non-embeddable on constrained
targets and would create a dependency cycle with the TLS stack (which itself
uses DNS). The split is the correct architectural boundary.

### Security model

Encryption alone (DoT/DoH) protects privacy: a passive observer cannot read
your queries. It does not prevent an active attacker from forging responses if
the nameserver itself is compromised or if the encrypted channel terminates at
an untrusted resolver.

DNSSEC validation provides data integrity: the cryptographic chain from the
DNS root to the leaf record proves the answer is what the zone owner published.
It does not encrypt anything.

DANE combines both: DNSSEC-authenticated TLSA records let a TLS client verify
a server certificate without the WebPKI, or use DNSSEC to constrain which
WebPKI CAs may issue for a name.

Applications that need all three properties — privacy, integrity, and
certificate binding — should configure DoT or DoH transport, enable DNSSEC
validation, and call `validate_dane` before establishing TLS connections.

---

## 2. Transport Configuration

```ferrum
// Transport selection for the secure resolver.
// Udp and Tcp are provided for parity and as fallback paths.
// Production security-conscious code should use DoT or DoH.
enum ResolverTransport {
    // Plain UDP (RFC 1035). Falls back to TCP on truncated responses.
    // No confidentiality. Default for stdlib Resolver.
    Udp,

    // Plain TCP (RFC 1035). Larger response support. Still no confidentiality.
    Tcp,

    // DNS-over-TLS (RFC 7858). Encrypts queries to the nameserver.
    // Default port is 853; override for non-standard deployments.
    DoT {
        server:          SocketAddr,
        cert_validation: TlsCertValidation,
    },

    // DNS-over-HTTPS (RFC 8484). DNS carried in HTTPS POST or GET requests.
    // url must be an https:// URI pointing to a /dns-query endpoint.
    // client is a configured HTTPS client from ferrum_extlib_http.
    DoH {
        url:    Uri,
        client: HttpClient,
    },
}

// How the resolver validates the nameserver's TLS certificate.
// Applies only to DoT and DoH transports.
enum TlsCertValidation {
    // Validate using the system trust store. Suitable for public resolvers
    // like 1.1.1.1:853 or dns.google:853.
    SystemRoots,

    // Pin a specific CA certificate. Use when your nameserver uses a
    // private CA (corporate resolver, private DNS infrastructure).
    PinnedCa(CertDer),

    // Pin the resolver's exact end-entity certificate (SPKI hash).
    // Maximum pinning; fails if the resolver rotates its certificate.
    PinnedSpkiSha256([u8; 32]),

    // Disable certificate validation. Only for testing and local development.
    // Never use in production — defeats the purpose of DoT/DoH.
    DisabledForTesting,
}

// Full configuration for SecureResolver.
type SecureResolverConfig {
    // How to reach nameservers.
    transport:      ResolverTransport,

    // Explicit nameserver list. If empty, falls back to system configuration.
    // For DoT: each entry's IP is used; the port in the transport overrides.
    // For DoH: ignored (the DoH URL specifies the resolver).
    nameservers:    Vec[SocketAddr],

    // Search domains applied to single-label queries.
    // Mirrors /etc/resolv.conf search directive.
    search_domains: Vec[String],

    // Threshold for applying search domains.
    // Queries with fewer than ndots dots are tried with search domains first.
    // IETF default is 1; most OS defaults are 1 or 5.
    ndots:          u8,

    // Per-query timeout. Applied independently to each UDP/TCP/DoT attempt.
    timeout:        Duration,

    // Number of retry attempts per nameserver before trying the next.
    retries:        u8,

    // Enable DNSSEC validation. Responses are validated against the
    // trust anchor chain. Unsigned zones still resolve; only bogus
    // signatures cause failure. Set dnssec_mode for stricter policy.
    dnssec:         bool,

    // DNSSEC policy when dnssec is true.
    dnssec_mode:    DnssecMode,

    // Enable DANE enforcement. Requires dnssec: true.
    // When true, connections through this resolver should call
    // validate_dane before establishing TLS.
    dane:           bool,

    // mDNS integration. When true and ferrum_extlib_mdns is available,
    // .local queries are automatically routed to the mDNS subsystem.
    // Queries for other names are unaffected.
    mdns:           bool,
}

impl SecureResolverConfig {
    // Sensible default: DoT with system roots, DNSSEC validation,
    // DANE enabled, 5s timeout, 3 retries.
    fn default_secure(server: SocketAddr): Self {
        SecureResolverConfig {
            transport:      ResolverTransport.DoT {
                                server,
                                cert_validation: TlsCertValidation.SystemRoots,
                            },
            nameservers:    Vec.new(),
            search_domains: Vec.new(),
            ndots:          1,
            timeout:        Duration.from_secs(5),
            retries:        3,
            dnssec:         true,
            dnssec_mode:    DnssecMode.Validate,
            dane:           true,
            mdns:           false,
        }
    }

    // Minimal configuration: DoH with DNSSEC required.
    fn default_doh(url: Uri, client: HttpClient): Self {
        SecureResolverConfig {
            transport:      ResolverTransport.DoH { url, client },
            nameservers:    Vec.new(),
            search_domains: Vec.new(),
            ndots:          1,
            timeout:        Duration.from_secs(10),
            retries:        2,
            dnssec:         true,
            dnssec_mode:    DnssecMode.Require,
            dane:           true,
            mdns:           false,
        }
    }
}
```

---

## 3. Secure Resolver

```ferrum
// SecureResolver wraps and extends stdlib net.Resolver with security layers:
// - DoT/DoH encrypted transports
// - Full DNSSEC chain validation
// - DANE/TLSA record access
// - Extended record type queries
// - Structured query/response with per-query DNSSEC status
// - TTL-based caching with DNSSEC-aware cache partitioning
// - Automatic mDNS routing for .local names
//
// SecureResolver is a capability, not a global. Construct it explicitly.
// Clone it cheaply to share across tasks — the cache is shared.
type SecureResolver  given [A: Allocator]

impl SecureResolver {
    // Construct a new resolver from configuration.
    // Establishes TLS connections for DoT, validates DoH URL for DoH.
    // Returns an error if TLS handshake fails or nameservers are unreachable.
    pub fn new(config: SecureResolverConfig): Result[Self, ResolverError] ! Net + Async

    // Load DNSSEC trust anchors from the IANA root zone anchor (RFC 7958).
    // Call this after new() if you want DNSSEC validation against the
    // live root. For embedded or offline use, call with_trust_anchors().
    pub fn load_root_trust_anchors(&mut self): Result[(), ResolverError] ! Net + Async

    // Install explicit trust anchors (DS records for the root or a private
    // root zone). Use for private DNS hierarchies or DNSSEC split horizons.
    // Each DnsKey in anchors must correspond to a zone in the hierarchy.
    pub fn with_trust_anchors(&mut self, anchors: Vec[TrustAnchor])

    // Generic async query — the primary API.
    // Returns all sections of the DNS response with per-response DNSSEC status.
    // Effects: Net (sends a DNS query), Async (awaits network I/O).
    pub fn query(
        &self,
        name:  &str,
        rtype: RecordType,
    ): Result[QueryResponse, ResolverError] ! Net + Async

    // Obtain a cancel token for the next query call.
    // Pass the token to another task; call token.cancel() to abort the query.
    // Guaranteed: after cancel() returns, the query future will not produce
    // any further output. No callback or continuation fires after cancellation.
    pub fn query_cancel_token(&self): CancelToken

    // Typed convenience queries — build on query() internally.
    pub fn lookup_ip(&self, host: &str): Result[Vec[IpAddr], ResolverError] ! Net + Async
    pub fn lookup_ip6(&self, host: &str): Result[Vec[Ipv6Addr], ResolverError] ! Net + Async
    pub fn lookup_mx(&self, name: &str): Result[Vec[MxRecord], ResolverError] ! Net + Async
    pub fn lookup_srv(
        &self,
        service: &str,
        proto:   &str,
        name:    &str,
    ): Result[Vec[SrvRecord], ResolverError] ! Net + Async
    pub fn lookup_tlsa(
        &self,
        port:  u16,
        proto: &str,
        name:  &str,
    ): Result[Vec[TlsaRecord], ResolverError] ! Net + Async
    pub fn lookup_txt(&self, name: &str): Result[Vec[String], ResolverError] ! Net + Async
    pub fn lookup_caa(&self, name: &str): Result[Vec[CaaRecord], ResolverError] ! Net + Async
    pub fn lookup_svcb(&self, name: &str): Result[Vec[SvcbRecord], ResolverError] ! Net + Async
    pub fn lookup_https(&self, name: &str): Result[Vec[HttpsRecord], ResolverError] ! Net + Async
    pub fn lookup_naptr(&self, name: &str): Result[Vec[NaptrRecord], ResolverError] ! Net + Async
    pub fn reverse_lookup(&self, ip: IpAddr): Result[Vec[String], ResolverError] ! Net + Async

    // Evict all cache entries. Useful after a network change event.
    pub fn flush_cache(&mut self)

    // Return current cache statistics.
    pub fn cache_stats(&self): CacheStats
}

// A cancel token for an in-flight query.
// Dropping the token without calling cancel() has no effect.
// Calling cancel() signals the query to abort at its next await point.
// After cancel() returns, the associated query Future resolves to
// Err(ResolverError.Cancelled) and no further side effects occur.
type CancelToken {
    pub fn cancel(self)       // Consume and cancel
    pub fn is_cancelled(&self): bool
}
```

---

## 4. Record Types

### 4.1 RecordType Enum

```ferrum
// All IANA-assigned DNS record types relevant to this module.
// Types not listed here are accessible via RecordType.Unknown(u16).
enum RecordType {
    A,          // IPv4 address (RFC 1035)
    Aaaa,       // IPv6 address (RFC 3596)
    Ns,         // Authoritative nameserver (RFC 1035)
    Cname,      // Canonical name alias (RFC 1035)
    Soa,        // Start of authority (RFC 1035)
    Ptr,        // Pointer (reverse DNS) (RFC 1035)
    Mx,         // Mail exchange (RFC 1035)
    Txt,        // Text record (RFC 1035)
    Srv,        // Service locator (RFC 2782)
    Naptr,      // Naming authority pointer (RFC 3403)
    Ds,         // Delegation signer (RFC 4034)
    Rrsig,      // Resource record signature (RFC 4034)
    Nsec,       // Next secure (RFC 4034)
    Nsec3,      // Next secure v3 (RFC 5155)
    Dnskey,     // DNS public key (RFC 4034)
    Tlsa,       // TLSA/DANE (RFC 6698)
    Caa,        // Certification authority authorization (RFC 8659)
    Https,      // HTTPS service binding (RFC 9460)
    Svcb,       // Service binding (RFC 9460)

    // Any other type, by IANA number.
    Unknown(u16),
}
```

### 4.2 ResourceRecord Union

```ferrum
// A single DNS resource record, parsed to its native type.
// Unknown or unparsed record types carry their wire-format data.
enum ResourceRecord {
    A(Ipv4Addr),
    Aaaa(Ipv6Addr),
    Ns(String),
    Cname(String),
    Ptr(String),
    Mx(MxRecord),
    Txt(Vec[Vec[u8]]),      // TXT strings are sequences of byte strings
    Srv(SrvRecord),
    Naptr(NaptrRecord),
    Ds(DsRecord),
    Rrsig(RrsigRecord),
    Nsec(NsecRecord),
    Nsec3(Nsec3Record),
    Dnskey(DnskeyRecord),
    Tlsa(TlsaRecord),
    Caa(CaaRecord),
    Https(HttpsRecord),
    Svcb(SvcbRecord),
    // Wire-format bytes for unknown or unparsed types.
    Unknown { rtype: u16, data: Vec[u8] },
}

// Common metadata present on every record, regardless of type.
type RrMeta {
    pub name:  String,      // Owner name
    pub class: DnsClass,    // Always In for Internet records
    pub ttl:   Duration,
}

enum DnsClass {
    In,    // Internet (RFC 1035) — the only class in common use
    Any,   // Wildcard query class
    Other(u16),
}
```

### 4.3 Typed Record Structs

```ferrum
// MX record — mail exchange.
type MxRecord {
    pub priority: u16,      // Lower values preferred
    pub exchange: String,   // Hostname of mail server
}

// SRV record (RFC 2782) — service location.
// Queried as _service._proto.name, e.g. _imaps._tcp.example.com
type SrvRecord {
    pub priority: u16,      // Lower values preferred
    pub weight:   u16,      // Load balancing among equal-priority records
    pub port:     u16,
    pub target:   String,   // Hostname of service endpoint
}

impl SrvRecord {
    // Weighted random selection from a slice of equal-priority SrvRecords,
    // following RFC 2782 selection algorithm. Returns the index chosen.
    pub fn select_weighted(records: &[SrvRecord]): Option[usize]
}

// TLSA record (RFC 6698) — DANE certificate association.
// Used to bind a TLS certificate to a DNS name under DNSSEC protection.
type TlsaRecord {
    pub usage:         TlsaUsage,
    pub selector:      TlsaSelector,
    pub matching_type: TlsaMatchingType,
    pub cert_data:     Vec[u8],     // Content depends on matching_type
}

enum TlsaUsage {
    PkixTa,     // 0: PKIX-TA — CA constraint (CA must be in PKIX chain)
    PkixEe,     // 1: PKIX-EE — EE certificate constraint
    DaneTa,     // 2: DANE-TA — trust anchor for this service only
    DaneEe,     // 3: DANE-EE — EE certificate, no PKIX required
}

enum TlsaSelector {
    FullCert,   // 0: Match against the full DER certificate
    Spki,       // 1: Match against the SubjectPublicKeyInfo only
}

enum TlsaMatchingType {
    Full,       // 0: cert_data is the full selected bytes
    Sha256,     // 1: cert_data is SHA-256 of the selected bytes
    Sha512,     // 2: cert_data is SHA-512 of the selected bytes
}

// SVCB record (RFC 9460) — service binding for arbitrary protocol endpoints.
type SvcbRecord {
    pub priority: u16,          // 0 = alias mode; >0 = service mode
    pub target:   String,       // Target hostname; "." means owner name
    pub params:   SvcbParams,   // Service parameters
}

// HTTPS record (RFC 9460) — SVCB subtype for HTTPS endpoints.
// Structurally identical to SvcbRecord; a distinct type prevents confusion
// between HTTPS-specific and generic SVCB usage.
type HttpsRecord {
    pub priority: u16,
    pub target:   String,
    pub params:   SvcbParams,
}

// Parsed SVCB/HTTPS service parameters (SvcParamKey values).
type SvcbParams {
    pub alpn:              Option[Vec[String]],     // Supported ALPN IDs
    pub no_default_alpn:   bool,
    pub port:              Option[u16],
    pub ipv4hint:          Vec[Ipv4Addr],
    pub ipv6hint:          Vec[Ipv6Addr],
    pub ech:               Option[Vec[u8]],         // Encrypted Client Hello config
    pub mandatory_keys:    Vec[u16],                // Must-understand keys
    pub unknown:           Vec[(u16, Vec[u8])],     // Unrecognized keys, raw
}

// NAPTR record (RFC 3403) — Naming Authority Pointer.
// Used in ENUM (telephone number mapping) and SIP service discovery.
type NaptrRecord {
    pub order:       u16,
    pub preference:  u16,
    pub flags:       String,        // "S", "A", "U", "P" or empty
    pub service:     String,        // Service/protocol identifier
    pub regexp:      String,        // Substitution regexp (may be empty)
    pub replacement: String,        // Next lookup target (if regexp empty)
}

// CAA record (RFC 8659) — Certification Authority Authorization.
// Restricts which CAs may issue certificates for a domain.
type CaaRecord {
    pub critical: bool,
    pub tag:      CaaTag,
    pub value:    String,
}

enum CaaTag {
    Issue,          // "issue" — CA authorized to issue DV certs
    IssueWild,      // "issuewild" — CA authorized to issue wildcard certs
    Iodef,          // "iodef" — incident report URL
    Other(String),  // Unknown tag
}

// DS record (RFC 4034) — Delegation Signer.
// Published in the parent zone to authenticate the child zone's DNSKEY.
type DsRecord {
    pub key_tag:     u16,
    pub algorithm:   DnsKeyAlgorithm,
    pub digest_type: DsDigestType,
    pub digest:      Vec[u8],
}

enum DsDigestType {
    Sha1,       // 1 — Deprecated; do not use for new zones
    Sha256,     // 2 — Recommended (RFC 4034)
    Sha384,     // 4 — Alternative (RFC 6605)
    Unknown(u8),
}

// DNSKEY record (RFC 4034) — DNS public key.
// Used to verify RRSIG records. The Zone Key bit indicates signing keys.
type DnskeyRecord {
    pub flags:     DnskeyFlags,
    pub protocol:  u8,          // Always 3 per RFC 4034
    pub algorithm: DnsKeyAlgorithm,
    pub public_key: Vec[u8],    // Algorithm-specific public key bytes
}

type DnskeyFlags {
    pub zone_key:    bool,      // Bit 7 — key signs zone data
    pub secure_entry_point: bool, // Bit 15 — key is a trust anchor (KSK)
    pub revoked:     bool,      // Bit 8 — key is revoked (RFC 5011)
}

enum DnsKeyAlgorithm {
    RsaSha256,      // 8 — RSA/SHA-256 (RFC 5702)
    RsaSha512,      // 10 — RSA/SHA-512 (RFC 5702)
    EcdsaP256Sha256, // 13 — ECDSA P-256/SHA-256 (RFC 6605)
    EcdsaP384Sha384, // 14 — ECDSA P-384/SHA-384 (RFC 6605)
    Ed25519,        // 15 — Ed25519 (RFC 8080)
    Ed448,          // 16 — Ed448 (RFC 8080)
    Unknown(u8),
}

// RRSIG record (RFC 4034) — Resource Record Signature.
// Covers a complete RRset; validated against the DNSKEY with matching key tag.
type RrsigRecord {
    pub type_covered: RecordType,
    pub algorithm:    DnsKeyAlgorithm,
    pub labels:       u8,           // Label count in owner name
    pub original_ttl: u32,
    pub expiration:   Timestamp,
    pub inception:    Timestamp,
    pub key_tag:      u16,
    pub signer_name:  String,
    pub signature:    Vec[u8],
}

// NSEC record (RFC 4034) — Next Secure.
// Provides authenticated denial of existence for a name or type.
type NsecRecord {
    pub next_name:  String,
    pub type_bitmap: Vec[RecordType],   // Types present at owner name
}

// NSEC3 record (RFC 5155) — Hashed Next Secure.
// Provides authenticated denial without exposing the zone contents.
type Nsec3Record {
    pub algorithm:   u8,
    pub flags:       u8,
    pub iterations:  u16,
    pub salt:        Vec[u8],
    pub next_hashed_owner: Vec[u8],
    pub type_bitmap: Vec[RecordType],
}
```

---

## 5. DNSSEC Validation

### 5.1 Validation Result

```ferrum
// Per-response DNSSEC validation status.
// Every QueryResponse carries one of these.
type DnssecResult {
    // True if the complete chain from root to leaf was validated
    // and every signature was current and correct.
    pub authenticated: bool,

    // True if a signature was present but failed to verify, or
    // a required record (DS, DNSKEY) was missing from a signed zone.
    // Bogus responses MUST NOT be used for any security decision.
    pub bogus:         bool,

    // True if the zone is known to be unsigned (no DS in parent,
    // no DNSKEY at zone apex). Not bogus — just unsigned.
    pub insecure:      bool,

    // The validated or as-received RRset.
    pub records:       Vec[ResourceRecord],

    // Detailed validation chain, one entry per delegation boundary.
    // Empty for insecure or unauthenticated zones.
    pub chain:         Vec[DnssecChainLink],

    // If bogus: the specific error that caused validation to fail.
    pub bogus_reason:  Option[DnssecValidationError],
}

// One link in the DNSSEC authentication chain.
type DnssecChainLink {
    pub zone:    String,
    pub ds:      Vec[DsRecord],         // DS records anchoring this zone
    pub dnskey:  Vec[DnskeyRecord],     // ZSK/KSK records at this zone
    pub rrsig:   Vec[RrsigRecord],      // Signatures over this zone's data
}
```

### 5.2 Validation Errors

```ferrum
enum DnssecValidationError {
    // A signature's expiration timestamp is in the past.
    // Check system clock; if clock is correct, the zone operator has not
    // re-signed in time.
    SignatureExpired { signer: String, expired_at: Timestamp },

    // A signature's inception timestamp is in the future.
    // Indicates a clock skew or a pre-published signature.
    SignatureNotYetValid { signer: String, valid_from: Timestamp },

    // Signature failed cryptographic verification.
    BogusSignature { key_tag: u16, algorithm: DnsKeyAlgorithm },

    // No DNSKEY found with the key tag referenced in a RRSIG or DS.
    KeyNotFound { key_tag: u16, zone: String },

    // DNSKEY algorithm is unsupported or deprecated.
    UnsupportedAlgorithm(DnsKeyAlgorithm),

    // DS digest does not match any DNSKEY in the child zone.
    DsKeyMismatch { parent_zone: String, child_zone: String },

    // A signed zone is missing its NSEC/NSEC3 chain.
    // Required to provide authenticated denial of existence.
    MissingDenialOfExistence { zone: String },

    // The DS record at the parent zone is absent,
    // breaking the chain of trust from the trust anchor.
    MissingDs { parent_zone: String, child_zone: String },

    // Validation exceeded configured depth or recursion limit.
    ChainTooDeep { depth: usize },
}
```

### 5.3 Trust Anchor Management

```ferrum
// A trust anchor is a DNSKEY (or its DS digest) that is accepted without
// a parent signature — typically the IANA root KSK, or a private zone root.
type TrustAnchor {
    pub zone:    String,
    pub dnskey:  DnskeyRecord,
    pub ds:      Option[DsRecord],  // If DS form is preferred for matching
}

impl TrustAnchor {
    // The IANA root trust anchors as of the KSK-2024 rollover.
    // Embed these for offline environments that cannot fetch from data.iana.org.
    pub const IANA_ROOT: [TrustAnchor; 1]

    // Parse a trust anchor from the RFC 7958 XML format
    // (data.iana.org/root-anchors/root-anchors.xml).
    pub fn from_iana_xml(xml: &str): Result[Vec[TrustAnchor], TrustAnchorError]

    // Parse a trust anchor from the BIND-style zone key record format.
    pub fn from_zone_key(zone: &str, dnskey_rdata: &str): Result[Self, TrustAnchorError]
}

enum TrustAnchorError {
    ParseError(String),
    UnsupportedAlgorithm(DnsKeyAlgorithm),
    InvalidKeyData,
}
```

---

## 6. DANE Validation

```ferrum
// Result of a DANE/TLSA validation attempt for a specific TLS connection.
type DaneValidationResult {
    // True if at least one TLSA record matched the presented certificate.
    pub matched:           bool,

    // The TLSA records that were fetched and used in the check.
    pub tlsa_records:      Vec[TlsaRecord],

    // True if the TLSA RRset was DNSSEC-authenticated.
    // A DANE result is only meaningful if this is true.
    // A match against unvalidated TLSA records provides no security benefit.
    pub dnssec_validated:  bool,

    // Usage that produced the match, if matched is true.
    pub matched_usage:     Option[TlsaUsage],
}

// Validate a server's TLS certificate against a set of TLSA records.
//
// cert: the DER-encoded end-entity certificate as presented by the server.
// chain: the server's full certificate chain, DER-encoded, leaf first.
// tlsa_records: TLSA records for the service (should come from lookup_tlsa
//              with DNSSEC validation confirmed).
//
// Returns DaneValidationResult. The caller must also check dnssec_validated
// on the result; a match without DNSSEC validation is not secure.
//
// Pure: no network access. All inputs must be fetched before calling.
pub fn validate_dane(
    cert:         &[u8],
    chain:        &[Vec[u8]],
    tlsa_records: &[TlsaRecord],
): Result[DaneValidationResult, DaneError]

enum DaneError {
    // No TLSA records were provided.
    NoTlsaRecords,

    // The certificate could not be parsed.
    CertParseError(String),

    // All TLSA records use unsupported usage, selector, or matching types.
    // At least one record must use a supported combination.
    NoSupportedRecords,

    // Usage 0 or 1 requires a complete PKIX chain. The chain was incomplete
    // or contained an unparseable certificate.
    ChainIncomplete,

    // An internal cryptographic error during hash or signature computation.
    CryptoError(String),
}
```

---

## 7. Full Async Query API

### 7.1 QueryResponse

```ferrum
// The full response to a DNS query, including all sections and
// per-response DNSSEC status.
type QueryResponse {
    // Name as queried (after CNAME following if any).
    pub queried_name:  String,

    // The authoritative answer section.
    pub answers:       Vec[ResourceRecord],

    // Authority section (NS records pointing to nameservers, NSEC/NSEC3
    // for denial of existence).
    pub authority:     Vec[ResourceRecord],

    // Additional section (glue A/AAAA records for nameservers, etc.).
    pub additional:    Vec[ResourceRecord],

    // DNSSEC status for the answer RRset.
    pub dnssec:        DnssecResult,

    // True if this response was served from the local TTL cache.
    pub from_cache:    bool,

    // Minimum TTL across all answer records. The caller may cache results
    // for this duration; SecureResolver handles TTL decrement internally.
    pub min_ttl:       Duration,
}
```

### 7.2 Primary Query Method

```ferrum
// Perform a fully async DNS query.
//
// name: the DNS name to query. Relative names are completed against
//       search_domains following the ndots rule.
// rtype: the record type to query.
//
// Returns the full query response including DNSSEC status.
// If dnssec is enabled in the config, bogus responses return
// Err(ResolverError.DnssecBogus) regardless of dnssec_mode.
// With DnssecMode.Require, unsigned (insecure) responses also
// return Err(ResolverError.DnssecInsecure).
//
// Cancellation: call query_cancel_token() before this call to obtain a
// token. Pass the token to another task. Calling token.cancel() causes
// this future to resolve to Err(ResolverError.Cancelled) at the next
// await point, with no partial output.
pub fn query(
    &self,
    name:  &str,
    rtype: RecordType,
): Result[QueryResponse, ResolverError] ! Net + Async
```

### 7.3 Cancellation Protocol

```ferrum
// Usage pattern for cancellable queries:
//
//   let resolver = SecureResolver.new(config).await?
//   let token = resolver.query_cancel_token()
//
//   // In another task or after a timeout:
//   scope s {
//       let query_task = s.spawn(resolver.query("example.com", RecordType.A))
//       let cancel_task = s.spawn(timeout_then_cancel(token, Duration.from_secs(3)))
//       match query_task.await {
//           Ok(response)                     => use(response),
//           Err(ResolverError.Cancelled)     => handle_cancel(),
//           Err(e)                           => handle_error(e),
//       }
//   }
//
// The CancelToken is Send and may be moved to any task.
// After cancel() returns, the query future is guaranteed not to fire
// any callbacks or produce any further output. This guarantee holds
// even if the query was already in the middle of a TLS read.
```

---

## 8. mDNS Routing

When `SecureResolverConfig.mdns` is `true` and `ferrum_extlib_mdns` is
available at link time, the `SecureResolver` automatically routes queries
for names ending in `.local` (and `254.169.in-addr.arpa` for reverse lookups
on link-local addresses) to the mDNS subsystem rather than to the configured
nameservers.

```ferrum
// Routing rules applied before sending any query to a nameserver:
//
//   name ends with ".local"
//     -> routed to ferrum_extlib_mdns::MdnsResolver
//     -> DNSSEC is not applicable (mDNS is unsigned by design)
//     -> DnssecResult.insecure = true for all .local responses
//
//   name ends with "254.169.in-addr.arpa" (link-local reverse)
//     -> routed to mDNS
//
//   all other names
//     -> routed to configured nameservers via configured transport

// If ferrum_extlib_mdns is not linked, .local queries proceed to the
// configured nameserver (which will likely return NXDOMAIN).
// No compile error is produced; the feature degrades gracefully.
```

---

## 9. Caching

The cache is internal to `SecureResolver` and shared across all clones of the
same resolver instance. It is safe for concurrent access from multiple tasks.

```ferrum
// Cache semantics:
//
// Positive entries (records returned by the server):
//   Cached for the minimum TTL across all records in the answer section.
//   DNSSEC-authenticated entries are stored in a separate partition from
//   unauthenticated entries. A cache hit from the authenticated partition
//   is never downgraded to serve an unauthenticated query result.
//
// Negative entries (NXDOMAIN / NODATA):
//   Cached for the TTL of the SOA record in the authority section.
//   Negative entries are also partitioned by DNSSEC status.
//   An authenticated NXDOMAIN prevents cache poisoning: a subsequent
//   positive answer from a different nameserver is rejected.
//
// DNSSEC cache partitioning:
//   Authenticated (AD bit set and chain validated):
//     Served only when DNSSEC is enabled in config.
//   Unauthenticated:
//     Served when DNSSEC is disabled, or for insecure zones.
//
// TTL management:
//   TTLs are decremented in real time. A cached entry whose TTL reaches
//   zero is evicted before the next query can return it.
//
// mDNS entries:
//   .local responses are cached separately with their own TTL tracking.
//   They are never served in response to non-.local queries.

type CacheStats {
    pub total_entries:         usize,
    pub authenticated_entries: usize,
    pub negative_entries:      usize,
    pub hits:                  u64,
    pub misses:                u64,
    pub evictions:             u64,
}
```

---

## 10. Error Types

```ferrum
// Top-level error type for all resolver operations.
enum ResolverError {
    // The queried name does not exist (RCODE 3 / NXDOMAIN).
    // Contains the SOA from the authority section if present.
    NxDomain { name: String, soa: Option[ResourceRecord] },

    // The nameserver returned a server failure (RCODE 2 / SERVFAIL).
    // Often indicates a DNSSEC validation failure at the upstream resolver.
    ServFail { name: String },

    // The query did not complete within the configured timeout after all
    // retry attempts were exhausted.
    Timeout { name: String, elapsed: Duration },

    // A DNSSEC signature was present but failed to verify, or the chain
    // of trust could not be completed. Do not use the response data.
    DnssecBogus { name: String, reason: DnssecValidationError },

    // The zone exists but is unsigned, and DnssecMode.Require is configured.
    DnssecInsecure { name: String },

    // Transport-level failure: TLS handshake failed, TCP connection refused,
    // DoH returned a non-200 status, etc.
    TransportFailed { nameserver: SocketAddr, detail: String },

    // The nameserver's TLS certificate failed validation.
    // Only possible with DoT or DoH transports.
    CertificateError { nameserver: SocketAddr, detail: String },

    // The query was cancelled via CancelToken.cancel().
    Cancelled,

    // A name in the query or response is malformed (label too long,
    // total name exceeds 255 octets, invalid encoding).
    InvalidName(String),

    // The response message was structurally malformed and could not be parsed.
    MalformedResponse { detail: String },

    // The configured nameserver list is empty and no system configuration
    // could be found.
    NoNameservers,

    // DNSSEC trust anchors have not been loaded. Call load_root_trust_anchors()
    // or with_trust_anchors() before querying with dnssec: true.
    NoTrustAnchors,

    // An I/O error at the socket layer.
    Io(IoError),
}
```

---

## 11. Example Usage

### 11.1 DoT Lookup with DNSSEC Validation

```ferrum
use ferrum_extlib_dns_secure.{SecureResolver, SecureResolverConfig, RecordType}

pub fn resolve_with_dnssec(hostname: &str): Result[Vec[IpAddr], ResolverError] ! Net + Async {
    let server = SocketAddr.from_str("9.9.9.9:853")?
    let config = SecureResolverConfig.default_secure(server)
    let mut resolver = SecureResolver.new(config).await?
    resolver.load_root_trust_anchors().await?

    let response = resolver.query(hostname, RecordType.Aaaa).await?

    if response.dnssec.bogus {
        return Err(ResolverError.DnssecBogus {
            name:   hostname.to_string(),
            reason: response.dnssec.bogus_reason.unwrap(),
        })
    }

    if !response.dnssec.authenticated {
        // Zone is unsigned. Proceed only if your policy allows it.
        log.warn("DNSSEC not available for {hostname}: treating as insecure")
    }

    let addrs = response.answers
        .iter()
        .filter_map(|rr| match rr {
            ResourceRecord.Aaaa(addr) => Some(IpAddr.V6(addr)),
            ResourceRecord.A(addr)    => Some(IpAddr.V4(addr)),
            _                         => None,
        })
        .collect()

    Ok(addrs)
}
```

### 11.2 DANE TLSA Retrieval for Email (SMTP)

DANE for email uses TLSA records at `_25._tcp.<mx-hostname>` to authenticate
the MX server's TLS certificate, reducing reliance on the WebPKI for SMTP
security (RFC 7672).

```ferrum
use ferrum_extlib_dns_secure.{
    SecureResolver, SecureResolverConfig, RecordType,
    TlsaRecord, DaneValidationResult, validate_dane,
}

pub fn fetch_smtp_tlsa(
    resolver: &SecureResolver,
    mx_host:  &str,
) : Result[DaneValidationResult, ResolverError] ! Net + Async {
    // Retrieve TLSA records for SMTP (port 25, TCP).
    let tlsa_records = resolver.lookup_tlsa(25, "tcp", mx_host).await?

    // Retrieve the actual server certificate over SMTP STARTTLS.
    // (TLS connection setup elided — depends on ferrum_extlib_tls.)
    let (cert_der, chain_der) = connect_smtp_and_get_cert(mx_host).await?

    // Validate: purely local, no network access.
    let result = validate_dane(&cert_der, &chain_der, &tlsa_records)?

    if !result.dnssec_validated {
        // A TLSA match without DNSSEC is not secure — an attacker could
        // forge both the TLSA record and the certificate.
        log.warn("TLSA records for {mx_host} are not DNSSEC-authenticated")
    }

    if result.matched && result.dnssec_validated {
        log.info("DANE validation passed for {mx_host}")
    } else if !result.matched {
        log.warn("DANE validation failed: no TLSA record matched for {mx_host}")
    }

    Ok(result)
}
```

### 11.3 SRV Record Lookup for Service Discovery

```ferrum
use ferrum_extlib_dns_secure.{SecureResolver, SrvRecord}

// Discover IMAPS endpoints for a domain using SRV records (RFC 6186).
pub fn discover_imap(
    resolver: &SecureResolver,
    domain:   &str,
) : Result[SocketAddr, ResolverError] ! Net + Async {
    let records = resolver.lookup_srv("imaps", "tcp", domain).await?

    if records.is_empty() {
        return Err(ResolverError.NxDomain {
            name: format("_imaps._tcp.{domain}"),
            soa:  None,
        })
    }

    // Sort by priority (lowest first), then apply weighted random selection.
    let mut by_priority = records.clone()
    by_priority.sort_by_key(|r| r.priority)

    let lowest_priority = by_priority[0].priority
    let candidates: Vec[SrvRecord] = by_priority
        .into_iter()
        .take_while(|r| r.priority == lowest_priority)
        .collect()

    let idx = SrvRecord.select_weighted(&candidates)
        .ok_or(ResolverError.NxDomain {
            name: format("_imaps._tcp.{domain}"),
            soa:  None,
        })?

    let chosen = &candidates[idx]
    let addrs  = resolver.lookup_ip(&chosen.target).await?
    let ip     = addrs.first().ok_or(ResolverError.NxDomain {
        name: chosen.target.clone(),
        soa:  None,
    })?

    Ok(SocketAddr.new(ip, chosen.port))
}
```

### 11.4 Cancellable Query with Timeout

```ferrum
use ferrum_extlib_dns_secure.{SecureResolver, RecordType, ResolverError}

pub fn query_with_timeout(
    resolver: &SecureResolver,
    name:     &str,
    rtype:    RecordType,
    limit:    Duration,
): Result[QueryResponse, ResolverError] ! Net + Async {
    let token = resolver.query_cancel_token()

    scope s {
        let query  = s.spawn(resolver.query(name, rtype))
        let watcher = s.spawn({
            sleep(limit).await
            token.cancel()
        })

        let result = query.await
        // Watcher is cancelled automatically when scope exits.
        result
    }
}
```

---

## 12. Dependencies

`ferrum_extlib_dns_secure` depends on:

| Dependency | Role |
|---|---|
| `ferrum_extlib_tls` | TLS connections for DoT transport; DNSSEC signature verification (RSA, ECDSA, Ed25519); DANE certificate parsing and hash computation |
| `ferrum_extlib_http` | HTTPS client for DoH transport (`HttpClient` type in `ResolverTransport.DoH`) |
| Ferrum stdlib `net` | `UdpSocket`, `TcpStream`, `SocketAddr`, `IpAddr` — underlying transport for UDP and plain TCP modes |
| Ferrum stdlib `crypto` | SHA-256 and SHA-512 for DS and TLSA matching type hash comparison |
| Ferrum stdlib `async` | Task scheduling, `scope`, `sleep`, `timeout` — all async query I/O |
| `ferrum_extlib_mdns` | Optional; if linked, `.local` queries are routed to the mDNS subsystem |

`ferrum_extlib_tls` and `ferrum_extlib_http` are compile-time required
dependencies even if the application only uses UDP or TCP transports, because
the DoT and DoH transport paths are compiled into the same crate. If binary
size is a constraint, a `no_tls` feature flag can exclude DoT/DoH at compile
time; this also disables DNSSEC validation (which depends on the same
cryptographic library).

---

*End of ferrum_extlib_dns_secure design document*
