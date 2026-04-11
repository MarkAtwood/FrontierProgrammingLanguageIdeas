# Ferrum Standard Library — crypto, testing

**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)

---

## 24. crypto — Cryptographic Primitives

Cryptographic primitives belong in the stdlib to ensure correct, audited, non-footgun implementations. This is not a full cryptographic library — it is the foundation that TLS libraries and other crates build on.

**Policy:** All deprecated or broken algorithms have no implementation. No MD5. No SHA1 for new code (only for legacy interop). No DES. No RC4. No ECB mode. The library does not enable subtle misuse.

```ferrum
// Hash functions
trait Hasher[const OUTPUT_LEN: usize] {
    fn update(&mut self, data: &[u8])
    fn finalize(self): [u8; OUTPUT_LEN]
    fn reset(&mut self)
}

fn hash[H: Hasher<N>, const N: usize](data: &[u8]): [u8; N]
    // Convenience: create, update, finalize

struct Sha256   // implements Hasher<32>
struct Sha384   // implements Hasher<48>
struct Sha512   // implements Hasher<64>
struct Sha3_256 // implements Hasher<32>
struct Sha3_512 // implements Hasher<64>
struct Blake3   // implements Hasher<32> (variable output via Blake3Xof)
struct Blake2b  // implements Hasher<64>
struct Blake2s  // implements Hasher<32>

// HMAC
type Hmac[H: Hasher<N>, const N: usize]
impl[H: Hasher<N>, const N: usize] Hmac[H, N] {
    fn new(key: &[u8]): Self
    fn update(&mut self, data: &[u8])
    fn finalize(self): [u8; N]
    fn verify(self, tag: &[u8; N]): Result[(), MacError]
}

// AEAD ciphers — the safe interface for symmetric encryption
trait Aead {
    const KEY_LEN:   usize
    const NONCE_LEN: usize
    const TAG_LEN:   usize

    fn encrypt_in_place(
        &self,
        key:        &[u8; Self.KEY_LEN],
        nonce:      &[u8; Self.NONCE_LEN],
        aad:        &[u8],
        plaintext:  &[u8],
        ciphertext: &mut Vec[u8],    // ciphertext.len() == plaintext.len() + TAG_LEN
    ): Result[(), AeadError]

    fn decrypt_in_place(
        &self,
        key:        &[u8; Self.KEY_LEN],
        nonce:      &[u8; Self.NONCE_LEN],
        aad:        &[u8],
        ciphertext: &[u8],           // includes appended tag
        plaintext:  &mut Vec[u8],
    ): Result[(), AeadError]
}

struct Aes128Gcm      // AES-128-GCM, key 16, nonce 12, tag 16
struct Aes256Gcm      // AES-256-GCM, key 32, nonce 12, tag 16
struct ChaCha20Poly1305  // key 32, nonce 12, tag 16
struct XChaCha20Poly1305 // key 32, nonce 24, tag 16 (extended nonce — safe with random nonces)

// NOTE: No raw AES-CBC, AES-CTR, or ChaCha20 without authentication.
// Unauthenticated encryption is not exposed. If you need it, use posix/unsafe.

// Safe AEAD — nonce reuse is structurally impossible
//
// The high-level API generates nonces internally from a per-context counter
// plus random base. Applications cannot pass a nonce. This eliminates the
// nonce-reuse vulnerability class through the API surface.
//
// For protocol implementation (TLS record layer) where nonce management is
// the protocol's responsibility, use the raw Aead trait above.

type SafeAead[A: Aead] {
    key:     [u8; A.KEY_LEN],
    counter: u64,
    base:    [u8; 4],   // random, set on construction
}

impl[A: Aead] SafeAead[A] {
    fn new(key: [u8; A.KEY_LEN]): Result[Self, CryptoError] ! IO
        // Initializes counter to 0, base from SystemRng

    fn seal(&mut self, aad: &[u8], plaintext: &[u8]): Result[Vec[u8], AeadError]
        // Generates nonce internally: base || counter++
        // Returns ciphertext with prepended nonce and appended tag
        // Nonce reuse impossible: counter is monotonic, never exposed

    fn open(&self, aad: &[u8], ciphertext: &[u8]): Result[Vec[u8], AeadError]
        // Extracts nonce from ciphertext prefix, decrypts
        // Stateless on receiver side — any message decryptable

    fn seal_in_place(&mut self, aad: &[u8], buf: &mut Vec[u8]): Result[(), AeadError]
        // Encrypts in place, prepends nonce, appends tag

    fn messages_remaining(&self): u64
        // Returns 2^64 - counter; warns if < 2^32
}

// Key derivation
struct HkdfSha256
struct HkdfSha512
struct Pbkdf2HmacSha256
struct Argon2id   // for passwords — memory hard
struct Scrypt

impl HkdfSha256 {
    fn extract(salt: &[u8], ikm: &[u8]): PrkSha256
    fn expand(prk: &PrkSha256, info: &[u8], okm: &mut [u8]): Result[(), HkdfError]
        requires okm.len() <= 255 * 32
}

impl Argon2id {
    fn new(params: Argon2Params): Self
    fn hash_password(self, password: &[u8], salt: &[u8; 16]): [u8; 32]
    fn verify(self, password: &[u8], salt: &[u8; 16], expected: &[u8; 32]): bool
}

// Asymmetric cryptography
// Signatures
struct Ed25519
impl Ed25519 {
    fn generate_keypair(): (Ed25519PublicKey, Ed25519SecretKey) ! IO
    fn sign(sk: &Ed25519SecretKey, msg: &[u8]): [u8; 64]
    fn verify(pk: &Ed25519PublicKey, msg: &[u8], sig: &[u8; 64]): Result[(), SigError]
}

struct P256   // NIST P-256 / secp256r1
struct X25519 // ECDH

// Random number generation
type SystemRng  // OS CSPRNG (/dev/urandom, getrandom, BCryptGenRandom)
impl SystemRng {
    fn new(): Result[Self, RngError] ! IO
    fn fill_bytes(&mut self, buf: &mut [u8]) ! IO
    fn next_u32(&mut self): u32 ! IO
    fn next_u64(&mut self): u64 ! IO
}

// Constant-time comparison — prevents timing side channels
fn ct_eq(a: &[u8], b: &[u8]): bool
fn ct_select(condition: bool, a: u64, b: u64): u64
```

---

## 25. testing — Test Framework

Tests are first-class. No external harness required.

```ferrum
// Unit tests
@test
fn test_add() {
    assert_eq!(2 + 2, 4)
}

// Expected panic
@test @should_panic(message = "divide by zero")
fn test_div_zero() {
    let _ = 1 / 0
}

// Test with result — can use ?
@test
fn test_file_read(): Result[(), IoError] ! IO {
    let content = fs.read_text("test_data/sample.txt")?
    assert!(content.contains("expected"))
    Ok(())
}

// Parameterized tests
@test @cases(
    (0, "zero"),
    (1, "one"),
    (-1, "negative one"),
)
fn test_name_of(input: i32, expected: &str) {
    assert_eq!(name_of(input), expected)
}

// Property-based tests (uses contracts for generators)
@property_test
fn test_sort_idempotent(mut v: Vec[i32]) {
    v.sort()
    let sorted = v.clone()
    v.sort()
    assert_eq!(v, sorted)
}

// Benchmark
@bench
fn bench_hash(b: &mut Bencher) {
    let data = vec![0u8; 1024]
    b.iter(|| Sha256.update(&data))
}

// Test utilities
fn assert_eq![T: Debug + PartialEq](left: T, right: T)
fn assert_ne![T: Debug + PartialEq](left: T, right: T)
fn assert!(cond: bool)
fn assert_matches!(expr, pattern)
fn assert_approx_eq!(a: f64, b: f64, epsilon: f64)  // for floats

// Mocking (trait-based, no macro magic)
// Test doubles are just types that implement the trait under test.
// No framework needed for pure trait implementations.

// Test organization
// Tests co-located with source in test blocks, or in tests/ directory
// Integration tests: tests/integration_test.fe
// Doc tests: code examples in /// doc comments are run as tests
```

---

## 26. C Mistakes Avoided: Reference Table

This appendix maps every C/POSIX mistake to the Ferrum solution, for compiler and stdlib authors to verify completeness.

| C Mistake | Where It Bites | Ferrum Solution | Module |
|---|---|---|---|
| `errno` global mutable error state | Thread safety, error loss | All errors are `Result` return values | Every module |
| Null-terminated strings; strlen scans every time | Buffer overflows, O(n) unknowingly | `&str` carries length; `CStr` for C interop only | `core.str` |
| `fread` returns `size_t`, EOF via `feof()` | Confused error handling | `ReadResult` enum: `Data(n)`, `Eof`, `Err` | `io` |
| Text vs binary mode conflation on Windows | Silent data corruption | `TextReader`/`BinaryReader` are distinct types | `text`, `binary` |
| `printf` format string as runtime string | Format string injection | Format strings are compile-time literals | `fmt` |
| `malloc` returns uninitialized memory | Uninitialized reads | Allocator returns zeroed by default; uninit is `unsafe` | `alloc` |
| `memcpy` UB on overlapping regions | Silent memory corruption | `copy` vs `copy_overlapping` are separate functions | `core.ptr` |
| `time_t` 32-bit on some platforms (Y2K38) | Year 2038 overflow | `Timestamp` uses `i64` nanoseconds always | `time` |
| Raw signal handlers (almost nothing is async-signal-safe) | Deadlocks, data corruption | Signals convert to channel messages; no raw handlers | `sys.posix` |
| Locale-dependent `ctype.h` | Encoding bugs, non-reproducible behavior | All locale operations are explicit and parameterized | `text` |
| No ownership semantics on returned pointers | Memory leaks, double-free | All returned heap values are owned | Language |
| `unsigned` / `signed` confusion in sizes | Wrap-around bugs | `usize` for sizes; `isize` for offsets; no silent mixing | Language |
| `strtol` uses sentinel for error (LONG_MAX + errno) | Easy to forget checking | All parse operations return `Result` | `core.str` |
| Socket API: 15+ steps for HTTP | Productivity sink | HTTP is a first-class type | `net.http` |
| `set_current_dir` is process-global (thread-unsafe) | Heisenbugs | Not in cross-platform API; only in `sys.posix` with annotation | `sys.posix` |
| `setenv` is not thread-safe | Heisenbugs | `set_env_var` warns when called after thread spawn | `sys.posix` |
| `FILE*` not typed by capability or mode | Read/write confusion | `Read`, `Write`, `Seek` are separate traits | `io` |
| Blocking `read` returns -1 for EINTR | Must retry loop at every call site | `ReadResult::Interrupted` is handled internally by `read_exact` | `io` |
| EOF and error conflated in `getchar` return | -1 is both EOF and error | `ReadResult` enum distinguishes both | `io` |
| `int` is platform-width | Portability bugs | No `int` type; all integers are explicit-width | Language |
| `realloc` returns `NULL` on failure but old pointer is lost | Memory leak | `realloc` returns `Result`; old pointer preserved on `Err` | `alloc` |
| No generic `min`/`max` (need `fmin`, `imin`, etc.) | Code duplication | Generic `Ord::min`, `Ord::max`, `f32::min_num` etc. | `core.cmp` |
| `strncpy` does not null-terminate if source is too long | Buffer bugs | Not implemented; use `&str` slicing | `core.str` |
| Global mutable `locale` state | Non-reproducible behavior | No global locale; all locale-sensitive ops take a locale parameter | `text` |
| `NaN != NaN` makes sorting/hashing unpredictable | Silent comparison bugs | `f32::total_cmp` for total order; `f32::min_num` for NaN-non-propagating | `math` |
| `SIGPIPE` crashes program on broken pipe | Surprised programs | `SIGPIPE` default-ignored; broken pipe returns `IoError::BrokenPipe` | `sys.posix` |
| `fork` in multi-threaded programs is unsafe | Deadlocks | `fork` is in `sys.posix`, `! Unsafe + IO`; documented hazards | `sys.posix` |
| Implicit `int` return type for `main` | Portability | `main(): ()` or `main(): ExitCode`; no implicit numeric return | Language |
| `clock()` measures CPU time, not wall time, platform-dependently | Wrong measurements | `Instant` (monotonic), `Timestamp` (wall), clearly separate | `time` |

---

*End of Ferrum Standard Library 0.1*
