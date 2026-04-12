# Ferrum Extended Standard Library — IMAP

**Module path:** `extlib::imap`
**Implements:** RFC 9051 (IMAP4rev2), RFC 3501 (IMAP4rev1 compatibility), RFC 2177 (IDLE),
RFC 6851 (MOVE, mandatory in RFC 9051), RFC 4315 (UID EXPUNGE), RFC 2342 (NAMESPACE,
mandatory in RFC 9051), RFC 9051 §6.3.11 (LIST-STATUS, mandatory in RFC 9051),
RFC 9051 §6.3.10 (STATUS=SIZE, mandatory in RFC 9051)
**Dependencies:** `extlib::tls`, `extlib::connect`; stdlib `async`, `net`, `io`

---

## 1. Overview and Rationale

### Why IMAP Is Not in the Standard Library

The stdlib `net` module provides TCP streams and TLS connections. IMAP is not there.
This is deliberate.

**Most programs do not access email.** A web server, a CLI tool, a data pipeline —
none of these need IMAP. Pulling in a full IMAP implementation as part of every Ferrum
program would impose binary size, compile time, and dependency surface on code that
will never call it. The extlib exists precisely for high-value protocol libraries that
are common enough to warrant a first-party, audited implementation but not universal
enough to belong in the stdlib.

**IMAP depends on TLS.** RFC 9051 §11.1 deprecates unencrypted IMAP. The implicit TLS
port (993, IMAPS) and STARTTLS upgrade path both require a TLS engine. Ferrum's `extlib::tls`
provides that engine. Requiring `extlib::tls` as a transitive dependency of the stdlib
is not acceptable; it carries CA trust anchors, cipher suite policy, and significant
code size. The extlib layer exists for modules with non-trivial dependency trees.

**IMAP is protocol-complex.** IMAP4rev2 (RFC 9051) is a stateful, multiplexed
protocol with five connection states (Not Authenticated, Authenticated, Selected,
Logout, and the implicit Not Connected state), literal string synchronization,
sequence number vs. UID duality, optional extension advertisement, and server-initiated
untagged responses that can arrive during any client command. Implementing it correctly
is non-trivial. A first-party audited implementation prevents the class of bugs that
arise from hand-rolled IMAP parsers.

### RFC Baseline

This module targets RFC 9051 (IMAP4rev2, 2021) as its primary baseline. RFC 9051
obsoletes RFC 3501 and makes several extensions mandatory that were optional in
IMAP4rev1:

- **MOVE** (RFC 6851) — atomic move without copy-then-expunge races
- **STATUS=SIZE** — mailbox size in bytes in STATUS responses
- **LIST-STATUS** — STATUS data piggybacked on LIST responses, eliminating
  round-trip storms for mailbox enumeration
- **NAMESPACE** (RFC 2342) — server advertises personal, shared, and other namespace
  prefixes and delimiters

Servers that advertise `IMAP4rev1` but not `IMAP4rev2` are supported in compatibility
mode: the client detects the capability set and avoids RFC 9051-only commands.

### What This Module Provides

- IMAP client with full RFC 9051 command set
- Authentication: LOGIN, PLAIN, XOAUTH2, EXTERNAL (certificate-based)
- Mailbox management: LIST (with LIST-STATUS), STATUS, CREATE, DELETE, RENAME,
  SUBSCRIBE, UNSUBSCRIBE, NAMESPACE
- Message access: FETCH, UID FETCH, streaming large body sections via `AsyncRead`
- Search: SEARCH, UID SEARCH with a composable criteria builder
- Flag and message operations: STORE, COPY, MOVE, EXPUNGE, UID EXPUNGE, APPEND
- Server push: IDLE (RFC 2177) with automatic re-IDLE before the 29-minute limit
- IMAP server with pluggable backend trait and a built-in `MemoryBackend` for testing

### What This Module Does Not Provide

- An SMTP client (see `extlib::smtp`)
- MIME parsing of message bodies (see `extlib::mime`)
- Message thread rendering (THREAD extension, RFC 5256) — planned post-1.0
- IMAP QUOTA (RFC 9208) — planned post-1.0
- POP3

---

## 2. Connection and Authentication

### 2.1 Security Model

```ferrum
// Determines how the TCP connection is secured.
enum ImapSecurity {
    // Implicit TLS from the first byte — IMAPS (port 993).
    // This is the preferred and recommended mode. No cleartext window.
    Imaps {
        tls_config: TlsClientConfig,
    },

    // Connect in plaintext, then upgrade to TLS via STARTTLS (port 143).
    // The client aborts if the server does not offer STARTTLS capability.
    // No plaintext fallback is offered — silent fallback is a downgrade
    // attack surface. Clients that need a plaintext fallback must use Plain
    // explicitly and accept the security consequences.
    StartTls {
        tls_config: TlsClientConfig,
    },

    // No TLS. Acceptable only for localhost connections or
    // isolated test networks. Not acceptable over any public network.
    Plain,
}
```

### 2.2 Authentication

```ferrum
// Authentication mechanism sent after the TLS layer is established.
enum ImapAuth {
    // IMAP LOGIN command (RFC 3501 §6.2.3).
    // Sends credentials in cleartext inside the TLS tunnel.
    // Refused by the client if TLS is not active.
    Login {
        username: String,
        password: String,
    },

    // AUTHENTICATE PLAIN (RFC 4616).
    // Same security profile as Login when TLS is in use; preferred because
    // it is a standard SASL mechanism. Refused without TLS.
    Plain {
        username: String,
        password: String,
    },

    // AUTHENTICATE XOAUTH2 — OAuth2 bearer token.
    // token_fn is called at authentication time to fetch the current token,
    // allowing token refresh without reconstructing the client.
    XOAuth2 {
        username: String,
        token_fn: Box[dyn Fn() -> Result[String, ImapError] + Send + Sync],
    },

    // AUTHENTICATE EXTERNAL — identity is derived from the TLS client
    // certificate. The username hint is sent as the authorization identity;
    // pass an empty string to use the identity from the certificate directly.
    External {
        authzid: String,
    },
}
```

### 2.3 Configuration

```ferrum
struct ImapConfig {
    pub security:          ImapSecurity,
    pub auth:              ImapAuth,

    // Wall-clock timeout for the initial TCP connect and TLS handshake.
    // Default: 30 seconds.
    pub connect_timeout:   Duration,

    // Timeout for each individual IMAP command (FETCH, SEARCH, etc.).
    // The IDLE command uses its own timeout; see §8.
    // Default: 60 seconds.
    pub command_timeout:   Duration,

    // Timeout for reading greeting and capability advertisement after
    // the TCP connection is established but before authentication.
    // Default: 15 seconds.
    pub greeting_timeout:  Duration,
}

impl ImapConfig {
    // Convenience constructor: IMAPS on port 993 with LOGIN authentication.
    fn new_imaps(
        username: &str,
        password: &str,
        tls_config: TlsClientConfig,
    ): Self

    // Convenience constructor: STARTTLS on port 143 with AUTH PLAIN.
    fn new_starttls(
        username: &str,
        password: &str,
        tls_config: TlsClientConfig,
    ): Self
}
```

### 2.4 Connecting

```ferrum
// The top-level client type. Holds a not-yet-authenticated connection.
type ImapClient

impl ImapClient {
    // Connect to an IMAP server and authenticate.
    //
    // Steps performed:
    //   1. TCP connect (via extlib::connect, happy eyeballs where applicable)
    //   2. TLS handshake (IMAPS) or STARTTLS upgrade
    //   3. Read server greeting and initial CAPABILITY response
    //   4. Issue CAPABILITY command if not advertised in greeting
    //   5. Send authentication command (LOGIN, AUTHENTICATE, etc.)
    //   6. Return an authenticated ImapSession on success
    //
    // Effects: Async (suspends on I/O), Net (opens socket, sends packets)
    fn connect(
        host:   &str,
        port:   u16,
        config: ImapConfig,
    ): Result[ImapSession, ImapError] ! Async + Net
}
```

### 2.5 Session

```ferrum
// An authenticated IMAP session in the Authenticated state (RFC 9051 §3.3).
// From here the caller can manage mailboxes or SELECT/EXAMINE a mailbox.
struct ImapSession {
    // Server capability set as reported after authentication.
    // Refreshed automatically when the server sends updated CAPABILITY responses.
    pub fn capabilities(&self): &CapabilitySet
}

type CapabilitySet

impl CapabilitySet {
    fn has(&self, cap: &str): bool
        // Case-insensitive capability lookup, e.g. "IDLE", "MOVE", "IMAP4rev2"
    fn has_auth(&self, mechanism: &str): bool
        // Check for AUTH=MECHANISM, e.g. "XOAUTH2", "PLAIN"
    fn is_imap4rev2(&self): bool
        // True if "IMAP4rev2" is present
}
```

---

## 3. Mailbox Operations

All methods in this section operate on an `ImapSession` in the Authenticated or
Selected state. They do not require a mailbox to be selected.

### 3.1 LIST and LIST-STATUS

```ferrum
impl ImapSession {
    // LIST command with piggybacked STATUS data (RFC 9051 mandatory LIST-STATUS).
    //
    // reference: the mailbox name reference, typically "" (empty, meaning root)
    // pattern: glob pattern — "*" matches any string including delimiter,
    //          "%" matches any string except delimiter
    //
    // When the server advertises IMAP4rev2 (or the LIST-STATUS extension),
    // the status_items slice is sent as a STATUS= selection option and
    // STATUS data is returned inline. On servers that do not support
    // LIST-STATUS, the library issues individual STATUS commands for each
    // returned mailbox and merges the results transparently; callers see
    // no difference.
    //
    // To skip status data entirely, pass an empty slice for status_items.
    fn list(
        &mut self,
        reference:    &str,
        pattern:      &str,
        status_items: &[StatusItem],
    ): Result[Vec[MailboxInfo], ImapError] ! Async

    // LSUB — list subscribed mailboxes (RFC 3501; still present in RFC 9051).
    fn lsub(
        &mut self,
        reference: &str,
        pattern:   &str,
    ): Result[Vec[MailboxInfo], ImapError] ! Async
}

// Information about a single mailbox returned by LIST or LSUB.
struct MailboxInfo {
    // Full mailbox name (e.g., "INBOX", "INBOX/Sent", "Archive/2024").
    pub name:       String,

    // Hierarchy delimiter character used in this namespace,
    // or None for flat namespaces with no hierarchy.
    pub delimiter:  Option[char],

    // Server-reported attributes for this mailbox.
    pub attributes: Vec[MailboxAttribute],

    // STATUS data returned inline by LIST-STATUS, or fetched separately.
    // None when status_items was empty.
    pub status:     Option[MailboxStatus],
}

enum MailboxAttribute {
    // Selection
    Noselect,       // mailbox cannot be selected — it is a container only
    HasChildren,    // has inferiors (child mailboxes exist)
    HasNoChildren,  // has no inferiors
    Marked,         // has messages since last session (server-defined)
    Unmarked,       // no messages since last session

    // RFC 6154 (Special-Use)
    All,            // \All — all messages (virtual mailbox)
    Archive,        // \Archive
    Drafts,         // \Drafts
    Flagged,        // \Flagged
    Junk,           // \Junk (spam)
    Sent,           // \Sent
    Trash,          // \Trash

    Subscribed,     // returned by LSUB or LIST (SUBSCRIBED) selection option

    // Any other attribute the server reports that this library does not
    // recognize. Preserved for forward-compatibility.
    Other(String),
}
```

### 3.2 STATUS

```ferrum
impl ImapSession {
    // STATUS command — query metadata about a mailbox without selecting it.
    // Includes STATUS=SIZE when the server advertises IMAP4rev2 and Size
    // is included in items.
    fn status(
        &mut self,
        mailbox: &str,
        items:   &[StatusItem],
    ): Result[MailboxStatus, ImapError] ! Async
}

enum StatusItem {
    Messages,     // total number of messages
    Recent,       // number of messages with \Recent flag
    Unseen,       // number of messages without \Seen flag
    UidNext,      // predicted next UID
    UidValidity,  // UID validity value
    Size,         // total size of all messages in bytes (RFC 9051 §6.3.10)
}

struct MailboxStatus {
    // Each field is None if the corresponding StatusItem was not requested
    // or if the server did not return a value for it.
    pub messages:     Option[u32],
    pub recent:       Option[u32],
    pub unseen:       Option[u32],
    pub uidnext:      Option[u32],
    pub uidvalidity:  Option[u32],
    // Total size of the mailbox in bytes. Present only when Size was
    // requested and the server supports STATUS=SIZE (mandatory in RFC 9051).
    pub size:         Option[u64],
}
```

### 3.3 Mailbox Management

```ferrum
impl ImapSession {
    // CREATE — create a new mailbox.
    fn create(&mut self, name: &str): Result[(), ImapError] ! Async

    // DELETE — delete a mailbox. The mailbox must not be selected.
    fn delete(&mut self, name: &str): Result[(), ImapError] ! Async

    // RENAME — rename a mailbox. Renaming INBOX is server-defined behavior;
    // it typically creates a new mailbox with the target name and leaves INBOX empty.
    fn rename(&mut self, from: &str, to: &str): Result[(), ImapError] ! Async

    // SUBSCRIBE / UNSUBSCRIBE — manage the subscription list.
    fn subscribe(&mut self, mailbox: &str): Result[(), ImapError] ! Async
    fn unsubscribe(&mut self, mailbox: &str): Result[(), ImapError] ! Async

    // NAMESPACE — query server namespace information (mandatory in RFC 9051).
    // Returns personal, shared (other users'), and public namespaces.
    fn namespace(&mut self): Result[NamespaceResponse, ImapError] ! Async
}

// Response to the NAMESPACE command (RFC 2342, mandatory in RFC 9051).
struct NamespaceResponse {
    // Namespaces accessible to the authenticated user as their own mailboxes.
    // Typically one entry: prefix="" delimiter="/" for most servers.
    pub personal: Vec[Namespace],

    // Namespaces for other users' mailboxes, if the server exposes them.
    // Prefix typically "Other Users/" or "user/".
    pub other_users: Vec[Namespace],

    // Shared namespaces visible to all or a group of users.
    pub shared: Vec[Namespace],
}

struct Namespace {
    // The prefix string that must be prepended to a mailbox name in this
    // namespace to form a full mailbox name (e.g., "", "INBOX.", "user/").
    pub prefix:    String,

    // The hierarchy delimiter used within this namespace, or None.
    pub delimiter: Option[char],
}
```

---

## 4. Selected Mailbox State

IMAP distinguishes between the Authenticated state and the Selected state. The
Selected state is entered by SELECT or EXAMINE and gives access to message operations.
This module models the distinction with separate types so that the compiler enforces
that message fetch, search, store, and expunge operations are only callable on a
selected session.

### 4.1 Entering and Leaving the Selected State

```ferrum
impl ImapSession {
    // SELECT — open a mailbox for read-write access.
    // Consumes the ImapSession and returns a SelectedSession on success.
    // The original session is recoverable via SelectedSession::close() or
    // SelectedSession::unselect() (RFC 9051 §6.4.2).
    fn select(
        self,
        mailbox: &str,
    ): Result[SelectedSession, ImapError] ! Async

    // EXAMINE — open a mailbox for read-only access.
    // Same as select() but the server will not modify any message flags.
    fn examine(
        self,
        mailbox: &str,
    ): Result[SelectedSession, ImapError] ! Async
}

// Data returned by SELECT or EXAMINE.
struct SelectData {
    // Total number of messages in the mailbox.
    pub exists:           u32,

    // Number of messages with \Recent flag.
    pub recent:           u32,

    // Predicted UID of the next message to be added.
    pub uidnext:          Option[u32],

    // UID validity value. If this changes between sessions, all cached UIDs
    // are invalid and must be discarded.
    pub uidvalidity:      u32,

    // Sequence number of the first unseen message, if reported.
    pub first_unseen:     Option[u32],

    // Flags the server permits clients to set permanently on messages.
    pub permanent_flags:  Vec[Flag],

    // True when SELECT was used (read-write); false when EXAMINE was used.
    pub read_write:       bool,
}

// An IMAP session in the Selected state.
// Provides all ImapSession methods plus message-level operations.
struct SelectedSession {
    // Metadata returned by the SELECT or EXAMINE command.
    pub fn select_data(&self): &SelectData

    // Name of the currently selected mailbox.
    pub fn mailbox_name(&self): &str

    // Transition back to Authenticated state.
    // CLOSE implicitly expunges messages flagged \Deleted (read-write only).
    // Use unselect() to leave without expunging.
    fn close(self): Result[ImapSession, ImapError] ! Async

    // UNSELECT (RFC 3691) — leave the Selected state without expunging.
    // Falls back to SELECT on a dummy nonexistent mailbox on servers that
    // do not advertise UNSELECT capability, then issues EXAMINE on INBOX
    // to avoid side effects. This is transparent to callers.
    fn unselect(self): Result[ImapSession, ImapError] ! Async

    // Select a different mailbox without returning to Authenticated state first.
    fn reselect(self, mailbox: &str): Result[SelectedSession, ImapError] ! Async
}

// SelectedSession also exposes all ImapSession mailbox management methods.
// They are available even while a mailbox is selected, because IMAP permits
// management commands (LIST, STATUS, CREATE, etc.) in the Selected state.
impl SelectedSession {
    fn list(
        &mut self,
        reference:    &str,
        pattern:      &str,
        status_items: &[StatusItem],
    ): Result[Vec[MailboxInfo], ImapError] ! Async

    fn status(
        &mut self,
        mailbox: &str,
        items:   &[StatusItem],
    ): Result[MailboxStatus, ImapError] ! Async

    fn namespace(&mut self): Result[NamespaceResponse, ImapError] ! Async

    // ... (create, delete, rename, subscribe, unsubscribe also available)
}
```

---

## 5. Fetching Messages

### 5.1 Sequence Sets

```ferrum
// A set of message sequence numbers or UIDs for use in FETCH, STORE, COPY,
// MOVE, and EXPUNGE commands.
//
// Sequence numbers are 1-based. The special value * means the largest
// number in use (the last message in the mailbox, or the last UID assigned).
type SequenceSet

impl SequenceSet {
    fn single(n: u32): Self              // e.g., 1
    fn range(from: u32, to: u32): Self   // e.g., 1:10
    fn from_star(from: u32): Self        // e.g., 5:* (from n to last)
    fn star(): Self                      // * (last message only)
    fn all(): Self                       // 1:* (all messages)

    // Add additional numbers or ranges, building a comma-separated set.
    // e.g., SequenceSet::single(1).add(3).add_range(5, 10) => "1,3,5:10"
    fn add(self, n: u32): Self
    fn add_range(self, from: u32, to: u32): Self

    // Parse from IMAP sequence-set string syntax ("1,3:5,10:*").
    fn parse(s: &str): Result[Self, ImapError]
}

impl Display for SequenceSet  // renders as canonical IMAP sequence-set syntax
```

### 5.2 Fetch Items

```ferrum
// Data items to request in a FETCH command.
enum FetchItem {
    // Message envelope: From, To, Cc, Bcc, Subject, Date, Message-ID,
    // In-Reply-To, Reply-To, Sender (RFC 9051 §7.5.2).
    Envelope,

    // High-level MIME structure without body content.
    BodyStructure,

    // Body section content. See BodySection for section addressing.
    // partial: Some((offset, length)) to fetch a byte range.
    BodySection {
        section: BodySection,
        partial: Option[(u64, u64)],
    },

    // Like BodySection but does not implicitly set the \Seen flag.
    BodySectionPeek {
        section: BodySection,
        partial: Option[(u64, u64)],
    },

    // System and keyword flags on the message.
    Flags,

    // Server-internal receipt timestamp (not the Date: header).
    InternalDate,

    // Size of the full RFC 5322 message in bytes.
    Rfc822Size,

    // The UID of the message. Always included in UID FETCH responses.
    Uid,
}

// Addressing a section within a MIME message body.
enum BodySection {
    // The entire message as an RFC 5322 message including headers.
    Full,

    // All header fields of the message (or a specific MIME part).
    Header,

    // Specific header fields. FieldsNot is the complement.
    HeaderFields(Vec[String]),
    HeaderFieldsNot(Vec[String]),

    // The body of the message (or a MIME part) excluding headers.
    Text,

    // A specific MIME part by 1-based part number.
    // Part "1" is the first body part of a multipart message.
    // Part "1.2" is the second sub-part of the first part. Etc.
    Part(Vec[u32>]),

    // Header of a nested message/rfc822 part.
    PartHeader(Vec[u32]),

    // Text body of a nested message/rfc822 part.
    PartText(Vec[u32]),
}
```

### 5.3 Fetch Responses

```ferrum
// Response data for one message from a FETCH command.
struct FetchResponse {
    // Sequence number (always present).
    pub sequence_num:  u32,

    // UID of the message (present when UID FETCH was used or Uid was requested).
    pub uid:           Option[u32],

    // Message flags at fetch time.
    pub flags:         Option[Vec[Flag]],

    // Parsed envelope data.
    pub envelope:      Option[Envelope],

    // MIME body structure.
    pub body_structure: Option[BodyStructure],

    // Body section data, keyed by the section that was requested.
    pub sections:      HashMap[BodySection, Bytes],

    // Server-internal receipt timestamp.
    pub internal_date: Option[Timestamp],

    // Size of the full RFC 5322 message in octets.
    pub size:          Option[u32],
}

// RFC 5322 envelope fields as parsed by the server.
struct Envelope {
    pub date:        Option[String],
    pub subject:     Option[String],
    pub from:        Vec[Address],
    pub sender:      Vec[Address],
    pub reply_to:    Vec[Address],
    pub to:          Vec[Address],
    pub cc:          Vec[Address],
    pub bcc:         Vec[Address],
    pub in_reply_to: Option[String],
    pub message_id:  Option[String],
}

struct Address {
    pub display_name: Option[String],
    pub mailbox:      Option[String],  // local part
    pub host:         Option[String],  // domain
}

impl Address {
    fn addr_spec(&self): Option[String]
        // Returns "mailbox@host" when both are present, None for group markers.
}
```

### 5.4 FETCH Commands

```ferrum
impl SelectedSession {
    // FETCH — retrieve data by sequence number.
    // Processes all untagged FETCH responses and returns them collected.
    fn fetch(
        &mut self,
        sequence: SequenceSet,
        items:    &[FetchItem],
    ): Result[Vec[FetchResponse], ImapError] ! Async

    // UID FETCH — retrieve data by UID. Preferred over FETCH for stability:
    // UIDs are stable across sessions as long as UIDVALIDITY has not changed,
    // while sequence numbers shift whenever messages are expunged.
    fn uid_fetch(
        &mut self,
        uids:  SequenceSet,
        items: &[FetchItem],
    ): Result[Vec[FetchResponse], ImapError] ! Async

    // Stream a single body section as AsyncRead, avoiding buffering the
    // entire body in memory. Useful for large attachments.
    // Does not set \Seen implicitly (uses BODY.PEEK internally).
    fn fetch_body_section_stream(
        &mut self,
        uid:     u32,
        section: BodySection,
    ): Result[impl AsyncRead, ImapError] ! Async
}
```

---

## 6. Searching

### 6.1 SearchCriteria Builder

```ferrum
// Composable search criteria, mapping to IMAP SEARCH keys.
// Criteria are combined with AND semantics unless .or() or .not() is used.
type SearchCriteria

impl SearchCriteria {
    // Match all messages. Equivalent to an empty search (ALL key).
    fn all(): Self

    // Match messages with the \Seen flag.
    fn seen(): Self

    // Match messages without the \Seen flag.
    fn unseen(): Self

    // Match messages with the \Flagged flag.
    fn flagged(): Self

    // Match messages with the \Deleted flag.
    fn deleted(): Self

    // Match messages with the \Draft flag.
    fn draft(): Self

    // Match messages with the \Answered flag.
    fn answered(): Self

    // Match messages with the \Recent flag.
    fn recent(): Self

    // Match messages where the From: header contains the given string
    // (case-insensitive substring match, server-defined).
    fn from(addr: &str): Self

    // Match messages where the To: header contains the given string.
    fn to(addr: &str): Self

    // Match messages where the Cc: header contains the given string.
    fn cc(addr: &str): Self

    // Match messages where the Subject: header contains the given string.
    fn subject(s: &str): Self

    // Match messages where any header or body contains the given string.
    fn text(s: &str): Self

    // Match messages where the body contains the given string.
    fn body(s: &str): Self

    // Match messages with an internal date on or after the given date.
    fn since(date: Date): Self

    // Match messages with an internal date strictly before the given date.
    fn before(date: Date): Self

    // Match messages with an internal date on the given date.
    fn on(date: Date): Self

    // Match messages with an RFC 5322 Date: header on or after the given date.
    fn sent_since(date: Date): Self

    // Match messages with an RFC 5322 Date: header strictly before the given date.
    fn sent_before(date: Date): Self

    // Match messages whose RFC 5322 size is larger than n bytes.
    fn larger(n: u32): Self

    // Match messages whose RFC 5322 size is smaller than n bytes.
    fn smaller(n: u32): Self

    // Match messages with the given keyword flag set.
    fn keyword(flag: &str): Self

    // Match messages in the given UID set.
    fn uid(set: SequenceSet): Self

    // Match messages in the given sequence number set.
    fn sequence(set: SequenceSet): Self

    // Logical NOT of the given criteria.
    fn not(criteria: SearchCriteria): Self

    // Logical OR of two criteria sets.
    fn or(a: SearchCriteria, b: SearchCriteria): Self

    // Logical AND: combine two criteria (implicit in IMAP search, explicit here).
    // Used to build compound criteria programmatically.
    fn and(self, other: SearchCriteria): Self
}
```

### 6.2 Search Commands

```ferrum
impl SelectedSession {
    // SEARCH — returns sequence numbers of matching messages.
    // Sequence numbers are valid only for the current session; they shift
    // when messages are expunged. Prefer uid_search() for stability.
    fn search(
        &mut self,
        criteria: SearchCriteria,
    ): Result[Vec[u32], ImapError] ! Async

    // UID SEARCH — returns UIDs of matching messages.
    // UIDs are stable across sessions as long as UIDVALIDITY is unchanged.
    fn uid_search(
        &mut self,
        criteria: SearchCriteria,
    ): Result[Vec[u32], ImapError] ! Async
}
```

---

## 7. Flag and Message Operations

### 7.1 Flags

```ferrum
// System flags and user-defined keyword flags.
enum Flag {
    // RFC 9051 §2.3.2 system flags
    Seen,           // \Seen — message has been read
    Answered,       // \Answered — message has been replied to
    Flagged,        // \Flagged — marked for special attention
    Deleted,        // \Deleted — marked for expunge
    Draft,          // \Draft — not yet sent
    Recent,         // \Recent — arrived since last session; server-set, not client-settable

    // User-defined keyword flag (must match atom syntax: no spaces or special chars).
    Keyword(String),
}

impl Display for Flag   // renders as "\Seen", "\Flagged", or the keyword string
```

### 7.2 STORE — Modifying Flags

```ferrum
// Determines whether STORE sets, adds to, or removes from the flag set.
enum StoreOp {
    // Replace the message's flag set entirely with the given flags.
    Set,

    // Add the given flags without removing others.
    Add,

    // Remove the given flags, leaving others untouched.
    Remove,
}

impl SelectedSession {
    // STORE — modify flags by sequence number.
    // Pass silent: true to use .SILENT variants (no FETCH response returned).
    fn store(
        &mut self,
        sequence: SequenceSet,
        flags:    &[Flag],
        op:       StoreOp,
        silent:   bool,
    ): Result[Vec[FetchResponse], ImapError] ! Async

    // UID STORE — modify flags by UID.
    fn uid_store(
        &mut self,
        uids:   SequenceSet,
        flags:  &[Flag],
        op:     StoreOp,
        silent: bool,
    ): Result[Vec[FetchResponse], ImapError] ! Async
}
```

### 7.3 COPY, MOVE, EXPUNGE

```ferrum
impl SelectedSession {
    // COPY — copy messages to another mailbox by sequence number.
    fn copy(
        &mut self,
        sequence: SequenceSet,
        mailbox:  &str,
    ): Result[CopyResponse, ImapError] ! Async

    // UID COPY — copy by UID.
    fn uid_copy(
        &mut self,
        uids:    SequenceSet,
        mailbox: &str,
    ): Result[CopyResponse, ImapError] ! Async

    // MOVE — atomically move messages to another mailbox (RFC 6851).
    // Mandatory in RFC 9051. On servers that advertise only IMAP4rev1 without
    // the MOVE extension, the library emulates move as COPY + STORE +Deleted
    // + EXPUNGE transparently, but does not guarantee atomicity.
    fn move_(
        &mut self,
        sequence: SequenceSet,
        mailbox:  &str,
    ): Result[MoveResponse, ImapError] ! Async

    // UID MOVE — move by UID.
    fn uid_move(
        &mut self,
        uids:    SequenceSet,
        mailbox: &str,
    ): Result[MoveResponse, ImapError] ! Async

    // EXPUNGE — permanently removes all messages flagged \Deleted.
    // Returns the sequence numbers of expunged messages.
    // Warning: expunge shifts sequence numbers for all messages with higher
    // numbers. Revalidate any cached sequence numbers after expunge.
    fn expunge(&mut self): Result[Vec[u32], ImapError] ! Async

    // UID EXPUNGE — expunge specific UIDs only (RFC 4315).
    // Requires the UIDPLUS capability. Only the specified messages are removed,
    // even if other messages are also flagged \Deleted.
    fn uid_expunge(&mut self, uids: SequenceSet): Result[Vec[u32], ImapError] ! Async
}

// COPYUID / MOVEUID response data from the server (RFC 4315 UIDPLUS).
// Present only when the server advertises UIDPLUS capability.
struct CopyResponse {
    // UIDVALIDITY of the destination mailbox.
    pub uidvalidity:  Option[u32],
    // UID set of source messages copied.
    pub source_uids:  Option[SequenceSet],
    // UID set assigned to the copies in the destination mailbox.
    pub dest_uids:    Option[SequenceSet],
}

struct MoveResponse {
    pub uidvalidity:  Option[u32],
    pub source_uids:  Option[SequenceSet],
    pub dest_uids:    Option[SequenceSet],
}
```

### 7.4 APPEND

```ferrum
impl ImapSession {
    // APPEND — upload a raw RFC 5322 message to a mailbox.
    // The message is uploaded verbatim; no parsing is performed.
    // date: the internal date to assign; None uses the current server time.
    fn append(
        &mut self,
        mailbox: &str,
        flags:   &[Flag],
        date:    Option[Timestamp],
        message: &[u8],
    ): Result[AppendResult, ImapError] ! Async
}

// APPEND result data (RFC 4315 UIDPLUS).
// Fields are None on servers that do not advertise UIDPLUS.
struct AppendResult {
    // UIDVALIDITY of the mailbox the message was appended to.
    pub uidvalidity: Option[u32],
    // UID assigned to the appended message.
    pub uid:         Option[u32],
}
```

---

## 8. IDLE — Server Push for New Messages

RFC 2177 IDLE allows the client to park the connection in a waiting state, receiving
unsolicited server updates (new messages, flag changes, expunges) without polling.
RFC 2177 recommends re-issuing IDLE at least every 29 minutes to prevent server-side
timeout; this library handles that automatically.

### 8.1 Entering IDLE

```ferrum
impl SelectedSession {
    // IDLE — enter the idle waiting state.
    // Returns an IdleHandle that the caller uses to receive events and
    // eventually terminate the idle.
    //
    // Requires the IDLE capability. Returns ImapError::UnsupportedCapability
    // on servers that do not advertise IDLE.
    //
    // The library will re-issue IDLE transparently after 28 minutes to
    // stay within the RFC 2177 recommendation of re-idling before 29 minutes.
    // Callers see no interruption from this re-idle cycle.
    fn idle(&mut self): Result[IdleHandle, ImapError] ! Async
}
```

### 8.2 IdleHandle

```ferrum
// A handle to an active IDLE session.
// Borrow of the SelectedSession is implicit — no other commands can be
// issued while IDLE is active. Call done() to terminate IDLE and release
// the session for further commands.
type IdleHandle[&'session mut SelectedSession]

impl IdleHandle {
    // Wait for the next server notification. Suspends until an untagged
    // response arrives or the automatic re-idle fires (transparent to caller).
    //
    // Returns None only when done() has been called on another task/handle
    // or the server sends BYE.
    fn wait(&mut self): Result[IdleEvent, ImapError] ! Async

    // Terminate IDLE by sending the DONE continuation.
    // After this call the SelectedSession is usable again.
    fn done(self): Result[(), ImapError] ! Async
}

// An event received from the server while in IDLE.
enum IdleEvent {
    // EXISTS n — total message count changed to n.
    // A new value greater than the previous exists count indicates new mail.
    Exists(u32),

    // EXPUNGE n — message with sequence number n was expunged.
    // Sequence numbers above n are decremented.
    Expunge(u32),

    // FETCH n (FLAGS ...) — flag changes on message n.
    FlagsChanged {
        sequence_num: u32,
        flags:        Vec[Flag],
    },

    // RECENT n — number of \Recent messages changed.
    Recent(u32),

    // Any other untagged response received during IDLE.
    // Preserved for forward-compatibility with extensions.
    Other {
        tag:  String,
        data: String,
    },
}
```

---

## 9. IMAP Server

The server side exposes a trait-based backend model. Implementors provide a
`MailboxBackend` and the library handles connection management, protocol framing,
capability advertisement, tag tracking, and literal synchronization.

### 9.1 Server Configuration

```ferrum
struct ImapServerConfig {
    // TLS configuration for IMAPS (mandatory; the server always requires TLS).
    pub tls_config:       TlsServerConfig,

    // Advertise STARTTLS in addition to implicit TLS on the same port.
    // When true, plaintext connections are accepted on the bind address and
    // STARTTLS upgrades them. Default: false.
    pub enable_starttls:  bool,

    // Capabilities to advertise in CAPABILITY responses, in addition to
    // the capabilities this library implements unconditionally.
    // Use this to advertise backend-specific capabilities.
    pub extra_caps:       Vec[String],

    // Maximum literal size the server will accept from clients.
    // Clients that send a larger literal receive NO [TOOBIG]. Default: 50 MiB.
    pub max_literal_size: usize,

    // Per-connection idle timeout. Connections that send no command for this
    // duration are terminated with a BYE. RFC 9051 §5.4 requires at least
    // 30 minutes. Default: 30 minutes.
    pub idle_timeout:     Duration,
}
```

### 9.2 Server Binding and Runtime

```ferrum
type ImapServer

impl ImapServer {
    // Bind to a local address and prepare to accept connections.
    // Does not start the accept loop.
    fn bind(
        addr:    SocketAddr,
        config:  ImapServerConfig,
        backend: impl MailboxBackend,
    ): Result[Self, ImapError] ! Async + Net

    // Accept connections and drive protocol sessions until an unrecoverable
    // error occurs. Spawns one async task per accepted connection.
    // Never returns Ok(()).
    fn run(&mut self): Result[(), ImapError] ! Async + Net
}
```

### 9.3 MailboxBackend Trait

```ferrum
// The storage backend for an IMAP server.
// Every method corresponds to one or more IMAP commands.
// Methods are called with a reference to the authenticated session context,
// which carries the username and any session-scoped state the backend needs.
trait MailboxBackend: Send + Sync + 'static {
    type Error: Into[ImapError] + Send

    // LIST — enumerate mailboxes matching the pattern.
    fn list(
        &self,
        ctx:       &SessionContext,
        reference: &str,
        pattern:   &str,
    ): impl Future[Output=Result[Vec[MailboxInfo], Self::Error]] ! Async

    // STATUS — return metadata for a mailbox without selecting it.
    fn status(
        &self,
        ctx:     &SessionContext,
        mailbox: &str,
        items:   &[StatusItem],
    ): impl Future[Output=Result[MailboxStatus, Self::Error]] ! Async

    // SELECT / EXAMINE — open a mailbox; returns select data and a backend handle.
    fn select(
        &self,
        ctx:       &SessionContext,
        mailbox:   &str,
        read_only: bool,
    ): impl Future[Output=Result[(SelectData, Box[dyn SelectedBackend]), Self::Error]] ! Async

    // APPEND — store a message in a mailbox.
    fn append(
        &self,
        ctx:     &SessionContext,
        mailbox: &str,
        flags:   &[Flag],
        date:    Option[Timestamp],
        message: &[u8],
    ): impl Future[Output=Result[AppendResult, Self::Error]] ! Async

    // CREATE / DELETE / RENAME / SUBSCRIBE / UNSUBSCRIBE
    fn create(&self, ctx: &SessionContext, name: &str):
        impl Future[Output=Result[(), Self::Error]] ! Async
    fn delete(&self, ctx: &SessionContext, name: &str):
        impl Future[Output=Result[(), Self::Error]] ! Async
    fn rename(&self, ctx: &SessionContext, from: &str, to: &str):
        impl Future[Output=Result[(), Self::Error]] ! Async
    fn subscribe(&self, ctx: &SessionContext, mailbox: &str):
        impl Future[Output=Result[(), Self::Error]] ! Async
    fn unsubscribe(&self, ctx: &SessionContext, mailbox: &str):
        impl Future[Output=Result[(), Self::Error]] ! Async

    // NAMESPACE
    fn namespace(&self, ctx: &SessionContext):
        impl Future[Output=Result[NamespaceResponse, Self::Error]] ! Async
}

// Backend operations available only in the Selected state.
// Returned by MailboxBackend::select().
trait SelectedBackend: Send + Sync {
    type Error: Into[ImapError] + Send

    fn fetch(
        &self,
        sequence: &SequenceSet,
        by_uid:   bool,
        items:    &[FetchItem],
    ): impl Future[Output=Result[Vec[FetchResponse], Self::Error]] ! Async

    fn search(
        &self,
        criteria: &SearchCriteria,
        by_uid:   bool,
    ): impl Future[Output=Result[Vec[u32>, Self::Error]] ! Async

    fn store(
        &self,
        sequence: &SequenceSet,
        flags:    &[Flag],
        op:       StoreOp,
        by_uid:   bool,
        silent:   bool,
    ): impl Future[Output=Result[Vec[FetchResponse], Self::Error]] ! Async

    fn copy(
        &self,
        sequence: &SequenceSet,
        dest:     &str,
        by_uid:   bool,
    ): impl Future[Output=Result[CopyResponse, Self::Error]] ! Async

    fn move_(
        &self,
        sequence: &SequenceSet,
        dest:     &str,
        by_uid:   bool,
    ): impl Future[Output=Result[MoveResponse, Self::Error]] ! Async

    fn expunge(
        &self,
        uids: Option[&SequenceSet],  // None = all \Deleted; Some = UID EXPUNGE
    ): impl Future[Output=Result[Vec[u32>, Self::Error]] ! Async
}

// Per-connection context passed to every backend call.
struct SessionContext {
    pub username:   String,
    pub peer_addr:  SocketAddr,
}
```

### 9.4 Built-in Backends

```ferrum
// In-memory backend for unit tests and integration tests.
// Stores messages and flags in a HashMap. Not persistent across process restarts.
// Thread-safe; shared via Arc for use across async tasks.
type MemoryBackend

impl MemoryBackend {
    fn new(): Self

    // Pre-populate a mailbox with messages for test setup.
    fn add_mailbox(&mut self, name: &str)
    fn add_message(
        &mut self,
        mailbox: &str,
        flags:   &[Flag],
        date:    Timestamp,
        raw:     &[u8],
    ): u32  // returns assigned UID
}

impl MailboxBackend for MemoryBackend { ... }
```

---

## 10. Error Types

```ferrum
enum ImapError {
    // The named mailbox does not exist on the server.
    // Server response code: [NONEXISTENT]
    NoSuchMailbox { name: String },

    // A command was syntactically or semantically rejected (BAD response).
    BadCommand { command: String, reason: String },

    // Authentication failed (NO [AUTHENTICATIONFAILED]).
    AuthenticationFailed { reason: String },

    // A capability required by the called method is absent.
    // e.g., calling idle() on a server without IDLE capability.
    UnsupportedCapability { capability: String },

    // A TLS error occurred during connection establishment or data transfer.
    TlsError { reason: String },

    // The underlying TCP connection failed or was reset.
    NetworkError(IoError),

    // The server sent a BYE response and closed the connection.
    ServerBye { message: String },

    // The server sent data that did not conform to RFC 9051 grammar.
    ProtocolViolation { context: String, detail: String },

    // A command or connection-level deadline was exceeded.
    Timeout { operation: String, duration: Duration },

    // The server returned a NO response (command failed, not a protocol error).
    // message may carry a [RESP-CODE] from the server.
    CommandFailed { command: String, message: String },

    // The server's UIDVALIDITY changed since the last session.
    // All cached UIDs for this mailbox must be discarded.
    UidValidityChanged { mailbox: String, old: u32, new_: u32 },

    // The connection was closed by the local application before the
    // command completed (e.g., the IdleHandle was dropped without done()).
    ConnectionClosed,
}

impl Display for ImapError { ... }
impl Error   for ImapError { ... }
```

---

## 11. Example Usage

### 11.1 Fetch Unread Messages

Connect to an IMAP server, open the inbox, find all unseen messages, and fetch
their envelopes and text bodies.

```ferrum
use extlib::imap::{
    ImapClient, ImapConfig, ImapSecurity, ImapAuth,
    BodySection, FetchItem, SearchCriteria, StatusItem,
}
use extlib::tls::TlsClientConfig
use std::time::Duration

fn fetch_unread(
    host:     &str,
    username: &str,
    password: &str,
): Result[(), ImapError] ! Async + Net {
    let tls = TlsClientConfig::default()

    let config = ImapConfig {
        security:        ImapSecurity::Imaps { tls_config: tls },
        auth:            ImapAuth::Login {
                             username: username.to_string(),
                             password: password.to_string(),
                         },
        connect_timeout: Duration::from_secs(15),
        command_timeout: Duration::from_secs(60),
        greeting_timeout: Duration::from_secs(15),
    }

    let mut session = ImapClient::connect(host, 993, config).await?

    // Report mailbox statistics before selecting
    let status = session.status(
        "INBOX",
        &[StatusItem::Messages, StatusItem::Unseen, StatusItem::Size],
    ).await?
    println("INBOX: {} messages, {} unseen, {} bytes",
        status.messages.unwrap_or(0),
        status.unseen.unwrap_or(0),
        status.size.unwrap_or(0),
    )

    let mut sel = session.select("INBOX").await?
    println("selected: exists={} uidvalidity={}",
        sel.select_data().exists,
        sel.select_data().uidvalidity,
    )

    // Find unseen messages
    let uids = sel.uid_search(SearchCriteria::unseen()).await?
    if uids.is_empty() {
        println("no unread messages")
        sel.close().await?
        return Ok(())
    }

    println("found {} unread messages", uids.len())

    // Fetch envelopes and text bodies for all unseen messages
    let seq = SequenceSet::from_iter(uids.iter().copied())
    let responses = sel.uid_fetch(&seq, &[
        FetchItem::Uid,
        FetchItem::Envelope,
        FetchItem::Flags,
        FetchItem::BodySectionPeek {
            section: BodySection::Text,
            partial: None,
        },
    ]).await?

    for msg in &responses {
        if let Some(env) = &msg.envelope {
            let subject = env.subject.as_deref().unwrap_or("(no subject)")
            let from = env.from.first()
                .and_then(|a| a.addr_spec())
                .unwrap_or_else(|| "(unknown)".to_string())
            println("UID {}: from={} subject={}",
                msg.uid.unwrap_or(0), from, subject)
        }
    }

    sel.close().await?
    Ok(())
}
```

### 11.2 IDLE — Wait for New Mail

Enter IDLE on the selected inbox and process each new message as it arrives.

```ferrum
use extlib::imap::{
    ImapClient, ImapConfig, ImapSecurity, ImapAuth,
    FetchItem, BodySection, IdleEvent, SequenceSet,
}

fn watch_inbox(config: ImapConfig): Result[(), ImapError] ! Async + Net {
    let mut session = ImapClient::connect("mail.example.com", 993, config).await?
    let mut sel = session.select("INBOX").await?

    let mut known_exists = sel.select_data().exists
    println("watching INBOX, currently {} messages", known_exists)

    loop {
        let mut idle = sel.idle().await?

        loop {
            let event = idle.wait().await?

            match event {
                IdleEvent::Exists(n) if n > known_exists => {
                    // New messages arrived. Terminate IDLE, fetch them, then re-IDLE.
                    idle.done().await?

                    let new_uids = sel.uid_search(
                        SearchCriteria::uid(
                            SequenceSet::range(known_exists + 1, n)
                        )
                    ).await?

                    let responses = sel.uid_fetch(
                        &SequenceSet::from_iter(new_uids.iter().copied()),
                        &[FetchItem::Uid, FetchItem::Envelope],
                    ).await?

                    for msg in &responses {
                        if let Some(env) = &msg.envelope {
                            let subject = env.subject.as_deref().unwrap_or("(no subject)")
                            println("new message: {}", subject)
                        }
                    }

                    known_exists = n
                    break  // break inner loop to re-IDLE
                }
                IdleEvent::Expunge(n) => {
                    println("message {} was expunged", n)
                    if known_exists > 0 { known_exists -= 1 }
                }
                IdleEvent::FlagsChanged { sequence_num, flags } => {
                    println("message {} flags changed: {:?}", sequence_num, flags)
                }
                _ => {}
            }
        }
        // Loop back to re-enter IDLE
    }
}
```

### 11.3 Move a Message to a Different Mailbox

Search for messages from a sender, mark them read, and move them to an archive folder.

```ferrum
use extlib::imap::{
    ImapClient, ImapConfig, SearchCriteria,
    StoreOp, Flag, SequenceSet,
}

fn archive_sender(
    config: ImapConfig,
    sender: &str,
    dest:   &str,
): Result[(), ImapError] ! Async + Net {
    let mut session = ImapClient::connect("mail.example.com", 993, config).await?
    let mut sel = session.select("INBOX").await?

    let uids = sel.uid_search(SearchCriteria::from(sender)).await?

    if uids.is_empty() {
        println("no messages from {}", sender)
        sel.close().await?
        return Ok(())
    }

    let set = SequenceSet::from_iter(uids.iter().copied())

    // Mark all found messages as read
    sel.uid_store(&set, &[Flag::Seen], StoreOp::Add, true).await?

    // Move atomically to the archive folder (MOVE command, RFC 6851)
    let result = sel.uid_move(&set, dest).await?

    println(
        "moved {} messages from {} to {}",
        uids.len(), sender, dest
    )

    if let (Some(sv), Some(du)) = (result.uidvalidity, result.dest_uids) {
        println("destination UIDVALIDITY={} UIDs={}", sv, du)
    }

    sel.close().await?
    Ok(())
}
```

### 11.4 Search by Date Range and Sender, Then Download Attachments

```ferrum
use extlib::imap::{
    ImapClient, ImapConfig, SearchCriteria,
    FetchItem, BodySection, BodyStructure,
}
use std::time::Date
use std::io::AsyncReadExt

fn download_attachments_since(
    config: ImapConfig,
    sender: &str,
    since:  Date,
    outdir: &str,
): Result[(), ImapError] ! Async + Net + IO {
    let mut session = ImapClient::connect("mail.example.com", 993, config).await?
    let mut sel = session.select("INBOX").await?

    let criteria = SearchCriteria::from(sender)
        .and(SearchCriteria::since(since))

    let uids = sel.uid_search(criteria).await?
    println("found {} messages", uids.len())

    for uid in uids {
        let single = SequenceSet::single(uid)

        // Fetch body structure to find attachment part numbers
        let responses = sel.uid_fetch(&single, &[
            FetchItem::BodyStructure,
            FetchItem::Envelope,
        ]).await?

        let msg = match responses.first() {
            Some(m) => m,
            None    => continue,
        }

        let structure = match &msg.body_structure {
            Some(s) => s,
            None    => continue,
        }

        // Walk the body structure to find attachment parts
        for (part_num, part) in structure.attachment_parts() {
            let filename = part.filename().unwrap_or("attachment")
            let dest_path = format!("{}/{}-{}", outdir, uid, filename)

            // Stream the attachment to disk without loading it into memory
            let mut reader = sel.fetch_body_section_stream(
                uid,
                BodySection::Part(part_num),
            ).await?

            let mut file = std::fs::File::create(&dest_path).await?
            std::io::copy_async(&mut reader, &mut file).await?
            println("saved {} bytes to {}", file.metadata().await?.len(), dest_path)
        }
    }

    sel.close().await?
    Ok(())
}
```

### 11.5 Simple In-Memory IMAP Server for Testing

```ferrum
use extlib::imap::{
    ImapServer, ImapServerConfig, MemoryBackend,
    Flag,
}
use extlib::tls::TlsServerConfig
use std::net::SocketAddr
use std::time::Timestamp

fn run_test_server(): Result[(), ImapError] ! Async + Net {
    let mut backend = MemoryBackend::new()

    backend.add_mailbox("INBOX")
    backend.add_mailbox("Sent")

    backend.add_message(
        "INBOX",
        &[],
        Timestamp::now(),
        b"From: alice@example.com\r\nTo: bob@example.com\r\nSubject: Hello\r\n\r\nHi Bob!\r\n",
    )

    backend.add_message(
        "INBOX",
        &[Flag::Seen],
        Timestamp::now(),
        b"From: carol@example.com\r\nTo: bob@example.com\r\nSubject: Meeting\r\n\r\nSee you there.\r\n",
    )

    let tls = TlsServerConfig::self_signed_for_testing()

    let config = ImapServerConfig {
        tls_config:       tls,
        enable_starttls:  false,
        extra_caps:       vec![],
        max_literal_size: 10 * 1024 * 1024,
        idle_timeout:     Duration::from_secs(1800),
    }

    let addr: SocketAddr = "127.0.0.1:10993".parse().unwrap()

    let mut server = ImapServer::bind(addr, config, backend).await?
    println("test IMAP server listening on {}", addr)
    server.run().await
}
```

---

## 12. Dependencies

| Dependency | Role |
|---|---|
| `extlib::tls` | `TlsClientConfig`, `TlsServerConfig`, TLS handshake engine for IMAPS and STARTTLS |
| `extlib::connect` | Happy eyeballs, SO_ERROR checking, SRV lookup for `_imap._tcp` and `_imaps._tcp`, DANE enforcement via TLSA records |
| `std::net` | `TcpStream`, `SocketAddr`, `IpAddr`, `IoError`, `AsyncRead`, `AsyncWrite` |
| `std::async` | `Future`, async runtime, `scope`, structured concurrency for IDLE and streaming fetch |
| `std::io` | `AsyncRead`, `AsyncReadExt`, `copy_async` — used in streaming body section fetch |
| `std::time` | `Duration`, `Timestamp`, `Date`, `Instant` — timeouts, internal dates, search date criteria |
| `std::collections` | `HashMap` — `FetchResponse::sections` storage |
| `std::bytes` | `Bytes` — zero-copy body section storage in fetch responses |

### Feature Flags

```
imap-server    = []                  // enables ImapServer, MailboxBackend, MemoryBackend
                                     // always included; no extra deps beyond core
imap-idle      = []                  // IDLE support; always included
imap-streaming = ["std::io"]         // fetch_body_section_stream; AsyncRead-based
                                     // large body delivery without full buffering
```

A client-only build that imports `extlib::imap` gets the full client (connect,
authenticate, select, fetch, search, store, copy, move, expunge, append, idle) with
no server code linked unless `imap-server` is explicitly required.

---

*Part of the Ferrum extended standard library — CCSP (Communication, Cryptography and
Systems Protocols) suite.*
*See also: `extlib::smtp`, `extlib::mime`, `extlib::tls`, `extlib::connect`*
