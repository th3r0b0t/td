# Talking MTProto 2.0 Without TDLib

> **A complete, step-by-step guide to implementing the Telegram MTProto 2.0 protocol from
> scratch — enough to create an authorization key, log in with a phone number, and send a
> text message — in C, Rust, or any other language.**

This guide was written by reading the reference implementation contained in this
repository (TDLib) line by line, and cross-checking it against the published protocol
description at <https://core.telegram.org/mtproto/description>. Every non-obvious claim in
this documentation carries a citation of the form `path/to/file.cpp:123`, pointing at the
exact source in this repository that the claim was derived from. **When the published
documentation and the code disagree, the code wins** — Telegram's servers implement the
code, not the prose. Places where they differ are flagged explicitly.

---

## Who this is for

You want to write a Telegram client (or bot, or userbot, or protocol analyzer) that speaks
directly to Telegram's servers over a socket, without linking TDLib and without using any
existing MTProto library. You are comfortable with:

* binary wire formats and byte-level thinking,
* modular arithmetic on 2048-bit numbers,
* AES, SHA-1, SHA-256, HMAC, PBKDF2 (you do **not** need to implement them — use OpenSSL,
  libsodium, `ring`, `RustCrypto`, etc.),
* sockets and asynchronous I/O.

You do **not** need to know anything about Telegram or MTProto beforehand. Everything is
built up from first principles.

---

## What you will have built at the end

A program that:

1. opens a TCP connection to a Telegram datacenter,
2. performs the Diffie–Hellman handshake and derives a 2048-bit **authorization key**,
3. establishes an encrypted **session**,
4. requests an SMS/app login code for a phone number,
5. signs in (including the two-factor / SRP password path),
6. resolves a `@username` to a peer,
7. sends `Hello, world` to that peer,
8. reads back the server's confirmation.

---

## How to read this guide

The chapters build strictly on one another. Read them in order the first time. Afterwards,
the [appendices](appendix/) are the part you will keep coming back to.

| # | Chapter | What it covers |
|---|---------|----------------|
| **0** | [Introduction](00-introduction/) | The mental model, the vocabulary, the layer cake |
| **1** | [Serialization (TL)](01-serialization/) | How every byte on the wire is laid out |
| **2** | [Transport](02-transport/) | Getting bytes to a datacenter: framings, obfuscation |
| **3** | [Authorization key](03-auth-key/) | The DH handshake — the hardest part, done in 3 round trips |
| **4** | [Encrypted messages](04-encrypted-messages/) | `msg_key`, the KDF, AES-IGE, the envelope |
| **5** | [The session layer](05-session/) | `msg_id`, `seq_no`, containers, acks, salts, service messages |
| **6** | [The API layer](06-api-layer/) | `initConnection`, `invokeWithLayer`, RPC, errors, DC migration |
| **7** | [Logging in](07-login/) | `auth.sendCode` → `auth.signIn` → 2FA (SRP) |
| **8** | [Sending a message](08-sending-a-message/) | Resolving peers, `messages.sendMessage`, updates |
| **9** | [Advanced topics](09-advanced/) | Perfect forward secrecy, temporary keys, multi-DC |
| **10** | [Implementation guides](10-implementation/) | Concrete C and Rust structure, testing, debugging |
| **A–E** | [Appendices](appendix/) | Constructor IDs, DC addresses, RSA keys, errors, source map |

### Chapter index

<details>
<summary><b>Full file listing</b></summary>

```
howto/
├── README.md                                  ← you are here
│
├── 00-introduction/
│   ├── 01-what-you-are-building.md            The problem, the shape of the solution
│   ├── 02-glossary.md                         Every term used in this guide
│   └── 03-architecture.md                     The five layers and how they nest
│
├── 01-serialization/
│   ├── 01-tl-language.md                      Reading a .tl schema
│   ├── 02-wire-encoding.md                    int, long, string, bytes, vector, flags
│   └── 03-worked-examples.md                  Byte-by-byte serialization walkthroughs
│
├── 02-transport/
│   ├── 01-datacenters.md                      Where to connect, DC ids, bootstrap IPs
│   ├── 02-tcp-framings.md                     Abridged / Intermediate / Padded / Full
│   ├── 03-obfuscation.md                      obfuscated2, MTProxy, fake-TLS
│   └── 04-transport-errors.md                 Bare negative int32 error frames
│
├── 03-auth-key/
│   ├── 01-overview.md                         What an auth key is and why DH
│   ├── 02-step1-req-pq.md                     req_pq_multi → resPQ, factoring pq
│   ├── 03-step2-req-dh-params.md              RSA_PAD, p_q_inner_data, server_DH_inner_data
│   ├── 04-step3-set-client-dh-params.md       g_b, dh_gen_ok, deriving the key and salt
│   ├── 05-security-checks.md                  Every check you MUST implement
│   └── 06-worked-example.md                   A complete annotated handshake
│
├── 04-encrypted-messages/
│   ├── 01-envelope.md                         The byte layout of an encrypted message
│   ├── 02-msg-key-and-kdf.md                  msg_key, KDF v2, the x=0/x=8 rule
│   ├── 03-aes-ige.md                          The IGE block cipher mode
│   └── 04-encrypt-decrypt-recipes.md          Step-by-step pseudocode both ways
│
├── 05-session/
│   ├── 01-msg-id-and-seq-no.md                Message identifiers and sequence numbers
│   ├── 02-containers-and-acks.md              msg_container, msgs_ack, batching
│   ├── 03-service-messages.md                 Every service message, and your response
│   ├── 04-salts-and-time.md                   Server salt, clock sync, future_salts
│   └── 05-reliability.md                      Resends, dedup, quick acks, ping
│
├── 06-api-layer/
│   ├── 01-init-connection.md                  invokeWithLayer + initConnection
│   ├── 02-rpc-results-and-errors.md           rpc_result, rpc_error, gzip
│   └── 03-migration-and-multi-dc.md           PHONE_MIGRATE_X, exportAuthorization
│
├── 07-login/
│   ├── 01-flow-overview.md                    The whole state machine on one page
│   ├── 02-api-credentials.md                  api_id / api_hash, test DCs
│   ├── 03-send-code.md                        auth.sendCode and code delivery types
│   ├── 04-sign-in.md                          auth.signIn, auth.signUp
│   ├── 05-two-factor-srp.md                   The SRP algorithm, in full
│   └── 06-after-login.md                      What to do with the session afterwards
│
├── 08-sending-a-message/
│   ├── 01-peers-and-access-hashes.md          The peer model
│   ├── 02-resolving-a-recipient.md            contacts.resolveUsername, users.getUsers
│   ├── 03-send-message.md                     messages.sendMessage field by field
│   └── 04-reading-updates.md                  Making sense of the Updates response
│
├── 09-advanced/
│   ├── 01-perfect-forward-secrecy.md          Temporary keys and auth.bindTempAuthKey
│   └── 02-connection-management.md            Reconnects, multiple sessions, timeouts
│
├── 10-implementation/
│   ├── 01-project-skeleton.md                 Module breakdown, shared by C and Rust
│   ├── 02-c-guide.md                          C-specific advice with OpenSSL
│   ├── 03-rust-guide.md                       Rust-specific advice
│   └── 04-testing-and-debugging.md            How to debug an encrypted protocol
│
└── appendix/
    ├── A-constructor-ids.md                   Every constructor ID you need
    ├── B-datacenters-and-keys.md              DC addresses and server RSA public keys
    ├── C-error-reference.md                   Error codes and strings, and what to do
    ├── D-tdlib-source-map.md                  Where each concept lives in this repo
    └── E-implementation-checklist.md          A tick-list before you ship
```

</details>

---

## The 60-second version

If you only read one page, read this one.

```
                    YOUR PROGRAM
                         │
    ┌────────────────────┼─────────────────────────────────┐
    │  API layer         │  telegram_api.tl objects        │
    │                    │  e.g. messages.sendMessage      │
    ├────────────────────┼─────────────────────────────────┤
    │  Session layer     │  msg_id, seq_no, salt,          │
    │                    │  containers, acks, RPC          │
    ├────────────────────┼─────────────────────────────────┤
    │  Crypto layer      │  auth_key_id ‖ msg_key ‖        │
    │                    │  AES-256-IGE( … )               │
    ├────────────────────┼─────────────────────────────────┤
    │  Transport layer   │  length-prefixed frames,        │
    │                    │  optional AES-CTR obfuscation   │
    ├────────────────────┼─────────────────────────────────┤
    │  TCP               │  149.154.167.51:443             │
    └────────────────────┴─────────────────────────────────┘
```

1. **Connect** to a datacenter (e.g. `149.154.167.51:443`, DC 2 — see
   [appendix B](appendix/B-datacenters-and-keys.md)).
2. **Choose a framing.** Send the 4-byte magic `EE EE EE EE`; from then on every packet is
   `u32 length_LE ‖ payload`. (`td/mtproto/TcpTransport.cpp:75-78`)
3. **Create an auth key.** Three unencrypted round trips (`req_pq_multi` → `req_DH_params`
   → `set_client_DH_params`) give you a shared 256-byte secret. This is *the* hard part;
   [chapter 3](03-auth-key/) walks it byte by byte.
4. **Encrypt everything after that.** Every message becomes
   `auth_key_id(8) ‖ msg_key(16) ‖ AES-256-IGE(salt ‖ session_id ‖ msg_id ‖ seq_no ‖ len ‖ body ‖ padding)`.
   (`td/mtproto/Transport.cpp:36-75`)
5. **Wrap your first API call** in `invokeWithLayer(layer, initConnection(…, query))`.
   (`td/telegram/net/MtprotoHeader.cpp:32-35`)
6. **Log in:** `auth.sendCode` → user types the code → `auth.signIn` → (maybe)
   `account.getPassword` + `auth.checkPassword`.
7. **Send:** `contacts.resolveUsername` → `messages.sendMessage`.

Everything else in this guide is detail — but in cryptographic protocols the detail *is*
the protocol. Skipping a validation step in chapter 3 turns a secure channel into a
man-in-the-middle's playground.

---

## Conventions used throughout

| Notation | Meaning |
|----------|---------|
| `a ‖ b` | byte concatenation |
| `x[i..j)` | bytes `i` (inclusive) through `j` (exclusive) of `x`, zero-based |
| `LE` / `BE` | little-endian / big-endian |
| `0x…` | hexadecimal literal |
| `int`, `long` | 32-bit and 64-bit **signed little-endian** integers |
| `int128`, `int256` | 16- and 32-byte opaque values, **not** integers, no byte swapping |
| `SHA1(x)` | 20-byte digest |
| `SHA256(x)` | 32-byte digest |
| `file.cpp:12-34` | a citation into this repository |

> **⚠ Security note boxes** flag things that are *mandatory* for security. Ignoring one
> does not make your client fail to work — it makes it insecure, silently.

> **💡 Implementation note boxes** flag things that will save you hours of debugging.

---

## A word of warning before you start

Writing an MTProto client is not hard because the cryptography is exotic — it is standard
AES, SHA and Diffie–Hellman. It is hard because:

* **There is almost no error feedback.** A wrong byte anywhere produces either silence, a
  TCP reset, or a bare `-404`. Chapter [10.4](10-implementation/04-testing-and-debugging.md)
  is about making this tractable.
* **Byte order is inconsistent by design.** TL integers are little-endian; big numbers
  (`pq`, `p`, `q`, `dh_prime`, `g_a`, `g_b`) are big-endian byte strings; the fake-TLS
  record length is big-endian. Getting this wrong is the single most common bug.
* **Several fields look like integers but are not.** `int128`/`int256` nonces are opaque
  byte strings; treating them as numbers and byte-swapping them breaks the handshake.
* **The specification is a moving target.** The API layer (currently **229**, see
  `td/telegram/Version.h:13`) changes every few weeks. The *MTProto* layer below it is
  stable and is what this guide focuses on.

Take it one chapter at a time, and validate each layer before building the next.

Start here → [00-introduction/01-what-you-are-building.md](00-introduction/01-what-you-are-building.md)
