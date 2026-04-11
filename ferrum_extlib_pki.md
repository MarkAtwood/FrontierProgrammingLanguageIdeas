# Ferrum Extended Library — PKI Module

**Module path:** `extlib.pki` (provided by `lib_ccsp_pki`)
**Part of:** Ferrum Extended Standard Library (extlib)
**RFC references:** RFC 2986 (PKCS#10 CSR), RFC 5280 (X.509), RFC 7292 (PKCS#12),
RFC 6698 (DANE/TLSA), RFC 5652 (CMS)

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [Dependencies](#2-dependencies)
3. [Key Generation](#3-key-generation)
4. [Certificate Signing Requests (CSR)](#4-certificate-signing-requests-csr)
5. [Certificate Signing](#5-certificate-signing)
6. [PKCS#12 — Certificate and Key Bundles](#6-pkcs12--certificate-and-key-bundles)
7. [Certificate Authority Operations](#7-certificate-authority-operations)
8. [DANE Helper](#8-dane-helper)
9. [Error Types](#9-error-types)
10. [Example Usage](#10-example-usage)

---

## 1. Overview and Rationale

### What the stdlib provides

The Ferrum standard library's `crypto` module provides raw asymmetric key primitives:
Ed25519 key generation and signing, X25519 key agreement, P-256 ECDSA signing, and
system CSPRNG access. These are the correct building blocks for writing cryptographic
protocols.

What the stdlib deliberately omits:

- **Certificate Signing Requests (PKCS#10, RFC 2986):** Structured DER/PEM encoding of
  a public key plus a subject name plus extensions, signed by the corresponding private key.
  Requires ASN.1 encoding and X.509 schema knowledge.
- **X.509 certificate signing:** Constructing and signing a certificate from a CSR requires
  a CA key, serial number management, extension handling, and correct DER encoding of the
  `TBSCertificate` structure.
- **PKCS#12 bundles (RFC 7292):** Encrypted containers that carry a certificate chain and
  private key together. Used by virtually every platform that imports TLS credentials.
  Requires CMS, AES-256-CBC, SHA-256 MAC, and correct safe-bag encoding.
- **CRL construction:** Building and signing certificate revocation lists for small deployments
  and testing.

### Why this is an extended library, not stdlib

PKI infrastructure is not universally needed. Embedded targets, command-line tools, and most
application code never issue or sign certificates. Putting this in the stdlib would impose
ASN.1 parsing, CMS, and BigUint arithmetic on every Ferrum binary.

The module also depends on three other extlib modules — `extlib.asn1`, `extlib.cert`, and
`extlib.crypto` (for RSA legacy support) — which themselves cannot be in the stdlib for the
same layering reasons.

The split mirrors the stdlib's own design principle: `crypto` provides raw key operations;
this module composes them into PKI workflows.

### Key algorithm defaults rationale

Algorithm choices are not neutral. This module's defaults are:

| Algorithm | Role | Rationale |
|---|---|---|
| **Ed25519** | Primary signing | Fast, small signatures (64 bytes), strong security, no parameter choices to get wrong. Deterministic — no nonce bias risk. Preferred for all new deployments. |
| **X25519** | Key agreement | The ECDH counterpart to Ed25519. Same performance profile. Used in TLS 1.3, DTLS 1.3, QUIC. |
| **ECDSA P-256** | Interoperability fallback | Required by many TLS stacks, FIDO2, and enterprise PKI infrastructure that does not yet support Ed25519. |
| **ECDSA P-384** | Higher-security interop | Used where P-256 is considered marginal (government, defense, finance). |
| **RSA-2048** | Legacy only | For infrastructure that cannot accept ECDSA or EdDSA (old Java keystores, some RADIUS servers, legacy PKCS#12 consumers). Explicit `Rsa { bits: RsaBits.Rsa2048 }` required — not a default. |
| **RSA-3072 / RSA-4096** | Legacy only | Longer key lengths for legacy contexts with longer validity periods. |
| **RSA-1024** | Absent | Broken. Not representable in the type. Code that tries to generate RSA-1024 does not compile. |

RSA-1024 is not rejected at runtime — it is absent from the `RsaBits` type. There is no
`WeakKey` variant to suppress or a configuration flag to enable it. The type system enforces
the policy.

---

## 2. Dependencies

```
extlib.pki
    extlib.asn1         — DER encoding, OID representation, ASN.1 constructed types
    extlib.cert         — Certificate, CertChain, DistinguishedName, Extension,
                          GeneralName, KeyUsage, BasicConstraints, SubjectAltName
    extlib.crypto       — RsaPrivateKey, RsaPublicKey (RSA legacy support only;
                          Ed25519/ECDSA come from std.crypto)
    std.crypto          — Ed25519, X25519, P256, P384, SystemRng, Sha256, BigUint
    std.alloc           — Vec, Arc, Box, String
    std.time            — Timestamp, Duration
```

The `extlib.cert` module provides the `Certificate` type used throughout. `extlib.pki` does
not re-implement X.509 parsing — it constructs and signs certificates using the types
`extlib.cert` defines.

---

## 3. Key Generation

### 3.1 KeyAlgorithm

```ferrum
// The set of key algorithms this module can generate.
// RSA-1024 is absent — not configurable, not suppressible, does not exist.
enum KeyAlgorithm {
    Ed25519,
    X25519,
    EcdsaP256,
    EcdsaP384,
    Rsa { bits: RsaBits },
}

// RSA key sizes available. 1024 is not a variant.
enum RsaBits {
    Rsa2048,
    Rsa3072,
    Rsa4096,
}
```

### 3.2 KeyPair

```ferrum
// A matched private/public key pair produced by generate_key_pair.
// The algorithm field records what was generated for later use in CSR signing.
type KeyPair {
    pub algorithm:   KeyAlgorithm,
    pub private_key: PrivateKey,
    pub public_key:  PublicKey,
}
```

### 3.3 generate_key_pair

```ferrum
// Generate a fresh key pair from the system CSPRNG.
//
// The ! Unsafe effect is required because this function calls the platform
// CSPRNG (SystemRng) and is marked Unsafe to make the entropy source explicit
// at call sites. Callers without ! Unsafe in their own effect set must
// explicitly escalate or delegate to an async runtime entry point that
// already holds Unsafe.
//
// RSA generation is slow (50–300 ms depending on key size and target).
// Ed25519 and ECDSA generation are fast (< 1 ms).
pub fn generate_key_pair(alg: KeyAlgorithm): Result[KeyPair, PkiError] ! Unsafe
```

### 3.4 PrivateKey

```ferrum
// Opaque wrapper around a private key.
// The in-memory representation is not accessible — callers cannot accidentally
// serialize or log a private key. Use the explicit to_pkcs8_* methods when
// persistence is intentional.
//
// Memory is zeroed on drop (equivalent to explicit_bzero / SecureZero semantics,
// resistant to dead-store elimination).
type PrivateKey   // opaque

impl PrivateKey {
    // Serialize to PKCS#8 DER (RFC 5958 OneAsymmetricKey format).
    // The returned bytes own their memory; the caller is responsible for
    // zeroing after use if passing to untrusted code.
    pub fn to_pkcs8_der(&self): Vec[u8]

    // Serialize to PKCS#8 PEM (base64-wrapped DER with -----BEGIN PRIVATE KEY-----).
    pub fn to_pkcs8_pem(&self): String

    // Parse a PKCS#8 DER private key.
    // Returns WeakKeyRejected if the encoded key is RSA < 2048 bits.
    pub fn from_pkcs8_der(bytes: &[u8]): Result[PrivateKey, PkiError]

    // Parse a PKCS#8 PEM private key (accepts one PEM block; ignores surrounding whitespace).
    pub fn from_pkcs8_pem(pem: &str): Result[PrivateKey, PkiError]

    // The algorithm of this key.
    pub fn algorithm(&self): KeyAlgorithm
}

// PrivateKey does not implement Display, Debug with key material, or Clone.
// These omissions are intentional.
impl Drop for PrivateKey {
    fn drop(&mut self)
        // Zeroes the key material before freeing memory.
}
```

### 3.5 PublicKey

```ferrum
// A public key — freely copyable and loggable.
type PublicKey {
    pub algorithm: KeyAlgorithm,
}

impl PublicKey {
    // Serialize to SubjectPublicKeyInfo DER (RFC 5480).
    pub fn to_spki_der(&self): Vec[u8]

    // Serialize to PEM (-----BEGIN PUBLIC KEY-----).
    pub fn to_pem(&self): String

    // Parse a SubjectPublicKeyInfo DER public key.
    pub fn from_spki_der(bytes: &[u8]): Result[PublicKey, PkiError]

    // Parse a PEM public key.
    pub fn from_pem(pem: &str): Result[PublicKey, PkiError]
}

impl Clone for PublicKey { ... }
impl Display for PublicKey { ... }   // displays algorithm + hex fingerprint (SHA-256 of SPKI DER)
```

---

## 4. Certificate Signing Requests (CSR)

A CSR (PKCS#10, RFC 2986) packages a public key, a subject distinguished name, and
optional extensions into a DER structure, then signs the whole thing with the corresponding
private key. The signature proves the requestor controls the private key. A CA verifies
the CSR's signature before issuing a certificate.

### 4.1 DistinguishedNameBuilder

```ferrum
// Builds an X.509 DistinguishedName (DN) from typed components.
// Avoids raw OID strings at call sites.
type DistinguishedNameBuilder

impl DistinguishedNameBuilder {
    pub fn new(): Self

    // CN — commonName (OID 2.5.4.3)
    pub fn common_name(mut self, cn: &str): Self

    // O — organizationName (OID 2.5.4.10)
    pub fn organization(mut self, org: &str): Self

    // C — countryName (OID 2.5.4.6) — must be a two-letter ISO 3166-1 alpha-2 code
    pub fn country(mut self, c: &str): Self

    // ST — stateOrProvinceName (OID 2.5.4.8)
    pub fn state(mut self, st: &str): Self

    // L — localityName (OID 2.5.4.7)
    pub fn locality(mut self, l: &str): Self

    // OU — organizationalUnitName (OID 2.5.4.11)
    pub fn organizational_unit(mut self, ou: &str): Self

    // emailAddress (OID 1.2.840.113549.1.9.1) — deprecated in certs per RFC 5280
    // but still accepted in CSR subjects for compatibility
    pub fn email(mut self, e: &str): Self

    // Build the DistinguishedName.
    // Returns Err(InvalidDn) if country is set but is not exactly 2 ASCII letters,
    // or if no attributes have been set.
    pub fn build(self): Result[DistinguishedName, PkiError]
}
```

### 4.2 GeneralName

```ferrum
// Subject Alternative Name value — matches the X.509 GeneralName type.
enum GeneralName {
    // dNSName — e.g. "example.com", "*.example.com"
    Dns(String),

    // iPAddress — IPv4 or IPv6 address (no subnet mask)
    Ip(IpAddr),

    // rfc822Name — email address, e.g. "user@example.com"
    Email(String),

    // uniformResourceIdentifier — e.g. "https://example.com"
    Uri(String),
}
```

### 4.3 KeyUsage

```ferrum
// X.509 KeyUsage bits (RFC 5280 §4.2.1.3).
// Set the bits relevant to the key's intended use.
// Ed25519 and ECDSA signing keys: DigitalSignature.
// RSA encryption keys: KeyEncipherment.
// CA keys: KeyCertSign + CrlSign.
type KeyUsage {
    pub digital_signature:  bool,   // bit 0
    pub content_commitment: bool,   // bit 1 (formerly nonRepudiation)
    pub key_encipherment:   bool,   // bit 2
    pub data_encipherment:  bool,   // bit 3
    pub key_agreement:      bool,   // bit 4
    pub key_cert_sign:      bool,   // bit 5
    pub crl_sign:           bool,   // bit 6
    pub encipher_only:      bool,   // bit 7
    pub decipher_only:      bool,   // bit 8
}

impl KeyUsage {
    // Preset: DigitalSignature only — correct for Ed25519, ECDSA TLS leaf certs
    const DIGITAL_SIGNATURE: Self

    // Preset: KeyCertSign + CrlSign — correct for CA certificates
    const CA: Self

    // Preset: DigitalSignature + KeyEncipherment — for RSA TLS leaf certs
    const RSA_TLS: Self
}
```

### 4.4 Extended Key Usage OIDs

```ferrum
// Common EKU OIDs — pass to CsrBuilder.extended_key_usage().
// Custom OIDs can be constructed via extlib.asn1.Oid.
const EKU_SERVER_AUTH:   Oid   // 1.3.6.1.5.5.7.3.1 — TLS server authentication
const EKU_CLIENT_AUTH:   Oid   // 1.3.6.1.5.5.7.3.2 — TLS client authentication
const EKU_CODE_SIGNING:  Oid   // 1.3.6.1.5.5.7.3.3 — code signing
const EKU_EMAIL_PROTECT: Oid   // 1.3.6.1.5.5.7.3.4 — email protection (S/MIME)
const EKU_TIME_STAMP:    Oid   // 1.3.6.1.5.5.7.3.8 — timestamping
const EKU_OCSP_SIGN:     Oid   // 1.3.6.1.5.5.7.3.9 — OCSP signing
```

### 4.5 CsrBuilder

```ferrum
type CsrBuilder

impl CsrBuilder {
    // Begin building a CSR. The key pair provides both the public key to embed
    // and the private key to sign the request structure with.
    pub fn new(key_pair: &KeyPair): Self

    // Set the subject DN. Required — build() returns Err(InvalidCsr) if omitted.
    pub fn subject(mut self, dn: DistinguishedNameBuilder): Self

    // Add a Subject Alternative Name extension entry.
    // Call multiple times to add multiple SANs.
    pub fn san(mut self, name: GeneralName): Self

    // Set the KeyUsage extension (marked critical).
    // If not called, KeyUsage is omitted from the CSR extensions.
    pub fn key_usage(mut self, usage: KeyUsage): Self

    // Add an Extended Key Usage OID.
    // Call multiple times for multiple EKUs.
    pub fn extended_key_usage(mut self, oid: Oid): Self

    // Add a raw DER-encoded extension (for extensions not covered by the typed API).
    pub fn raw_extension(mut self, oid: Oid, critical: bool, value: &[u8]): Self

    // Sign the CertificationRequestInfo with the key pair's private key and
    // DER-encode the result. Returns Err if subject was not set or signing failed.
    pub fn build(self): Result[Csr, PkiError]
}
```

### 4.6 Csr

```ferrum
// A parsed and signature-verified PKCS#10 CSR.
// Constructed by CsrBuilder.build() or Csr.from_der()/from_pem().
type Csr

impl Csr {
    // Parse a DER-encoded CSR. Verifies the self-signature.
    pub fn from_der(bytes: &[u8]): Result[Csr, PkiError]

    // Parse a PEM-encoded CSR (-----BEGIN CERTIFICATE REQUEST-----).
    pub fn from_pem(pem: &str): Result[Csr, PkiError]

    // Return the raw DER bytes.
    pub fn to_der(&self): &[u8]

    // Return PEM representation.
    pub fn to_pem(&self): String

    // The subject DN encoded in this CSR.
    pub fn subject(&self): &DistinguishedName

    // The public key embedded in this CSR.
    pub fn public_key(&self): &PublicKey

    // The SANs in the SubjectAltName extension, if present.
    pub fn san(&self): &[GeneralName]

    // The KeyUsage bits in the extension, if present.
    pub fn key_usage(&self): Option[KeyUsage]

    // The Extended Key Usage OIDs in the extension, if present.
    pub fn extended_key_usage(&self): &[Oid]

    // Re-verify the CSR's self-signature.
    // Returns true if valid, false if the signature does not match.
    // Note: from_der and from_pem already verify on parse; this method exists
    // for explicit re-verification after deserialization from untrusted storage.
    pub fn verify_signature(&self): bool
}
```

---

## 5. Certificate Signing

### 5.1 SigningParams

```ferrum
// Parameters controlling what a CA puts into a signed certificate.
// The CA may add or replace SANs from the CSR; the CA controls the final cert.
type SigningParams

impl SigningParams {
    pub fn new(): Self

    // Certificate validity window. Both are required; build() returns Err if omitted.
    pub fn not_before(mut self, t: Timestamp): Self
    pub fn not_after(mut self, t: Timestamp): Self

    // Serial number. Use generate_serial() to produce a cryptographically random value
    // (RFC 5280 §4.1.2.2 requires serials to be positive integers < 2^159).
    // If not set, generate_serial() is called automatically.
    pub fn serial(mut self, n: BigUint): Self

    // Mark this as a CA certificate (sets BasicConstraints cA=TRUE, critical).
    // Default: false.
    pub fn is_ca(mut self, ca: bool): Self

    // Path length constraint for CA certificates (BasicConstraints pathLenConstraint).
    // Ignored if is_ca is false.
    // None means no constraint (intermediate CAs may issue further intermediates).
    pub fn path_len_constraint(mut self, len: Option[u32]): Self

    // Copy the SANs from the CSR into the issued certificate.
    // Default: true. Set to false if the CA will supply its own SANs via override_san.
    pub fn copy_san_from_csr(mut self, copy: bool): Self

    // Replace the CSR's SANs with this explicit list (implies copy_san_from_csr false).
    // The CA is responsible for verifying domain control before calling this.
    pub fn override_san(mut self, names: Vec[GeneralName]): Self

    // Append additional SANs beyond what the CSR contains.
    // Combined with copy_san_from_csr(true) (the default), the issued cert
    // contains CSR SANs plus the additional ones.
    pub fn additional_san(mut self, name: GeneralName): Self

    // Append a raw DER-encoded extension.
    pub fn additional_extensions(mut self, exts: Vec[Extension]): Self
}
```

### 5.2 generate_serial

```ferrum
// Produce a cryptographically random serial number meeting RFC 5280 §4.1.2.2:
// a positive integer no larger than 20 octets (159 bits), using SystemRng.
pub fn generate_serial(): BigUint ! Unsafe
```

### 5.3 CertSigner

```ferrum
type CertSigner

impl CertSigner {
    // Create a signer from a CA certificate and its private key.
    // The CA cert is used as the issuer field of signed certificates.
    pub fn new(ca_cert: &Certificate, ca_key: &PrivateKey): Self

    // Sign a CSR and produce a certificate.
    // The CSR's self-signature is verified before signing.
    // Returns Err(InvalidCsr) if the CSR signature is invalid.
    pub fn sign_csr(
        &self,
        csr:    &Csr,
        params: &SigningParams,
    ): Result[Arc[Certificate], PkiError]

    // Produce a self-signed certificate from a key pair.
    // No CSR is needed — the issuer and subject are the same DN.
    // Common for DANE usage 3 leaf certificates and test infrastructure.
    // The key pair signs its own public key and the provided subject DN.
    pub fn sign_self_signed(
        key_pair: &KeyPair,
        subject:  DistinguishedNameBuilder,
        params:   &SigningParams,
    ): Result[Arc[Certificate], PkiError]
}
```

The `sign_self_signed` function is a free function in the module — it does not require an
existing CA, since the certificate is simultaneously its own issuer and subject.

---

## 6. PKCS#12 — Certificate and Key Bundles

PKCS#12 (RFC 7292) is the standard format for transporting a certificate chain and private
key together as an encrypted, authenticated bundle. It is used by Java keystores, Windows
certificate stores, iOS/macOS credential import, nginx/Apache deployment workflows, and
many RADIUS and VPN server configurations.

### 6.1 Default cipher policy

The module uses **PKCS#12 v1.1** (RFC 7292 Appendix B) cipher defaults:

| Component | Algorithm |
|---|---|
| Key encryption | AES-256-CBC + PBKDF2-HMAC-SHA-256 |
| MAC | HMAC-SHA-256 over the AuthSafe content |
| Certificate bags | AES-256-CBC |

Legacy PKCS#12 containers encrypted with RC2, 3DES, or SHA-1 MACs are **rejected on import**
with `PkiError.Pkcs12DecryptFailed` or `PkiError.Pkcs12MacFailed`. There is no option to
accept them. Operators must re-encrypt legacy containers before importing.

### 6.2 Pkcs12Builder

```ferrum
type Pkcs12Builder

impl Pkcs12Builder {
    pub fn new(): Self

    // The end-entity certificate. Required; build() returns Err if omitted.
    pub fn cert(mut self, cert: Arc[Certificate]): Self

    // The private key corresponding to the end-entity certificate. Required.
    pub fn key(mut self, key: PrivateKey): Self

    // Intermediate CA certificates to include in the chain, in order from
    // end-entity toward (but not including) the root.
    // May be called multiple times; certs are appended.
    pub fn chain(mut self, certs: Vec[Arc[Certificate]]): Self

    // Human-readable label stored in the FriendlyName SafeBag attribute.
    // Optional. Shown by keystore UIs (Keychain, certmgr.msc, etc.).
    pub fn friendly_name(mut self, name: &str): Self

    // Encryption password. Required; empty string ("") is accepted but strongly
    // discouraged — most platform keystore importers will warn or refuse it.
    pub fn password(mut self, password: &str): Self

    // Build and encrypt the PKCS#12 DER bundle.
    // The private key is consumed and zeroed after encryption.
    pub fn build(self): Result[Vec[u8], PkiError]
}
```

### 6.3 Pkcs12

```ferrum
// A decrypted and parsed PKCS#12 bundle.
type Pkcs12 {
    pub cert:          Arc[Certificate],
    pub key:           PrivateKey,
    pub chain:         Vec[Arc[Certificate]],
    pub friendly_name: Option[String],
}

impl Pkcs12 {
    // Decrypt and parse a PKCS#12 DER bundle.
    //
    // Rejects:
    //   - RC2 or 3DES encrypted SafeBags  → PkiError.Pkcs12DecryptFailed
    //   - SHA-1 MACs                       → PkiError.Pkcs12MacFailed
    //   - Wrong password (MAC mismatch)    → PkiError.Pkcs12MacFailed
    //   - RSA keys < 2048 bits             → PkiError.WeakKeyRejected
    pub fn from_der(bytes: &[u8], password: &str): Result[Pkcs12, PkiError]
}
```

---

## 7. Certificate Authority Operations

`Ca` is a higher-level wrapper combining the root (or intermediate) certificate and its
private key. It provides a unified interface for root generation, intermediate issuance,
leaf signing, and CRL construction.

### 7.1 CaParams

```ferrum
// Parameters for generating a new CA certificate (root or intermediate).
type CaParams

impl CaParams {
    pub fn new(): Self

    // Subject DN for the generated CA certificate.
    pub fn subject(mut self, dn: DistinguishedNameBuilder): Self

    // Validity period. Both required.
    pub fn not_before(mut self, t: Timestamp): Self
    pub fn not_after(mut self, t: Timestamp): Self

    // Key algorithm to generate for this CA. Default: KeyAlgorithm.Ed25519.
    pub fn key_algorithm(mut self, alg: KeyAlgorithm): Self

    // BasicConstraints pathLenConstraint (cA=TRUE is implied).
    // None means unconstrained — intermediate CAs may issue further intermediates.
    // 0 means this CA may only issue end-entity certificates.
    pub fn path_len_constraint(mut self, len: Option[u32]): Self

    // Include a Subject Key Identifier extension (recommended by RFC 5280).
    // Default: true.
    pub fn subject_key_identifier(mut self, include: bool): Self
}
```

### 7.2 Ca

```ferrum
type Ca

impl Ca {
    // Construct a Ca from an existing root certificate and private key.
    // Use this when loading a CA from persistent storage.
    pub fn new(root_cert: Arc[Certificate], root_key: PrivateKey): Self

    // Generate a new root CA: key pair + self-signed certificate.
    // Returns the Ca (for subsequent signing) and the root certificate (for distribution).
    // The root certificate is self-signed — the CA is the issuer of its own certificate.
    pub fn generate_root(
        params: &CaParams,
    ): Result[(Ca, Arc[Certificate]), PkiError] ! Unsafe

    // Issue an intermediate CA certificate signed by this Ca.
    // Returns the intermediate Ca (ready for use) and its certificate (for chain building).
    // The intermediate's key pair is freshly generated using params.key_algorithm.
    pub fn create_intermediate(
        &self,
        params: &CaParams,
    ): Result[(Ca, Arc[Certificate]), PkiError] ! Unsafe

    // Sign a CSR from an end-entity or subordinate.
    pub fn sign_csr(
        &self,
        csr:    &Csr,
        params: &SigningParams,
    ): Result[Arc[Certificate], PkiError]

    // The CA's own certificate (for including in chains).
    pub fn certificate(&self): Arc[Certificate]
}
```

### 7.3 CertRevocationList (CRL)

The CRL builder supports RFC 5280 §5 CRL construction for small deployments, test
infrastructure, and private CA scenarios. It does not implement a full CRL distribution
infrastructure — for high-volume CRL serving or OCSP, use a dedicated CA system.

```ferrum
// RevocationReason codes (RFC 5280 §5.3.1).
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

type CrlBuilder

impl CrlBuilder {
    // Begin building a CRL. The issuer certificate and key will sign the CRL.
    pub fn new(issuer: &Certificate, issuer_key: &PrivateKey): Self

    // thisUpdate field (RFC 5280 §5.1.2.4) — when this CRL was issued.
    // Required; build() returns Err if omitted.
    pub fn this_update(mut self, t: Timestamp): Self

    // nextUpdate field — when the next CRL will be issued.
    // Required by RFC 5280 for conforming CAs.
    pub fn next_update(mut self, t: Timestamp): Self

    // CRL number extension (RFC 5280 §5.2.3) — monotonically increasing integer.
    // Optional. Recommended for CAs that distribute CRLs publicly.
    pub fn crl_number(mut self, n: BigUint): Self

    // Mark a certificate as revoked.
    // May be called multiple times. Entries are sorted by serial in the output.
    pub fn revoke(
        mut self,
        serial:          BigUint,
        reason:          RevocationReason,
        revocation_time: Timestamp,
    ): Self

    // Sign and DER-encode the CRL.
    pub fn build(self): Result[Vec[u8], PkiError]
}
```

---

## 8. DANE Helper

DANE (DNS-Based Authentication of Named Entities, RFC 6698) publishes TLSA records in
DNS to bind TLS certificates to hostnames. DANE usage 3 (DANE-EE) is the simplest and
most robust mode: the TLSA record matches the end-entity certificate directly, bypassing
the WebPKI entirely. Self-signed certificates are fully valid under DANE-EE.

### 8.1 Types

```ferrum
// RFC 6698 §2.1.1 — Certificate Usage field.
enum DaneUsage {
    // Usage 0 — PKIX-TA: the TLSA record constrains which CA may sign the cert.
    PkixTrustAnchor,

    // Usage 1 — PKIX-EE: the TLSA record identifies the end-entity cert,
    //           but PKIX validation must still succeed.
    PkixEndEntity,

    // Usage 2 — DANE-TA: the TLSA record identifies a trust anchor; PKIX not required.
    DaneTrustAnchor,

    // Usage 3 — DANE-EE: the TLSA record identifies the end-entity cert directly.
    //           No PKIX validation. Self-signed certs are valid. Recommended for
    //           SMTP MTA-STS, XMPP, and private service deployments.
    DomainIssuedCert,
}

// RFC 6698 §2.1.2 — Selector field.
enum DaneSelector {
    // Selector 0 — full certificate DER.
    FullCert,

    // Selector 1 — SubjectPublicKeyInfo DER (recommended — survives cert renewal
    //              if the public key is retained).
    SubjectPublicKeyInfo,
}

// RFC 6698 §2.1.3 — Matching Type field.
enum DaneMatchingType {
    // Matching Type 0 — full byte comparison (rarely used; large DNS records).
    Full,

    // Matching Type 1 — SHA-256 of the selected content. Standard for most TLSA records.
    Sha256,

    // Matching Type 2 — SHA-512 of the selected content.
    Sha512,
}

// A TLSA record ready for publication in DNS.
// The caller is responsible for formatting this into a zone file or DNS API call.
type TlsaRecord {
    pub usage:         DaneUsage,
    pub selector:      DaneSelector,
    pub matching_type: DaneMatchingType,
    // The certificate association data — the raw bytes, SHA-256, or SHA-512 of
    // the selected content, depending on matching_type.
    pub data:          Vec[u8],
}

impl TlsaRecord {
    // Format as a DNS RDATA presentation string, e.g.:
    //   "3 1 1 abc123..."
    // Ready to insert into a zone file or DNS API.
    pub fn to_rdata_string(&self): String
}
```

### 8.2 dane_tlsa_from_cert

```ferrum
// Compute a TLSA record from a certificate.
//
// Common usage: DANE-EE with SPKI selector and SHA-256:
//   dane_tlsa_from_cert(&cert, DaneUsage.DomainIssuedCert,
//                       DaneSelector.SubjectPublicKeyInfo,
//                       DaneMatchingType.Sha256)
//
// The SubjectPublicKeyInfo selector is preferred over FullCert because it allows
// certificate renewal (rotating the validity period) without changing the TLSA
// record, as long as the public key is preserved.
pub fn dane_tlsa_from_cert(
    cert:          &Certificate,
    usage:         DaneUsage,
    selector:      DaneSelector,
    matching_type: DaneMatchingType,
): TlsaRecord
```

---

## 9. Error Types

```ferrum
enum PkiError {
    // Key generation failed — usually a CSPRNG failure.
    KeyGenerationFailed(String),

    // The provided key bytes could not be parsed or are structurally invalid.
    InvalidKey(String),

    // Signing operation failed (e.g., key algorithm mismatch, hardware token error).
    SignatureFailed(String),

    // PKCS#12 decryption failed: wrong password, or the container uses a rejected
    // legacy cipher (RC2, 3DES).
    Pkcs12DecryptFailed,

    // PKCS#12 MAC verification failed: wrong password, or the container uses
    // a SHA-1 MAC (rejected by policy).
    Pkcs12MacFailed,

    // Key rejected because it is too weak. Triggered by RSA keys under 2048 bits
    // found during from_pkcs8_der or Pkcs12::from_der import.
    // RSA-1024 cannot be generated (it is absent from RsaBits), but it can arrive
    // via import; that path is where this error fires.
    WeakKeyRejected { algorithm: String, bits: usize },

    // CSR self-signature invalid, or required CSR fields (subject) missing.
    InvalidCsr(String),

    // PEM block was malformed, had the wrong label, or contained invalid base64.
    InvalidPem(String),

    // ASN.1 DER encoding or decoding error. Wraps the extlib.asn1 error.
    Asn1Error(Asn1Error),

    // The provided distinguished name had an invalid field (e.g., country not 2 letters).
    InvalidDn(String),

    // Signing parameters were incomplete (e.g., not_before/not_after not set).
    InvalidParams(String),
}

impl Display for PkiError { ... }
impl Error   for PkiError { ... }
```

---

## 10. Example Usage

### 10.1 Generate an Ed25519 CA and issue a server certificate

```ferrum
use extlib.pki.{
    Ca, CaParams, CsrBuilder, SigningParams,
    DistinguishedNameBuilder, GeneralName, KeyAlgorithm,
    EKU_SERVER_AUTH, generate_serial,
}
use std.time.Timestamp

fn build_pki(): Result[(), PkiError] ! Unsafe {
    // Generate root CA — Ed25519 by default
    let ca_params = CaParams.new()
        .subject(
            DistinguishedNameBuilder.new()
                .common_name("Example Root CA")
                .organization("Example Corp")
                .country("US")
        )
        .not_before(Timestamp.now())
        .not_after(Timestamp.now().add_years(10))
        .path_len_constraint(Some(1))

    let (root_ca, root_cert) = Ca.generate_root(&ca_params)?

    // Generate server key pair and CSR
    let server_kp = generate_key_pair(KeyAlgorithm.Ed25519)?

    let csr = CsrBuilder.new(&server_kp)
        .subject(
            DistinguishedNameBuilder.new()
                .common_name("api.example.com")
                .organization("Example Corp")
                .country("US")
        )
        .san(GeneralName.Dns("api.example.com".to_string()))
        .san(GeneralName.Dns("api-staging.example.com".to_string()))
        .extended_key_usage(EKU_SERVER_AUTH)
        .build()?

    // CA signs the CSR
    let signing_params = SigningParams.new()
        .not_before(Timestamp.now())
        .not_after(Timestamp.now().add_years(1))
        .serial(generate_serial()?)
        .copy_san_from_csr(true)

    let server_cert = root_ca.sign_csr(&csr, &signing_params)?

    // server_cert is Arc[Certificate], ready for use with extlib.tls
    log.info("issued: subject={server_cert.subject()} serial={server_cert.serial()}")

    Ok(())
}
```

### 10.2 Export a cert+key bundle to PKCS#12

```ferrum
use extlib.pki.{Pkcs12Builder, KeyAlgorithm, generate_key_pair}
use std.fs

fn export_pkcs12(
    cert:  Arc[Certificate],
    key:   PrivateKey,
    chain: Vec[Arc[Certificate]],
    path:  &str,
): Result[(), PkiError] ! Unsafe + IO {
    let der = Pkcs12Builder.new()
        .cert(cert)
        .key(key)
        .chain(chain)
        .friendly_name("api.example.com")
        .password("correct-horse-battery-staple")
        .build()?

    fs.write(path, &der).map_err(|e| PkiError.Asn1Error(e.into()))?
    Ok(())
}
```

### 10.3 Import a PKCS#12 bundle

```ferrum
use extlib.pki.{Pkcs12, PkiError}
use std.fs

fn load_pkcs12(path: &str, password: &str): Result[Pkcs12, PkiError] ! IO {
    let bytes = fs.read(path).map_err(|e| PkiError.InvalidPem(e.to_string()))?
    // Rejects RC2, 3DES, SHA-1 MACs, and RSA < 2048 automatically.
    Pkcs12.from_der(&bytes, password)
}
```

### 10.4 Create a self-signed certificate for DANE usage 3

DANE-EE (usage 3) does not require a CA chain. The certificate is self-signed and
identified directly by the TLSA record in DNS. Combined with DNSSEC, this provides
strong authentication without the WebPKI.

```ferrum
use extlib.pki.{
    CertSigner, CaParams, SigningParams, DistinguishedNameBuilder,
    KeyAlgorithm, GeneralName, KeyUsage, EKU_SERVER_AUTH,
    generate_key_pair, generate_serial,
    dane_tlsa_from_cert, DaneUsage, DaneSelector, DaneMatchingType,
}
use std.time.Timestamp

fn create_dane_cert(hostname: &str): Result[(), PkiError] ! Unsafe {
    let kp = generate_key_pair(KeyAlgorithm.Ed25519)?

    let subject = DistinguishedNameBuilder.new()
        .common_name(hostname)

    let params = SigningParams.new()
        .not_before(Timestamp.now())
        .not_after(Timestamp.now().add_years(2))
        .serial(generate_serial()?)
        .override_san(vec![GeneralName.Dns(hostname.to_string())])

    // Self-signed — no CA needed for DANE-EE
    let cert = CertSigner.sign_self_signed(&kp, subject, &params)?

    // Compute the TLSA record (3 1 1 — DANE-EE, SPKI, SHA-256)
    let tlsa = dane_tlsa_from_cert(
        &cert,
        DaneUsage.DomainIssuedCert,
        DaneSelector.SubjectPublicKeyInfo,
        DaneMatchingType.Sha256,
    )

    // Publish tlsa.to_rdata_string() to DNS for _443._tcp.<hostname>
    log.info("TLSA RDATA: {tlsa.to_rdata_string()}")
    log.info("Publish to DNS: _443._tcp.{hostname}. IN TLSA {tlsa.to_rdata_string()}")

    Ok(())
}
```

### 10.5 Build a two-tier CA hierarchy (root + intermediate)

```ferrum
use extlib.pki.{Ca, CaParams, SigningParams, CsrBuilder,
                 DistinguishedNameBuilder, GeneralName,
                 KeyAlgorithm, EKU_CLIENT_AUTH, generate_serial}
use std.time.Timestamp

fn two_tier_ca(): Result[(), PkiError] ! Unsafe {
    // Root CA — 10 year validity, constrained to issue only intermediates
    let root_params = CaParams.new()
        .subject(DistinguishedNameBuilder.new()
            .common_name("Corp Root CA")
            .organization("Corp")
            .country("DE"))
        .not_before(Timestamp.now())
        .not_after(Timestamp.now().add_years(10))
        .path_len_constraint(Some(0))   // intermediates may not issue further CAs

    let (root_ca, _root_cert) = Ca.generate_root(&root_params)?

    // Intermediate CA — 5 year validity, path length 0 (issues leaf certs only)
    let int_params = CaParams.new()
        .subject(DistinguishedNameBuilder.new()
            .common_name("Corp Device CA")
            .organization("Corp")
            .country("DE"))
        .not_before(Timestamp.now())
        .not_after(Timestamp.now().add_years(5))
        .path_len_constraint(Some(0))

    let (intermediate_ca, _int_cert) = root_ca.create_intermediate(&int_params)?

    // Issue a client certificate from the intermediate
    let device_kp = generate_key_pair(KeyAlgorithm.EcdsaP256)?
    let csr = CsrBuilder.new(&device_kp)
        .subject(DistinguishedNameBuilder.new()
            .common_name("device-42.corp.example")
            .organization("Corp"))
        .extended_key_usage(EKU_CLIENT_AUTH)
        .build()?

    let leaf_params = SigningParams.new()
        .not_before(Timestamp.now())
        .not_after(Timestamp.now().add_years(1))
        .serial(generate_serial()?)

    let _leaf_cert = intermediate_ca.sign_csr(&csr, &leaf_params)?

    Ok(())
}
```

---

*End of extlib.pki design document.*

*See also:*
- *[ferrum_extlib_dtls.md](ferrum_extlib_dtls.md) — DTLS module (uses extlib.pki for certificate handling)*
- *[ferrum_extlib_connect.md](ferrum_extlib_connect.md) — connect module (DANE enforcement at connection time)*
- *[ferrum_extlib_dns_secure.md](ferrum_extlib_dns_secure.md) — DNSSEC resolver (required for DANE TLSA validation)*
- *[ferrum-stdlib-crypto-testing.md](ferrum-stdlib-crypto-testing.md) — stdlib crypto primitives (Ed25519, ECDSA, SystemRng)*
- *[ferrum-stdlib.md](ferrum-stdlib.md) — Standard library index*
