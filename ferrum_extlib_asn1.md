# Ferrum Extended Library — asn1: ASN.1/DER Codec

**Module path:** `extlib.asn1`
**Layer:** Extended standard library (not stdlib; requires optional dependency)
**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)

---

## 1. Overview and Rationale

### What ASN.1 Is

Abstract Syntax Notation One (ASN.1) is an ITU-T standard for describing data structures exchanged between systems, independent of machine architecture and programming language. It is the substrate of PKI: X.509 certificates, PKCS#8 private key files, PKCS#12 keystores, CMS (Cryptographic Message Syntax) signed and enveloped data, OCSP, and the structures inside TLS all encode their wire format in ASN.1.

ASN.1 defines both the notation (the schema language) and a family of encoding rules. The three most common encoding rule families are:

- **BER (Basic Encoding Rules):** The original family. Allows multiple valid encodings for the same value (indefinite-length form, non-minimal integers, extra leading zero bytes). Flexible but ambiguous.
- **DER (Distinguished Encoding Rules):** A strict subset of BER. Every value has exactly one valid encoding. Required by X.509, PKCS, and TLS because signature verification requires byte-exact canonicality: signing a certificate and verifying it must hash the same bytes.
- **CER (Canonical Encoding Rules):** An alternative canonical subset of BER, used by some LDAP implementations. Not used in PKI.

### Why DER Is the Safe Default

DER's value is canonicality. An X.509 certificate's signature covers the DER encoding of the TBSCertificate structure. If a library accepts BER-encoded certificates and normalizes them silently, two different byte strings can represent the same logical certificate — which breaks signature verification and opens the door to certificate substitution attacks. DER eliminates this class of bug by making every valid structure have exactly one encoding.

This library treats DER as the default and correct encoding rule set. BER parsing is available as an explicit opt-in for legacy data (see section 7). Code that does not opt in cannot produce or consume BER-only encodings.

### Why Extlib, Not Stdlib

The Ferrum standard library covers cryptographic primitives (hashing, HMAC, key derivation, symmetric encryption, signatures) that are widely needed and have stable, bounded interfaces. ASN.1 is not in that category for two reasons.

**Scope:** The only programs that need ASN.1 are those working with PKI — TLS stack implementations, certificate authorities, PKCS key import/export, CMS signers/verifiers. This is a small fraction of all Ferrum programs. Imposing the codec and all its associated types on every binary would be hostile to programs that never touch a certificate.

**Dependency scope:** A correct, hardened ASN.1 decoder has significant implementation surface and depends on `extlib.bigint` for arbitrary-precision integers. Keeping these as explicit extlib dependencies makes the cost visible and controllable.

Programs implementing TLS, PKCS, CMS, or X.509 opt in explicitly:

```toml
[dependencies]
extlib.asn1 = { version = "1.0" }
```

### No Code Generation

Many ASN.1 toolchains provide a compiler: you write a `.asn` schema file, run the compiler, and get generated source code. `extlib.asn1` takes no such approach.

There is no separate schema language. There is no schema compiler. There is no generated code. Schemas are expressed directly in Ferrum, using the types and traits defined in this module. A SEQUENCE is a Ferrum struct with `#[derive(DerDecode, DerEncode)]`. A CHOICE is a Ferrum enum. Tagging is expressed via `Explicit` and `Implicit` wrapper types. The result is schemas that are checked by the Ferrum compiler, refactorable with standard tools, and readable without knowledge of a separate DSL.

---

## 2. Core Types

### 2.1 `Oid` — Object Identifier

An ASN.1 OBJECT IDENTIFIER is a sequence of non-negative integers (arcs) that uniquely identifies an object in a globally-registered tree. OIDs appear in `AlgorithmIdentifier` (which hash or signature algorithm), attribute types in Distinguished Names, extended key usage extensions, and many other PKI structures.

```ferrum
/// An ASN.1 OBJECT IDENTIFIER.
///
/// Stored as a compact arc sequence. Immutable after construction.
/// Implements Eq, Hash, Display (dotted notation), Debug.
pub struct Oid { ... }

impl Oid {
    /// Parse from DER-encoded bytes (tag + length + value).
    pub fn parse_der(input: &[u8]): Result[(Self, &[u8]), Asn1Error]

    /// Encode to DER bytes (tag + length + value).
    pub fn to_der(self: &Self): Vec[u8] ! Alloc

    /// Dotted-decimal string representation, e.g. "1.2.840.113549.1.1.11".
    pub fn to_string(self: &Self): String ! Alloc

    /// The arc sequence as a slice of u32 values.
    /// The first two arcs are packed (first * 40 + second) in DER encoding
    /// but are unpacked here for clarity.
    pub fn arcs(self: &Self): &[u32]
}

impl Eq for Oid { ... }
impl Hash for Oid { ... }
impl Display for Oid { ... }
impl Debug for Oid { ... }
```

OIDs can be constructed as constants using the `oid!` compiler intrinsic, which validates the arc sequence and builds the internal representation at compile time:

```ferrum
// Compile-time OID constants
const OID_RSA_ENCRYPTION:       Oid = oid!(1, 2, 840, 113549, 1, 1, 1)
const OID_SHA256_WITH_RSA:      Oid = oid!(1, 2, 840, 113549, 1, 1, 11)
const OID_EC_PUBLIC_KEY:        Oid = oid!(1, 2, 840, 10045, 2, 1)
const OID_ECDSA_WITH_SHA256:    Oid = oid!(1, 2, 840, 10045, 4, 3, 2)
const OID_SHA256:               Oid = oid!(2, 16, 840, 1, 101, 3, 4, 2, 1)
const OID_SHA384:               Oid = oid!(2, 16, 840, 1, 101, 3, 4, 2, 2)
const OID_SHA512:               Oid = oid!(2, 16, 840, 1, 101, 3, 4, 2, 3)
const OID_COMMON_NAME:          Oid = oid!(2, 5, 4, 3)
const OID_COUNTRY:              Oid = oid!(2, 5, 4, 6)
const OID_ORGANIZATION:         Oid = oid!(2, 5, 4, 10)
const OID_SUBJECT_KEY_ID:       Oid = oid!(2, 5, 29, 14)
const OID_KEY_USAGE:            Oid = oid!(2, 5, 29, 15)
const OID_BASIC_CONSTRAINTS:    Oid = oid!(2, 5, 29, 19)
const OID_EXT_KEY_USAGE:        Oid = oid!(2, 5, 29, 37)
const OID_PKCS8_VERSION:        Oid = oid!(1, 2, 840, 113549, 1, 9, 14)
```

The `oid!` form is a built-in compiler intrinsic, not a user-defined macro. No user-defined macros exist in Ferrum.

### 2.2 `Tag` — ASN.1 Tag

```ferrum
/// ASN.1 tag class.
pub enum TagClass {
    Universal,
    Application,
    ContextSpecific,
    Private,
}

/// A fully-decoded ASN.1 tag.
pub struct Tag {
    pub class:       TagClass,
    pub constructed: bool,
    pub number:      u32,   // tag number; high-tag-number form supported
}

impl Tag {
    /// Encode to wire form (one or more bytes per X.690 §8.1.2).
    pub fn to_bytes(self: &Self): [u8; 4]   // max 4 bytes; actual len in result

    /// Universal tag for SEQUENCE (0x10, constructed).
    pub const SEQUENCE:     Tag = Tag { class: TagClass.Universal, constructed: true,  number: 16 }
    /// Universal tag for SET (0x11, constructed).
    pub const SET:          Tag = Tag { class: TagClass.Universal, constructed: true,  number: 17 }
    /// Universal tag for OCTET STRING (0x04, primitive).
    pub const OCTET_STRING: Tag = Tag { class: TagClass.Universal, constructed: false, number: 4  }
    /// Universal tag for OID (0x06, primitive).
    pub const OID:          Tag = Tag { class: TagClass.Universal, constructed: false, number: 6  }
    /// Universal tag for NULL (0x05, primitive).
    pub const NULL:         Tag = Tag { class: TagClass.Universal, constructed: false, number: 5  }
    /// Universal tag for BOOLEAN (0x01, primitive).
    pub const BOOLEAN:      Tag = Tag { class: TagClass.Universal, constructed: false, number: 1  }
    /// Universal tag for INTEGER (0x02, primitive).
    pub const INTEGER:      Tag = Tag { class: TagClass.Universal, constructed: false, number: 2  }
    /// Universal tag for BIT STRING (0x03, primitive).
    pub const BIT_STRING:   Tag = Tag { class: TagClass.Universal, constructed: false, number: 3  }
    /// Universal tag for UTF8String (0x0C, primitive).
    pub const UTF8_STRING:  Tag = Tag { class: TagClass.Universal, constructed: false, number: 12 }
}

impl Eq for Tag { ... }
impl Debug for Tag { ... }
impl Display for Tag { ... }
```

### 2.3 `Length` — Definite Length

```ferrum
/// A decoded ASN.1 length value.
///
/// DER mandates definite-length encoding. Indefinite-length form (0x80)
/// is rejected by DerReader and returned as Asn1Error::IndefiniteLength.
/// BerReader accepts indefinite lengths.
pub struct Length {
    value: usize,
}

impl Length {
    pub fn new(value: usize): Self
    pub fn value(self: &Self): usize { self.value }

    /// Encode as DER length bytes (short or long form).
    pub fn to_bytes(self: &Self): [u8; 5]   // max 5 bytes
}
```

### 2.4 `AnyRef<'a>` — Zero-Copy Opaque ANY

```ferrum
/// A borrowed slice of DER bytes representing an ASN.1 ANY value.
///
/// Captures the tag, length, and value octets together, zero-copy.
/// Used to defer decoding of ANY DEFINED BY fields until the OID
/// is known, and to represent unknown extension values.
///
/// Does not allocate. Lifetime is tied to the input buffer.
pub struct AnyRef['a] {
    tag:   Tag,
    bytes: &'a [u8],    // full TLV (tag + length + value)
}

impl['a] AnyRef['a] {
    /// The tag of this value.
    pub fn tag(self: &Self): Tag { self.tag }

    /// The full DER encoding (tag + length + value bytes).
    pub fn as_bytes(self: &Self): &'a [u8] { self.bytes }

    /// The value octets only (length and tag stripped).
    pub fn value_bytes(self: &Self): &'a [u8]

    /// Decode this AnyRef as a specific type.
    pub fn decode['b, T: DerDecode['b]](self: &'b Self): Result[T, Asn1Error]
        where 'a: 'b
}
```

### 2.5 `Explicit[T, TAG]` — Explicit Context Tagging

```ferrum
/// Wrapper for an explicitly context-tagged value.
///
/// ASN.1 explicit tagging wraps the inner encoding in an additional
/// TL (tag + length) envelope with the given context tag number.
///
/// Example: [0] EXPLICIT INTEGER  →  Explicit[Integer, 0]
pub struct Explicit[T, const TAG: u8] {
    pub inner: T,
}

impl[T: DerEncode, const TAG: u8] DerEncode for Explicit[T, TAG] { ... }
impl['a, T: DerDecode['a], const TAG: u8] DerDecode['a] for Explicit[T, TAG] { ... }
```

### 2.6 `Implicit[T, TAG]` — Implicit Context Tagging

```ferrum
/// Wrapper for an implicitly context-tagged value.
///
/// ASN.1 implicit tagging replaces the inner type's universal tag with
/// the given context tag number, keeping the inner value encoding.
///
/// Example: [1] IMPLICIT OCTET STRING  →  Implicit[OctetString, 1]
///
/// Note: implicit tagging of CHOICE or ANY is not permitted in ASN.1
/// and will produce a compile-time error via trait bounds.
pub struct Implicit[T: ImplicitTaggable, const TAG: u8] {
    pub inner: T,
}

/// Marker trait for types that support implicit tagging.
/// NOT implemented by ChoiceValue or AnyRef (which cannot be implicitly tagged).
pub marker trait ImplicitTaggable {}

impl[T: DerEncode + ImplicitTaggable, const TAG: u8] DerEncode for Implicit[T, TAG] { ... }
impl['a, T: DerDecode['a] + ImplicitTaggable, const TAG: u8] DerDecode['a] for Implicit[T, TAG] { ... }
```

---

## 3. DER/BER Encoding Traits

### 3.1 `DerEncode`

```ferrum
/// Types that can encode themselves to DER.
pub trait DerEncode {
    /// Encode to a new byte vector.
    fn der_encode(self: &Self): Vec[u8] ! Alloc

    /// Encode into an existing byte vector, appending.
    /// Preferred in hot paths to avoid repeated allocation.
    fn der_encode_to(self: &Self, buf: &mut Vec[u8]) ! Alloc
}
```

### 3.2 `DerDecode`

```ferrum
/// Types that can decode themselves from DER.
///
/// The lifetime 'a refers to the input buffer. Types that borrow from
/// the input (e.g., AnyRef, &[u8] fields) carry 'a. Owned types
/// implement DerDecode['static].
pub trait DerDecode['a]: Sized {
    /// Decode from the front of `input`.
    ///
    /// Returns (value, remaining_bytes) on success.
    /// Returns Asn1Error on any encoding violation.
    /// Rejects trailing data within the value's scope but not beyond it —
    /// the caller decides whether trailing bytes after the value are an error.
    fn der_decode(input: &'a [u8]): Result[(Self, &'a [u8]), Asn1Error]
}
```

### 3.3 `#[derive(DerEncode, DerDecode)]`

The compiler can derive `DerEncode` and `DerDecode` for structs and enums that follow ASN.1 SEQUENCE and CHOICE patterns, respectively.

**Struct (SEQUENCE):**

```ferrum
/// A PKCS#10 CertificationRequestInfo as an example schema.
#[derive(DerEncode, DerDecode)]
pub struct CertificationRequestInfo {
    pub version:             Integer,
    pub subject:             Name,
    pub subject_public_key:  SubjectPublicKeyInfo,
    // [0] IMPLICIT Attributes OPTIONAL
    pub attributes:          Option[Implicit[Attributes, 0]],
}
```

The derive macro generates:

- `DerEncode`: encodes each field in declaration order inside a SEQUENCE TLV.
- `DerDecode`: reads a SEQUENCE TLV, then reads each field in order; `Option[...]` fields use `read_optional`.

**Enum (CHOICE):**

```ferrum
#[derive(DerEncode, DerDecode)]
pub enum Time {
    UtcTime(UtcTime),
    GeneralizedTime(GeneralizedTime),
}
```

The derive reads the next tag from the input and dispatches to the matching variant. All variants must have distinct tags.

**Limitations:** The derive does not handle complex DEFAULT values, SET OF with canonical ordering checks, or SEQUENCE OF with length constraints. These require manual `impl` blocks.

---

## 4. Built-In ASN.1 Types

All types in this section implement both `DerEncode` and `DerDecode`.

### 4.1 Primitive Scalar Types

```ferrum
/// ASN.1 BOOLEAN.
/// DER requires 0xFF for TRUE and 0x00 for FALSE.
/// BER accepts any non-zero byte for TRUE; NonCanonical error in DER mode.
pub struct Boolean { pub value: bool }

/// ASN.1 INTEGER with arbitrary precision.
/// Backed by extlib.bigint.BigInt. Can represent RSA moduli and exponents.
/// DER requires minimal encoding (no unnecessary leading zero or 0xFF bytes).
pub struct Integer { inner: BigInt }

impl Integer {
    pub fn from_i64(v: i64): Self
    pub fn from_u64(v: u64): Self
    pub fn from_bigint(v: BigInt): Self
    pub fn to_i64(self: &Self): Result[i64, Asn1Error]
    pub fn to_u64(self: &Self): Result[u64, Asn1Error]
    pub fn as_bigint(self: &Self): &BigInt
}

/// ASN.1 BIT STRING.
/// Stores the unused-bits count and the byte content.
/// DER requires unused bits to be zero.
pub struct BitString {
    pub unused_bits: u8,     // 0–7; number of unused bits in the final byte
    pub bytes:       Vec[u8],
}

impl BitString {
    pub fn from_bytes(bytes: Vec[u8]): Self          // unused_bits = 0
    pub fn from_bytes_unused(bytes: Vec[u8], unused: u8): Result[Self, Asn1Error]
    pub fn as_bytes(self: &Self): &[u8]
    pub fn bit_len(self: &Self): usize
    pub fn get_bit(self: &Self, index: usize): Option[bool]
}

/// ASN.1 OCTET STRING.
pub struct OctetString { pub bytes: Vec[u8] }

impl OctetString {
    pub fn new(bytes: Vec[u8]): Self
    pub fn as_bytes(self: &Self): &[u8]
    pub fn into_bytes(self: Self): Vec[u8]
}

/// ASN.1 NULL. No value; encodes as 05 00.
pub type Null
```

### 4.2 String Types

ASN.1 defines many character string types, the legacy of an era when character encoding was not settled. All of them decode to `String` (Ferrum's UTF-8 type). Encoders produce the correct wire encoding from a `String`.

```ferrum
pub struct Utf8String      { pub value: String }   // Universal tag 12
pub struct PrintableString { pub value: String }   // Universal tag 19; subset of ASCII
pub struct Ia5String       { pub value: String }   // Universal tag 22; 7-bit ASCII
pub struct BmpString       { pub value: String }   // Universal tag 30; UCS-2 (UTF-16 BE)
pub struct UniversalString { pub value: String }   // Universal tag 28; UCS-4
pub struct VisibleString   { pub value: String }   // Universal tag 26; visible ASCII
pub struct GeneralString   { pub value: String }   // Universal tag 27; ISO 646
pub struct TeletexString   { pub value: String }   // Universal tag 20; T.61; legacy only
```

Decoding notes:
- `BmpString` wire bytes are UTF-16 BE; the decoder converts to UTF-8. Surrogate pairs are rejected.
- `UniversalString` wire bytes are UCS-4 (big-endian code points); decoded to UTF-8.
- `PrintableString` decoding validates that the input contains only the PrintableString character set; any other byte is `Asn1Error::InvalidValue`.
- `Ia5String` decoding validates that all bytes are in the range 0x00–0x7F.
- `TeletexString` decoding accepts bytes but performs best-effort T.61-to-UTF-8 conversion; avoid in new code.

### 4.3 Time Types

```ferrum
use std.time.Timestamp

/// ASN.1 UTCTime. Format: YYMMDDHHMMSSZ or with timezone offset.
/// Years 00–49 are interpreted as 2000–2049; 50–99 as 1950–1999 (RFC 5280 §4.1.2.5.1).
/// DER requires the Z (UTC) suffix.
pub struct UtcTime { pub value: Timestamp }

/// ASN.1 GeneralizedTime. Format: YYYYMMDDHHMMSS[.fff]Z.
/// DER prohibits fractional seconds and requires the Z suffix.
pub struct GeneralizedTime { pub value: Timestamp }
```

### 4.4 Constructed Types

```ferrum
/// ASN.1 SEQUENCE — a fixed-ordered collection of fields.
/// Use #[derive(DerDecode, DerEncode)] on structs for SEQUENCE schemas.
/// This type is the generic SEQUENCE OF wrapper.
pub struct SequenceOf[T] { pub items: Vec[T] }

impl[T: DerEncode] DerEncode for SequenceOf[T] { ... }
impl['a, T: DerDecode['a]] DerDecode['a] for SequenceOf[T] { ... }

/// ASN.1 SET OF — like SEQUENCE OF, but DER requires elements sorted
/// by encoded byte value (X.690 §11.6).
pub struct SetOf[T] { pub items: Vec[T] }

impl[T: DerEncode] DerEncode for SetOf[T] { ... }
impl['a, T: DerDecode['a]] DerDecode['a] for SetOf[T] { ... }

/// ASN.1 OPTIONAL field wrapper inside a SEQUENCE.
/// In derive context, use Option[T] directly in the struct field.
/// This type alias makes the intent explicit in manual impl blocks.
pub type Optional[T] = Option[T]
```

---

## 5. DER Reader

`DerReader` provides a cursor-based API for reading a DER-encoded input buffer. It enforces strict DER at every read. Nesting is tracked by depth counter; any input that would exceed 64 levels of nesting is immediately rejected as an attack.

```ferrum
/// A DER reader for a borrowed byte slice.
///
/// The reader maintains:
///   - a cursor into the input
///   - a nesting depth counter (max 64)
///
/// All methods reject indefinite-length encoding, non-canonical
/// tag/length encodings, and out-of-bounds lengths.
pub struct DerReader['a] { ... }

impl['a] DerReader['a] {
    /// Construct a reader for the given bytes.
    pub fn new(input: &'a [u8]): Self

    /// True if all input has been consumed.
    pub fn is_empty(self: &Self): bool

    /// Peek at the tag of the next TLV without consuming it.
    /// Returns Asn1Error::UnexpectedEof if the input is empty.
    pub fn peek_tag(self: &Self): Result[Tag, Asn1Error]

    /// Read and decode the next element as type T.
    ///
    /// Advances the cursor past the TLV. Returns the decoded value.
    pub fn read_element[T: DerDecode['a]](self: &mut Self): Result[T, Asn1Error]

    /// Read the next element as T if the next tag matches; otherwise return None.
    ///
    /// Does not advance the cursor on None.
    pub fn read_optional[T: DerDecode['a]](self: &mut Self): Result[Option[T], Asn1Error]

    /// Read a SEQUENCE TLV and call `f` with a reader scoped to the sequence body.
    ///
    /// Increments the nesting depth counter. Returns Asn1Error::NestingTooDeep
    /// at depth 64 without calling `f`.
    pub fn read_sequence[T](
        self: &mut Self,
        f: impl Fn(&mut DerReader['a]): Result[T, Asn1Error],
    ): Result[T, Asn1Error]

    /// Read a SET TLV and call `f` with a reader scoped to the set body.
    ///
    /// Increments the nesting depth counter.
    pub fn read_set[T](
        self: &mut Self,
        f: impl Fn(&mut DerReader['a]): Result[T, Asn1Error],
    ): Result[T, Asn1Error]

    /// Read an explicitly context-tagged value.
    ///
    /// Expects a context-specific tag with `tag_number`, then calls `f`
    /// with a reader scoped to the tagged content.
    pub fn read_context_explicit[T](
        self:       &mut Self,
        tag_number: u8,
        f:          impl Fn(&mut DerReader['a]): Result[T, Asn1Error],
    ): Result[T, Asn1Error]

    /// Read the next TLV as a raw AnyRef without decoding it.
    ///
    /// Returns a zero-copy reference into the input. Use for ANY or
    /// ANY DEFINED BY fields where the type is determined by context.
    pub fn read_any(self: &mut Self): Result[AnyRef['a], Asn1Error]

    /// Current nesting depth. 0 at the top level.
    pub fn depth(self: &Self): u8

    /// Maximum nesting depth. Hard-coded to 64. Not configurable.
    pub const MAX_DEPTH: u8 = 64
}
```

The 64-level nesting limit is not configurable. Any legitimate ASN.1 structure used in PKI fits well within this limit. Accepting deeper nesting would invite stack exhaustion and recursive descent amplification attacks.

---

## 6. ANY DEFINED BY with OID Dispatch

ASN.1's `ANY DEFINED BY` construct allows a value's type to be determined by another field — typically an OID naming an algorithm or attribute type. A dispatch table maps OIDs to decoder functions.

### 6.1 `OidDispatch[T]`

```ferrum
/// A table mapping OIDs to decoder functions for ANY DEFINED BY dispatch.
///
/// `T` is the target type produced by each decoder.
/// Built once and typically stored as a static.
pub struct OidDispatch[T] { ... }

impl[T] OidDispatch[T] {
    /// Construct a dispatch table from a slice of (OID, decoder) pairs.
    pub fn new(entries: &[(Oid, fn(&[u8]): Result[T, Asn1Error])]): Self ! Alloc

    /// Look up a decoder for `oid`.
    pub fn get(self: &Self, oid: &Oid): Option[fn(&[u8]): Result[T, Asn1Error]]
}
```

### 6.2 `dispatch_any_defined_by`

```ferrum
/// Decode an AnyRef using the decoder registered for `oid` in `table`.
///
/// Returns Asn1Error::UnknownOid if the OID is not in the table.
/// The UnknownOid error carries the OID so the caller can decide
/// whether to log it, skip it, or propagate it.
///
/// Unknown OIDs are surfaced, never swallowed silently.
pub fn dispatch_any_defined_by['a, T](
    oid:   &Oid,
    any:   AnyRef['a],
    table: &OidDispatch[T],
): Result[T, Asn1Error]
```

### 6.3 Example: AlgorithmIdentifier Parameters Dispatch

```ferrum
use extlib.asn1.{
    self, AnyRef, Asn1Error, Null, OctetString, Oid, OidDispatch,
    dispatch_any_defined_by,
}

// The set of algorithm parameter types we understand.
enum AlgParams {
    Absent,
    Null,
    EcNamedCurve(Oid),
    RsaPssParams(RsaPssParams),
}

fn decode_ec_params(bytes: &[u8]): Result[AlgParams, Asn1Error] {
    let (oid, _) = Oid.parse_der(bytes)?
    Ok(AlgParams.EcNamedCurve(oid))
}

fn decode_rsa_pss_params(bytes: &[u8]): Result[AlgParams, Asn1Error] {
    let (params, _) = RsaPssParams.der_decode(bytes)?
    Ok(AlgParams.RsaPssParams(params))
}

static PARAM_DISPATCH: OidDispatch[AlgParams] = OidDispatch.new(&[
    (OID_EC_PUBLIC_KEY, decode_ec_params),
    (OID_RSA_PSS,       decode_rsa_pss_params),
])

fn decode_alg_params(algorithm: &Oid, params: Option[AnyRef]): Result[AlgParams, Asn1Error] {
    match params {
        None => Ok(AlgParams.Absent),
        Some(any) => {
            if any.tag() == Tag.NULL {
                return Ok(AlgParams.Null)
            }
            dispatch_any_defined_by(algorithm, any, &PARAM_DISPATCH)
        },
    }
}
```

### 6.4 Example: PKCS#8 PrivateKey Dispatch

```ferrum
use extlib.asn1.{OidDispatch, dispatch_any_defined_by, AnyRef, Asn1Error}

enum PrivateKeyData {
    Rsa(RsaPrivateKey),
    EcdsaP256(EcdsaP256Key),
    Ed25519(Ed25519Key),
}

static PRIVKEY_DISPATCH: OidDispatch[PrivateKeyData] = OidDispatch.new(&[
    (OID_RSA_ENCRYPTION, |bytes| {
        let (k, _) = RsaPrivateKey.der_decode(bytes)?
        Ok(PrivateKeyData.Rsa(k))
    }),
    (OID_EC_PUBLIC_KEY, |bytes| {
        let (k, _) = EcdsaP256Key.der_decode(bytes)?
        Ok(PrivateKeyData.EcdsaP256(k))
    }),
    (OID_ED25519, |bytes| {
        let (k, _) = Ed25519Key.der_decode(bytes)?
        Ok(PrivateKeyData.Ed25519(k))
    }),
])

fn decode_pkcs8_key(info: &PrivateKeyInfo): Result[PrivateKeyData, Asn1Error] {
    dispatch_any_defined_by(
        &info.private_key_algorithm.algorithm,
        info.private_key_bytes,
        &PRIVKEY_DISPATCH,
    )
}
```

---

## 7. BER Mode (Opt-In)

BER parsing is available for interoperability with legacy systems that produce non-canonical encodings. It is never the default.

```ferrum
/// A BER reader. Accepts:
///   - Indefinite-length encoding (0x80 end-of-contents)
///   - Non-minimal length encodings (e.g. 0x81 0x05 instead of 0x05)
///   - Leading zero bytes in INTEGER beyond DER minimum
///   - Duplicate members in SET
///   - Constructed encoding of primitive types
///
/// BER is explicit opt-in. Accidentally using BerReader instead of
/// DerReader is visible at the call site.
pub struct BerReader['a] { ... }

impl['a] BerReader['a] {
    pub fn new(input: &'a [u8]): Self

    /// Return a DerReader that reads from the remaining input.
    /// Any BER-only encoding encountered after this point is an error.
    pub fn strict_der(self: Self): DerReader['a]

    pub fn is_empty(self: &Self): bool
    pub fn peek_tag(self: &Self): Result[Tag, Asn1Error]

    pub fn read_element[T: DerDecode['a]](self: &mut Self): Result[T, Asn1Error]
    pub fn read_optional[T: DerDecode['a]](self: &mut Self): Result[Option[T], Asn1Error]

    pub fn read_sequence[T](
        self: &mut Self,
        f: impl Fn(&mut BerReader['a]): Result[T, Asn1Error],
    ): Result[T, Asn1Error]

    pub fn read_any(self: &mut Self): Result[AnyRef['a], Asn1Error]

    pub fn depth(self: &Self): u8
    pub const MAX_DEPTH: u8 = 64
}
```

`BerReader` uses the same nesting depth limit as `DerReader`. BER's indefinite-length nesting cannot be used to bypass this limit.

When `BerReader` successfully reads a value that DER would reject (non-minimal length, constructed primitive, etc.), it internally notes the deviation. Callers who care can check `BerReader::is_canonical()` after reading to determine whether the input was valid DER or required BER relaxation.

---

## 8. DER Writer

`DerWriter` constructs a DER-encoded byte sequence. The API is stateful and append-only. All output is well-formed DER by construction.

```ferrum
/// A DER encoder that builds output into an internal buffer.
pub struct DerWriter { ... }

impl DerWriter {
    /// Construct an empty writer.
    pub fn new(): Self ! Alloc

    /// Construct a writer with pre-allocated capacity.
    pub fn with_capacity(cap: usize): Self ! Alloc

    /// Write a single DER-encodable value, appending its TLV to the buffer.
    pub fn write[T: DerEncode](self: &mut Self, value: &T) ! Alloc

    /// Write a SEQUENCE TLV. Calls `f` with a writer for the body.
    ///
    /// The body is written to a temporary buffer, then the SEQUENCE tag
    /// and computed length are prepended. The result is appended to `self`.
    pub fn write_sequence(
        self: &mut Self,
        f: impl Fn(&mut DerWriter): Result[(), Asn1Error],
    ): Result[(), Asn1Error] ! Alloc

    /// Write a SET TLV (like write_sequence but with SET tag).
    pub fn write_set(
        self: &mut Self,
        f: impl Fn(&mut DerWriter): Result[(), Asn1Error],
    ): Result[(), Asn1Error] ! Alloc

    /// Write an explicitly context-tagged TLV wrapping the body.
    pub fn write_context_explicit(
        self:       &mut Self,
        tag_number: u8,
        f:          impl Fn(&mut DerWriter): Result[(), Asn1Error],
    ): Result[(), Asn1Error] ! Alloc

    /// Write raw bytes without any tag/length wrapping.
    /// Use only when you have a pre-encoded TLV (e.g., AnyRef).
    pub fn write_raw(self: &mut Self, bytes: &[u8]) ! Alloc

    /// Consume the writer and return the accumulated DER bytes.
    pub fn finish(self: Self): Vec[u8]
}
```

---

## 9. Common PKI Structures

The following structures appear in nearly every PKI application. They are provided as built-in types rather than requiring each user to re-derive them. All implement `DerEncode` and `DerDecode`.

### 9.1 `AlgorithmIdentifier`

```ferrum
/// ASN.1 AlgorithmIdentifier (RFC 5280 §4.1.1.2).
///
/// AlgorithmIdentifier ::= SEQUENCE {
///     algorithm   OBJECT IDENTIFIER,
///     parameters  ANY DEFINED BY algorithm OPTIONAL
/// }
#[derive(DerEncode, DerDecode)]
pub struct AlgorithmIdentifier['a] {
    pub algorithm:  Oid,
    pub parameters: Option[AnyRef['a]],   // zero-copy; caller dispatches
}
```

### 9.2 `SubjectPublicKeyInfo`

```ferrum
/// ASN.1 SubjectPublicKeyInfo (RFC 5280 §4.1.2.7).
///
/// SubjectPublicKeyInfo ::= SEQUENCE {
///     algorithm          AlgorithmIdentifier,
///     subjectPublicKey   BIT STRING
/// }
#[derive(DerEncode, DerDecode)]
pub struct SubjectPublicKeyInfo['a] {
    pub algorithm:          AlgorithmIdentifier['a],
    pub subject_public_key: BitString,
}
```

### 9.3 `PrivateKeyInfo`

```ferrum
/// ASN.1 PrivateKeyInfo (PKCS#8, RFC 5958 §2).
///
/// OneAsymmetricKey ::= SEQUENCE {
///     version                  Version,       -- 0 or 1
///     privateKeyAlgorithm      AlgorithmIdentifier,
///     privateKey               OCTET STRING,  -- contains key-type-specific encoding
///     attributes           [0] Attributes OPTIONAL,
///     publicKey            [1] BIT STRING OPTIONAL  -- version 1 only
/// }
///
/// The `private_key` field is an OCTET STRING whose content is key-type-specific.
/// Use dispatch_any_defined_by with the algorithm OID to decode it.
#[derive(DerEncode, DerDecode)]
pub struct PrivateKeyInfo['a] {
    pub version:               Integer,
    pub private_key_algorithm: AlgorithmIdentifier['a],
    pub private_key:           OctetString,
    pub attributes:            Option[Implicit[AnyRef['a], 0]],
    pub public_key:            Option[Implicit[BitString, 1]],
}
```

### 9.4 `DigestInfo`

```ferrum
/// ASN.1 DigestInfo (PKCS#1 §9.2, used in RSA signatures).
///
/// DigestInfo ::= SEQUENCE {
///     digestAlgorithm  AlgorithmIdentifier,
///     digest           OCTET STRING
/// }
#[derive(DerEncode, DerDecode)]
pub struct DigestInfo['a] {
    pub digest_algorithm: AlgorithmIdentifier['a],
    pub digest:           OctetString,
}
```

---

## 10. Error Types

```ferrum
/// All errors produced by the ASN.1 codec.
pub enum Asn1Error {
    /// The next tag in the input does not match what was expected.
    UnexpectedTag {
        expected: Tag,
        found:    Tag,
    },

    /// A length value is invalid or inconsistent with the remaining input.
    InvalidLength,

    /// Bytes remain in the input after decoding is complete, where none are expected.
    TrailingData,

    /// The nesting depth counter reached MAX_DEPTH (64).
    /// The input is rejected without further parsing.
    NestingTooDeep,

    /// An OID was encountered that is not registered in the dispatch table.
    /// The OID is included so the caller can log or handle it specifically.
    UnknownOid(Oid),

    /// A string value contains bytes that are not valid UTF-8, or a
    /// character set-constrained type (PrintableString, IA5String) contains
    /// a disallowed character.
    InvalidUtf8,

    /// A time value (UTCTime or GeneralizedTime) is syntactically invalid
    /// or represents an impossible date.
    InvalidTime,

    /// A value is structurally valid but semantically incorrect.
    /// The message names the violated constraint (e.g., "INTEGER not minimal").
    InvalidValue(&'static str),

    /// The input is valid BER but not valid DER.
    /// Produced by DerReader when it encounters indefinite-length encoding,
    /// non-minimal lengths, or other BER-only constructs.
    /// Not produced by BerReader.
    NonCanonical,

    /// Unexpected end of input.
    UnexpectedEof,

    /// Indefinite-length encoding encountered in DER mode.
    IndefiniteLength,
}

impl Display for Asn1Error { ... }
impl Error for Asn1Error {}
```

Error handling discipline: `UnknownOid` carries the OID and is always propagated to the caller. The library never silently ignores an unrecognized OID. Callers decide whether an unknown OID is fatal (strict PKI validation) or ignorable (best-effort parsing).

---

## 11. Security Properties

### Strict DER by Default

`DerReader` enforces every constraint in X.690 §11 (DER requirements):

- Tags encoded in minimum bytes (no padding).
- Lengths encoded in minimum form (short form when value ≤ 127; long form with minimum length bytes otherwise).
- INTEGERs encoded without unnecessary leading 0x00 or 0xFF bytes.
- BOOLEAN TRUE encoded as 0xFF, not any other non-zero value.
- BIT STRINGs with unused bits set to zero.
- UTCTime and GeneralizedTime use the Z suffix; GeneralizedTime has no fractional seconds in DER.
- SET OF members sorted by encoded byte value.

Any violation returns `Asn1Error::NonCanonical`. There is no lenient mode for `DerReader`. Use `BerReader` when lenient parsing is required, and do so explicitly.

### Bounded Nesting

The nesting depth counter is checked before entering any `read_sequence`, `read_set`, or `read_context_explicit` call. The check happens before allocating any memory for the nested structure. At depth 64, the input is rejected immediately. This prevents:

- Stack exhaustion from recursive descent.
- Allocation amplification from deeply-nested SEQUENCE-of-SEQUENCE structures.
- CPU exhaustion from O(depth) tag-length parsing overhead on adversarial input.

The limit is intentionally not configurable. Any legitimate PKI structure fits within 64 levels. A configurable limit would invite misconfiguration.

### Zero-Copy `AnyRef`

`AnyRef<'a>` holds a reference into the input buffer. No heap allocation occurs when capturing an opaque ANY field. The input buffer's lifetime governs the `AnyRef` lifetime via the `'a` parameter, preventing use-after-free without runtime overhead.

### No Allocation During Parsing Tags and Lengths

Tag and length parsing produces `Tag` and `Length` values directly. No heap memory is allocated to parse a TLV header. Allocation occurs only when materializing the value (e.g., constructing a `String` from a UTF8String, or a `Vec[u8]` from an OCTET STRING).

### No Implicit BER Normalization

`DerReader` never silently normalizes a non-canonical encoding to its canonical form. Normalization-on-read has historically been used to craft inputs that pass signature verification against one encoding while having a different byte representation than the signed form. `DerReader` fails hard on non-canonical input rather than normalizing it.

---

## 12. Example Usage

### 12.1 Decode an X.509 TBSCertificate (Simplified)

This example shows a partial TBSCertificate schema, DER-decoded from a certificate byte slice:

```ferrum
use extlib.asn1.{
    DerReader, Asn1Error, Oid, Integer, BitString,
    AlgorithmIdentifier, SubjectPublicKeyInfo, AnyRef,
    Explicit, UtcTime, GeneralizedTime,
}

enum CertTime {
    Utc(UtcTime),
    Generalized(GeneralizedTime),
}

struct Validity {
    not_before: CertTime,
    not_after:  CertTime,
}

fn decode_cert_time(r: &mut DerReader): Result[CertTime, Asn1Error] {
    match r.peek_tag()? {
        Tag.UTC_TIME         => Ok(CertTime.Utc(r.read_element[UtcTime]()?)),
        Tag.GENERALIZED_TIME => Ok(CertTime.Generalized(r.read_element[GeneralizedTime]()?)),
        found                => Err(Asn1Error.UnexpectedTag {
            expected: Tag.UTC_TIME,
            found,
        }),
    }
}

struct TbsCertificate['a] {
    pub version:               Option[i64],          // [0] EXPLICIT INTEGER DEFAULT 0
    pub serial_number:         Integer,
    pub signature:             AlgorithmIdentifier['a],
    pub issuer_raw:            AnyRef['a],            // Name — opaque for this example
    pub not_before:            CertTime,
    pub not_after:             CertTime,
    pub subject_raw:           AnyRef['a],            // Name — opaque for this example
    pub subject_public_key:    SubjectPublicKeyInfo['a],
    pub extensions_raw:        Option[AnyRef['a]],    // [3] EXPLICIT Extensions OPTIONAL
}

fn decode_tbs_certificate['a](input: &'a [u8]): Result[TbsCertificate['a], Asn1Error] {
    let mut r = DerReader.new(input)

    r.read_sequence(|r| {
        // version [0] EXPLICIT INTEGER DEFAULT 0
        let version = if r.peek_tag()? == Tag::context_explicit(0) {
            Some(r.read_context_explicit(0, |r| {
                let v: Integer = r.read_element()?
                v.to_i64()
            })?)
        } else {
            None
        }

        let serial_number:      Integer               = r.read_element()?
        let signature:          AlgorithmIdentifier   = r.read_element()?
        let issuer_raw:         AnyRef                = r.read_any()?      // skip Name
        let validity = r.read_sequence(|r| {
            let not_before = decode_cert_time(r)?
            let not_after  = decode_cert_time(r)?
            Ok(Validity { not_before, not_after })
        })?
        let subject_raw:        AnyRef                = r.read_any()?      // skip Name
        let subject_public_key: SubjectPublicKeyInfo  = r.read_element()?

        // Skip optional issuerUniqueID [1] and subjectUniqueID [2]
        if r.peek_tag()? == Tag::context_implicit(1) { let _: AnyRef = r.read_any()? }
        if r.peek_tag()? == Tag::context_implicit(2) { let _: AnyRef = r.read_any()? }

        // extensions [3] EXPLICIT Extensions OPTIONAL
        let extensions_raw = if !r.is_empty() {
            Some(r.read_context_explicit(3, |r| r.read_any())?)
        } else {
            None
        }

        Ok(TbsCertificate {
            version,
            serial_number,
            signature,
            issuer_raw,
            not_before: validity.not_before,
            not_after:  validity.not_after,
            subject_raw,
            subject_public_key,
            extensions_raw,
        })
    })
}
```

### 12.2 Dispatch on AlgorithmIdentifier OID

```ferrum
use extlib.asn1.{
    AlgorithmIdentifier, OidDispatch, AnyRef, Asn1Error,
    dispatch_any_defined_by, Oid,
}

enum SignatureAlgorithm {
    Rsa,
    RsaPss { hash: Oid, salt_len: u32 },
    EcdsaWithSha256,
    EcdsaWithSha384,
    Ed25519,
}

fn rsa_pss_decoder['a](bytes: &'a [u8]): Result[SignatureAlgorithm, Asn1Error] {
    // decode RSA-PSS parameters — simplified
    let mut r = DerReader.new(bytes)
    let params = r.read_sequence(|r| {
        let hash_alg: AlgorithmIdentifier = r.read_element()?
        // ... decode mask and salt ...
        Ok((hash_alg.algorithm.clone(), 32u32))
    })?
    Ok(SignatureAlgorithm.RsaPss { hash: params.0, salt_len: params.1 })
}

static SIG_ALG_DISPATCH: OidDispatch[SignatureAlgorithm] = OidDispatch.new(&[
    (OID_SHA256_WITH_RSA,   |_| Ok(SignatureAlgorithm.Rsa)),
    (OID_RSA_PSS,           rsa_pss_decoder),
    (OID_ECDSA_WITH_SHA256, |_| Ok(SignatureAlgorithm.EcdsaWithSha256)),
    (OID_ECDSA_WITH_SHA384, |_| Ok(SignatureAlgorithm.EcdsaWithSha384)),
    (OID_ED25519,           |_| Ok(SignatureAlgorithm.Ed25519)),
])

fn decode_sig_alg['a](alg: AlgorithmIdentifier['a]): Result[SignatureAlgorithm, Asn1Error] {
    let any = alg.parameters.unwrap_or(AnyRef.null())
    dispatch_any_defined_by(&alg.algorithm, any, &SIG_ALG_DISPATCH)
}
```

### 12.3 Write a DER-Encoded PKCS#8 PrivateKeyInfo

```ferrum
use extlib.asn1.{DerWriter, Oid, Integer, OctetString, Null, AlgorithmIdentifier}

fn encode_pkcs8_ed25519(private_key_bytes: &[u8]): Result[Vec[u8], Asn1Error] ! Alloc {
    let mut w = DerWriter.new()

    w.write_sequence(|w| {
        // version = 0 (v1)
        w.write(&Integer.from_i64(0))

        // privateKeyAlgorithm AlgorithmIdentifier
        w.write_sequence(|w| {
            w.write(&OID_ED25519)  // Oid implements DerEncode
            // no parameters for Ed25519
            Ok(())
        })?

        // privateKey OCTET STRING (wraps the raw 32-byte scalar)
        w.write(&OctetString.new(private_key_bytes.to_vec()))

        Ok(())
    })?

    Ok(w.finish())
}
```

---

## 13. Dependencies

### Required

- `std.core` — slice types, `Result`, `Option`, `String`, basic traits.
- `std.alloc` — `Vec`, `Box`, heap allocation for `Integer`, `String`, and `OctetString` values.
- `extlib.bigint` — `BigInt` backing the `Integer` type for arbitrary-precision ASN.1 INTEGER values. RSA moduli and private exponents require this. For small integers (serial numbers, version fields), `Integer.to_i64()` or `to_u64()` extracts the value without retaining a `BigInt`.

### No Other Dependencies

This module does not depend on any networking, I/O, or crypto extlib modules. It is a pure codec: bytes in, structured values out; structured values in, bytes out.
