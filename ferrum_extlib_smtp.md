# Ferrum Extended Standard Library — SMTP

**Module path:** `extlib::smtp`
**Implements:** RFC 5321 (SMTP), RFC 5322 (Internet Message Format), RFC 6376 (DKIM),
RFC 2920 (PIPELINING), RFC 1870 (SIZE), RFC 6531 (SMTPUTF8), DANE for outbound MX
**Dependencies:** `extlib::tls`, `extlib::dns_secure`; stdlib `crypto`, `async`, `net`

---

## 1. Overview and Rationale

### Why SMTP Is Not in the Standard Library

The stdlib `net` module provides TCP streams, TLS connections, and DNS resolution.
SMTP is not there. This is deliberate.

**Most programs do not send email.** A web server, a command-line tool, a database
driver — none of these need SMTP. Pulling in a full SMTP implementation as part of
every Ferrum program would impose binary size, compile time, and dependency surface on
code that will never call it. The extlib is for domain-specific protocol libraries that
are common enough to warrant a first-party, audited implementation but not universal
enough to belong in the stdlib.

**SMTP has a severe CVE history.** Sendmail is one of the most consistently
vulnerability-ridden pieces of infrastructure software ever written, with critical
CVEs spanning four decades (CVE-1999-0206, CVE-2002-0906, CVE-2003-0161,
CVE-2006-0058, CVE-2014-3956, and many others). The root causes recur:
hand-written address parsers operating on byte buffers without length tracking,
manual MIME boundary search with off-by-one errors, sprintf into fixed-size
stack buffers, and state machines that accept out-of-order commands. These are
not implementation accidents — they are the natural consequence of writing SMTP
clients and servers in C close to the wire.

**A declarative API prevents an entire class of bugs.** This library does not expose
raw SMTP command streams or byte-level message construction. Application code
assembles a `Message` using a typed builder; the library serializes it. There is no
way to produce a malformed `Content-Type` header, inject a newline into an address,
or construct a MIME boundary that conflicts with body content through the public API.
Buffer management is invisible to callers.

### What This Library Provides

- RFC 5322 message construction with automatic MIME multipart handling
- SMTP submission client with STARTTLS, implicit TLS (SMTPS), PIPELINING
- AUTH PLAIN and XOAUTH2
- SMTP extensions: SIZE, 8BITMIME, SMTPUTF8, PIPELINING, STARTTLS, AUTH
- Outbound MTA delivery with MX lookup and DANE enforcement
- DKIM signing (Ed25519) and verification

### What This Library Does Not Provide

- An IMAP or POP3 client (separate extlib modules)
- A full MTA (mail queuing, retry scheduling, bounce handling)
- Inbound spam filtering
- Message parsing of arbitrary received email (use `extlib::mime` for that)

---

## 2. Message Construction (RFC 5322)

### 2.1 EmailAddress

```ferrum
// An RFC 5321/5322 email address.
// Represents display-name + addr-spec as a validated pair.
// The addr-spec (local@domain) is always valid ASCII per RFC 5321.
// For Unicode local parts, see SmtputfAddress.
struct EmailAddress {
    pub display_name: Option[String],
    pub local:        String,    // left of @, ASCII only
    pub domain:       String,    // right of @, ASCII only (ACE-encoded if IDN)
}

enum AddressError {
    EmptyLocalPart,
    EmptyDomain,
    InvalidCharacter { position: usize, ch: char },
    LocalPartTooLong,          // RFC 5321 §4.5.3: 64 octets max
    DomainTooLong,             // RFC 5321 §4.5.3: 253 chars max
    QuotedStringUnclosed,
    CommentUnclosed,
    MalformedDisplayName,
}

impl EmailAddress {
    // Parse an RFC 5322 address — "Alice <alice@example.com>" or "alice@example.com"
    fn parse(s: &str): Result[Self, AddressError]

    // Construct directly from parts (validates each)
    fn new(local: &str, domain: &str): Result[Self, AddressError]
    fn with_display(display: &str, local: &str, domain: &str): Result[Self, AddressError]

    // addr-spec only: "alice@example.com"
    fn addr_spec(&self): String

    // RFC 5322 formatted: "Alice <alice@example.com>" or "alice@example.com"
    fn to_header_value(&self): String
}

impl Display for EmailAddress    // same as to_header_value
impl Debug for EmailAddress
impl PartialEq for EmailAddress  // compares addr-spec case-insensitively on domain
impl Hash for EmailAddress
```

### 2.2 SmtputfAddress (RFC 6531)

```ferrum
// An internationalized email address where the local part may contain Unicode.
// Requires the SMTPUTF8 SMTP extension. Domain is always ACE-encoded for DNS
// but displayed as U-label for humans.
struct SmtputfAddress {
    pub display_name: Option[String],
    pub local:        String,    // Unicode, NFC-normalized
    pub domain:       String,    // Unicode U-label form
}

enum SmtputfError {
    AddressError(AddressError),
    LocalPartNotNfcNormalized,
    BidiViolation,              // RFC 5893 bidirectional constraint
    InvalidLabel,
}

impl SmtputfAddress {
    fn parse(s: &str): Result[Self, SmtputfError]
    fn addr_spec_utf8(&self): String       // Unicode form: "用户@例子.广告"
    fn addr_spec_ace(&self): String        // ACE form for SMTP wire: "用户@xn--fsqu00a.xn--..."
    fn to_ascii_address(&self): Result[EmailAddress, SmtputfError]
        // Converts to ASCII-compatible address if local part is ASCII
}
```

### 2.3 MessageBuilder and Message

```ferrum
// Builds an RFC 5322 message. The builder enforces structural constraints:
// - At least one From address is required before build()
// - At least one To, Cc, or Bcc address is required before build()
// - Date is set to current time if not provided
// - Message-ID is generated if not provided
// - Content-Type and MIME structure are determined automatically
struct MessageBuilder {
    // Addressing
    fn from(self, addr: EmailAddress): Self
    fn to(self, addr: EmailAddress): Self
    fn cc(self, addr: EmailAddress): Self
    fn bcc(self, addr: EmailAddress): Self
        // Bcc addresses are used for the SMTP RCPT TO envelope but are
        // not included in the serialized message headers.
    fn reply_to(self, addr: EmailAddress): Self

    // Internationalized addressing (requires SMTPUTF8 support on server)
    fn from_utf8(self, addr: SmtputfAddress): Self
    fn to_utf8(self, addr: SmtputfAddress): Self
    fn cc_utf8(self, addr: SmtputfAddress): Self
    fn bcc_utf8(self, addr: SmtputfAddress): Self

    // Header fields
    fn subject(self, subject: &str): Self
    fn message_id(self, id: &str): Self
        // Must be globally unique; caller is responsible for uniqueness.
        // If not called, a Message-ID is generated automatically using
        // a cryptographic random component.
    fn date(self, ts: Timestamp): Self
        // If not called, uses current time at build() invocation.
    fn in_reply_to(self, message_id: &str): Self
    fn references(self, message_ids: &[&str]): Self

    // Arbitrary headers — name and value are validated for RFC 5322 compliance.
    // Attempting to set a structured header (From, To, Date, etc.) through
    // this method returns Err(HeaderError::StructuredHeaderRequired).
    fn header(self, name: &str, value: &str): Result[Self, HeaderError]

    // Body content
    // If only text_body is set: Content-Type: text/plain
    // If only html_body is set: Content-Type: text/html
    // If both are set: multipart/alternative (text first, html second per RFC 2046)
    // If attachments are added: multipart/mixed wrapping the above
    fn text_body(self, content: &str): Self
    fn html_body(self, content: &str): Self

    // Attachments
    fn attachment(self, filename: &str, content_type: &str, data: &[u8]): Self
        // content_type must be a valid MIME type ("image/png", "application/pdf", etc.)
        // data is encoded as base64 in the MIME part
    fn inline_attachment(self, content_id: &str, content_type: &str, data: &[u8]): Self
        // Content-Disposition: inline; Content-ID for use in HTML <img src="cid:...">

    // Build the message, consuming the builder.
    // Returns Err if required fields are missing or any field is invalid.
    fn build(self): Result[Message, MessageBuildError]
}

impl MessageBuilder {
    fn new(): Self
}

enum MessageBuildError {
    MissingFrom,
    MissingRecipient,            // no To, Cc, or Bcc set
    HeaderError(HeaderError),
    InvalidContentType { content_type: String },
    InvalidAttachmentFilename { filename: String },
}

enum HeaderError {
    InvalidName { name: String },
    InvalidValue { name: String, value: String },
    StructuredHeaderRequired { name: String },
    FoldedValueTooLong,
}

// An immutable, fully-constructed RFC 5322 message.
// The internal representation is the serialized byte form, which is computed
// once at build() time. All accessors are zero-copy views into that buffer.
struct Message {
    pub fn from_addrs(&self): &[EmailAddress]
    pub fn to_addrs(&self): &[EmailAddress]
    pub fn cc_addrs(&self): &[EmailAddress]
    pub fn bcc_addrs(&self): &[EmailAddress]
        // Returns the Bcc addresses for use by the SMTP client.
        // These do not appear in as_bytes().
    pub fn subject(&self): Option[&str]
    pub fn message_id(&self): Option[&str]
    pub fn date(&self): Option[Timestamp]
    pub fn header(&self, name: &str): Option[&str]

    // The serialized RFC 5322 message bytes, suitable for DATA command transmission.
    // Bcc headers are stripped. Line endings are CRLF. Dot-stuffing is applied.
    pub fn as_bytes(&self): &[u8]

    // Size in bytes (for the SMTP SIZE extension pre-check)
    pub fn byte_size(&self): usize

    // True if any address (from, to, cc, bcc) is a SmtputfAddress.
    // Determines whether SMTPUTF8 is required for transmission.
    pub fn requires_smtputf8(&self): bool
}
```

---

## 3. SMTP Client

### 3.1 Configuration

```ferrum
// Security model for the connection to the submission server.
enum SmtpSecurity {
    // No TLS. Suitable only for localhost submission (port 25) or
    // internal networks where eavesdropping is not a concern.
    // Not suitable for submission over the public internet.
    Plain,

    // STARTTLS upgrade after initial plaintext greeting (port 587).
    // Aborts if the server does not offer STARTTLS.
    // Does not fall back to plaintext — fallback is a downgrade attack surface.
    StartTls {
        tls_config: TlsClientConfig,
    },

    // Implicit TLS from the start (port 465 / RFC 8314).
    // Preferred for submission; no cleartext window.
    Smtps {
        tls_config: TlsClientConfig,
    },
}

// Authentication mechanism.
enum SmtpAuth {
    // No authentication. Suitable only for open relays or localhost.
    None,

    // AUTH PLAIN (RFC 4616) — username and password, base64-encoded.
    // Only safe over TLS. The client refuses to send PLAIN credentials
    // unless the connection is TLS-protected.
    Plain {
        username: String,
        password: String,
    },

    // AUTH XOAUTH2 — token is fetched by calling token_fn at auth time.
    // The function is called each time authentication is needed, which
    // allows token refresh without reconstructing the client.
    XOAuth2 {
        username: String,
        token_fn: Box[dyn Fn() -> String + Send + Sync],
    },
}

struct SmtpClientConfig {
    pub server:           String,
    pub port:             u16,
    pub security:         SmtpSecurity,
    pub auth:             SmtpAuth,

    // Enable DANE verification for the submission server.
    // Requires dns_secure to be available and the server's domain to be
    // DNSSEC-signed. If DANE is enabled and DNSSEC validation fails,
    // the connection is refused rather than falling back.
    pub dane:             bool,

    pub connect_timeout:  Duration,
    pub command_timeout:  Duration,    // per-command timeout (EHLO, MAIL FROM, etc.)
    pub data_timeout:     Duration,    // timeout for the DATA transfer phase

    // Maximum message size this client will attempt to send.
    // If the server advertises a smaller SIZE, the client checks before sending.
    // If not set, relies entirely on the server's advertised limit.
    pub max_message_size: Option[usize],
}

impl SmtpClientConfig {
    // Convenience constructor for a typical submission server.
    // Uses SMTPS on port 465 with AUTH PLAIN.
    fn new_submission(
        server: &str,
        username: &str,
        password: &str,
        tls_config: TlsClientConfig,
    ): Self

    // Convenience constructor for localhost submission (port 25, no auth, no TLS).
    fn new_local(): Self
}
```

### 3.2 SmtpClient

```ferrum
struct SmtpClient { ... }

impl SmtpClient {
    // Establish a connection to the server and negotiate capabilities.
    // Performs EHLO, optionally STARTTLS, and AUTH.
    // The connection is held open for subsequent send() calls.
    fn connect(config: SmtpClientConfig): Result[Self, SmtpError] ! Async + Net

    // Send a single message.
    // Issues MAIL FROM, RCPT TO (for each To/Cc/Bcc address), DATA.
    // If the server advertised SIZE, checks message.byte_size() before sending.
    // If the server advertised SMTPUTF8 and message.requires_smtputf8() is false,
    // the SMTPUTF8 parameter is omitted for compatibility.
    fn send(&mut self, message: &Message): Result[SendReceipt, SmtpError] ! Async + Net

    // Send multiple messages in a single connection, using PIPELINING (RFC 2920)
    // when the server supports it. MAIL FROM + RCPT TO commands for all messages
    // are batched and sent before reading responses. Falls back to sequential
    // send if the server does not advertise PIPELINING.
    //
    // Returns one SendResult per message. Individual message failures do not
    // abort the batch — all messages are attempted and their results collected.
    fn send_batch(
        &mut self,
        messages: &[Message],
    ): Result[Vec[SendResult], SmtpError] ! Async + Net

    // Verify a recipient address (VRFY command). Many servers disable VRFY;
    // returns Err(SmtpError::CommandNotSupported) in that case.
    fn verify_recipient(&mut self, addr: &EmailAddress): Result[VrfyResult, SmtpError] ! Async + Net

    // Gracefully close the connection (QUIT command).
    fn quit(self): Result[(), SmtpError] ! Async + Net

    // Return the set of SMTP extensions advertised by the server in EHLO.
    fn server_capabilities(&self): &SmtpCapabilities
}

struct SmtpCapabilities {
    pub pipelining:   bool,
    pub starttls:     bool,
    pub size_limit:   Option[usize],    // advertised SIZE value
    pub eight_bitmime: bool,
    pub smtputf8:     bool,
    pub auth_plain:   bool,
    pub auth_xoauth2: bool,
    pub server_name:  String,           // from EHLO response
}

// Result of a successful send.
struct SendReceipt {
    // The Message-ID of the sent message (from the message headers).
    pub message_id:      Option[String],
    // The final 250 response from the server after DATA + "." terminator.
    pub server_response: String,
    // The server's queue identifier, if the server provides one in the response.
    pub queue_id:        Option[String],
}

// Result for one message in a batch send.
enum SendResult {
    Sent(SendReceipt),
    Failed {
        message_id: Option[String],
        error:      SmtpError,
    },
}

enum VrfyResult {
    Valid(EmailAddress),
    Ambiguous(Vec[EmailAddress]),
    Unknown,
}
```

---

## 4. PIPELINING (RFC 2920)

When `send_batch` is called and the server advertises `PIPELINING`, the client
batches multiple commands before waiting for responses. The protocol sequence
for two messages (message A to alice@example.com, message B to bob@example.com)
looks like this:

```
Client sends in one write:
  MAIL FROM:<sender@example.org>
  RCPT TO:<alice@example.com>
  DATA

Server responds (client reads all three):
  250 OK
  250 OK
  354 Start input

Client sends body of message A, then:
  .
  MAIL FROM:<sender@example.org>
  RCPT TO:<bob@example.com>
  DATA

Server responds:
  250 OK (end of message A)
  250 OK
  250 OK
  354 Start input

Client sends body of message B, then:
  .

Server responds:
  250 OK (end of message B)
```

The benefit is fewer round trips, which matters when sending to many recipients
over a high-latency link to a remote submission server.

**Error handling in pipelined mode:** If a `RCPT TO` is rejected (e.g., `550 No such
user`), that specific recipient is recorded as a failure in the `SendResult` for that
message. Other recipients and messages in the batch continue. A `MAIL FROM` rejection
aborts only that message's transaction.

---

## 5. SMTP Extensions

The following extensions are negotiated automatically during EHLO and used
transparently when available:

| Extension | RFC | Client behavior |
|---|---|---|
| `PIPELINING` | RFC 2920 | Used in `send_batch`; falls back to sequential without it |
| `SIZE` | RFC 1870 | `MAIL FROM` includes `SIZE=n`; pre-checks against `max_message_size` |
| `8BITMIME` | RFC 6152 | `MAIL FROM` includes `BODY=8BITMIME` when message has non-ASCII body |
| `SMTPUTF8` | RFC 6531 | `MAIL FROM` includes `SMTPUTF8` when `message.requires_smtputf8()` is true |
| `STARTTLS` | RFC 3207 | Upgraded immediately after EHLO when `SmtpSecurity::StartTls` is configured |
| `AUTH` | RFC 4954 | Authenticates using the configured `SmtpAuth` mechanism |

---

## 6. Outbound MX Delivery with DANE

`SmtpMtaClient` is for programs that act as mail transfer agents: they look up
the MX records for a recipient domain and deliver directly, rather than relaying
through a submission server.

**Most applications should use `SmtpClient` instead.** Use `SmtpMtaClient` only
if you are building MTA-like software (message queue processor, mail gateway,
notification system that sends without a relay).

### 6.1 MX Resolution

```ferrum
// Result of MX record lookup and DANE TLSA retrieval for one MX host.
struct MxTarget {
    pub hostname:    String,
    pub priority:    u16,
    pub addresses:   Vec[IpAddr],      // A/AAAA records for this MX hostname
    pub tlsa:        Option[TlsaSet],  // DANE TLSA records if DNSSEC-validated
}

// A set of TLSA records for a host:port combination.
struct TlsaSet {
    pub records:        Vec[TlsaRecord],
    pub dnssec_valid:   bool,   // true if DNSSEC chain validated end-to-end
}

struct TlsaRecord {
    pub usage:      TlsaUsage,
    pub selector:   TlsaSelector,
    pub matching:   TlsaMatching,
    pub data:       Vec[u8],
}

enum TlsaUsage {
    PkixTa,     // 0 — CA constraint (PKIX chain must include this trust anchor)
    PkixEe,     // 1 — Service certificate constraint
    DaneTa,     // 2 — Trust anchor assertion (no PKIX chain needed)
    DaneEe,     // 3 — Domain-issued certificate (no PKIX chain needed)
}

enum TlsaSelector {
    FullCert,     // 0 — full DER certificate
    SubjectPubKey // 1 — SubjectPublicKeyInfo
}

enum TlsaMatching {
    Full,    // 0 — exact match
    Sha256,  // 1
    Sha512,  // 2
}
```

### 6.2 SmtpMtaClient

```ferrum
struct SmtpMtaClientConfig {
    // TLS configuration used when connecting to MX hosts.
    pub tls_config:        TlsClientConfig,

    // Enforce DANE when DNSSEC-validated TLSA records are found.
    // If true and TLSA records exist but the certificate does not match,
    // the connection is refused and delivery fails for that MX.
    // If false, DANE validation errors are logged but not fatal.
    pub dane_enforce:      bool,

    // Fall back to opportunistic TLS (STARTTLS without DANE) when the MX
    // host has no TLSA records or DNSSEC is not available.
    pub opportunistic_tls: bool,

    pub connect_timeout:   Duration,
    pub command_timeout:   Duration,
    pub data_timeout:      Duration,
    pub max_message_size:  Option[usize],
}

struct SmtpMtaClient { ... }

impl SmtpMtaClient {
    fn new(config: SmtpMtaClientConfig): Self

    // Resolve MX records for to_domain, select a target using priority/weight,
    // establish a connection, and deliver the message.
    //
    // MX selection: lowest-priority MX hosts are tried first (RFC 5321 §5).
    // Within the same priority, hosts are tried in random order. On connection
    // failure, the next host is tried until all are exhausted.
    //
    // DANE: if an MX host has DNSSEC-validated TLSA records and dane_enforce
    // is true, TLS is mandatory and the certificate is validated against TLSA.
    // If DANE validation fails, that MX is skipped (not fallen back to plain TLS).
    //
    // Returns DeliveryResult describing which MX was used and the outcome.
    fn deliver(
        &self,
        message: &Message,
        to_domain: &str,
    ): Result[DeliveryResult, SmtpError] ! Async + Net

    // Resolve and return the MX targets for a domain without delivering.
    // Useful for pre-flight checks or diagnostic tooling.
    fn resolve_mx(
        &self,
        domain: &str,
    ): Result[Vec[MxTarget], SmtpError] ! Async + Net
}

struct DeliveryResult {
    pub mx_hostname:      String,
    pub mx_ip:            IpAddr,
    pub tls_used:         bool,
    pub dane_validated:   bool,
    pub receipt:          SendReceipt,
}
```

---

## 7. DKIM (RFC 6376)

### 7.1 DKIM Signing

```ferrum
// DKIM signer using Ed25519 keys.
// Ed25519 is preferred over RSA for DKIM because it produces shorter signatures
// (88 bytes base64 vs ~344 bytes for RSA-2048), has no padding oracle attacks,
// and is faster to sign and verify.
struct DkimSigner {
    selector:    String,
    domain:      String,
    private_key: Ed25519SecretKey,
}

impl DkimSigner {
    fn new(selector: &str, domain: &str, private_key: Ed25519SecretKey): Self

    // Sign the message, adding a DKIM-Signature header.
    // The header is prepended to the message's existing headers.
    // Signed headers: From, To, Subject, Date, Message-ID, Content-Type.
    // Body canonicalization: relaxed. Header canonicalization: relaxed.
    // Uses SHA-256 as the hash algorithm (ed25519-sha256).
    fn sign(&self, message: &mut Message): Result[(), DkimError]

    // Sign with explicit header list (advanced use).
    fn sign_headers(
        &self,
        message: &mut Message,
        headers_to_sign: &[&str],
    ): Result[(), DkimError]
}

enum DkimError {
    SigningFailed,
    InvalidSelector { selector: String },
    InvalidDomain { domain: String },
    MessageAlreadySigned,
    HeaderCanonalizationFailed { header: String },
}
```

### 7.2 DKIM Verification

```ferrum
// Verify the DKIM-Signature(s) on a received message.
// Fetches the public key from DNS using a TXT record query on
// <selector>._domainkey.<domain>. Requires network access.
struct DkimVerifier { ... }

impl DkimVerifier {
    fn new(): Self

    fn verify(
        &self,
        message: &Message,
    ): Result[DkimVerifyResult, DkimError] ! Async + Net
}

enum DkimVerifyResult {
    // The signature is cryptographically valid, the DNS key was found,
    // and the message content has not been altered.
    Valid {
        domain:   String,
        selector: String,
    },

    // The signature structure was valid but the cryptographic check failed.
    // The message may have been tampered with in transit.
    Invalid {
        domain:   String,
        selector: String,
        reason:   DkimInvalidReason,
    },

    // The message has no DKIM-Signature header.
    NoSignature,

    // A DKIM-Signature header was present but the DNS TXT record for the
    // signing key does not exist. The key may have been rotated or the
    // signature may be forged with a non-existent selector.
    KeyNotFound {
        domain:   String,
        selector: String,
    },
}

enum DkimInvalidReason {
    SignatureMismatch,
    BodyHashMismatch,
    SignatureExpired { expired_at: Timestamp },
    KeyAlgorithmMismatch { expected: String, found: String },
    RequiredHeaderMissing { header: String },
    MalformedSignatureHeader,
}
```

---

## 8. SMTP Server (Basic)

This is a minimal SMTP server sufficient for test harnesses, internal mail
acceptance endpoints, and simple gatewaying. It is not a full MTA.

```ferrum
struct SmtpServerConfig {
    // The SMTP banner string returned in the 220 greeting.
    // Example: "mail.example.com ESMTP ready"
    pub banner:            String,

    // TLS configuration. If None, the server does not offer STARTTLS.
    pub tls_config:        Option[TlsServerConfig],

    // Called when a client issues AUTH. If None, AUTH is not advertised.
    pub auth_handler:      Option[Box[dyn AuthHandler + Send + Sync]>,

    pub max_message_size:  usize,
    pub max_recipients:    usize,       // per-message RCPT TO limit
    pub command_timeout:   Duration,    // timeout per command from client
    pub data_timeout:      Duration,    // timeout for DATA reception
}

// Implemented by the application to handle incoming messages.
// Called once per received message after the DATA "." terminator is received
// and the complete message has been parsed.
trait MessageHandler {
    fn accept(
        &self,
        envelope: &Envelope,
        message: &Message,
    ): impl Future[Output=AcceptResult] ! Async
}

// The SMTP envelope for an incoming message.
struct Envelope {
    pub mail_from:        EmailAddress,
    pub rcpt_to:          Vec[EmailAddress],
    pub source_ip:        IpAddr,
    // The username the client authenticated as, if AUTH succeeded.
    pub authenticated_as: Option[String],
    // True if the client issued EHLO with SMTPUTF8.
    pub smtputf8:         bool,
}

// The handler's response to an incoming message.
enum AcceptResult {
    // Accept the message. The server responds with 250 OK.
    Accepted { queue_id: String },
    // Reject with a permanent failure. The server responds with 5xx.
    Rejected { code: u16, message: String },
    // Reject with a transient failure (try again later). Responds with 4xx.
    Deferred { code: u16, message: String },
}

// Implemented by the application to handle AUTH.
trait AuthHandler {
    fn authenticate(
        &self,
        mechanism: &str,
        credentials: &AuthCredentials,
    ): impl Future[Output=AuthResult] ! Async
}

struct AuthCredentials {
    pub username: String,
    pub password: Option[String],   // None for token-based mechanisms
    pub token:    Option[String],
}

enum AuthResult {
    Authenticated { username: String },
    Failed,
    Error { message: String },
}

struct SmtpServer { ... }

impl SmtpServer {
    // Bind and start listening. Does not block — returns immediately.
    // Call serve() to run the accept loop.
    fn bind(
        addr: SocketAddr,
        config: SmtpServerConfig,
        handler: Box[dyn MessageHandler + Send + Sync],
    ): Result[Self, SmtpError] ! Async + Net

    // Run the accept loop. Blocks (or awaits) indefinitely.
    // Each accepted connection is handled concurrently.
    fn serve(&self): Result[(), SmtpError] ! Async + Net

    fn local_addr(&self): SocketAddr
}
```

---

## 9. Error Types

```ferrum
enum SmtpError {
    // TCP connection to the server could not be established.
    ConnectionFailed {
        host:   String,
        port:   u16,
        source: IoError,
    },

    // EHLO or other connection-phase failure.
    HandshakeFailed { message: String },

    // TLS negotiation failed (certificate error, protocol error, etc.).
    TlsError {
        host:   String,
        source: TlsError,
    },

    // DANE validation failed: TLSA records exist and were DNSSEC-validated,
    // but the server's certificate does not match any TLSA record.
    DaneError {
        host:    String,
        message: String,
    },

    // AUTH command failed (wrong credentials, unsupported mechanism, etc.).
    AuthFailed { message: String },

    // The server rejected MAIL FROM, RCPT TO, or DATA with a 5xx response.
    MessageRejected {
        code:    u16,
        message: String,
    },

    // A specific recipient was rejected (4xx or 5xx on RCPT TO).
    RecipientNotAccepted {
        recipient: EmailAddress,
        code:      u16,
        message:   String,
    },

    // Message is larger than max_message_size or the server's advertised SIZE.
    SizeExceeded {
        message_size:   usize,
        size_limit:     usize,
    },

    // SMTPUTF8 is required for this message but the server does not support it.
    SmtputfNotSupported,

    // A network error occurred during an established connection.
    NetworkError(IoError),

    // The server sent a response that does not conform to RFC 5321.
    ProtocolError { response: String },

    // An operation timed out.
    Timeout { operation: String },

    // DNS resolution failed (for MX lookup or DANE).
    DnsError(DnsError),

    // An SMTP command was issued that the server does not support.
    CommandNotSupported { command: String },
}

impl Error for SmtpError
impl Display for SmtpError
impl Debug for SmtpError
```

---

## 10. Example Usage

### 10.1 Send an HTML Email with Attachment via STARTTLS

```ferrum
use extlib::smtp::{
    EmailAddress, MessageBuilder, SmtpClient, SmtpClientConfig,
    SmtpSecurity, SmtpAuth,
}
use extlib::tls::TlsClientConfig

fn send_report(
    report_pdf: &[u8],
    recipient: &str,
) : Result[(), SmtpError] ! Async + Net {

    let from = EmailAddress::parse("reports@example.com")?
    let to   = EmailAddress::parse(recipient)?

    let message = MessageBuilder::new()
        .from(from.clone())
        .to(to)
        .subject("Monthly Report — April 2026")
        .text_body("Please find the monthly report attached.")
        .html_body("<p>Please find the monthly report attached.</p>")
        .attachment("report-2026-04.pdf", "application/pdf", report_pdf)
        .build()?

    let tls = TlsClientConfig::default_with_system_roots()

    let config = SmtpClientConfig {
        server:           "smtp.example.com".to_string(),
        port:             587,
        security:         SmtpSecurity::StartTls { tls_config: tls },
        auth:             SmtpAuth::Plain {
                              username: "reports@example.com".to_string(),
                              password: "hunter2".to_string(),
                          },
        dane:             false,
        connect_timeout:  Duration::from_secs(10),
        command_timeout:  Duration::from_secs(30),
        data_timeout:     Duration::from_secs(120),
        max_message_size: None,
    }

    let mut client = SmtpClient::connect(config).await?
    let receipt    = client.send(&message).await?
    client.quit().await?

    println("Sent; queue-id: {}", receipt.queue_id.unwrap_or("(none)"))
    Ok(())
}
```

### 10.2 MTA Delivery with DKIM Signing and DANE

```ferrum
use extlib::smtp::{
    EmailAddress, MessageBuilder, SmtpMtaClient, SmtpMtaClientConfig,
    DkimSigner,
}
use stdlib::crypto::Ed25519

fn deliver_with_dkim(
    to: EmailAddress,
    subject: &str,
    body: &str,
    dkim_private_key: Ed25519SecretKey,
) : Result[DeliveryResult, SmtpError] ! Async + Net {

    let from = EmailAddress::parse("noreply@sending.example")?

    let mut message = MessageBuilder::new()
        .from(from)
        .to(to.clone())
        .subject(subject)
        .text_body(body)
        .build()?

    // Sign before delivery
    let signer = DkimSigner::new("2026a", "sending.example", dkim_private_key)
    signer.sign(&mut message)?

    let to_domain = to.domain.clone()

    let config = SmtpMtaClientConfig {
        tls_config:        TlsClientConfig::default_with_system_roots(),
        dane_enforce:      true,
        opportunistic_tls: true,
        connect_timeout:   Duration::from_secs(15),
        command_timeout:   Duration::from_secs(30),
        data_timeout:      Duration::from_secs(300),
        max_message_size:  Some(25 * 1024 * 1024),  // 25 MiB
    }

    let mta = SmtpMtaClient::new(config)
    mta.deliver(&message, &to_domain).await
}
```

### 10.3 Minimal Test SMTP Server

```ferrum
use extlib::smtp::{
    SmtpServer, SmtpServerConfig, Envelope, Message,
    MessageHandler, AcceptResult,
}
use stdlib::sync::Mutex

struct CollectingHandler {
    received: Mutex[Vec[Message]],
}

impl MessageHandler for CollectingHandler {
    fn accept(
        &self,
        _envelope: &Envelope,
        message: &Message,
    ): impl Future[Output=AcceptResult] ! Async {
        {
            self.received.lock().push(message.clone())
            AcceptResult::Accepted { queue_id: "test-001".to_string() }
        }
    }
}

fn run_test_server(): Result[SmtpServer, SmtpError] ! Async + Net {
    let handler = Box::new(CollectingHandler { received: Mutex::new(vec[]) })

    let config = SmtpServerConfig {
        banner:           "localhost test ESMTP".to_string(),
        tls_config:       None,
        auth_handler:     None,
        max_message_size: 10 * 1024 * 1024,
        max_recipients:   100,
        command_timeout:  Duration::from_secs(30),
        data_timeout:     Duration::from_secs(60),
    }

    SmtpServer::bind(
        SocketAddr::from_str("127.0.0.1:2525")?,
        config,
        handler,
    )
}
```

---

## 11. Dependencies

### Extlib Dependencies

| Module | Used for |
|---|---|
| `extlib::tls` | `TlsClientConfig`, `TlsServerConfig`, `TlsStream`, `TlsError` for STARTTLS and SMTPS |
| `extlib::dns_secure` | DNSSEC-validated MX and TLSA record resolution in `SmtpMtaClient` and `DkimVerifier` |

### Stdlib Dependencies

| Module | Used for |
|---|---|
| `stdlib::crypto` | `Ed25519SecretKey`, `Ed25519PublicKey` for DKIM; `Sha256` for body hash |
| `stdlib::async` | `Future`, `Runtime`, `.await`, `Duration` |
| `stdlib::net` | `TcpStream`, `SocketAddr`, `IpAddr`, `IoError` |
| `stdlib::time` | `Timestamp`, `Duration` for date headers and timeouts |
| `stdlib::alloc` | `String`, `Vec`, `Box` |

### No Further Dependencies

This module does not depend on `extlib::http`, `extlib::json`, or any serialization
framework. It has no build-time code generation step. All MIME boundary generation,
base64 encoding, and header folding are implemented within the module using stdlib
primitives.
