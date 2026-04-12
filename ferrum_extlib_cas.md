# Ferrum Extended Library — `ccsp.cas`: Content-Addressable Storage

**Module path:** `extlib.ccsp.cas`
**Part of:** Extended Standard Library (not `std` or `core`)
**Companion:** [Ferrum Standard Library](ferrum-stdlib.md)
**Dependencies (core):** `std.crypto` (SHA-256), `std.fs`, `std.alloc`
**Dependencies (remote backend, optional):** `std.async`, `std.net`

---

## 1. Overview and Rationale

### What Content-Addressable Storage Is

In a content-addressable store, the address of an object **is** the hash of its content. You do not choose where to put something; the store computes the location from what you put there. This property has two immediate consequences:

**Identity implies integrity.** If you read an object at address `A` and its SHA-256 hash equals `A`, you know the content is exactly what was stored, with no modification in transit or at rest. There is no separate checksum step; the address is the checksum.

**Writes are idempotent.** Writing the same content twice produces the same address and the same stored object. There is no duplication, no conflict, no version number needed for immutable data. The same blob stored in ten different package versions occupies one slot in the store.

**Content can be verified by anyone.** Knowing the address is sufficient to verify any copy of the data, regardless of where it came from. This makes CAS stores naturally suited to distributed and replicated settings.

### The Git Object Model

`extlib.ccsp.cas` uses the Git object model as its on-disk format. Git has stored the source history of millions of projects for over twenty years. Its object model is well-specified, battle-tested, and understood by a large ecosystem of tools. Using a compatible format means:

- Existing Git tooling can inspect, verify, and debug a `cas` store.
- The object types (blob, tree, commit, tag) map naturally to the use cases described below.
- Migration from or to a Git repository is straightforward.

The one intentional deviation: this module uses **SHA-256**, not SHA-1. Git migrated to SHA-256 (git format version 1) for exactly the same reasons this module uses it from the start: SHA-1 is cryptographically broken for collision resistance, and any content-addressable system used for security-sensitive work (package management, build caches, config auditing) must not rely on it.

### Use Cases

**Build caches.** A build system hashes all inputs to a build step — source files, compiler binary, flags — and uses the resulting tree hash as the cache key. If the output object exists in the store under that key, the step is skipped. Outputs are themselves stored as blobs or trees, addressed by their own content.

**Package stores.** Immutable packages are stored once by content hash. Multiple versions of a package that share files share the blobs. Installing, upgrading, and rolling back are pointer operations on commit chains, not file copies.

**Configuration management.** Each configuration snapshot is a commit pointing at a tree of config files. The commit chain is a rollback log. Every prior state is recoverable as long as the commit hash is known, with hash-verified integrity at every step.

**Distributed synchronization.** Two nodes synchronizing content can compare object IDs without transferring data. Only objects absent from the destination need to be sent. The hash verifies correctness on arrival.

### Why Extended Library, Not Stdlib

The `cas` module depends on `std.crypto` for SHA-256, which is in stdlib. But:

- Most programs do not need a content-addressable object store. The stdlib should not carry it.
- The remote backend depends on `std.async` and `std.net`, which are heavyweight.
- The Git-compatible wire format involves a degree of format complexity (object header framing, zlib compression, tree entry sorting rules) that belongs behind an explicit dependency.

Programs that need CAS — build systems, package managers, config management tools, synchronization daemons — opt in by depending on `extlib.ccsp.cas`.

---

## 2. Core Types

### 2.1 `ObjectId`

An `ObjectId` is a SHA-256 hash. It is the address and the identity of a stored object.

```ferrum
/// SHA-256 content address. 32 bytes. Equality is constant-time.
///
/// An ObjectId is both the address of an object in the store and its
/// integrity check. Knowing the id is sufficient to verify any copy
/// of the object's content.
struct ObjectId {
    bytes: [u8; 32],
}

impl ObjectId {
    /// Construct from raw bytes.
    fn from_bytes(bytes: [u8; 32]): Self

    /// Parse a 64-character lowercase hex string.
    /// Returns Err if the string is not exactly 64 lowercase hex digits.
    fn from_hex(s: &str): Result[Self, CasError]

    /// Render as a 64-character lowercase hex string.
    fn to_hex(&self): String

    /// The raw 32-byte representation.
    fn as_bytes(&self): &[u8; 32]
}

// Display: lowercase hex, 64 characters.
impl Display for ObjectId { ... }

// Equality is constant-time to prevent timing side channels.
// ObjectId values appear in security-sensitive contexts (package verification,
// build attestation); timing-variable equality would be a vulnerability.
impl PartialEq for ObjectId { ... }   // constant-time
impl Eq for ObjectId { ... }
impl Hash for ObjectId { ... }
```

`ObjectId` implements `Copy`. It is 32 bytes; copying is cheaper than managing a reference lifetime in the calling patterns where it appears.

### 2.2 `ObjectKind`

```ferrum
/// The four object types in the Git object model.
enum ObjectKind {
    /// Raw bytes. Any content. The leaf node of every tree.
    Blob,

    /// A sorted list of named entries, each pointing at a Blob, Tree,
    /// or submodule Commit. The directory node.
    Tree,

    /// A snapshot: points at a Tree, zero or more parent Commits,
    /// author/committer metadata, and a message.
    Commit,

    /// An annotated tag: points at any object, with tagger metadata
    /// and a message.
    Tag,
}

impl ObjectKind {
    /// The string used in Git object headers: "blob", "tree", "commit", "tag".
    fn as_str(&self): &'static str
}

impl Display for ObjectKind { ... }
```

### 2.3 `Object` and `OwnedObject`

```ferrum
/// A borrowed view of a stored object: kind + raw bytes.
///
/// The lifetime 'a ties this view to the storage that holds the data.
/// Use OwnedObject when you need to store or return the object
/// independently of the backing store.
struct Object[' a] {
    kind: ObjectKind,
    data: &'a [u8],
}

impl[' a] Object[' a] {
    fn kind(&self): ObjectKind
    fn data(&self): &[u8]
    fn id(&self): ObjectId     // recomputes SHA-256; not cached
    fn to_owned(&self): OwnedObject
}

/// An owned object: kind + heap-allocated bytes.
struct OwnedObject {
    kind: ObjectKind,
    data: Vec[u8],
}

impl OwnedObject {
    fn kind(&self): ObjectKind
    fn data(&self): &[u8]
    fn id(&self): ObjectId     // recomputes SHA-256; not cached
    fn as_ref[' a](&' a self): Object[' a]
}
```

---

## 3. Object Types

The Git object model defines four concrete types. This section describes each type, its serialization format, and how to construct it.

All `serialize()` methods produce an `OwnedObject` whose `.data` field contains the Git-format wire bytes. The Git format is:

```
"<kind> <size>\0<content>"
```

where `<kind>` is `blob`, `tree`, `commit`, or `tag`, `<size>` is the decimal byte count of `<content>`, and `\0` is a null byte. The SHA-256 of this complete byte sequence is the object ID.

### 3.1 Blobs

A blob is raw bytes. It carries no metadata; it is simply content.

```ferrum
/// Convenience: produce the (id, object) pair for a blob in one call.
///
/// This is equivalent to constructing the Git blob wire format
/// ("blob <n>\0<data>"), hashing it, and wrapping it in an OwnedObject.
/// The caller can then pass the OwnedObject to ObjectStore::write_raw.
fn blob_from_bytes(data: &[u8]): (ObjectId, OwnedObject)
```

The `write_blob` high-level function (§8) wraps this and writes directly to a store.

### 3.2 Trees

A tree is a sorted list of named entries. Each entry points at a blob, a sub-tree, a symlink target (blob), or a submodule commit.

```ferrum
/// The permission/type mode of a tree entry.
/// These are the exact mode values used in the Git tree format.
enum TreeEntryMode {
    Blob,             // 100644 — regular file
    BlobExecutable,   // 100755 — executable file
    Tree,             // 040000 — subdirectory
    Symlink,          // 120000 — symbolic link (target stored as blob)
    Commit,           // 160000 — submodule (gitlink)
}

impl TreeEntryMode {
    /// The octal string written into the Git tree format.
    fn as_git_mode_str(&self): &'static str
}

/// One entry in a tree object.
struct TreeEntry {
    mode: TreeEntryMode,
    name: String,     // single path component; must not contain '/' or '\0'
    id:   ObjectId,
}

/// A tree object: a sorted directory listing.
///
/// Git requires entries to be sorted. The sort key for files is the
/// entry name; the sort key for directories is the entry name with
/// a '/' appended. This matches `git ls-tree` order.
struct Tree {
    entries: Vec[TreeEntry],
}

impl Tree {
    /// Construct, sorting entries into Git tree order.
    /// Returns CasError::ParseError if any entry name contains '/' or '\0'.
    fn new(entries: Vec[TreeEntry]): Result[Self, CasError]

    /// Serialize to the Git tree wire format, returning an OwnedObject.
    fn serialize(&self): OwnedObject

    /// Compute the object ID without materializing the full OwnedObject.
    fn id(&self): ObjectId
}
```

### 3.3 Commits

A commit records a snapshot of a tree, zero or more parent commits, authorship, and a message.

```ferrum
/// Author or committer identity with timestamp.
struct CommitAuthor {
    name:      String,
    email:     String,
    timestamp: i64,       // Unix epoch seconds
    offset:    i16,       // UTC offset in minutes (e.g. -480 for UTC-8)
}

impl CommitAuthor {
    fn new(name: String, email: String, timestamp: i64, offset: i16): Self

    /// Format as the Git "ident line": "Name <email> <ts> <+HHMM>"
    fn to_git_ident(&self): String
}

/// A commit object.
struct Commit {
    /// The tree this commit snapshots.
    tree:     ObjectId,

    /// Parent commits. Empty for an initial commit. Two for a merge commit.
    parents:  Vec[ObjectId],

    /// Who wrote the content.
    author:   CommitAuthor,

    /// Who created this commit object (may differ from author after rebase).
    committer: CommitAuthor,

    /// Commit message. Conventionally: subject line, blank line, body.
    message: String,

    /// Arbitrary key-value headers inserted after "committer" and before
    /// the blank line that precedes the message. Used by git-notes,
    /// signed commits (gpgsig), and application-defined metadata.
    extra_headers: Vec[(String, String)],
}

impl Commit {
    fn new(tree: ObjectId, parents: Vec[ObjectId],
           author: CommitAuthor, committer: CommitAuthor,
           message: String): Self

    /// Serialize to the Git commit wire format.
    fn serialize(&self): OwnedObject

    /// Compute the object ID.
    fn id(&self): ObjectId
}
```

### 3.4 Tags

An annotated tag points at any object with tagger identity and a message. Lightweight tags (bare refs pointing at commits) are outside the scope of this module — they are a ref-management concept, not an object type.

```ferrum
/// An annotated tag object.
struct Tag {
    /// The object being tagged. May be any ObjectKind.
    object:  ObjectId,
    kind:    ObjectKind,

    /// The tag name (e.g. "v1.2.3"). Not a ref path; just the name.
    name:    String,

    tagger:  CommitAuthor,
    message: String,
}

impl Tag {
    fn serialize(&self): OwnedObject
    fn id(&self): ObjectId
}
```

---

## 4. The `ObjectStore` Trait

All backends implement `ObjectStore`. Code that is generic over the backend takes `impl ObjectStore` or `&impl ObjectStore`.

```ferrum
trait ObjectStore {
    /// Write an object of the given kind with the given content.
    ///
    /// Computes the Git-format SHA-256 hash (over the "kind size\0content"
    /// header), stores the object, and returns the id.
    ///
    /// Idempotent: writing the same content twice is not an error.
    /// The second write may be a no-op if the object already exists.
    fn write(&self, kind: ObjectKind, data: &[u8]): Result[ObjectId, CasError] ! IO

    /// Read a stored object by id.
    ///
    /// Recomputes the hash after reading. Returns CasError::HashMismatch
    /// if the stored bytes do not hash to the requested id.
    fn read(&self, id: &ObjectId): Result[OwnedObject, CasError] ! IO

    /// Read and verify, returning only the raw content bytes.
    ///
    /// Equivalent to read(id)?.data, but may be more efficient on backends
    /// that can stream without fully buffering. Hash is verified before
    /// any bytes are returned to the caller.
    fn read_bytes(&self, id: &ObjectId): Result[Vec[u8], CasError] ! IO

    /// Check whether an object exists without reading it.
    fn exists(&self, id: &ObjectId): Result[bool, CasError] ! IO

    /// Delete an object.
    ///
    /// Returns Ok(()) if the object did not exist (idempotent delete).
    /// Callers are responsible for reachability; the store does not
    /// enforce referential integrity. Use gc (§5.4) for safe collection.
    fn delete(&self, id: &ObjectId): Result[(), CasError] ! IO
}
```

### Writing Raw Objects

For callers that have already serialized an `OwnedObject` (e.g. from `Tree::serialize()`), there is a convenience method provided by a blanket impl on `ObjectStore`:

```ferrum
// Provided by blanket impl:
fn write_object(&self, obj: &OwnedObject): Result[ObjectId, CasError] ! IO
    // Equivalent to self.write(obj.kind, obj.data())
```

---

## 5. Filesystem Backend

The filesystem backend stores objects in the Git loose-object layout. An existing Git repository's `objects/` directory is a valid `FsStore`, and vice versa.

### 5.1 Opening and Initializing

```ferrum
struct FsStore { ... }

impl FsStore {
    /// Open an existing store rooted at `path`.
    ///
    /// Expects `path/objects/` to exist. Returns CasError::Io if the
    /// directory is absent or unreadable.
    fn open(path: &Path): Result[Self, CasError] ! IO

    /// Initialize a new empty store at `path`.
    ///
    /// Creates `path/objects/` and the two-character prefix directories
    /// lazily (they are created on first write, not eagerly at init time).
    /// Returns CasError::Io if `path` does not exist or is not writable.
    fn init(path: &Path): Result[Self, CasError] ! IO
}

impl ObjectStore for FsStore { ... }
```

### 5.2 On-Disk Layout

Objects are stored in a two-level directory structure, compatible with Git's loose-object format:

```
{root}/
  objects/
    ab/
      cdef1234...   ← full file name is the remaining 62 hex digits
    fe/
      dcba9876...
    pack/           ← reserved for future pack-file support; not used
    info/           ← reserved
```

The first two hex characters of the object ID name the directory. The remaining 62 characters name the file. This limits any single directory to at most 256 entries at the first level, avoiding filesystem performance problems on stores with millions of objects.

This layout is identical to Git's `$GIT_DIR/objects/` loose format.

### 5.3 Atomic Writes and Compression

**Atomic writes.** Objects are first written to a temporary file in `{root}/objects/tmp/`, then renamed into their final location. POSIX `rename(2)` is atomic within a filesystem; a reader can never observe a partial write. On Windows, `MoveFileExW` with `MOVEFILE_REPLACE_EXISTING` provides the same guarantee.

If the target path already exists (duplicate write), the rename is still attempted. If it fails because the file exists, the write is considered successful — the object is already present with the correct content (content-addressing guarantees this).

**Compression.** Objects are stored zlib-compressed by default, matching Git. The compression level is configurable:

```ferrum
impl FsStore {
    /// Set zlib compression level for new writes. Default: 1 (fast).
    /// 0 = no compression (larger files, faster writes).
    /// 9 = maximum compression (smallest files, slowest writes).
    fn with_compression(self, level: u8): Self
}
```

Existing objects are read regardless of their compression state; the store detects zlib headers automatically.

### 5.4 Garbage Collection

```ferrum
struct GcStats {
    objects_deleted: usize,
    bytes_freed:     u64,
}

impl FsStore {
    /// Delete all objects not in `reachable`.
    ///
    /// The caller is responsible for computing the reachable set, typically
    /// by starting from known roots and walking with walk_tree / read_commit.
    /// Objects in `reachable` that do not exist in the store are silently
    /// ignored (not an error).
    fn gc(&self, reachable: &[ObjectId]): Result[GcStats, CasError] ! IO
}
```

GC does not lock the store. For concurrent workloads, callers should ensure no concurrent writers are adding objects that reference as-yet-unwritten objects when GC runs.

---

## 6. In-Memory Backend

The in-memory backend stores all objects in a `HashMap`. It is intended for tests, for build-step caches that do not need to outlive the process, and for any context where filesystem I/O is unavailable or undesirable.

```ferrum
struct MemStore { ... }

impl MemStore {
    /// Create an empty in-memory store.
    fn new(): Self

    /// Create with a pre-allocated capacity for `n` objects.
    fn with_capacity(n: usize): Self

    /// Number of objects currently stored.
    fn object_count(&self): usize

    /// Total bytes of object content (not counting HashMap overhead).
    fn total_bytes(&self): usize

    /// Snapshot all stored object IDs. Useful in tests to verify
    /// exactly which objects were written.
    fn all_ids(&self): Vec[ObjectId]
}

impl ObjectStore for MemStore { ... }
```

`MemStore` does not implement the `IO` effect on its `ObjectStore` methods — the effect annotation is part of the trait signature and is present for interface uniformity. The implementation performs no actual I/O.

---

## 7. Remote and Caching Backends

### 7.1 `RemoteStore` Trait

A remote backend transfers objects over a network. It uses `Async + Net` effects instead of `IO`.

```ferrum
trait RemoteStore {
    /// Fetch an object from the remote by id.
    ///
    /// Returns CasError::NotFound if the remote does not have the object.
    /// The implementation must verify the hash of received bytes before
    /// returning Ok; hash verification is not optional.
    fn fetch(&self, id: &ObjectId): Result[OwnedObject, CasError] ! Async + Net

    /// Push an object to the remote.
    ///
    /// Idempotent: pushing an already-present object is not an error.
    fn push(&self, obj: &OwnedObject): Result[ObjectId, CasError] ! Async + Net

    /// Check whether the remote has an object, without fetching it.
    fn has(&self, id: &ObjectId): Result[bool, CasError] ! Async + Net
}
```

### 7.2 `CachingStore`

`CachingStore` composes a local `ObjectStore` and a `RemoteStore`. Reads check the local store first; on a miss, the object is fetched from the remote and inserted into the local store. Writes go to the local store and optionally to the remote.

```ferrum
struct CachingStore[L: ObjectStore, R: RemoteStore] {
    local:  L,
    remote: R,
}

impl[L: ObjectStore, R: RemoteStore] CachingStore[L, R] {
    fn new(local: L, remote: R): Self

    /// Access the local store directly (e.g. for gc or inspection).
    fn local(&self): &L

    /// Access the remote store directly.
    fn remote(&self): &R
}

impl[L: ObjectStore, R: RemoteStore] ObjectStore for CachingStore[L, R] {
    // read: checks local first; on miss, fetches from remote and caches locally.
    // write: writes to local; does NOT automatically push to remote.
    // exists: checks local first; on miss, checks remote.
    // delete: deletes from local only.
}
```

For explicit push-to-remote, use `sync` (§7.3).

### 7.3 `sync`

```ferrum
struct SyncStats {
    objects_pushed:  usize,
    objects_already_present: usize,
    bytes_transferred: u64,
}

/// Push all objects reachable from `roots` from `local` to `remote`.
///
/// Walks the reachable closure of each root (following tree entries and
/// commit parents), checks whether each object exists on the remote,
/// and pushes objects that are absent. Objects already present on the
/// remote are skipped.
fn sync(
    local:   &impl ObjectStore,
    remote:  &impl RemoteStore,
    roots:   &[ObjectId],
): Result[SyncStats, CasError] ! IO + Async + Net
```

---

## 8. High-Level Object Operations

The high-level API composes `ObjectStore` operations with serialization and type-checked parsing. These functions are the primary interface for application code.

### 8.1 Writing

```ferrum
/// Serialize and store a blob. Returns its ObjectId.
fn write_blob(
    store: &impl ObjectStore,
    data:  &[u8],
): Result[ObjectId, CasError] ! IO

/// Validate, serialize, and store a tree.
///
/// Validates that no entry name contains '/' or '\0'.
/// Sorts entries into Git tree order before serializing.
fn write_tree(
    store:   &impl ObjectStore,
    entries: Vec[TreeEntry],
): Result[ObjectId, CasError] ! IO

/// Serialize and store a commit.
fn write_commit(
    store:  &impl ObjectStore,
    commit: Commit,
): Result[ObjectId, CasError] ! IO

/// Serialize and store an annotated tag.
fn write_tag(
    store: &impl ObjectStore,
    tag:   Tag,
): Result[ObjectId, CasError] ! IO
```

### 8.2 Reading

```ferrum
/// Read and verify a blob, returning its raw bytes.
///
/// Returns CasError::Corrupt if the stored object's kind is not Blob.
fn read_blob(
    store: &impl ObjectStore,
    id:    &ObjectId,
): Result[Vec[u8], CasError] ! IO

/// Read and parse a tree object.
///
/// Returns CasError::Corrupt if the stored object's kind is not Tree,
/// or CasError::ParseError if the tree bytes are malformed.
fn read_tree(
    store: &impl ObjectStore,
    id:    &ObjectId,
): Result[Tree, CasError] ! IO

/// Read and parse a commit object.
fn read_commit(
    store: &impl ObjectStore,
    id:    &ObjectId,
): Result[Commit, CasError] ! IO

/// Read and parse a tag object.
fn read_tag(
    store: &impl ObjectStore,
    id:    &ObjectId,
): Result[Tag, CasError] ! IO
```

### 8.3 Tree Traversal

```ferrum
/// Walk a tree to find the object at `path`.
///
/// `path` is a slash-separated relative path (e.g. "src/main.fe").
/// Each component must match a tree entry name exactly.
///
/// Returns the ObjectId of the blob or subtree at the path.
/// Returns CasError::NotFound if any path component is absent.
/// Returns CasError::Corrupt if a non-terminal path component
/// refers to an object that is not a tree.
fn walk_tree(
    store: &impl ObjectStore,
    root:  &ObjectId,
    path:  &str,
): Result[ObjectId, CasError] ! IO

/// Read the blob at `path` within a tree, returning its content.
///
/// Equivalent to walk_tree then read_blob.
fn read_path(
    store: &impl ObjectStore,
    root:  &ObjectId,
    path:  &str,
): Result[Vec[u8], CasError] ! IO
```

---

## 9. Batch Operations

### 9.1 Reachable Closure

```ferrum
struct CopyStats {
    objects_copied:        usize,
    objects_already_present: usize,
    bytes_copied:          u64,
}

/// Copy all objects reachable from `roots` from `src` to `dst`.
///
/// Starting from each root, follows:
/// - Commit → tree, parents (recursively)
/// - Tree → entries (blobs and subtrees)
/// - Tag → tagged object
///
/// Objects already present in `dst` are skipped (existence check before copy).
/// The traversal is depth-first; cycles in the object graph are impossible
/// by construction (SHA-256 makes cycles cryptographically infeasible).
fn copy_closure(
    src:   &impl ObjectStore,
    dst:   &impl ObjectStore,
    roots: &[ObjectId],
): Result[CopyStats, CasError] ! IO
```

`copy_closure` is the building block for both local backup and network sync. `sync` (§7.3) is `copy_closure` with the destination being a `RemoteStore`.

### 9.2 Listing Reachable Objects

```ferrum
/// Enumerate all objects reachable from `roots`, returning their IDs.
///
/// Useful for computing the set to pass to FsStore::gc, or for
/// prefetching a closure before going offline.
fn reachable_objects(
    store: &impl ObjectStore,
    roots: &[ObjectId],
): Result[Vec[ObjectId], CasError] ! IO
```

---

## 10. Error Types

```ferrum
/// All errors from the cas module.
enum CasError {
    /// Requested object is not in the store.
    NotFound(ObjectId),

    /// Object was read but its SHA-256 does not match the requested id.
    /// This indicates corruption or tampering.
    HashMismatch {
        id:       ObjectId,   // the id that was requested
        computed: ObjectId,   // the hash of the bytes that were actually read
    },

    /// Underlying I/O error (filesystem, network, OS).
    Io(IoError),

    /// Object exists and hashes correctly, but its content is structurally
    /// invalid for its declared kind (e.g. a tree with non-sorted entries,
    /// a commit with a malformed ident line).
    Corrupt {
        id:     ObjectId,
        reason: &'static str,
    },

    /// A structural parsing error on data that came from the caller
    /// (e.g. a path component containing '/', a tree entry name with '\0',
    /// a hex string that is not 64 characters).
    ParseError(&'static str),

    /// An object's kind did not match what the caller expected.
    /// For example, read_blob was called on a tree's ObjectId.
    KindMismatch {
        id:       ObjectId,
        expected: ObjectKind,
        found:    ObjectKind,
    },
}

impl Display for CasError { ... }
impl From[IoError] for CasError { ... }
```

---

## 11. Example Usage

### 11.1 Build Cache

Hash all inputs to a compilation step. If the output is already in the store, skip recompilation.

```ferrum
use extlib::ccsp::cas::{FsStore, ObjectStore, write_blob, write_tree, write_commit}
use extlib::ccsp::cas::{read_blob, walk_tree, TreeEntry, TreeEntryMode, Commit, CommitAuthor}
use std::crypto::Sha256

fn hash_inputs(sources: &[&Path], flags: &[&str]): ObjectId ! IO {
    let mut hasher = Sha256::new()
    for src in sources {
        let bytes = std::fs::read(src)?
        hasher.update(&bytes)
    }
    for flag in flags {
        hasher.update(flag.as_bytes())
        hasher.update(b"\n")
    }
    ObjectId::from_bytes(hasher.finalize())
}

fn build_cached(
    store:   &FsStore,
    sources: &[&Path],
    flags:   &[&str],
    compile: fn(&[&Path], &[&str]): Result[Vec[u8], BuildError],
): Result[Vec[u8], CasError] ! IO {
    let input_id = hash_inputs(sources, flags)?

    // Check cache
    match store.exists(&input_id)? {
        true => {
            return read_blob(store, &input_id)
        }
        false => {}
    }

    // Cache miss: compile and store output
    let output = compile(sources, flags)
        .map_err(|e| CasError::Corrupt { id: input_id, reason: "compile failed" })?
    let output_id = write_blob(store, &output)?

    // Store the input→output mapping as a commit
    let mut entries = Vec::new()
    entries.push(TreeEntry { mode: TreeEntryMode::Blob, name: "output".to_string(), id: output_id })
    let tree_id = write_tree(store, entries)?

    let now = std::time::unix_now()
    let author = CommitAuthor::new("build-cache".to_string(), "".to_string(), now, 0)
    let commit = Commit::new(tree_id, vec![input_id], author.clone(), author, "cached build".to_string())
    write_commit(store, commit)?

    Ok(output)
}
```

### 11.2 Immutable Package Store

Packages are stored once. Installing a package is writing its blob and recording the id. Multiple versions sharing files share blobs automatically.

```ferrum
use extlib::ccsp::cas::{FsStore, ObjectStore, write_blob, read_blob, ObjectId, CasError}

struct PackageStore {
    store: FsStore,
}

impl PackageStore {
    fn open(path: &Path): Result[Self, CasError] ! IO {
        Ok(PackageStore { store: FsStore::open(path)? })
    }

    /// Add a package tarball. Returns the content id.
    fn add(&self, tarball: &[u8]): Result[ObjectId, CasError] ! IO {
        write_blob(&self.store, tarball)
    }

    /// Retrieve a package tarball by its content id.
    fn get(&self, id: &ObjectId): Result[Vec[u8], CasError] ! IO {
        read_blob(&self.store, id)
    }
}
```

### 11.3 Configuration Rollback

Each configuration snapshot is a commit. Rolling back is following parent links.

```ferrum
use extlib::ccsp::cas::{FsStore, write_blob, write_tree, write_commit, read_commit}
use extlib::ccsp::cas::{TreeEntry, TreeEntryMode, Commit, CommitAuthor, ObjectId, CasError}

fn snapshot_config(
    store:    &FsStore,
    files:    &[(&str, &[u8])],
    parent:   Option[ObjectId],
    message:  &str,
) -> Result[ObjectId, CasError] ! IO {
    let mut entries = Vec::new()
    for (name, content) in files {
        let blob_id = write_blob(store, content)?
        entries.push(TreeEntry {
            mode: TreeEntryMode::Blob,
            name: name.to_string(),
            id:   blob_id,
        })
    }
    let tree_id = write_tree(store, entries)?

    let now = std::time::unix_now()
    let author = CommitAuthor::new("config-mgmt".to_string(), "".to_string(), now, 0)
    let parents = match parent {
        Some(id) => vec![id],
        None     => vec![],
    }
    let commit = Commit::new(tree_id, parents, author.clone(), author, message.to_string())
    write_commit(store, commit)
}

fn rollback(
    store:   &FsStore,
    current: &ObjectId,
    steps:   usize,
) -> Result[ObjectId, CasError] ! IO {
    let mut id = *current
    for _ in 0..steps {
        let commit = read_commit(store, &id)?
        match commit.parents.first() {
            Some(parent) => { id = *parent }
            None         => { return Err(CasError::NotFound(id)) }
        }
    }
    Ok(id)
}
```

---

## 12. Dependencies

### Core (always required)

| Dependency | What it provides |
|---|---|
| `std.crypto` | `Sha256` hasher for computing and verifying object IDs |
| `std.fs` | `Path`, `File`, atomic rename, directory creation |
| `std.alloc` | `Vec[u8]`, `String`, `HashMap` for MemStore |

### Remote backend (optional, feature-gated)

| Dependency | What it provides |
|---|---|
| `std.async` | `Async` effect, `Future`, `.await` |
| `std.net` | `Net` effect, `TcpStream`, connection primitives |

The remote backend types (`RemoteStore`, `CachingStore`, `sync`) are only compiled when the `remote` feature of `extlib.ccsp.cas` is enabled. Code that uses only `FsStore` or `MemStore` does not pay for the async or networking machinery.

```
# Cargo-equivalent dependency declaration
[dependencies]
extlib-ccsp-cas = { version = "0.1", features = [] }          # core only
extlib-ccsp-cas = { version = "0.1", features = ["remote"] }  # core + remote backend
```

---

## 13. Design Notes and Non-Decisions

**No ref management.** This module stores and retrieves objects by hash. Named references (`main`, `v1.2.3`, `HEAD`) are a higher-level concept and are deliberately out of scope. A ref is a file that maps a name to a hash; applications that need refs can implement them trivially on top of `FsStore`.

**No pack files.** Git pack files consolidate many loose objects into one compressed binary with a delta chain, dramatically reducing disk usage for large histories. This module stores only loose objects. Pack file support is reserved for a future `extlib.ccsp.cas.pack` extension. The loose-object layout is sufficient for the described use cases and keeps the implementation auditable.

**No locking.** The `ObjectStore` trait does not provide a lock operation. Concurrent writers are safe because writes are idempotent and atomic at the filesystem level. Concurrent GC with concurrent writes requires application-level coordination; the module does not impose a specific locking strategy.

**SHA-256 only.** There is no SHA-1 compatibility mode. Code that needs to read SHA-1-addressed Git repositories should use a dedicated Git library, not this module. Supporting both hash functions in the same store would require threading a "hash variant" through every API, complicating all call sites for a use case that is not this module's concern.

**Hash verification on every read.** `read` and `read_bytes` always recompute the hash and compare it to the requested id. There is no `read_unchecked`. The overhead is one SHA-256 pass over the object data, which is appropriate for a module whose security property is "content addresses are integrity checks." Callers who genuinely need to skip verification can read the raw bytes from the filesystem directly, outside this module, making the bypass explicit and visible in code review.
