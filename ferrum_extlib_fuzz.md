# Ferrum Extended Library — Fuzzing Infrastructure

**Module path:** `extlib.fuzz`
**Part of:** Ferrum Extended Standard Library (extlib)
**Engines:** LibFuzzer (LLVM), AFL++, honggfuzz
**OSS-Fuzz:** Supported

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [FuzzTarget Trait](#2-fuzztarget-trait)
3. [LibFuzzer Integration](#3-libfuzzer-integration)
4. [AFL++ Integration](#4-afl-integration)
5. [Fuzz Corpus Management](#5-fuzz-corpus-management)
6. [Crash Triage](#6-crash-triage)
7. [OSS-Fuzz Integration](#7-oss-fuzz-integration)
8. [Built-in Fuzz Targets](#8-built-in-fuzz-targets)
9. [Differential Fuzzing](#9-differential-fuzzing)
10. [Coverage Reporting](#10-coverage-reporting)
11. [Structured Fuzz Input](#11-structured-fuzz-input)
12. [Example Usage](#12-example-usage)
13. [Dependencies](#13-dependencies)

---

## 1. Overview and Rationale

### Fuzzing vs. Property-Based Testing

The Ferrum standard library provides `@property_test` for property-based testing. A
`@property_test` function expresses an invariant — a logical property that must hold for
all inputs in a defined domain — and the framework generates random inputs drawn from that
domain to attempt to falsify it. This is valuable for checking algorithmic correctness:
sorting stability, codec round-trips, arithmetic identities.

Fuzzing is a different technique with a different threat model. Where property testing
generates inputs according to a schema, coverage-guided fuzzing mutates raw byte
sequences and uses feedback from the program's execution trace — branch coverage, edge
coverage — to steer mutation toward unexplored program states. The fuzzer does not know
that the input is supposed to be valid UTF-8, or that the ASN.1 length field must match
the payload, or that the TLS record version must be `0x0303`. It does not care. It pushes
raw bytes into a parser and observes which paths execute. When it finds a mutation that
covers a new branch, it adds that input to the corpus and mutates it further.

This produces qualitatively different findings:

- **Edge cases in parser combinators** that no reasonable property generator would
  construct: tag bytes with high bit set, length fields encoded in non-minimal form,
  indefinite-length BER sequences where DER was expected, DNS names with maximum-length
  labels, TLS extensions with zero-length data, JSON strings containing valid CESU-8
  surrogates.
- **State machine confusion**: inputs that transition a parser through a legal sequence of
  states to reach an unintended terminal state that a well-formed document never visits.
- **Allocation amplification**: a short input with deeply nested arrays, or a length field
  claiming a 4 GB allocation for a 12-byte packet.
- **Interaction effects**: a corpus input that triggers one code path combined with a
  mutation that triggers a second code path, producing a crash only at their intersection.

Property tests check invariants for inputs that the test author imagined. Coverage-guided
fuzzing finds bugs in inputs the test author did not imagine. Both are necessary.

### Why Production CVEs Come from OSS-Fuzz

The OSS-Fuzz program (Google, launched 2016) runs coverage-guided fuzzing continuously
against open-source critical infrastructure. At the time of writing it has found over
10,000 vulnerabilities. The pattern is consistent: memory-safety bugs in C parsers, panic
paths in Rust parsers that authors believed were unreachable, integer overflows in length
calculations, and resource exhaustion bugs in recursive descent parsers. These are not
obscure edge cases — they are CVEs that would have been exploitable in production.

A Ferrum program with memory safety guarantees from the ownership system cannot have
spatial memory-safety bugs (buffer overflows, use-after-free). However, it can still have:

- **Panic paths** reachable from untrusted input (index out of bounds, integer overflow in
  debug mode, unwrap on None in a parser branch)
- **Resource exhaustion** (unbounded allocation, stack overflow from recursive descent
  without depth limits)
- **Logic errors** that cause a security-relevant parser to accept invalid input
- **Correctness bugs** that cause two implementations to disagree on validity

OSS-Fuzz integration means these bugs are found before they ship, rather than after.

### Why Extlib, Not Stdlib

The fuzzing infrastructure module has a large dependency footprint: it pulls in every
parser and decoder in the extended library so it can fuzz all of them. It requires
FFI bindings to LibFuzzer and AFL++ runtime support. It produces standalone fuzz
binaries that are only built when explicitly requested with `--fuzz-target=<name>`. None
of this belongs in a production binary or in the standard library.

`extlib.fuzz` is explicitly not for production use. It is a development and CI tool. No
production binary should link against it. The `@cfg(fuzzing)` gate enforced by the
compiler ensures that fuzz target code is dead in non-fuzz builds.

### Relationship to `@property_test`

`extlib.fuzz` is a complement to `@property_test`, not a replacement. The recommended
approach for any parser or decoder in extlib:

1. Write `@property_test` functions for round-trip correctness (encode then decode
   produces the original value, decode then encode produces the original bytes).
2. Write `@property_test` functions for invariants on valid input (parsing a valid DER
   integer never returns negative length, etc.).
3. Register a `FuzzTarget` implementation that feeds raw arbitrary bytes to the parser and
   checks that no panic, no UB, and no unbounded allocation occurs.

The property tests run in seconds during `ferrum test`. The fuzz targets run for hours or
days in CI and on OSS-Fuzz. Both are necessary; neither subsumes the other.

---

## 2. FuzzTarget Trait

`FuzzTarget` is the core abstraction. Every fuzz target in `extlib.fuzz` implements it.
Third-party crates can implement it to integrate with the same infrastructure.

```ferrum
/// A single fuzz target: a named entry point that accepts arbitrary bytes
/// and checks a parser or decoder for safety invariants.
///
/// Implementors must satisfy the contracts documented on `fuzz()`.
pub trait FuzzTarget {
    /// A short, unique identifier used as the binary name and corpus directory name.
    /// Must be a valid filename component (ASCII alphanumeric and underscores only).
    /// Example: "asn1_der", "json_dom", "tls_record"
    fn name(): &'static str

    /// A human-readable description of what this target fuzzes and what it checks.
    /// Displayed in `ferrum fuzz list` output and in OSS-Fuzz project metadata.
    fn description(): &'static str

    /// Feed one test case to the target.
    ///
    /// # Contracts
    ///
    /// This function MUST:
    /// - Accept any byte sequence without undefined behavior.
    /// - Accept any byte sequence without panicking. Parse errors, validation
    ///   failures, and authentication failures MUST be returned as `Result::Err`,
    ///   never as a panic.
    /// - Complete in bounded time relative to `data.len()`. Recursive parsers
    ///   MUST enforce a depth limit. Allocation MUST be proportional to
    ///   `data.len()`, not to a length field inside `data`.
    /// - Not retain references into `data` after returning. The fuzzer reuses
    ///   the buffer.
    ///
    /// This function MUST NOT:
    /// - Allocate memory unboundedly (e.g., allocate `claimed_len` bytes where
    ///   `claimed_len` comes from an untrusted length field without bounding it
    ///   against `data.len()`).
    /// - Call `std::process::exit` or equivalent.
    /// - Spawn threads or tasks.
    /// - Perform I/O (network, filesystem) other than reading `data`.
    ///
    /// # Performance
    ///
    /// This function is called millions of times per second. It MUST NOT
    /// perform heap allocation in the common (rejection) path if avoidable.
    /// Stack-only fast rejection of clearly-invalid input is preferred.
    fn fuzz(data: &[u8])
}
```

### The `fuzz()` Contract in Detail

The most important contract is: **`fuzz()` must never panic on any input**. In Ferrum,
panic is not a recoverable error; it terminates the process. A fuzzer that triggers a
panic has found a bug. The fuzzer infrastructure treats any non-zero exit (panic, signal,
or assertion failure) as a finding and saves the input that caused it.

Corollary: every call to `.unwrap()`, `.expect()`, indexing with `[n]`, and integer
arithmetic on untrusted data inside a `fuzz()` implementation is a potential bug. The
correct pattern is always `Result`-returning functions that propagate errors to the caller
of `fuzz()`, where they are silently discarded. The fuzzer does not care about the error
message; it cares whether the process crashed.

The bounded-allocation contract is equally important. Consider a DER parser that reads a
4-byte length field: `let buf = Vec::with_capacity(claimed_len)`. If `claimed_len` comes
from the input without being bounded against `data.len()`, a 5-byte input can request a
4 GB allocation. On most systems this will succeed in virtual address space terms, then
immediately OOM-kill the process. This is a denial-of-service bug. The correct pattern is
`let claimed_len = claimed_len.min(data.len())` before any allocation.

---

## 3. LibFuzzer Integration

LibFuzzer is an in-process, coverage-guided fuzzer built into LLVM. It instruments the
binary at compile time with `SanitizerCoverage` and drives mutation inside the same
process, avoiding fork overhead. It is the primary fuzzing engine for OSS-Fuzz.

### Entry Point

```ferrum
use extlib.fuzz.{FuzzTarget, FuzzCorpus}

/// Run a fuzz target under LibFuzzer.
///
/// This function does not return. It initializes the LibFuzzer runtime, calls
/// `LLVMFuzzerTestOneInput` in a loop, and exits when the fuzzer terminates.
///
/// The `main()` function of a LibFuzzer fuzz binary is generated by the
/// `--fuzz-target=<name>` compiler flag; user code calls this function from
/// the generated main.
pub fn libfuzzer_main(target: &impl FuzzTarget) ! Unsafe
```

### Compilation

Fuzzer binaries are not produced by a normal `ferrum build`. They require an explicit
flag:

```
ferrum build --fuzz-target=asn1_der
```

This flag:

1. Sets `@cfg(fuzzing)` and `@cfg(libfuzzer)` for the compilation.
2. Links `libFuzzer.a` (from the LLVM toolchain).
3. Enables `SanitizerCoverage` instrumentation (`-fsanitize-coverage=trace-pc-guard,
   trace-cmp,indirect-calls`).
4. Generates a `main()` that calls `libfuzzer_main(target)` where `target` is the
   `FuzzTarget` registered under the given name.
5. Produces a standalone binary named `fuzz-<target-name>`.

The `@cfg(fuzzing)` gate ensures that `extlib.fuzz` and its dependencies are only
compiled when explicitly building a fuzz target. Normal production builds are unaffected.

### `LLVMFuzzerTestOneInput` Bridge

The LibFuzzer runtime calls `LLVMFuzzerTestOneInput(data: *const u8, size: usize)` on
each generated input. The bridge function converts the raw pointer to a Ferrum slice and
calls `target.fuzz(data)`. Any return from `fuzz()` is treated as success (the input did
not crash the target). A panic unwinds to the bridge, which reports the crash to
LibFuzzer.

```ferrum
// Generated by the compiler for each --fuzz-target=<name> build.
// Not part of the public API.
@cfg(libfuzzer)
@export("LLVMFuzzerTestOneInput")
fn llvm_fuzzer_test_one_input(data: *const u8, size: usize): i32 ! Unsafe {
    let slice = unsafe { std.slice.from_raw_parts(data, size) }
    REGISTERED_TARGET.fuzz(slice)
    0
}
```

### Standard LibFuzzer Flags

`FuzzCorpus` provides the recommended LibFuzzer command-line flags for a given target:

```ferrum
impl FuzzCorpus {
    /// Standard LibFuzzer flags for running this corpus.
    ///
    /// Returns arguments suitable for passing directly to the fuzz binary:
    ///   -max_len=65536 -timeout=10 -jobs=4 -workers=4 <corpus_dir>
    pub fn libfuzzer_flags(self: &Self): Vec[String] ! Alloc
}
```

The defaults reflect conservative values suitable for CI:

| Flag | Default | Rationale |
|------|---------|-----------|
| `-max_len` | 65536 | 64 KiB is sufficient for any realistic parser input; larger inputs slow the fuzzer without finding more bugs |
| `-timeout` | 10 | 10 seconds per input; triggers on infinite loops from recursion bugs |
| `-jobs` | 4 | Number of parallel fuzzer instances; adjust to available cores |
| `-workers` | 4 | Matches jobs for in-process parallelism |
| `-use_value_profile=1` | enabled | Use value comparison profiling to escape magic byte checks faster |

---

## 4. AFL++ Integration

AFL++ (American Fuzzy Lop plus plus) is a fork-server and persistent-mode fuzzer. It
instruments the target at compile time and communicates with the fuzzer process over
shared memory and pipes. AFL++ is the recommended engine for local long-running fuzzing
campaigns and is supported alongside LibFuzzer on OSS-Fuzz.

### Entry Point

```ferrum
use extlib.fuzz.{FuzzTarget, FuzzCorpus}

/// Run a fuzz target under AFL++ persistent mode.
///
/// Uses the `__AFL_FUZZ_TESTCASE_BUF` shared memory interface to receive
/// test cases from AFL++ without fork overhead. The loop runs until AFL++
/// terminates the process.
///
/// Requires compilation with `--fuzz-target=<name> --engine=afl`.
pub fn afl_main(target: &impl FuzzTarget) ! Unsafe
```

### Persistent Mode

AFL++ persistent mode avoids the overhead of forking a new process for each test case.
Instead, the fuzz target calls `__AFL_LOOP(N)` — a macro that returns `true` for the
next `N` iterations and then exits cleanly, allowing AFL++ to restore the fork-server
snapshot. Combined with the `__AFL_FUZZ_TESTCASE_BUF` shared memory buffer, this
eliminates stdin parsing overhead.

```ferrum
// AFL++ persistent-mode bridge (compiler-generated, not public API).
@cfg(afl)
fn afl_persistent_loop(target: &impl FuzzTarget) ! Unsafe {
    // __AFL_INIT() sets up the fork server.
    unsafe { afl_sys.afl_init() }

    // __AFL_FUZZ_TESTCASE_BUF is AFL++'s shared memory buffer.
    // __AFL_FUZZ_TESTCASE_LEN holds the current test case length.
    while unsafe { afl_sys.afl_loop(10_000) } {
        let data: &[u8] = unsafe {
            let buf = afl_sys.AFL_FUZZ_TESTCASE_BUF
            let len = afl_sys.AFL_FUZZ_TESTCASE_LEN
            std.slice.from_raw_parts(buf, len)
        }
        target.fuzz(data)
    }
}
```

### Coverage Instrumentation

AFL++ uses `__sanitizer_cov_trace_pc_guard` instrumentation, the same interface as
LibFuzzer's SanitizerCoverage. Compiling with `--engine=afl` passes
`-fsanitize-coverage=trace-pc-guard` to the Ferrum backend and links the AFL++
compiler runtime (`afl-compiler-rt.o`).

### Standard AFL++ Flags

```ferrum
impl FuzzCorpus {
    /// Standard AFL++ flags for running this corpus.
    ///
    /// Returns arguments suitable for passing to `afl-fuzz`:
    ///   -i <corpus_dir> -o <findings_dir> -t 10000 -- ./fuzz-<name>
    pub fn afl_flags(self: &Self, findings_dir: &Path): Vec[String] ! Alloc
}
```

| Flag | Default | Rationale |
|------|---------|-----------|
| `-i` | corpus directory | Seed corpus input directory |
| `-o` | findings directory | Output directory for crashes and hangs |
| `-t` | 10000 | Timeout per input in milliseconds (10 seconds) |
| `-m` | none | No memory limit; let the OS OOM-kill runaway allocations |
| `-Q` | off | QEMU mode off; use native instrumentation |

---

## 5. Fuzz Corpus Management

A fuzz corpus is a directory of seed inputs. The fuzzer starts from these seeds and
mutates them. A good seed corpus dramatically accelerates fuzzing: seeds that exercise
diverse code paths let the fuzzer start from interesting states rather than building
coverage from scratch. A poor seed corpus (one large file, a single zero-byte file)
wastes significant fuzzer compute before it discovers the first interesting paths.

```ferrum
use extlib.fuzz.{FuzzCorpus, FuzzError, MinimizeStats}
use std.fs.Path

/// A directory of seed inputs for a fuzz target.
///
/// Each file in the directory is one test case. Files are named by their
/// SHA-256 hash (hex, lowercase) to deduplicate automatically.
pub struct FuzzCorpus {
    dir: Path,
    target_name: String,
}

@derive(Debug)
pub struct MinimizeStats {
    /// Number of inputs in the corpus before minimization.
    pub before: usize,
    /// Number of inputs retained after minimization.
    pub after: usize,
    /// Whether the minimized corpus preserves all coverage seen in the original.
    pub coverage_preserved: bool,
}

impl FuzzCorpus {
    /// Open or create a corpus directory.
    ///
    /// If the directory does not exist, it is created. If it exists and
    /// contains files, those files become the initial seed corpus.
    pub fn new(dir: &Path): Result[FuzzCorpus, FuzzError] ! IO

    /// Add a single test case to the corpus.
    ///
    /// The test case is written to a file named by its SHA-256 hash.
    /// If an identical input is already present, this is a no-op (returns Ok).
    pub fn add_seed(self: &mut Self, data: &[u8]): Result[(), FuzzError] ! IO

    /// Import all files from a crash directory as new seeds.
    ///
    /// Crash directories produced by LibFuzzer or AFL++ contain inputs that
    /// triggered a crash in a previous target version. After fixing the crash,
    /// import these inputs as seeds so future fuzzing continues from these
    /// interesting states rather than rediscovering them.
    ///
    /// Returns the number of new seeds imported (excluding duplicates).
    pub fn import_from_crashes(
        self: &mut Self,
        crash_dir: &Path,
    ): Result[usize, FuzzError] ! IO

    /// Minimize the corpus.
    ///
    /// Runs all inputs through the target with coverage instrumentation.
    /// Removes inputs whose coverage is a strict subset of coverage already
    /// provided by other inputs. The output corpus covers all branches that
    /// the full corpus covered, with the minimum number of inputs.
    ///
    /// Uses the cmin (corpus minimization) algorithm: sort inputs by coverage
    /// contribution (descending), keep each input only if it adds at least one
    /// new branch to the accumulated coverage set.
    ///
    /// Writes the minimized corpus to `output_dir`. Does not modify `self.dir`.
    pub fn minimize(
        self: &Self,
        target: &impl FuzzTarget,
        output_dir: &Path,
    ): Result[MinimizeStats, FuzzError] ! IO

    /// The number of inputs in this corpus.
    pub fn len(self: &Self): usize

    /// Standard LibFuzzer flags for running this corpus.
    pub fn libfuzzer_flags(self: &Self): Vec[String] ! Alloc

    /// Standard AFL++ flags for running this corpus.
    pub fn afl_flags(self: &Self, findings_dir: &Path): Vec[String] ! Alloc
}
```

### Corpus Naming Convention

By convention, corpus directories are organized as:

```
corpus/
  asn1_der/        ← FuzzCorpus for Asn1FuzzTarget
    3a7f9c1b...    ← one seed, named by SHA-256
    9d2e4f00...
  json_dom/
    ...
  tls_record/
    ...
```

Seed inputs for each target in `extlib.fuzz` are provided by the module and committed to
version control. These cover the happy path (valid well-formed inputs) and known boundary
conditions (maximum-length fields, empty inputs, single-byte inputs).

---

## 6. Crash Triage

When a fuzzer finds an input that causes a crash (panic, signal, assertion failure), the
next steps are: confirm it is reproducible, minimize the input to its smallest
crash-inducing form, capture the stack trace, and determine the severity.

```ferrum
use extlib.fuzz.{FuzzTarget, FuzzError, CrashReport, Signal}

/// A fully triaged crash report.
@derive(Debug)
pub struct CrashReport {
    /// The original crashing input.
    pub input: Vec[u8],
    /// Stack trace captured at the crash point, formatted as a string.
    /// Empty if the crash occurred without a stack trace (e.g., SIGKILL from OOM).
    pub stack_trace: String,
    /// The signal or exit condition that terminated the process.
    pub signal: Signal,
    /// Whether the crash reproduced on 10 consecutive runs.
    pub reproducible: bool,
    /// The minimized crash input, if minimization was run and succeeded.
    pub minimized_input: Option[Vec[u8]],
}

/// The termination condition observed when the crash was triggered.
@derive(Debug, Clone, PartialEq)
pub enum Signal {
    /// Process panicked (Ferrum panic handler called, exit code 101).
    Panic,
    /// SIGSEGV: segmentation fault (should not occur in safe Ferrum; indicates
    /// a bug in an `! Unsafe` block or a stack overflow).
    Sigsegv,
    /// SIGABRT: abort (triggered by ASAN, UBSAN, or explicit abort()).
    Sigabrt,
    /// SIGFPE: floating-point exception (integer divide by zero, etc.).
    Sigfpe,
    /// Process exited with non-zero status code.
    NonzeroExit(i32),
    /// Process timed out (exceeded the configured per-input timeout).
    Timeout,
    /// Out of memory: allocation failed or OOM-killer terminated the process.
    OutOfMemory,
}

/// Triage a crash: reproduce it, capture the stack trace, and optionally minimize.
///
/// Runs `target.fuzz(crash_input)` in a subprocess with sanitizers enabled.
/// Captures the stack trace from AddressSanitizer or the Ferrum panic handler.
/// Checks reproducibility over 10 runs.
///
/// Does not minimize automatically; call `minimize_crash` separately.
pub fn triage_crash(
    target: &impl FuzzTarget,
    crash_input: &[u8],
): Result[CrashReport, FuzzError]

/// Minimize a crashing input using delta debugging.
///
/// Repeatedly attempts to remove bytes from `crash_input` while preserving
/// the crash. Uses a binary search strategy (ddmin algorithm): tries removing
/// halves, then quarters, then individual bytes.
///
/// Returns the smallest input known to reproduce the crash. The returned
/// input may not be globally minimal (delta debugging is not exhaustive), but
/// is substantially smaller in practice.
pub fn minimize_crash(
    target: &impl FuzzTarget,
    crash_input: &[u8],
): Result[Vec[u8], FuzzError]

/// Check whether `input` reproducibly crashes `target`.
///
/// Runs `target.fuzz(input)` in a subprocess `n` times.
/// Returns `true` if every run crashes with the same signal.
/// Returns `false` if any run completes without crashing.
pub fn reproduce_crash(
    target: &impl FuzzTarget,
    input: &[u8],
    n: u32,
): bool
```

### Triage Workflow

The recommended triage workflow after discovering a crash file `crash-abc123`:

```
# 1. Triage (captures stack trace, checks reproducibility)
ferrum fuzz triage --target=asn1_der --input=crash-abc123

# 2. Minimize (produces crash-abc123-minimized)
ferrum fuzz minimize --target=asn1_der --input=crash-abc123

# 3. Review the minimized input and stack trace, file a bug with both
```

The `ferrum fuzz` subcommand orchestrates these steps; `extlib.fuzz` provides the
underlying API for tools that need programmatic access.

---

## 7. OSS-Fuzz Integration

OSS-Fuzz is Google's continuous fuzzing infrastructure for critical open-source software.
It runs fuzz targets 24/7 against the current main branch, reports findings to
maintainers, and tracks coverage over time.

`extlib.fuzz` generates all artifacts OSS-Fuzz requires:

```ferrum
use extlib.fuzz.{FuzzTarget, FuzzError, OssFuzzConfig}
use std.fs.Path

/// Configuration for an OSS-Fuzz project.
pub struct OssFuzzConfig {
    /// OSS-Fuzz project name (must match the project directory name in
    /// the OSS-Fuzz repository).
    pub project_name: &'static str,
    /// Language identifier. Always "ferrum" for Ferrum projects.
    pub language: &'static str,
    /// Fuzzing engines to build for. OSS-Fuzz runs all registered engines.
    pub fuzzing_engines: &'static [&'static str],
    /// Sanitizers to build with. OSS-Fuzz builds once per sanitizer.
    pub sanitizers: &'static [&'static str],
    /// Primary maintainer email for OSS-Fuzz vulnerability reports.
    pub primary_contact: &'static str,
    /// Auto-CC emails for vulnerability reports.
    pub auto_ccs: &'static [&'static str],
}

/// Default OSS-Fuzz configuration for extlib.fuzz.
pub const EXTLIB_OSS_FUZZ_CONFIG: OssFuzzConfig = OssFuzzConfig {
    project_name: "ferrum-extlib",
    language: "ferrum",
    fuzzing_engines: &["libfuzzer", "afl", "honggfuzz"],
    sanitizers: &["address", "memory", "undefined"],
    primary_contact: "security@ferrum-lang.org",
    auto_ccs: &[],
}

/// Build all registered fuzz targets for OSS-Fuzz submission.
///
/// For each target in `targets`, produces:
/// - A standalone fuzz binary at `<out_dir>/fuzz-<target-name>`
/// - A seed corpus archive at `<out_dir>/fuzz-<target-name>_seed_corpus.zip`
/// - A dictionary file at `<out_dir>/fuzz-<target-name>.dict` (if the target
///   provides one via `FuzzTarget::dictionary()`)
///
/// Also produces:
/// - `<out_dir>/build.sh` — the build script OSS-Fuzz runs in its Docker container
/// - `<out_dir>/project.yaml` — OSS-Fuzz project metadata
///
/// The generated `build.sh` sets `$CC`, `$CXX`, `$CFLAGS` from the OSS-Fuzz
/// environment and calls `ferrum build --fuzz-target=<name>` for each target.
pub fn oss_fuzz_build_targets(
    targets: &[&dyn FuzzTarget],
    config: &OssFuzzConfig,
    out_dir: &Path,
): Result[(), FuzzError] ! IO

/// Upload a corpus to a GCS bucket for OSS-Fuzz corpus storage.
///
/// OSS-Fuzz stores corpora in GCS at:
///   gs://<project>-corpus.clusterfuzz-external.appspot.com/libFuzzer/<target>/
///
/// Uploading an enhanced corpus (after local fuzzing) reduces the time for
/// OSS-Fuzz to reach deep code paths from scratch.
pub fn upload_corpus(
    corpus: &FuzzCorpus,
    gcs_bucket: &str,
): Result[(), FuzzError] ! Async + Net
```

### Generated `project.yaml`

```yaml
# Generated by extlib.fuzz::oss_fuzz_build_targets()
homepage: "https://ferrum-lang.org"
language: ferrum
primary_contact: "security@ferrum-lang.org"
fuzzing_engines:
  - libfuzzer
  - afl
  - honggfuzz
sanitizers:
  - address
  - memory
  - undefined
```

### Generated `build.sh` Structure

The generated build script:

1. Invokes `ferrum build --release --fuzz-target=<name>` once per target, with
   `$CFLAGS`/`$CXXFLAGS` forwarded to the Ferrum compiler's C integration layer.
2. Copies each fuzz binary to `$OUT/fuzz-<name>`.
3. Zips each target's seed corpus into `$OUT/fuzz-<name>_seed_corpus.zip`.
4. Copies dictionary files to `$OUT/fuzz-<name>.dict`.

---

## 8. Built-in Fuzz Targets

`extlib.fuzz` ships a fuzz target for every parser and decoder in the extended library
and in the relevant standard library modules. Each target is registered by name and
available via `--fuzz-target=<name>`.

### `Asn1FuzzTarget` — `asn1_der`

```ferrum
/// Fuzzes extlib.asn1 DER and BER decoding.
///
/// Input: arbitrary bytes.
/// Checks: no panic, no UB, no infinite loop, bounded allocation,
///         error returned for invalid encodings rather than crash.
///
/// Also checks that successfully decoded values round-trip: encode(decode(x)) == x
/// for all inputs that parse successfully.
pub struct Asn1FuzzTarget {}

impl FuzzTarget for Asn1FuzzTarget {
    fn name(): &'static str { "asn1_der" }
    fn description(): &'static str {
        "DER/BER decoding of arbitrary byte sequences via extlib.asn1. \
         Checks no panic, bounded allocation, and encode-decode round-trip \
         for valid inputs."
    }
    fn fuzz(data: &[u8]) { ... }
}
```

The `fuzz()` implementation attempts to decode `data` as each of the following ASN.1
types in sequence: `Integer`, `OctetString`, `BitString`, `Oid`, `Sequence`, `Set`,
`ContextTagged`. Decode errors are discarded. Successful decodes are re-encoded and
the output compared to the decoded input bytes (round-trip check). Any discrepancy is
a panic, which the fuzzer records as a finding.

### `CertFuzzTarget` — `cert_der`

```ferrum
/// Fuzzes extlib.cert X.509 certificate parsing.
///
/// Input: raw DER bytes purporting to be a Certificate structure.
/// Checks: parse errors returned cleanly as Err, never as panic.
///         If parsing succeeds, validates internal consistency:
///         notBefore <= notAfter, signature algorithm matches TBSCertificate.
pub struct CertFuzzTarget {}

impl FuzzTarget for CertFuzzTarget {
    fn name(): &'static str { "cert_der" }
    fn description(): &'static str { ... }
    fn fuzz(data: &[u8]) { ... }
}
```

### `JsonFuzzTarget` — `json_dom` and `json_stream`

Two separate targets are registered:

**`json_dom`**: Fuzzes the DOM parser (`extlib.ccsp.json.parse()`). Checks that no input
causes a panic. Checks that security limits (max depth, max string length) are enforced:
if an input exceeds configured limits, `Err` is returned before substantial memory is
allocated.

**`json_stream`**: Fuzzes the streaming/SAX-style parser. In addition to the panic and
limit checks, this target runs the same input through both DOM and streaming parsers and
verifies they agree on whether the input is valid JSON. If one returns `Ok` and the other
returns `Err`, that is a finding.

```ferrum
pub struct JsonFuzzTarget { mode: JsonFuzzMode }

pub enum JsonFuzzMode { Dom, Stream }

impl FuzzTarget for JsonFuzzTarget {
    fn name(): &'static str {
        match self.mode {
            JsonFuzzMode.Dom    => "json_dom",
            JsonFuzzMode.Stream => "json_stream",
        }
    }
    fn fuzz(data: &[u8]) { ... }
}
```

### `TomlFuzzTarget` — `toml`

```ferrum
/// Fuzzes extlib.ccsp.toml parsing.
///
/// Input: arbitrary bytes (treated as UTF-8; invalid UTF-8 returns Err quickly).
/// Checks:
///   - No panic on any input.
///   - Invalid TOML returns Err.
///   - Valid TOML round-trips: parse then serialize then parse again produces
///     an equivalent value tree.
pub struct TomlFuzzTarget {}
```

### `DnsFuzzTarget` — `dns_response`

```ferrum
/// Fuzzes extlib.ccsp.dns_secure DNS response wire-format parsing.
///
/// Input: raw bytes in DNS wire format (as received from the network).
/// Checks: no panic on malformed responses. Malformed inputs return Err.
///         No compression pointer loop causes an infinite loop.
///         Name length limits (255 bytes per name, 63 bytes per label) enforced.
pub struct DnsFuzzTarget {}
```

The compression pointer loop check is critical: DNS wire format allows name compression
using pointer offsets. A crafted packet with two pointers that point to each other
(a cycle) must be detected and rejected within bounded iterations, not loop forever.

### `TlsRecordFuzzTarget` — `tls_record`

```ferrum
/// Fuzzes extlib.ccsp.tls TLS record layer parsing.
///
/// Input: raw bytes purporting to be one or more TLS records.
/// Checks: no panic on malformed records. Content type, version, and length
///         fields are fully untrusted. Length fields exceeding TLS record
///         size limits (16384 + 256 bytes per RFC 8449) are rejected as Err.
pub struct TlsRecordFuzzTarget {}
```

### `TlsHandshakeFuzzTarget` — `tls_handshake`

```ferrum
/// Fuzzes extlib.ccsp.tls TLS handshake message parsing.
///
/// Input: raw bytes purporting to be a handshake message body (without the
///         record layer header — starts at HandshakeType byte).
/// Checks: no panic on any input. All extension types handled. Unknown
///         extensions return Err rather than panic.
pub struct TlsHandshakeFuzzTarget {}
```

### `RegexFuzzTarget` — `regex_compile` and `regex_match`

Two separate targets:

**`regex_compile`**: Feeds the input (interpreted as UTF-8; invalid UTF-8 is rejected
immediately) as a regex pattern to `extlib.regex.Regex::new()`. Any syntactically valid
pattern must compile without panic. Compilation errors are valid outcomes. This finds
crashes in the regex parser for unusual but syntactically valid patterns (deeply nested
groups, very long character classes, patterns with many alternations).

**`regex_match`**: Takes a fixed compiled pattern (a structurally complex but known-safe
pattern: `(a+)+b`, compiled in NFA mode) and runs the input as the match subject. Because
`extlib.regex` uses a Thompson NFA, this must complete in O(n) time regardless of input.
This target's timeout check is its primary value: if the fuzzer's per-input timeout fires,
the NFA guarantee has been violated.

```ferrum
pub struct RegexFuzzTarget { mode: RegexFuzzMode }

pub enum RegexFuzzMode {
    Compile,
    Match { pattern: extlib.regex.Regex },
}
```

### `AeadFuzzTarget` — `aead_decrypt`

```ferrum
/// Fuzzes std.crypto AEAD decryption.
///
/// Input layout (fixed offsets):
///   bytes [0..32]   — AES-256-GCM key (32 bytes)
///   bytes [32..44]  — nonce (12 bytes)
///   bytes [44..]    — ciphertext + tag (variable length)
///
/// The target also runs ChaCha20-Poly1305 with the same key, nonce, and
/// ciphertext (key length is compatible: both use 32-byte keys and 12-byte nonces).
///
/// Checks: authentication failure returned as Err(AeadError::AuthenticationFailed),
///         never as panic. Short inputs (fewer than 44 + TAG_LEN bytes) return
///         Err immediately. The decryption output is discarded.
///
/// This target primarily checks that the authentication tag verification path
/// is panic-free for arbitrary ciphertext. It does not find AEAD correctness
/// bugs (those belong in property tests with known test vectors).
pub struct AeadFuzzTarget {}
```

### `CborFuzzTarget` — `cbor`

```ferrum
/// Fuzzes std.data.cbor CBOR decoding.
///
/// Input: raw bytes in CBOR encoding (RFC 7049 / RFC 8949).
/// Checks: no panic. Indefinite-length items bounded. Nested structure
///         depth limited (default: 64). Map keys deduplicated check:
///         if a map decodes successfully, no duplicate keys are present.
pub struct CborFuzzTarget {}
```

### `HttpRequestFuzzTarget` — `http_request`

```ferrum
/// Fuzzes std.http HTTP/1.1 request parsing.
///
/// Input: raw bytes purporting to be an HTTP/1.1 request (method, URI,
///         headers, optional body).
/// Checks: no panic. Request-line parsing rejects inputs that exceed
///         configurable URI length limits. Header count and total header
///         size limits enforced before allocation.
pub struct HttpRequestFuzzTarget {}
```

### `HttpResponseFuzzTarget` — `http_response`

```ferrum
/// Fuzzes std.http HTTP/1.1 response parsing.
///
/// Input: raw bytes purporting to be an HTTP/1.1 response (status line,
///         headers, optional body via Content-Length or chunked encoding).
/// Checks: no panic. Chunked encoding extension parsing does not loop.
///         Content-Length value is not trusted for pre-allocation beyond
///         a configured limit.
pub struct HttpResponseFuzzTarget {}
```

### `SmtpFuzzTarget` — `smtp_command`

```ferrum
/// Fuzzes extlib.ccsp.lineproto-based SMTP command parsing.
///
/// Input: raw bytes purporting to be one SMTP command line (EHLO, MAIL FROM,
///         RCPT TO, DATA, etc.) including CRLF terminator.
/// Checks: no panic. RFC 5321 line length limit (1000 bytes including CRLF)
///         enforced. Address parsing does not allocate unboundedly.
pub struct SmtpFuzzTarget {}
```

### `ImapFuzzTarget` — `imap_response`

```ferrum
/// Fuzzes extlib.ccsp.imap IMAP4 response parsing.
///
/// Input: raw bytes purporting to be one IMAP response line.
/// Checks: no panic. Literal size field (e.g., {12345}) not trusted for
///         pre-allocation beyond configured limit. Parenthesized list
///         nesting depth limited.
pub struct ImapFuzzTarget {}
```

### `PkiFuzzTarget` — `pki_import`

```ferrum
/// Fuzzes extlib.pki key and bundle import.
///
/// Two sub-modes, selected by the first byte of input:
///   0x00 — PKCS#8 private key import (DER): feeds remaining bytes to
///           extlib.pki.PrivateKey::from_pkcs8_der()
///   0x01 — PKCS#12 bundle parsing: feeds remaining bytes to
///           extlib.pki.Pkcs12::parse() with empty passphrase
///   other — no-op, returns immediately
///
/// Checks: no panic. PKCS#12 MAC verification failure returned as Err.
///         Deeply nested SafeContents structures do not overflow the stack.
pub struct PkiFuzzTarget {}
```

### Summary Table

| Target name | Module fuzzed | Primary check |
|---|---|---|
| `asn1_der` | `extlib.asn1` | No panic, bounded alloc, round-trip |
| `cert_der` | `extlib.cert` | No panic, internal consistency |
| `json_dom` | `extlib.ccsp.json` | No panic, depth limit enforced |
| `json_stream` | `extlib.ccsp.json` | DOM/stream agreement on validity |
| `toml` | `extlib.ccsp.toml` | No panic, round-trip on valid input |
| `dns_response` | `extlib.ccsp.dns_secure` | No panic, no pointer loop |
| `tls_record` | `extlib.ccsp.tls` | No panic, length limit enforced |
| `tls_handshake` | `extlib.ccsp.tls` | No panic, unknown extensions handled |
| `regex_compile` | `extlib.regex` | No panic on any valid pattern |
| `regex_match` | `extlib.regex` | NFA completes within timeout |
| `aead_decrypt` | `std.crypto` | Auth failure as Err, never panic |
| `cbor` | `std.data.cbor` | No panic, depth limit, no duplicate keys |
| `http_request` | `std.http` | No panic, length limits enforced |
| `http_response` | `std.http` | No panic, chunked encoding bounded |
| `smtp_command` | `extlib.ccsp.lineproto` | No panic, line length enforced |
| `imap_response` | `extlib.ccsp.imap` | No panic, literal size bounded |
| `pki_import` | `extlib.pki` | No panic, nesting depth bounded |

---

## 9. Differential Fuzzing

Differential fuzzing runs two implementations of the same interface on the same input and
flags any disagreement. This is particularly effective for catching bugs where one
implementation accepts input that the other rejects (a security boundary violation) or
where both accept but produce different output (a correctness bug).

```ferrum
use extlib.fuzz.{FuzzTarget, DiffFuzzTarget, DiffFinding}

/// A fuzz target that runs two implementations on each input and compares outcomes.
pub struct DiffFuzzTarget {
    target_name: &'static str,
    target_description: &'static str,
    a: Box[dyn FuzzTarget],
    b: Box[dyn FuzzTarget],
}

/// The outcome of one call to `fuzz()` on one implementation.
pub enum FuzzOutcome {
    /// Completed without panic (parse error, success, auth failure — all non-crash outcomes).
    Completed,
    /// Panicked (caught by the differential runner before it terminates the process).
    Panicked { message: String },
}

/// A disagreement between the two implementations on one input.
@derive(Debug)]
pub struct DiffFinding {
    pub input: Vec[u8],
    pub outcome_a: FuzzOutcome,
    pub outcome_b: FuzzOutcome,
    pub kind: DiffKind,
}

/// Classification of the disagreement.
pub enum DiffKind {
    /// One implementation panicked and the other did not.
    /// The panicking implementation has a bug; the other is the oracle.
    OneOnlyPanicked,
    /// Both completed, but one returned Ok and the other returned Err.
    /// This indicates a security-relevant validity disagreement.
    ValidityDisagreement,
    /// Both completed and returned Ok, but the parsed values differ.
    /// This indicates a correctness bug in one or both implementations.
    OutputDisagreement,
}

impl DiffFuzzTarget {
    /// Create a new differential fuzz target.
    ///
    /// `name` must be unique across all registered targets.
    /// `a` and `b` must implement the same semantic interface (same input format,
    /// same validity criteria).
    pub fn new(
        name: &'static str,
        description: &'static str,
        a: Box[dyn FuzzTarget],
        b: Box[dyn FuzzTarget],
    ): Self

    /// Run one input through both implementations.
    ///
    /// If a disagreement is detected, appends to `findings` and returns.
    /// This does not panic on disagreement — the caller collects findings
    /// and reports them after the fuzzing session.
    pub fn fuzz_diff(
        self: &Self,
        data: &[u8],
        findings: &mut Vec[DiffFinding],
    )
}

impl FuzzTarget for DiffFuzzTarget {
    fn name(): &'static str { self.target_name }
    fn description(): &'static str { self.target_description }

    /// Calls `fuzz_diff` with an internal findings buffer.
    /// Panics if a finding is detected, so the fuzzer records the input.
    fn fuzz(data: &[u8]) { ... }
}
```

### Use Case: DOM vs. Streaming JSON

The primary use case in `extlib.fuzz` is the `json_stream` target described in §8. The
DOM parser and streaming parser in `extlib.ccsp.json` are independent implementations
that share test suites but have different internal code paths. They must agree on whether
any given byte sequence is valid JSON. A differential fuzz target makes this invariant
continuously checked:

```ferrum
let json_diff = DiffFuzzTarget::new(
    "json_validity",
    "Check that DOM and streaming JSON parsers agree on validity for all inputs.",
    Box::new(JsonFuzzTarget { mode: JsonFuzzMode.Dom }),
    Box::new(JsonFuzzTarget { mode: JsonFuzzMode.Stream }),
)
```

### Use Case: Two ASN.1 Decoder Modes

BER decoding is a strict superset of DER decoding. For any input that is valid DER, the
DER decoder and the BER decoder must agree it is valid and must produce identical values.
A differential target between DER and BER modes with DER-valid seed corpus finds
violations of this subset relationship.

---

## 10. Coverage Reporting

Coverage reporting measures what fraction of the codebase a corpus exercises. It is
useful for evaluating whether a corpus is adequate before submitting to OSS-Fuzz, and for
identifying which code paths are never reached by any seed.

```ferrum
use extlib.fuzz.{FuzzTarget, CoverageReport, Location}
use std.io.Write

/// A source location (file, line, column).
@derive(Debug, Clone, PartialEq)]
pub struct Location {
    pub file: String,
    pub line: u32,
    pub column: u32,
}

/// Coverage summary from running a corpus through a target.
@derive(Debug)]
pub struct CoverageReport {
    /// Number of source lines executed by at least one input in the corpus.
    pub lines_covered: usize,
    /// Total number of instrumented source lines.
    pub total_lines: usize,
    /// Number of branches (if/else, match arms) covered by at least one input.
    pub branches_covered: usize,
    /// Total number of instrumented branches.
    pub total_branches: usize,
    /// Locations covered by the most recent run that were not covered before.
    /// Populated only when called incrementally (e.g., after adding new seeds).
    pub new_coverage: Vec[Location],
}

impl CoverageReport {
    /// Line coverage as a percentage (0.0 to 100.0).
    pub fn line_percent(self: &Self): f64 {
        (self.lines_covered as f64 / self.total_lines as f64) * 100.0
    }

    /// Branch coverage as a percentage (0.0 to 100.0).
    pub fn branch_percent(self: &Self): f64 {
        (self.branches_covered as f64 / self.total_branches as f64) * 100.0
    }

    /// Write a human-readable summary to `writer`.
    ///
    /// Output format:
    ///   Lines:    1234 / 1500 (82.3%)
    ///   Branches:  876 / 1200 (73.0%)
    ///   New coverage: 3 locations
    pub fn print_summary(
        self: &Self,
        writer: &mut impl Write,
    ): Result[(), std.io.IoError] ! IO
}

/// Run all inputs through `target` with source coverage instrumentation and
/// return a coverage report.
///
/// Requires compilation with `--coverage` (enables `llvm-cov` instrumentation
/// or `gcov` depending on the backend). If coverage instrumentation is not
/// available in the current build, returns an error.
///
/// The `inputs` slice is processed sequentially; coverage is accumulated
/// across all inputs.
pub fn run_with_coverage(
    target: &impl FuzzTarget,
    inputs: &[&[u8]],
): CoverageReport
```

### Integration with `llvm-cov`

When compiled with `--coverage`, the Ferrum compiler emits LLVM source-based coverage
instrumentation (`-fprofile-instr-generate -fcoverage-mapping`). After running
`run_with_coverage`, the raw profile data is merged with `llvm-profdata merge` and
converted to a coverage report with `llvm-cov report`. `extlib.fuzz` wraps this process
and parses the output into `CoverageReport`.

For projects using `gcov`-based toolchains, the `--coverage-format=gcov` build flag
switches to `gcov`/`lcov` output instead.

---

## 11. Structured Fuzz Input

Raw mutation fuzzing works well for binary formats. For targets that need to split their
input into multiple typed fields, raw mutation produces many inputs that fail fast on
the first field, never exercising the interaction between fields. `FuzzData` is a helper
that reads structured fields from a raw byte slice without panicking.

```ferrum
use extlib.fuzz.FuzzData

/// A cursor over raw fuzz input bytes that provides typed reads.
///
/// All read methods are infallible: when the underlying byte slice is exhausted,
/// they return zero, false, empty slice, or empty string as appropriate.
/// The cursor never panics, regardless of input.
pub type FuzzData<'a> {
    data: &'a [u8],
    pos: usize,
}

impl<'a> FuzzData<'a> {
    /// Wrap a raw byte slice for structured reading.
    pub fn new(data: &'a [u8]): Self {
        FuzzData { data, pos: 0 }
    }

    /// Read one byte. Returns 0 if exhausted.
    pub fn read_u8(self: &mut Self): u8

    /// Read two bytes (big-endian). Returns 0 if fewer than 2 bytes remain.
    pub fn read_u16(self: &mut Self): u16

    /// Read four bytes (big-endian). Returns 0 if fewer than 4 bytes remain.
    pub fn read_u32(self: &mut Self): u32

    /// Read eight bytes (big-endian). Returns 0 if fewer than 8 bytes remain.
    pub fn read_u64(self: &mut Self): u64

    /// Read one byte and return true if nonzero, false if zero or exhausted.
    pub fn read_bool(self: &mut Self): bool

    /// Read exactly `n` bytes. Returns a shorter slice if fewer remain.
    /// Returns an empty slice if exhausted.
    pub fn read_bytes(self: &mut Self, n: usize): &'a [u8]

    /// Return all remaining bytes without advancing the cursor.
    pub fn peek_remaining(self: &Self): &'a [u8]

    /// Return all remaining bytes and advance the cursor to the end.
    pub fn read_remaining(self: &mut Self): &'a [u8]

    /// Read bytes until a valid UTF-8 boundary is found.
    ///
    /// Reads up to `max_len` bytes from the current position.
    /// Truncates at the last valid UTF-8 character boundary within those bytes.
    /// Returns an empty string if no valid UTF-8 prefix exists.
    /// Never panics.
    pub fn read_utf8_str(self: &mut Self, max_len: usize): &'a str

    /// Number of bytes remaining.
    pub fn remaining(self: &Self): usize

    /// Whether all bytes have been consumed.
    pub fn is_exhausted(self: &Self): bool
}
```

### Design Note: Why Not Panic on Exhaustion?

`FuzzData` returns zero/false/empty rather than panicking when exhausted. This is
intentional. A fuzz target that panics when its input is too short is not finding a bug in
the code under test — it is finding a bug in the fuzz target itself. The fuzzer will
record that input as a crash in the target rather than in the parser being tested. All
defensive behavior must be in the fuzz target's own logic, not in the helpers.

### Use Case: Splitting Input into Fields

```ferrum
impl FuzzTarget for AeadFuzzTarget {
    fn fuzz(data: &[u8]) {
        let mut fd = FuzzData::new(data)
        let key   = fd.read_bytes(32)   // 32-byte key; empty slice if data < 32
        let nonce = fd.read_bytes(12)   // 12-byte nonce
        let ct    = fd.read_remaining() // remaining bytes = ciphertext + tag

        // read_bytes returns a slice possibly shorter than requested.
        // Callers that need exactly N bytes check the length explicitly
        // and return early (not panic) if insufficient.
        if key.len() < 32 || nonce.len() < 12 {
            return  // not enough data for a well-formed input; skip silently
        }

        // Safe: lengths checked above.
        let key_arr: &[u8; 32] = key.try_into().unwrap_or(return)
        let nonce_arr: &[u8; 12] = nonce.try_into().unwrap_or(return)

        let _ = std.crypto.Aes256Gcm::new().decrypt_in_place(
            key_arr, nonce_arr, &[], ct, &mut Vec::new()
        )
        // Err(AeadError::AuthenticationFailed) expected for random input.
        // Panic is a bug.
    }
}
```

---

## 12. Example Usage

### ASN.1 Fuzz Target (full implementation)

```ferrum
use extlib.fuzz.{FuzzTarget, FuzzData}
use extlib.asn1.{Any, DerDecode, DerEncode, Asn1Error}

pub struct Asn1FuzzTarget {}

impl FuzzTarget for Asn1FuzzTarget {
    fn name(): &'static str { "asn1_der" }

    fn description(): &'static str {
        "Fuzzes extlib.asn1 DER decoding. Attempts to decode input as each \
         primitive ASN.1 type. Checks no panic, bounded allocation, and that \
         successfully decoded values re-encode to bytes parseable as the same type."
    }

    fn fuzz(data: &[u8]) {
        // Attempt decode as raw Any (accepts any valid DER tag-length-value).
        match Any::from_der(data) {
            Result.Ok((any, _rest)) => {
                // Round-trip check: re-encode and re-decode.
                let reencoded = any.to_der()
                match Any::from_der(&reencoded) {
                    Result.Ok((any2, _)) => {
                        // The two decoded values must be identical.
                        // If not, panic — the fuzzer records this input.
                        assert_eq!(any, any2,
                            "ASN.1 round-trip mismatch: encode(decode(x)) != decode(encode(decode(x)))")
                    }
                    Result.Err(e) => {
                        // Encoding produced bytes that cannot be decoded — bug.
                        panic!("ASN.1 encode produced undecodable output: {e}")
                    }
                }
            }
            Result.Err(_) => {
                // Invalid DER — expected outcome for most random inputs.
                // Silently discard and return.
            }
        }
    }
}
```

### Differential JSON Fuzzing

```ferrum
use extlib.fuzz.{DiffFuzzTarget, FuzzCorpus}
use extlib.fuzz.targets.{JsonFuzzTarget, JsonFuzzMode}

fn build_json_diff_target(): DiffFuzzTarget {
    DiffFuzzTarget::new(
        "json_validity",
        "DOM and streaming JSON parsers must agree on whether each input is valid JSON.",
        Box::new(JsonFuzzTarget { mode: JsonFuzzMode.Dom }),
        Box::new(JsonFuzzTarget { mode: JsonFuzzMode.Stream }),
    )
}

fn run_json_differential_fuzz() ! IO {
    let target = build_json_diff_target()
    let corpus = FuzzCorpus::new(&Path::from("corpus/json_validity"))?

    // Seed with known-valid and known-invalid JSON.
    corpus.add_seed(b"null")?
    corpus.add_seed(b"true")?
    corpus.add_seed(b"42")?
    corpus.add_seed(b"\"hello\"")?
    corpus.add_seed(b"[1,2,3]")?
    corpus.add_seed(b"{\"k\":\"v\"}")?
    corpus.add_seed(b"{")?       // invalid: unclosed object
    corpus.add_seed(b"[1,]")?    // invalid: trailing comma

    // Run under LibFuzzer with standard flags.
    let flags = corpus.libfuzzer_flags()
    // flags: ["-max_len=65536", "-timeout=10", "-jobs=4", "-workers=4", "corpus/json_validity"]

    extlib.fuzz.libfuzzer_main(&target)
    // Does not return.
}
```

### Corpus Minimization Workflow

```ferrum
use extlib.fuzz.{FuzzCorpus, Asn1FuzzTarget}
use std.fs.Path

fn minimize_asn1_corpus() ! IO {
    let target = Asn1FuzzTarget {}
    let corpus = FuzzCorpus::new(&Path::from("corpus/asn1_der"))?
    let out    = Path::from("corpus/asn1_der_minimized")

    let stats = corpus.minimize(&target, &out)?

    println!("Corpus minimization complete:")
    println!("  Before: {} inputs", stats.before)
    println!("  After:  {} inputs", stats.after)
    println!("  Coverage preserved: {}", stats.coverage_preserved)

    // Replace old corpus with minimized version.
    // (Caller confirms before executing the rename.)
}
```

### OSS-Fuzz Build Script Generation

```ferrum
use extlib.fuzz.{oss_fuzz_build_targets, EXTLIB_OSS_FUZZ_CONFIG}
use extlib.fuzz.targets.*
use std.fs.Path

fn generate_oss_fuzz_artifacts() ! IO {
    let targets: Vec[&dyn FuzzTarget] = vec![
        &Asn1FuzzTarget {},
        &CertFuzzTarget {},
        &JsonFuzzTarget { mode: JsonFuzzMode.Dom },
        &JsonFuzzTarget { mode: JsonFuzzMode.Stream },
        &TomlFuzzTarget {},
        &DnsFuzzTarget {},
        &TlsRecordFuzzTarget {},
        &TlsHandshakeFuzzTarget {},
        &RegexFuzzTarget { mode: RegexFuzzMode.Compile },
        &RegexFuzzTarget { mode: RegexFuzzMode.Match {
            pattern: extlib.regex.Regex::new("(a+)+b").unwrap()
        }},
        &AeadFuzzTarget {},
        &CborFuzzTarget {},
        &HttpRequestFuzzTarget {},
        &HttpResponseFuzzTarget {},
        &SmtpFuzzTarget {},
        &ImapFuzzTarget {},
        &PkiFuzzTarget {},
    ]

    oss_fuzz_build_targets(
        &targets,
        &EXTLIB_OSS_FUZZ_CONFIG,
        &Path::from(std.env.var("OUT").unwrap_or("/tmp/oss-fuzz-out")),
    )?

    println!("OSS-Fuzz artifacts written.")
}
```

---

## 13. Dependencies

### Modules Being Fuzzed

| Module | Provided by |
|---|---|
| `extlib.asn1` | `lib_ccsp_asn1` |
| `extlib.cert` | `lib_ccsp_cert` |
| `extlib.ccsp.json` | `lib_ccsp_json` |
| `extlib.ccsp.toml` | `lib_ccsp_toml` |
| `extlib.ccsp.dns_secure` | `lib_ccsp_dns` |
| `extlib.ccsp.tls` | `lib_ccsp_tls` |
| `extlib.ccsp.lineproto` | `lib_ccsp_lineproto` (SMTP) |
| `extlib.ccsp.imap` | `lib_ccsp_imap` |
| `extlib.pki` | `lib_ccsp_pki` |
| `extlib.regex` | `lib_ccsp_regex` |
| `std.crypto` | Ferrum standard library |
| `std.data.cbor` | Ferrum standard library |
| `std.http` | Ferrum standard library |

### Standard Library Dependencies

| Module | Use |
|---|---|
| `std.testing` | `assert_eq!`, `assert!` macros used in round-trip checks |
| `std.io` | Corpus file I/O, crash report writing |
| `std.fs` | `Path`, directory iteration |
| `std.crypto.hash` | SHA-256 for corpus file naming |
| `std.process` | Subprocess execution for crash reproduction |

### External Fuzzing Engine Dependencies

All external dependencies are optional and gated on `@cfg` flags set by
`--fuzz-target` and `--engine` build flags. They are never linked into production
binaries.

| Dependency | Flag | Notes |
|---|---|---|
| LibFuzzer (`libFuzzer.a`) | `@cfg(libfuzzer)` | Bundled with LLVM toolchain; no separate install required if Ferrum uses LLVM backend |
| AFL++ compiler runtime (`afl-compiler-rt.o`) | `@cfg(afl)` | Requires AFL++ installed; `ferrum fuzz check-engines` verifies availability |
| honggfuzz runtime (`libhfuzz.a`) | `@cfg(honggfuzz)` | Optional; required only for OSS-Fuzz honggfuzz builds |

### `Cargo.toml`-equivalent Dependencies

```toml
[dependencies]
extlib.asn1       = { version = "1.0" }
extlib.cert       = { version = "1.0" }
extlib.ccsp.json  = { version = "1.0" }
extlib.ccsp.toml  = { version = "1.0" }
extlib.ccsp.dns   = { version = "1.0" }
extlib.ccsp.tls   = { version = "1.0" }
extlib.ccsp.imap  = { version = "1.0" }
extlib.pki        = { version = "1.0" }
extlib.regex      = { version = "1.0" }

[dev-dependencies]
# None: extlib.fuzz IS the dev/testing infrastructure, not a consumer of it.

[fuzz-dependencies]
# Linked only when building with --fuzz-target=<name>
libfuzzer-sys     = { optional = true }
afl-sys           = { optional = true }
honggfuzz-sys     = { optional = true }
```
