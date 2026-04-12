# Ferrum Extended Library — X.509 Certificate Module

**Module path:** `extlib.cert` (provided by `lib_ccsp_cert`)
**Part of:** Ferrum Extended Standard Library (extlib)

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [Dependencies](#2-dependencies)
3. [certparse — Certificate Parsing](#3-certparse--certificate-parsing)
4. [certval — Path Validation](#4-certval--path-validation-rfc-5280)
5. [ca-nav — Revocation and AIA Chasing](#5-ca-nav--revocation-and-aia-chasing)
6. [Trust Store Integration](#6-trust-store-integration)
7. [Certificate Fingerprinting](#7-certificate-fingerprinting)
8. [DANE Integration](#8-dane-integration)
9. [Error Types](#9-error-types)
10. [Example Usage](#10-example-usage)

---

## 1. Overview and Rationale

### Three-Layer Architecture

`extlib.cert` is organized into three modules with a strict one-way dependency:

```
ca-nav (revocation, AIA chasing)   — ! Async + Net
    └── certval (RFC 5280 path validation)   — pure + crypto
            └── certparse (DER/PEM parsing)  — pure
```

**certparse** parses DER bytes into an immutable, reference-counted `Certificate`. It is pure — no crypto, no network, no time. Any code that needs to inspect certificate fields (display, monitoring, custom logic) uses only this layer.

**certval** performs RFC 5280 path validation: signature verification, validity periods, name constraints, key usage, EKU, and policy OIDs. It requires crypto primitives but is otherwise pure. The caller supplies the time explicitly — `certval` never calls `Timestamp.now()` internally (see below).

**ca-nav** performs revocation checking. It fetches issuer certificates via AIA, queries OCSP responders, downloads CRLs, and caches results with TTL. It is async and requires network access. Its effect signature is `! Async + Net`.

### Why Explicit `at_time`

Every function in `certval` and `ca-nav` that is sensitive to time takes an explicit `at_time: Timestamp` parameter. No function in this library calls `Timestamp.now()` internally.

The reason is testability. A function that calls `Timestamp.now()` internally cannot be deterministically tested: the same test vector produces different results depending on when the test runs. Certificate validity periods are finite; any test that validates an actual certificate will eventually fail when the certificate expires — unless the test can supply the time.

With an explicit `at_time`, tests pin the clock to the moment the test vector was created and remain valid indefinitely. Expiry tests can pass a time known to be inside or outside the validity window. Revocation tests can simulate OCSP responses from any point in time.

Production code passes `Timestamp.now()` at the call site, making it visible to readers that the current time is a dependency. This is the same reason pure functions in cryptography libraries do not embed RNG calls.

### Why extlib, Not stdlib

X.509 parsing requires ASN.1 DER decoding, which `extlib.asn1` provides. Signature verification in `certval` requires RSA, ECDSA, and EdDSA, which `extlib.crypto` provides. The stdlib does not contain either. The full RFC 5280 state machine is also substantial enough that it belongs outside the standard library by size alone.

The three-layer split also means that a caller needing only certparse (e.g., a certificate inspection tool) does not pay for the crypto dependencies of certval, and a caller needing only certval does not pay for the network dependencies of ca-nav.

---

## 2. Dependencies

```
extlib.cert
    extlib.asn1         — DER decoder (BER subset), OID registry, BigUint encoding
    extlib.crypto       — RSA-PKCS1-v1.5, RSA-PSS, ECDSA (P-256, P-384, P-521),
                          Ed25519, Ed448, SHA-1, SHA-256, SHA-384, SHA-512
    std.alloc           — Arc, Vec, HashMap, Box
    std.async           — Future, AsyncRead (for ca-nav only)
    std.net             — HttpClient (for ca-nav only)
    std.time            — Timestamp, Duration
```

`extlib.asn1` and `extlib.crypto` are mandatory for certparse and certval respectively. `std.async` and `std.net` are only needed for ca-nav; code that links only certparse or certval does not pull in those stdlib modules.

---

## 3. certparse — Certificate Parsing

**Module path:** `extlib.cert.certparse`

### 3.1 Certificate

`Certificate` is reference-counted and immutable. Parsing is performed once; all field access thereafter is zero-cost. The parsed representation holds references into the original DER buffer where possible (strings, raw OIDs, raw extension values), and decoded copies where the wire format is not the final format (e.g., GeneralizedTime → Timestamp, BIT STRING → typed struct).

```ferrum
// Certificate is reference-counted: cloning an Arc[Certificate] is O(1).
// The inner value is immutable; no mut methods exist.
type Certificate

impl Certificate {
    // Parse a single DER-encoded certificate.
    pub fn from_der(bytes: &[u8]): Result[Arc[Certificate], CertParseError]

    // Parse one or more PEM-encoded certificates.
    // A PEM file containing a chain yields multiple Arc[Certificate] values,
    // one per -----BEGIN CERTIFICATE----- block, in document order.
    // Returns at least one certificate or an error.
    pub fn from_pem(pem: &str): Result[Vec[Arc[Certificate]], CertParseError]

    // ── Identity ────────────────────────────────────────────────────────────

    pub fn subject(&self): &DistinguishedName
    pub fn issuer(&self): &DistinguishedName
    pub fn serial(&self): &BigUint

    // ── Validity ────────────────────────────────────────────────────────────

    pub fn validity(&self): Validity

    // ── Public Key ──────────────────────────────────────────────────────────

    pub fn subject_public_key_info(&self): &SubjectPublicKeyInfo
    pub fn signature_algorithm(&self): &AlgorithmIdentifier

    // ── Basic Extensions ─────────────────────────────────────────────────────

    // True if BasicConstraints.cA is set and the keyCertSign key usage bit
    // is also set (or key usage is absent, which RFC 5280 treats as all bits
    // set for legacy reasons).
    pub fn is_ca(&self): bool

    pub fn basic_constraints(&self): Option[BasicConstraints]
    pub fn key_usage(&self): Option[KeyUsage]
    pub fn extended_key_usage(&self): Option[Vec[Oid]]

    // ── Name Extensions ───────────────────────────────────────────────────────

    pub fn subject_alt_names(&self): Vec[GeneralName]
    pub fn name_constraints(&self): Option[NameConstraints]

    // ── AIA / CDP ─────────────────────────────────────────────────────────────

    // Authority Information Access — contains caIssuers and OCSP URIs.
    pub fn authority_info_access(&self): Vec[AccessDescription]
    // CRL Distribution Points.
    pub fn crl_distribution_points(&self): Vec[CrlDistributionPoint]

    // ── Application Extensions ───────────────────────────────────────────────

    // DANE TLSA data embedded in the certificate (RFC 6698 / draft-ietf-dane-cert).
    // None if the TLSA extension is absent.
    pub fn dane_tlsa(&self): Option[TlsaData]

    // CAA records embedded in the certificate (informational; not the DNS CAA used
    // for issuance checks — see extlib.dns_secure for that).
    pub fn caa(&self): Vec[CaaRecord]

    // Certificate Policies — the policy OIDs asserted by this certificate.
    pub fn policy_oids(&self): Vec[Oid]

    // ── Raw Access ────────────────────────────────────────────────────────────

    // Zero-copy reference to the original DER bytes passed to from_der().
    // The certificate holds an Arc over the original allocation; this slice
    // is valid as long as any Arc[Certificate] referencing the same bytes
    // remains live.
    pub fn raw_der(&self): &[u8]

    // Convenience: DER of just the TBSCertificate (the signed portion).
    pub fn raw_tbs_der(&self): &[u8]

    // ── Fingerprinting ────────────────────────────────────────────────────────

    // SHA-256 of raw_der(). See §7.
    pub fn fingerprint_sha256(&self): [u8; 32]

    // SHA-256 of the raw SubjectPublicKeyInfo DER (DANE usage 1 / SPKI pin).
    pub fn spki_fingerprint_sha256(&self): [u8; 32]
}
```

### 3.2 Validity

```ferrum
type Validity {
    pub not_before: Timestamp,
    pub not_after:  Timestamp,
}

impl Validity {
    // True if at_time is within [not_before, not_after] inclusive.
    pub fn is_valid_at(&self, at_time: Timestamp): bool
}
```

### 3.3 DistinguishedName

`DistinguishedName` is an ordered list of typed attribute-value pairs. All string values are decoded to UTF-8: UTF8String and IA5String are passed through; PrintableString and TeletexString are decoded according to their character set mappings. Values that cannot be decoded to valid UTF-8 yield `CertParseError.NonUtf8String`.

```ferrum
type DistinguishedName {
    // Ordered list of relative distinguished names, each containing
    // one or more type-value pairs.
    pub rdns: Vec[RelativeDistinguishedName],
}

type RelativeDistinguishedName {
    pub attrs: Vec[AttributeTypeAndValue],
}

type AttributeTypeAndValue {
    pub oid:   Oid,
    pub value: String,   // always UTF-8
}

impl DistinguishedName {
    // First value of the given OID, or None if absent.
    pub fn get(&self, oid: &Oid): Option[&str]

    // Convenience accessors for common attributes.
    pub fn common_name(&self):            Option[&str]   // OID 2.5.4.3
    pub fn organization(&self):           Option[&str]   // OID 2.5.4.10
    pub fn organizational_unit(&self):    Option[&str]   // OID 2.5.4.11
    pub fn country(&self):                Option[&str]   // OID 2.5.4.6
    pub fn state_or_province(&self):      Option[&str]   // OID 2.5.4.8
    pub fn locality(&self):               Option[&str]   // OID 2.5.4.7
    pub fn email(&self):                  Option[&str]   // OID 1.2.840.113549.1.9.1

    // RFC 4514 one-line string representation ("CN=example.com,O=ACME,C=US")
    pub fn to_rfc4514(&self): String
}
```

### 3.4 GeneralName

Used in Subject Alternative Names, name constraints, and distribution point fields.

```ferrum
enum GeneralName {
    // dNSName — ASCII hostname or wildcard (e.g., "example.com", "*.example.com")
    DnsName(String),

    // iPAddress — decoded from the raw 4- or 16-byte encoding
    IpAddr(IpAddr),

    // rfc822Name — email address
    Email(String),

    // uniformResourceIdentifier
    Uri(String),

    // registeredID — arbitrary OID
    RegisteredId(Oid),

    // directoryName — a full DN
    DirectoryName(DistinguishedName),

    // otherName — typed extension value not otherwise recognized
    OtherName { type_id: Oid, value: Vec[u8] },
}
```

### 3.5 SubjectPublicKeyInfo

```ferrum
type SubjectPublicKeyInfo {
    pub algorithm: AlgorithmIdentifier,
    // Raw public key bytes (BIT STRING contents, decoded).
    pub key_bytes: Vec[u8],
}

type AlgorithmIdentifier {
    pub oid:        Oid,
    // DER encoding of the parameters field, or empty if absent.
    pub parameters: Vec[u8],
}
```

### 3.6 Extensions

```ferrum
type BasicConstraints {
    pub is_ca:              bool,
    // Maximum number of non-self-issued intermediate certificates that may
    // follow in a valid certification path. None means unconstrained.
    pub path_len_constraint: Option[u32],
}

type KeyUsage {
    pub digital_signature:  bool,
    pub content_commitment: bool,   // formerly non_repudiation
    pub key_encipherment:   bool,
    pub data_encipherment:  bool,
    pub key_agreement:      bool,
    pub key_cert_sign:      bool,
    pub crl_sign:           bool,
    pub encipher_only:      bool,
    pub decipher_only:      bool,
}

type NameConstraints {
    pub permitted_subtrees: Vec[GeneralSubtree],
    pub excluded_subtrees:  Vec[GeneralSubtree],
}

type GeneralSubtree {
    pub base:    GeneralName,
    // minimum and maximum are present in the ASN.1 but RFC 5280 requires
    // them to be absent (minimum = 0, maximum absent). They are decoded
    // but certval rejects certificates where they differ from the required
    // values.
    pub minimum: u32,           // always 0 in compliant certificates
    pub maximum: Option[u32],   // always None in compliant certificates
}

type AccessDescription {
    pub access_method:   Oid,     // id-ad-ocsp or id-ad-caIssuers
    pub access_location: GeneralName,
}

type CrlDistributionPoint {
    pub distribution_point: Option[DistributionPointName],
    pub reasons:            Option[ReasonFlags],
    pub crl_issuer:         Vec[GeneralName],
}

enum DistributionPointName {
    FullName(Vec[GeneralName]),
    NameRelativeToCRLIssuer(RelativeDistinguishedName),
}

type ReasonFlags {
    pub unused:           bool,
    pub key_compromise:   bool,
    pub ca_compromise:    bool,
    pub affiliation_changed: bool,
    pub superseded:       bool,
    pub cessation_of_operation: bool,
    pub certificate_hold: bool,
    pub privilege_withdrawn: bool,
    pub aa_compromise:    bool,
}

type TlsaData {
    pub usage:     u8,
    pub selector:  u8,
    pub matching:  u8,
    pub cert_data: Vec[u8],
}

type CaaRecord {
    pub flags: u8,
    pub tag:   String,
    pub value: String,
}
```

### 3.7 OID

```ferrum
// Opaque OID type. Display produces dotted-decimal ("2.5.4.3").
// Well-known OIDs are available as associated constants on Oid.
type Oid

impl Oid {
    pub fn from_dotted(s: &str): Result[Self, OidParseError]
    pub fn to_dotted(&self): String

    // Well-known OIDs
    const ID_CE_SUBJECT_ALT_NAME:         Self  // 2.5.29.17
    const ID_CE_BASIC_CONSTRAINTS:        Self  // 2.5.29.19
    const ID_CE_KEY_USAGE:                Self  // 2.5.29.15
    const ID_CE_EXT_KEY_USAGE:            Self  // 2.5.29.37
    const ID_CE_NAME_CONSTRAINTS:         Self  // 2.5.29.30
    const ID_CE_CRL_DISTRIBUTION_POINTS:  Self  // 2.5.29.31
    const ID_CE_CERTIFICATE_POLICIES:     Self  // 2.5.29.32
    const ID_AD_OCSP:                     Self  // 1.3.6.1.5.5.7.48.1
    const ID_AD_CA_ISSUERS:               Self  // 1.3.6.1.5.5.7.48.2
    const ID_KP_SERVER_AUTH:              Self  // 1.3.6.1.5.5.7.3.1
    const ID_KP_CLIENT_AUTH:              Self  // 1.3.6.1.5.5.7.3.2
    const ID_KP_CODE_SIGNING:             Self  // 1.3.6.1.5.5.7.3.3
    const ID_KP_EMAIL_PROTECTION:         Self  // 1.3.6.1.5.5.7.3.4
    const ID_KP_TIME_STAMPING:            Self  // 1.3.6.1.5.5.7.3.8
    const ID_KP_OCSP_SIGNING:             Self  // 1.3.6.1.5.5.7.3.9
    const ANY_POLICY:                     Self  // 2.5.29.32.0
}

impl Display for Oid { ... }
impl Eq for Oid { ... }
impl Hash for Oid { ... }
```

---

## 4. certval — Path Validation (RFC 5280)

**Module path:** `extlib.cert.certval`

### 4.1 CertValidator

`CertValidator` holds a set of trust anchors and performs RFC 5280 path validation. It is immutable after construction and can be shared across threads via `Arc`.

```ferrum
type CertValidator

impl CertValidator {
    // Construct a validator with the given trust anchors.
    // Trust anchors are the self-signed root CA certificates that are
    // trusted unconditionally (their signatures are not verified;
    // only their public keys are used to verify the next certificate
    // in a chain).
    pub fn new(trust_anchors: Vec[Arc[Certificate]]): Self

    // Validate a certificate chain against the trust anchors.
    //
    // chain[0] is the end-entity certificate.
    // chain[1..] are intermediate CA certificates in order from
    // end-entity toward the root. The root itself need not be present
    // in chain — the trust anchor is located by matching issuer/subject.
    //
    // at_time is used for validity period checks throughout the chain.
    // The caller must supply the current time; this function does not
    // call Timestamp.now() internally.
    //
    // Returns a ValidatedChain on success, or a CertValidationError
    // describing exactly which check failed and on which certificate.
    pub fn validate(
        &self,
        chain:   &[Arc[Certificate]],
        at_time: Timestamp,
        params:  &ValidationParams,
    ): Result[ValidatedChain, CertValidationError]
}
```

### 4.2 ValidationParams

`ValidationParams` is a builder-style configuration for a single validation call.

```ferrum
type ValidationParams

impl ValidationParams {
    pub fn new(): Self

    // Hostname to validate against the end-entity certificate's SANs.
    // If present, certval checks the dNSName SANs (or CN if SAN is absent,
    // for legacy compatibility) against the provided hostname using the
    // wildcard matching rules of RFC 6125.
    pub fn hostname(mut self, name: &str): Self

    // Required extended key usage OID. If present, certval checks that
    // the end-entity certificate asserts this EKU (or that EKU is absent,
    // which RFC 5280 §4.2.1.12 treats as any EKU for CA certs; this
    // function applies the check only to end-entity certificates).
    pub fn expected_eku(mut self, oid: Oid): Self

    // Certificate policy OIDs that must appear in every certificate in
    // the chain. If empty, policy checking is performed only to the
    // degree required by RFC 5280 §6.1 (no explicit required policy).
    pub fn required_policies(mut self, oids: &[Oid]): Self

    // Maximum number of intermediate CA certificates permitted between
    // the trust anchor and the end-entity certificate. Default: 8.
    // A value of 0 means only direct issuer-to-end-entity is permitted.
    pub fn max_chain_depth(mut self, n: u32): Self

    // Whether revocation status is checked as part of validation.
    // Default: Disabled (certval is pure; use ca-nav for revocation).
    // RevocationMode.Required causes certval to return
    // CertValidationError.RevocationNotChecked if ca-nav results are
    // not stapled into the chain (see ValidatedChain).
    pub fn check_revocation(mut self, mode: RevocationMode): Self
}

enum RevocationMode {
    // Do not check revocation. ValidatedChain.revocation is None.
    Disabled,
    // Check revocation if OCSP/CRL information is available; succeed
    // even if it is not available.
    Optional,
    // Revocation status must be known for every certificate in the chain.
    // Returns CertValidationError.RevocationNotChecked if unavailable.
    Required,
}
```

### 4.3 ValidatedChain

Returned on successful validation. Carries the ordered chain and metadata about what was validated.

```ferrum
type ValidatedChain {
    // Ordered chain: [0] is the end-entity, [last] is the certificate
    // immediately issued by the trust anchor.
    pub chain:        Vec[Arc[Certificate]],

    // The trust anchor that anchored this chain.
    pub trust_anchor: Arc[Certificate],

    // The time at which validation was performed (the at_time parameter).
    pub validated_at: Timestamp,

    // The set of policy OIDs that were valid at every certificate in the chain.
    // Empty if no explicit policy constraints were in effect.
    pub policy_oids:  Vec[Oid],

    // Revocation status, if checked. None when RevocationMode::Disabled.
    pub revocation:   Option[ChainRevocationStatus],
}

type ChainRevocationStatus {
    // Per-certificate revocation status, parallel to ValidatedChain.chain.
    pub statuses: Vec[RevocationStatus],
}
```

### 4.4 RFC 5280 Algorithm

`CertValidator::validate` implements the path validation algorithm from RFC 5280 §6.1 in full. The following checks are performed, in order, for each certificate in the chain from the trust anchor toward the end entity:

**Per-certificate checks:**
- Validity period: `not_before <= at_time <= not_after`
- Signature verification using the issuer's public key
- Basic constraints: intermediate certificates must have `is_ca = true`; path length constraint is enforced against `max_chain_depth`
- Key usage: `keyCertSign` must be set on CA certificates; `cRLSign` must be set on CRL signers
- Name constraints: permitted and excluded subtrees are propagated and checked against subject and SAN names
- Policy OIDs: the intersecting policy set is tracked through the chain

**End-entity checks:**
- Hostname matching (if `params.hostname` is set): dNSName SANs are checked first; CN fallback is applied only if no SAN extension is present
- EKU (if `params.expected_eku` is set): the required OID must appear in the EKU extension

**Chain construction:**
- Trust anchor location: the issuer of the top intermediate certificate is matched against the trust anchor set by (issuer DN, subject DN equality) and then by (authority key identifier, if present)
- Only certificates in the provided `chain` slice are used; the validator does not fetch missing intermediates (that is ca-nav's role)

---

## 5. ca-nav — Revocation and AIA Chasing

**Module path:** `extlib.cert.ca_nav`

### 5.1 RevocationChecker

`RevocationChecker` performs all network-dependent certificate status operations. It is constructed with an `HttpClient` and an optional cache.

```ferrum
type RevocationChecker

impl RevocationChecker {
    // Construct with an HTTP client. Uses an in-memory cache by default.
    pub fn new(http_client: HttpClient): Self

    // Attach a custom revocation cache (see §5.5).
    pub fn with_cache(mut self, cache: Arc[dyn RevocationCache]): Self

    // Staple an OCSP response obtained from a TLS handshake.
    // Stapled responses are checked before any network request is made.
    // Multiple responses may be stapled (one per certificate in the chain).
    // The response is validated against the chain before being stored;
    // invalid or irrelevant responses are silently discarded.
    pub fn add_stapled_ocsp(
        &mut self,
        response:    &[u8],
        for_cert:    &Arc[Certificate],
        at_time:     Timestamp,
    ): Result[(), RevocationError]

    // Check revocation status for every certificate in the validated chain.
    //
    // For each certificate, the checker:
    //   1. Returns the stapled OCSP response if one was provided and is
    //      still valid at at_time.
    //   2. Returns the cached status if present and the TTL has not expired.
    //   3. Queries the OCSP responder from the certificate's AIA extension.
    //   4. Falls back to downloading and checking the CRL.
    //   5. If neither is reachable, returns RevocationStatus.Unknown.
    //
    // If AIA caIssuers URIs are present and the issuer certificate is not
    // in the chain, the checker fetches and parses it automatically.
    // Fetched issuer certificates are validated against the already-validated
    // chain before use.
    //
    // at_time is used for OCSP response validity and CRL nextUpdate checks.
    pub fn check_revocation(
        &self,
        chain:   &ValidatedChain,
        at_time: Timestamp,
    ): Result[RevocationStatus, RevocationError] ! Async + Net

    // Fetch a missing issuer certificate from AIA caIssuers.
    // Returns the parsed, DER-decoded certificate.
    // Does not modify the RevocationChecker's internal state.
    pub fn fetch_issuer(
        &self,
        cert: &Arc[Certificate],
    ): Result[Arc[Certificate], RevocationError] ! Async + Net
}
```

### 5.2 RevocationStatus

```ferrum
enum RevocationStatus {
    // OCSP says Good, or CRL does not list the serial.
    Good,

    // OCSP says Revoked, or CRL lists the serial.
    Revoked {
        reason:          RevocationReason,
        revocation_time: Timestamp,
    },

    // OCSP response was Unknown, or neither OCSP nor CRL was reachable
    // and no stapled response was available.
    Unknown,
}

enum RevocationReason {
    Unspecified,
    KeyCompromise,
    CaCompromise,
    AffiliationChanged,
    Superseded,
    CessationOfOperation,
    CertificateHold,
    RemoveFromCrl,
    PrivilegeWithdrawn,
    AaCompromise,
}
```

### 5.3 OCSP Processing

When querying an OCSP responder, `RevocationChecker`:

1. Constructs an OCSP request (RFC 6960) containing the certificate's issuer name hash, issuer key hash, and serial number. The hash algorithm is SHA-1 as required by the OCSP protocol (SHA-1 is used here only as an identifier, not for security).
2. Sends a POST request to the OCSP URI from the certificate's AIA extension. The request body is the DER-encoded OCSPRequest.
3. Parses the OCSPResponse and verifies the responder's signature. The responder certificate must chain to the issuer certificate or be the issuer certificate itself.
4. Checks that `thisUpdate <= at_time <= nextUpdate`. Responses with a `nextUpdate` in the past are treated as `RevocationStatus.Unknown`.
5. Caches the result with TTL derived from `nextUpdate - at_time`.

```ferrum
// OCSP response parsing is internal; not exposed directly.
// The public surface is RevocationStatus from check_revocation().

// Nonce support: RevocationChecker includes a random nonce in OCSP requests
// by default. Responders that do not echo the nonce are accepted but logged
// (RFC 6960 §4.4.1 makes nonce optional for the responder).
```

### 5.4 CRL Processing

CRL checking is the fallback when OCSP is unavailable or fails.

1. Download the CRL from the `distributionPoint` URI in the certificate's CRL Distribution Points extension.
2. Verify the CRL's signature using the issuer's public key (or the key in the `cRLIssuer` field if present, which must itself be validated).
3. Check `thisUpdate <= at_time <= nextUpdate`.
4. Search the revokedCertificates list for the certificate's serial number.
5. Cache the parsed CRL with TTL from `nextUpdate - at_time`.

CRL size is bounded: CRLs larger than 64 MiB are rejected with `RevocationError.CrlTooLarge`. Delta CRLs (RFC 3280 §5.2.4) are not currently supported; the checker downloads only complete (base) CRLs.

### 5.5 RevocationCache

```ferrum
// Implement this trait to provide a custom revocation cache.
// The default implementation is an in-memory HashMap with TTL eviction.
trait RevocationCache: Send + Sync {
    // Look up a cached status for the certificate identified by its
    // SHA-256 fingerprint. Returns None on cache miss.
    fn get(
        &self,
        cert_sha256: &[u8; 32],
    ): Option[CachedRevocationEntry]

    // Store a status. The implementation is responsible for honouring
    // the provided TTL (entries must not be returned after expires_at).
    fn set(
        &mut self,
        cert_sha256: &[u8; 32],
        entry:       CachedRevocationEntry,
    )
}

type CachedRevocationEntry {
    pub status:     RevocationStatus,
    pub expires_at: Timestamp,     // from OCSP nextUpdate or CRL nextUpdate
    pub source:     RevocationSource,
}

enum RevocationSource {
    OcspStapled,
    OcspFetched,
    Crl,
}
```

---

## 6. Trust Store Integration

**Module path:** `extlib.cert.truststore`

### 6.1 SystemTrustStore

```ferrum
type SystemTrustStore

impl SystemTrustStore {
    // Load the OS trust store.
    //
    // Linux:   reads PEM bundles from /etc/ssl/certs/ca-certificates.crt
    //          (Debian/Ubuntu) or /etc/pki/tls/certs/ca-bundle.crt (RHEL/Fedora).
    //          Falls back to /etc/ssl/certs/ directory scan if the bundle is absent.
    // macOS:   reads from the Security.framework system and login keychains via
    //          SecTrustCopyAnchorCertificates().
    // Windows: reads from the CertStore "ROOT" system store via CertOpenSystemStore.
    //
    // The returned certificates are parsed and deduplicated by fingerprint.
    // Certificates that fail to parse are logged and skipped; a partial list
    // may be returned alongside the error.
    pub fn load(): Result[Vec[Arc[Certificate]], CertError] ! IO
}
```

### 6.2 TrustStoreBuilder

```ferrum
type TrustStoreBuilder

impl TrustStoreBuilder {
    pub fn new(): Self

    // Include all certificates from the OS trust store.
    pub fn with_system_certs(mut self): Result[Self, CertError] ! IO

    // Add every certificate from a PEM bundle file.
    pub fn with_pem_file(mut self, path: &str): Result[Self, CertError] ! IO

    // Add a single certificate.
    pub fn with_cert(mut self, cert: Arc[Certificate]): Self

    // Add a DER-encoded certificate.
    pub fn with_der(mut self, der: &[u8]): Result[Self, CertParseError]

    // Build the final trust anchor list. Deduplicates by fingerprint.
    pub fn build(self): Vec[Arc[Certificate]]
}
```

---

## 7. Certificate Fingerprinting

Fingerprints are derived from the full DER encoding of the certificate. They are always computed over `raw_der()` — the exact bytes that were parsed — not a re-encoded form. This ensures fingerprints match what OpenSSL and browsers report.

```ferrum
impl Certificate {
    // SHA-256 of raw_der(). This is the standard "certificate fingerprint"
    // shown by browsers and OpenSSL.
    pub fn fingerprint_sha256(&self): [u8; 32]

    // SHA-1 of raw_der(). Provided for interop with legacy systems that
    // display SHA-1 thumbprints. Not for security use.
    pub fn fingerprint_sha1(&self): [u8; 20]

    // SHA-256 of the SubjectPublicKeyInfo DER (DANE usage 1 / SPKI pinning).
    // This fingerprint is stable across certificate renewals as long as the
    // key pair is reused, which makes it suitable for HPKP-style pinning
    // and DANE usage 1 (PKIX-SPKI) and usage 3 (DANE-SPKI).
    pub fn spki_fingerprint_sha256(&self): [u8; 32]
}
```

---

## 8. DANE Integration

**Module path:** `extlib.cert.dane`

DANE (DNS-based Authentication of Named Entities, RFC 6698) allows a TLSA DNS record to constrain which certificates are acceptable for a given hostname and port, independent of (or in addition to) the PKIX trust store.

### 8.1 TlsaRecord

```ferrum
// A parsed TLSA record from DNS (typically provided by extlib.dns_secure).
type TlsaRecord {
    pub usage:     TlsaUsage,
    pub selector:  TlsaSelector,
    pub matching:  TlsaMatching,
    pub cert_data: Vec[u8],
}

enum TlsaUsage {
    PkixTa    = 0,   // PKIX trust anchor constraint
    PkixEe    = 1,   // PKIX end-entity constraint
    DaneTa    = 2,   // DANE trust anchor assertion
    DaneEe    = 3,   // DANE end-entity assertion (no PKIX required)
}

enum TlsaSelector {
    FullCert = 0,   // cert_data matches full certificate DER
    Spki     = 1,   // cert_data matches SubjectPublicKeyInfo DER
}

enum TlsaMatching {
    Full     = 0,   // cert_data is the full selected bytes
    Sha256   = 1,   // cert_data is SHA-256 of the selected bytes
    Sha512   = 2,   // cert_data is SHA-512 of the selected bytes
}
```

### 8.2 validate_dane_cert

```ferrum
// Check whether a certificate matches a TLSA record.
//
// This is a pure function: no network access, no time dependency.
// It does not perform PKIX path validation; for DANE usages that
// interact with PKIX (0 and 1), the caller is responsible for
// also running certval.
//
// Returns true if the certificate matches the record exactly.
// Returns false if the selector, matching type, or data do not match.
pub fn validate_dane_cert(cert: &Certificate, tlsa: &TlsaRecord): bool
```

### 8.3 Matching Logic

| Usage | Selector | Matching | Behaviour |
|-------|----------|----------|-----------|
| 0 PKIX-TA | 0 Full | 0 Full | TLSA data equals full DER of a trust anchor; that anchor is added to the PKIX path |
| 0 PKIX-TA | 0 Full | 1 SHA-256 | TLSA data equals SHA-256 of a trust anchor DER |
| 1 PKIX-EE | 1 SPKI | 1 SHA-256 | TLSA data equals SHA-256 of end-entity SPKI; PKIX validation still required |
| 2 DANE-TA | 0 Full | 0 Full | TLSA data equals full DER of a trust anchor; no PKIX required |
| 3 DANE-EE | 1 SPKI | 1 SHA-256 | TLSA data equals SHA-256 of end-entity SPKI; no PKIX, no hostname check |
| 3 DANE-EE | 0 Full | 1 SHA-256 | TLSA data equals SHA-256 of full end-entity cert |

All six selector × matching × usage combinations are supported. `validate_dane_cert` handles all four usages and both selectors and all three matching types.

For usage 3 (DANE-EE), hostname validation is not performed (per RFC 7671 §5.1): the TLSA record already binds the key to a specific service name via DNS.

---

## 9. Error Types

### 9.1 CertParseError

```ferrum
enum CertParseError {
    // DER structure is malformed (unexpected tag, truncated, etc.)
    DerMalformed { offset: usize, message: String },

    // Required field is absent in the ASN.1 structure
    MissingField(String),

    // A string field contained bytes that cannot be decoded to UTF-8
    NonUtf8String { field: String, bytes: Vec[u8] },

    // PEM block is malformed (bad header, invalid base64, etc.)
    PemMalformed(String),

    // PEM input contained no certificate blocks
    PemNoCertificate,

    // An extension was marked critical but is not understood
    UnrecognizedCriticalExtension(Oid),

    // Certificate version is not 1, 2, or 3
    UnsupportedVersion(u8),

    // The OID in the outer AlgorithmIdentifier does not match
    // the OID in TBSCertificate.signature
    SignatureAlgorithmMismatch,
}
```

### 9.2 CertValidationError

```ferrum
enum CertValidationError {
    // Certificate is expired: not_after < at_time
    Expired {
        cert:    Arc[Certificate],
        at_time: Timestamp,
    },

    // Certificate is not yet valid: at_time < not_before
    NotYetValid {
        cert:    Arc[Certificate],
        at_time: Timestamp,
    },

    // No trust anchor matched the issuer of the top intermediate certificate
    UnknownIssuer { cert: Arc[Certificate] },

    // Signature verification failed
    SignatureInvalid { cert: Arc[Certificate] },

    // Hostname does not match any SAN (or CN fallback)
    NameMismatch {
        expected: String,
        found:    Vec[GeneralName],
    },

    // The required key usage bit is not set
    KeyUsageMissing {
        cert:     Arc[Certificate],
        required: String,   // human-readable bit name
    },

    // The required EKU OID is absent from the end-entity certificate
    EkuMissing {
        cert:     Arc[Certificate],
        required: Oid,
    },

    // No policy OID intersection existed for the required policy set
    PolicyRejected {
        cert:     Arc[Certificate],
        required: Vec[Oid],
        present:  Vec[Oid],
    },

    // The chain depth exceeds max_chain_depth
    ChainTooLong {
        max_depth: u32,
        actual:    u32,
    },

    // A name constraint violation was found
    NameConstraintViolated {
        cert:       Arc[Certificate],   // the CA cert with the constraint
        name:       GeneralName,        // the violating name
        constraint: GeneralSubtree,
    },

    // An intermediate certificate does not have is_ca = true
    NotACa { cert: Arc[Certificate] },

    // BasicConstraints.pathLenConstraint was exceeded
    PathLenExceeded {
        cert:      Arc[Certificate],
        allowed:   u32,
        remaining: u32,
    },

    // RevocationMode::Required but revocation status could not be determined
    RevocationNotChecked { cert: Arc[Certificate] },

    // The chain slice is empty
    EmptyChain,
}
```

### 9.3 RevocationError

```ferrum
enum RevocationError {
    // HTTP request to OCSP responder or CRL distribution point failed
    HttpError { uri: String, cause: String },

    // OCSP response DER is malformed
    OcspMalformed(String),

    // OCSP response signature did not verify
    OcspSignatureInvalid,

    // OCSP response is outside its validity window (thisUpdate/nextUpdate)
    OcspStale { this_update: Timestamp, next_update: Option[Timestamp] },

    // CRL DER is malformed
    CrlMalformed(String),

    // CRL signature did not verify
    CrlSignatureInvalid,

    // CRL is outside its validity window
    CrlStale { this_update: Timestamp, next_update: Option[Timestamp] },

    // CRL exceeds the 64 MiB size limit
    CrlTooLarge { bytes: usize },

    // AIA caIssuers fetch returned a certificate that does not chain
    // to the already-validated chain
    IssuedCertMismatch,

    // The certificate has no AIA or CRL DP extension, so revocation
    // status cannot be determined by any automatic means
    NoRevocationEndpoint { cert: Arc[Certificate] },
}
```

### 9.4 CertError

```ferrum
enum CertError {
    Parse(CertParseError),
    Io(IoError),
    // OS trust store API returned an error (macOS/Windows)
    OsTrustStore(String),
}
```

All error types implement `Display` and `Error`.

---

## 10. Example Usage

### Parse and inspect a certificate

```ferrum
use extlib.cert.certparse.{Certificate, GeneralName}

fn print_sans(pem: &str): Result[(), CertParseError] {
    let certs = Certificate.from_pem(pem)?

    for cert in &certs {
        let subject = cert.subject().to_rfc4514()
        let sans    = cert.subject_alt_names()

        if sans.is_empty() {
            // Legacy: fall back to CN
            if let Some(cn) = cert.subject().common_name() {
                println!("CN fallback: {cn}")
            }
        } else {
            for san in &sans {
                match san {
                    GeneralName.DnsName(name)  => println!("DNS:{name}"),
                    GeneralName.IpAddr(addr)   => println!("IP:{addr}"),
                    GeneralName.Email(email)   => println!("email:{email}"),
                    GeneralName.Uri(uri)       => println!("URI:{uri}"),
                    _                          => {}
                }
            }
        }
    }

    Ok(())
}
```

### Validate a chain from a TLS handshake

```ferrum
use extlib.cert.certparse.Certificate
use extlib.cert.certval.{CertValidator, ValidationParams, RevocationMode}
use extlib.cert.truststore.TrustStoreBuilder
use extlib.cert.ca_nav.RevocationChecker
use std.time.Timestamp
use std.net.HttpClient

// At program startup: build the validator once and share it.
fn build_validator(): Result[CertValidator, CertError] ! IO {
    let anchors = TrustStoreBuilder.new()
        .with_system_certs()?
        .build()
    Ok(CertValidator.new(anchors))
}

// At TLS handshake completion: validate the peer chain.
fn validate_peer_chain(
    validator:      &CertValidator,
    chain_der:      Vec[Vec[u8]>,    // DER bytes per cert, leaf first
    hostname:       &str,
    stapled_ocsp:   Option[&[u8]>,  // from TLS handshake extension
    http_client:    HttpClient,
): Result[(), CertValidationError] ! Async + Net {
    // Parse the chain.
    let mut certs: Vec[Arc[Certificate]> = Vec.new()
    for der in &chain_der {
        certs.push(Certificate.from_der(der)?)
    }

    // Validate at the current time.
    let now    = Timestamp.now()
    let params = ValidationParams.new()
        .hostname(hostname)
        .expected_eku(Oid.ID_KP_SERVER_AUTH)
        .check_revocation(RevocationMode.Required)

    let validated = validator.validate(&certs, now, &params)?

    // Check revocation, using stapled OCSP if available.
    let mut checker = RevocationChecker.new(http_client)

    if let Some(ocsp_bytes) = stapled_ocsp {
        checker.add_stapled_ocsp(ocsp_bytes, &certs[0], now)?
    }

    let revocation = checker.check_revocation(&validated, now).await?

    match revocation {
        RevocationStatus.Good            => {}
        RevocationStatus.Unknown         => {
            // RevocationMode.Required would have already failed in validate();
            // this branch is reached only with RevocationMode.Optional.
        }
        RevocationStatus.Revoked { reason, revocation_time } => {
            return Err(CertValidationError.RevocationNotChecked {
                cert: certs[0].clone(),
            })
        }
    }

    Ok(())
}
```

### DANE validation (no network)

```ferrum
use extlib.cert.certparse.Certificate
use extlib.cert.dane.{TlsaRecord, TlsaUsage, TlsaSelector, TlsaMatching, validate_dane_cert}

// tlsa_records come from a DNSSEC-validated TLSA query via extlib.dns_secure.
fn check_dane(
    cert:         &Certificate,
    tlsa_records: &[TlsaRecord],
): bool {
    for tlsa in tlsa_records {
        if validate_dane_cert(cert, tlsa) {
            return true
        }
    }
    false
}
```

### Pin by SPKI fingerprint

```ferrum
use extlib.cert.certparse.Certificate

const PINNED_SPKI_SHA256: [u8; 32] = hex!(
    "25 84 7d 67 f7 24 05 d8 1f e0 be 2e 21 23 a0 eb"
    "20 09 ed e2 f2 20 59 4e b4 ec 62 f6 96 38 d7 5a"
)

fn verify_spki_pin(cert: &Certificate): Result[(), String> {
    let pin = cert.spki_fingerprint_sha256()
    if pin != PINNED_SPKI_SHA256 {
        return Err(format!("SPKI pin mismatch: got {pin:x?}"))
    }
    Ok(())
}
```

### Test with a fixed clock

```ferrum
// certval accepts at_time explicitly, so tests are deterministic.
// This test validates a real certificate as of the day it was issued,
// and will pass correctly regardless of when the test is run.
#[test]
fn test_expired_cert_is_rejected() {
    let cert_der  = include_bytes!("testdata/expired_cert.der")
    let issuer_der = include_bytes!("testdata/issuer.der")

    let cert   = Certificate.from_der(cert_der).unwrap()
    let issuer = Certificate.from_der(issuer_der).unwrap()

    let validator = CertValidator.new(vec![issuer.clone()])
    let params    = ValidationParams.new()

    // This timestamp is one day after not_after on the test certificate.
    let after_expiry = Timestamp.from_unix(1_800_000_000)

    let result = validator.validate(&[cert], after_expiry, &params)

    match result {
        Err(CertValidationError.Expired { .. }) => {}  // expected
        other => panic!("expected Expired, got {other:?}")
    }
}
```

---

## Dependencies Summary

| Layer | extlib deps | stdlib deps |
|---|---|---|
| certparse | `extlib.asn1` (DER decoder, OID) | `std.alloc` (Arc, Vec, String) |
| certval | `extlib.crypto` (RSA, ECDSA, Ed25519, SHA-*) | `std.alloc`, `std.time` (Timestamp) |
| ca-nav | — | `std.alloc`, `std.time`, `std.async`, `std.net` (HttpClient) |
| truststore | — | `std.alloc`, `std.io` (file reading) |
| dane | — | `std.alloc` |

`extlib.asn1` is a strict DER/BER decoder with an OID registry and arbitrary-precision integer support. It does not expose a general-purpose ASN.1 schema compiler; it provides the low-level primitives that certparse uses.

`extlib.crypto` provides signature verification algorithms. It does not generate keys or signatures in this context — only verification operations are used by certval and ca-nav.

---

*End of extlib.cert design document.*

*See also:*
- *[ferrum_extlib_dtls.md](ferrum_extlib_dtls.md) — DTLS, which uses extlib.cert for certificate validation*
- *[ferrum_extlib_connect.md](ferrum_extlib_connect.md) — connection dialing with DANE support*
- *[ferrum_extlib_dns_secure.md](ferrum_extlib_dns_secure.md) — DNSSEC resolver, TLSA record retrieval*
- *[ferrum-stdlib-async-net.md](ferrum-stdlib-async-net.md) — HttpClient, AsyncRead, AsyncWrite*
- *[ferrum-stdlib-util.md](ferrum-stdlib-util.md) — Timestamp, Duration*
