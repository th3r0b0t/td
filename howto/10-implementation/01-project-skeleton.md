# 10.1 — Project Skeleton

[← Previous](../09-advanced/02-connection-management.md) · [Index](../README.md) · [Next: C guide →](02-c-guide.md)

---

Language-independent structure: what modules you need, what each one owns, and what order
to build them in.

---

## 1. Module layout

```
mtproto/
├── crypto/
│   ├── aes_ige         AES-256-IGE encrypt/decrypt      → 4.3
│   ├── aes_ctr         AES-256-CTR for obfuscation      → 2.3
│   ├── rsa             Raw RSA (x^e mod n), RSA_PAD     → 3.2
│   ├── dh              Modular exponentiation + checks  → 3.5
│   ├── hash            SHA-1, SHA-256, HMAC, PBKDF2     → 4.2, 7.5
│   └── random          CSPRNG                           → everywhere
├── tl/
│   ├── serialize       int/long/double/string/bytes/vector → 1.2
│   ├── deserialize     the inverse, with bounds checks  → 1.2
│   └── schema          generated or hand-written types  → 1.1
├── transport/
│   ├── tcp             socket, connect, read/write loop → 9.2
│   ├── framing         Intermediate / PaddedIntermediate → 2.2
│   └── obfuscation     the 64-byte header, CTR streams  → 2.3
├── mtproto/
│   ├── handshake       req_pq → req_DH → set_client_DH  → 3.x
│   ├── envelope        encrypt/decrypt a message        → 4.x
│   ├── session         msg_id, seq_no, salt, acks       → 5.x
│   └── dispatch        rpc_result, service msgs, updates → 6.2
└── api/
    ├── connection      invokeWithLayer + initConnection → 6.1
    ├── auth            sendCode, signIn, SRP            → 7.x
    └── messages        resolve peer, sendMessage        → 8.x
```

Each row maps to a chapter. Build bottom-up.

---

## 2. Build order

Each step is independently testable. Do not proceed until the current step passes.

| # | Build | Test |
|---|-------|------|
| 1 | TL serialize/deserialize | Round-trip; compare with the byte layouts in [1.3](../01-serialization/03-worked-examples.md) |
| 2 | SHA-1/SHA-256, AES-IGE | Known-answer vectors; encrypt-then-decrypt |
| 3 | TCP + Intermediate framing | Connect to a DC; the socket stays open |
| 4 | Handshake | You obtain a 256-byte `auth_key` |
| 5 | Envelope | Encrypt then decrypt your own message |
| 6 | `initConnection` + `help.getConfig` | A parseable `config` reply |
| 7 | `auth.sendCode` | A code arrives |
| 8 | `auth.signIn` | `auth.authorization` |
| 9 | `messages.sendMessage` to self | The message appears in Saved Messages |

Step 6 is the milestone: reaching it means serialization, crypto, framing, the handshake,
and the envelope are **all** correct. Everything after it is ordinary API work.

---

## 3. Dependencies

| Need | C | Rust |
|------|---|------|
| Big integers | GMP, or OpenSSL `BN_*` | `num-bigint`, `rug` |
| AES | OpenSSL `AES_*` / `EVP_*` | `aes` + `ctr` crates |
| SHA-1/SHA-256 | OpenSSL | `sha1`, `sha2` |
| PBKDF2-HMAC-SHA512 | OpenSSL `PKCS5_PBKDF2_HMAC` | `pbkdf2` + `hmac` + `sha2` |
| CSPRNG | `getrandom(2)`, OpenSSL `RAND_bytes` | `rand::rngs::OsRng`, `getrandom` |
| gzip | zlib | `flate2` |
| Sockets | POSIX | `std::net`, or `tokio` |

OpenSSL alone covers every cryptographic need for C. AES-IGE is **not** in OpenSSL's public
API — you implement it over `AES_encrypt`/`AES_decrypt`, which is about 20 lines
([4.3](../04-encrypted-messages/03-aes-ige.md)).

> **⚠ Never write your own AES, SHA, or big-integer arithmetic.** Use a reviewed library.
> A constant-time modular exponentiation is not something to attempt for a first project.

---

## 4. Handling the schema

Three options:

**(a) Hand-write the ~30 constructors you need.** Fastest to a working client. Everything
required for login and one message is listed in
[Appendix A](../appendix/A-constructor-ids.md). Recommended.

**(b) Write a `.tl` parser and generate code.** The right answer for a real library, and a
substantial project in itself. TDLib's own parser is `td/generate/tl-parser/`.

**(c) Use an existing generator.** Several exist for Rust and Python. Convenient, but you
inherit their bugs and their layer.

For the purposes of this guide, (a).

---

## 5. Core state

```
struct Client {
    // Connection
    socket:              Socket,
    send_ctr, recv_ctr:  AesCtrState,     // only if obfuscated

    // Crypto
    auth_key:            [u8; 256],
    auth_key_id:         u64,             // SHA1(auth_key)[12..20)

    // Session
    session_id:          u64,             // random non-zero, per session
    server_salt:         u64,
    seq_no:              u32,             // increments by 2 per content message
    server_time_diff:    f64,
    last_msg_id:         u64,

    // Reliability
    pending:             Map<u64, PendingQuery>,
    to_ack:              Vec<u64>,

    // API
    dc_id:               i32,
    init_connection_sent: bool,
    user_hashes:         Map<i64, i64>,
}
```

That is the complete state of a working client. Note what is **not** there: no thread pool,
no actor system, no event bus.

---

## 6. Persistence

A minimal file:

```
magic       4 bytes    e.g. "TGMP"
version     4 bytes
dc_id       4 bytes
auth_key  256 bytes
salt        8 bytes
time_diff   8 bytes
user_id     8 bytes
```

288 bytes, mode `0600`. See [7.6](../07-login/06-after-login.md) for the security
requirements.

---

## 7. Error handling

Distinguish three classes, because they need different responses:

| Class | Examples | Response |
|-------|----------|----------|
| **Transport** | Connection refused, `-404`, malformed frame | Reconnect |
| **Protocol** | `bad_msg_notification`, `bad_server_salt` | Correct state, resend |
| **API** | `rpc_error` 400/401/420 | Surface to the caller, or wait |

Collapsing them into one "error" type produces a client that reconnects on a
`FLOOD_WAIT`, or gives up on a recoverable `bad_server_salt`.

---

## 8. Checklist

- [ ] Modules layered: crypto → TL → transport → MTProto → API
- [ ] Built and tested bottom-up
- [ ] `help.getConfig` succeeds before any login work begins
- [ ] Reviewed crypto libraries used, nothing hand-rolled except AES-IGE
- [ ] Only the ~30 needed constructors implemented
- [ ] Client state small and explicit
- [ ] Persistence file `0600` with a version field
- [ ] Transport, protocol, and API errors distinguished

---

[← Previous](../09-advanced/02-connection-management.md) · [Index](../README.md) · [Next: C guide →](02-c-guide.md)
