# 0.3 — Architecture: The Five Layers

[← Previous](02-glossary.md) · [Index](../README.md) · [Next: TL language →](../01-serialization/01-tl-language.md)

---

## 1. The layer cake

MTProto is best understood as five nested layers. Each one is a pure function of the layer
below it, which means you can — and should — implement and test them one at a time.

```
┌───────────────────────────────────────────────────────────────────────┐
│ L5  APPLICATION / API                                                 │
│     telegram_api.tl objects: auth.signIn, messages.sendMessage, …     │
│     Vocabulary: queries, results, errors, layers, flags               │
├───────────────────────────────────────────────────────────────────────┤
│ L4  SESSION                                                           │
│     msg_id, seq_no, session_id, server salt, containers, acks,        │
│     service messages, RPC correlation                                 │
├───────────────────────────────────────────────────────────────────────┤
│ L3  RECORD / CRYPTO                                                   │
│     auth_key_id ‖ msg_key ‖ AES-256-IGE(payload)                      │
│     (or, before a key exists: auth_key_id = 0, plaintext)             │
├───────────────────────────────────────────────────────────────────────┤
│ L2  TRANSPORT FRAMING                                                 │
│     length-prefixed packets; optional AES-CTR obfuscation;            │
│     optional fake-TLS wrapping                                        │
├───────────────────────────────────────────────────────────────────────┤
│ L1  TCP (or HTTP, or WebSocket)                                       │
└───────────────────────────────────────────────────────────────────────┘
```

Cross-cutting all of them: **L0, serialization (TL)**. TL is used to encode both the
low-level MTProto service messages (schema: `td/generate/scheme/mtproto_api.tl`) and the
high-level API (schema: `td/generate/scheme/telegram_api.tl`). It is not a "layer" in the
stack sense — it is the alphabet the other layers are written in.

---

## 2. Walking a message down the stack

Suppose you want to call `help.getConfig()`. Here is what happens to those bytes.

### L5 → L4: serialize the query

`help.getConfig#c4f9186b = Config;` has no parameters, so it serializes to exactly four
bytes:

```
6B 18 F9 C4                    ← 0xc4f9186b, little-endian
```

If this is the first query on the connection, it gets wrapped (see
[6.1](../06-api-layer/01-init-connection.md)):

```
0D 0D 9B DA                    ← invokeWithLayer#da9b0d0d
E5 00 00 00                    ← layer = 229
A9 5E CD C1                    ← initConnection#c1cd5ea9
02 00 00 00                    ← flags (bit 1 = params present)
…                              ← api_id, device_model, …, params
6B 18 F9 C4                    ← the actual query
```

Call this blob `body`, of length `len`.

### L4: wrap in a message

The session layer assigns a `msg_id` and a `seq_no`:

```
msg_id  = 0x67D9F1A400000000-ish   (unix_time × 2³², low bits fiddled)
seq_no  = 1                        (content-related → odd)
```

and produces the *plaintext payload*:

```
┌────────────┬────────────┬────────────┬─────────┬──────────┬───────────┬─────────┐
│ salt       │ session_id │ msg_id     │ seq_no  │ len      │ body      │ padding │
│ 8 bytes    │ 8 bytes    │ 8 bytes    │ 4 bytes │ 4 bytes  │ len bytes │ 12–1024 │
└────────────┴────────────┴────────────┴─────────┴──────────┴───────────┴─────────┘
```

(`td/mtproto/Transport.cpp:36-75`)

### L4 → L3: encrypt

```
msg_key = SHA256( auth_key[88 .. 120) ‖ plaintext )[8..24)
aes_key, aes_iv = KDF2(auth_key, msg_key, x=0)
ciphertext = AES-256-IGE-encrypt(plaintext, aes_key, aes_iv)
```

giving the *encrypted message*:

```
┌──────────────┬──────────┬──────────────┐
│ auth_key_id  │ msg_key  │ ciphertext   │
│ 8 bytes      │ 16 bytes │ n × 16 bytes │
└──────────────┴──────────┴──────────────┘
```

(`td/mtproto/Transport.cpp:148-164`, `td/mtproto/KDF.cpp:74-104`)

### L3 → L2: frame it

With the Intermediate framing:

```
┌───────────────────┬────────────────────┐
│ u32 length LE     │ encrypted message  │
└───────────────────┴────────────────────┘
```

(`td/mtproto/TcpTransport.cpp:50-73`)

If obfuscation is enabled, the whole byte stream (including this length field) is then
AES-CTR encrypted, and the connection began with a 64-byte obfuscation header.
(`td/mtproto/TcpTransport.cpp:80-143`)

### L2 → L1: `write()` to the socket.

Coming back up is the same in reverse, with validation at every step.

---

## 3. What lives where — a decision table

When you are debugging and something is wrong, this table tells you which layer to look at.

| Symptom | Layer | Likely cause |
|---------|-------|--------------|
| Connection resets immediately after connect | L2 | Wrong transport magic, or first obfuscation byte is a forbidden value |
| You receive a 4-byte packet containing `-404` | L3 | Server does not recognize your `auth_key_id` (key destroyed or never existed) |
| You receive a 4-byte packet containing `-429` | L2 | Rate-limited by the transport |
| Decryption produces garbage / `msg_key` mismatch | L3 | Wrong `x` offset, wrong KDF, wrong IGE implementation |
| `bad_msg_notification` code 16/17/20 | L4 | Your `msg_id` is outside the server's time window — sync your clock |
| `bad_msg_notification` code 18 | L4 | `msg_id` not divisible by 4 |
| `bad_msg_notification` code 32–35 | L4 | `seq_no` parity/ordering wrong |
| `bad_server_salt` | L4 | Normal; adopt the new salt and resend |
| `rpc_error(400, "CONNECTION_NOT_INITED")` | L5 | You forgot `invokeWithLayer`+`initConnection` on this connection |
| `rpc_error(400, "AUTH_KEY_UNREGISTERED")` (401) | L5 | Key is valid but not attached to a logged-in user |
| `rpc_error(303, "PHONE_MIGRATE_4")` | L5 | Wrong DC; redo everything on DC 4 |

Full list: [appendix C](../appendix/C-error-reference.md).

---

## 4. Two kinds of messages: plaintext and encrypted

The very first bytes of every MTProto message are an 8-byte `auth_key_id`. Its value
selects the message format:

| `auth_key_id` | Format | Used for |
|---------------|--------|----------|
| `0` | **Unencrypted**: `0(8) ‖ msg_id(8) ‖ length(4) ‖ body ‖ padding` | Only the three DH handshake round trips |
| non-zero | **Encrypted**: `auth_key_id(8) ‖ msg_key(16) ‖ AES-IGE(…)` | Everything else |

TDLib decides between them with a single test:
`packet_info->no_crypto_flag = as<int64>(message.begin()) == 0;`
(`td/mtproto/Transport.cpp:439`)

> **⚠ Security note.** Once you have an auth key, you must **never** accept an
> unencrypted message from the server again. A client that processes plaintext messages
> unconditionally can be fed forged service messages by anyone who can inject packets.
> Restrict the plaintext path to the handshake state machine only.

---

## 5. Connections, sessions, and auth keys — three different lifetimes

These three concepts are frequently conflated. They are not the same.

```
   auth key (per DC, persistent, survives restarts)
   └── session (random 64-bit id, holds seq_no and msg_id history)
       └── connection (one TCP socket)
```

* **Auth key** — created once per datacenter by the DH handshake. Persist it to disk. A
  desktop client typically keeps one for years.
* **Session** — a `session_id` plus the `seq_no` counter and the set of messages you have
  sent. If you lose track of your state (e.g. after a crash), start a *new* session rather
  than reusing a `seq_no`. The server tells you it has forgotten a session with
  `new_session_created`.
* **Connection** — one TCP socket. A session survives reconnects: you may open a new
  socket and keep the same `session_id` and `seq_no` counter. But each new *connection*
  needs the `initConnection` header again
  (`td/telegram/net/MtprotoHeader.cpp`, and the `need_header` logic at
  `td/mtproto/AuthData.h:169-195`).

A minimal client needs exactly one of each. TDLib runs several sessions per DC in parallel
(one for updates, several for uploads/downloads) — you do not need to.

---

## 6. Request/response is not the whole story

MTProto is **not** request/response. It is a bidirectional message stream on which an RPC
protocol is layered. At any moment the server may send you, unprompted:

* `new_session_created` — it lost your session,
* `bad_server_salt` — your salt expired,
* `msgs_ack` — confirmation of messages you sent,
* `msg_detailed_info` / `msg_new_detailed_info` — "I have an answer for you",
* `updateShort…` / `updates` — someone sent you a message,
* `pong` — reply to your keepalive.

Your read loop must therefore be a **dispatcher**, not a `read_response()` function.
Design for this from the start; retrofitting it is painful. The concrete list of everything
you can receive and what to do about it is [chapter 5.3](../05-session/03-service-messages.md).

---

## 7. Suggested module structure

Mapping the layers onto source files in your own project:

```
src/
├── tl/            L0   serializer, deserializer, generated schema bindings
├── crypto/        —    AES-IGE, AES-CTR, SHA1, SHA256, PBKDF2, bignum wrappers,
│                       factorization
├── transport/     L2   framing (intermediate), obfuscation, socket I/O
├── record/        L3   encrypt/decrypt an MTProto message
├── handshake/     L3   the three-step DH handshake
├── session/       L4   msg_id/seq_no, containers, acks, service messages, RPC map
└── api/           L5   login flow, sending messages
```

Chapter [10.1](../10-implementation/01-project-skeleton.md) expands this with the exact
functions each module should expose.

---

[← Previous](02-glossary.md) · [Index](../README.md) · [Next: TL language →](../01-serialization/01-tl-language.md)
