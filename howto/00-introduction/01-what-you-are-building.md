# 0.1 — What You Are Building

[← Back to index](../README.md) · [Next: Glossary →](02-glossary.md)

---

## 1. The problem

Telegram's servers do not speak HTTP+JSON, gRPC, or any other off-the-shelf protocol.
They speak **MTProto** — a custom binary protocol that bundles together:

* a **serialization format** (TL — "Type Language"),
* a **key agreement protocol** (Diffie–Hellman over a 2048-bit safe prime, with the
  server's half authenticated by RSA),
* a **record layer** (AES-256 in IGE mode with per-message keys),
* a **session/reliability layer** (message identifiers, sequence numbers, acknowledgements,
  containers, salts),
* a **request/response layer** (RPC over the above),
* and a **transport framing** with optional traffic obfuscation.

There is no TLS anywhere in the mandatory path. MTProto is its own transport security
protocol, running directly over a raw TCP socket (or HTTP, or UDP for calls).

TDLib — the library in this repository — implements all of the above and then adds a very
large amount of client-side business logic (a local database, update reconciliation, file
downloads, a high-level API). This guide throws away everything above the RPC layer and
tells you how to do the bottom five layers yourself.

---

## 2. What "sending one message" actually requires

Newcomers usually imagine this:

```
connect() → login() → send("hi") → done
```

The reality is closer to this:

```
1.  TCP connect to a datacenter                             ← chapter 2
2.  Send a 4-byte transport magic                           ← chapter 2
3.  req_pq_multi(nonce)                        [plaintext]  ← chapter 3
4.  ← resPQ(nonce, server_nonce, pq, fingerprints)
5.  Factor pq into p·q                                      ← chapter 3
6.  req_DH_params(…, RSA_PAD(p_q_inner_data)) [plaintext]
7.  ← server_DH_params_ok(encrypted_answer)
8.  AES-IGE-decrypt → server_DH_inner_data(g, dh_prime, g_a)
9.  Validate dh_prime is a safe prime, validate g_a          ← chapter 3 (MANDATORY)
10. b = random(2048); g_b = g^b mod dh_prime
11. set_client_DH_params(…, AES-IGE(client_DH_inner_data))  [plaintext]
12. ← dh_gen_ok(new_nonce_hash1)
13. auth_key = g_a^b mod dh_prime    (256 bytes)             ← chapter 3
    auth_key_id = SHA1(auth_key)[12..20)
    server_salt = new_nonce[0..8) XOR server_nonce[0..8)
--- everything from here on is encrypted -------------------- chapter 4
14. session_id = random(8)
15. invokeWithLayer(229, initConnection(api_id, …,           ← chapter 6
        help.getConfig()))
16. ← config(dc_options, …)   (also: new_session_created, salts)  ← chapter 5
17. auth.sendCode(phone, api_id, api_hash, codeSettings)     ← chapter 7
18. ← auth.sentCode(type, phone_code_hash, …)
    …or rpc_error("PHONE_MIGRATE_4") → reconnect to DC 4 and
      redo the ENTIRE handshake there                       ← chapter 6
19. (user reads the code)
20. auth.signIn(flags, phone, phone_code_hash, code)
21. ← auth.authorization(user)
    …or rpc_error(401, "SESSION_PASSWORD_NEEDED") →
      account.getPassword → SRP → auth.checkPassword        ← chapter 7
--- you are now logged in -----------------------------------
22. contacts.resolveUsername(flags, "durov")                 ← chapter 8
23. ← contacts.resolvedPeer(peer, chats, users)
24. messages.sendMessage(flags, inputPeerUser(id, access_hash),
        "Hello, world", random_id)
25. ← updateShortSentMessage(id, pts, date)
```

Plus, running concurrently the whole time:

* acknowledging every message the server sends you (`msgs_ack`), or the server will keep
  resending them;
* answering `bad_server_salt` by retrying with the new salt;
* answering `bad_msg_notification` (usually: your clock is wrong — fix your `msg_id`
  generation and resend);
* keeping the connection alive with `ping_delay_disconnect`;
* handling `new_session_created` (the server dropped your session; refetch state).

That is the scope of this guide.

---

## 3. Why not just use a library?

You should, if you can. There are mature MTProto implementations in most languages. Write
your own if:

* you are learning,
* you need an implementation with an auditable, minimal dependency surface,
* you are building tooling (traffic analysis, protocol fuzzing, an alternative server),
* the existing library in your language is unmaintained or does not expose what you need.

Do **not** write your own if you plan to ship a production client and you have not read
chapter [3.5 (security checks)](../03-auth-key/05-security-checks.md) carefully. An
MTProto implementation that skips the DH parameter validation is trivially
man-in-the-middleable by a hostile network operator, and it will look like it works
perfectly.

---

## 4. What is deliberately out of scope

| Topic | Why it is excluded |
|-------|--------------------|
| Secret chats (end-to-end encryption) | A separate protocol layered on top; see `td/mtproto/Transport.cpp:77-107` (`EndToEndHeader`) and MTProto's `mtproto/description_v2` |
| Voice/video calls | Uses a different transport entirely |
| File upload/download and CDN DCs | Same MTProto, different methods (`upload.*`); chapter 6.3 covers the DC-switching part you would need |
| The TDLib high-level API | This guide is about *replacing* it |
| MTProto 1.0 | Obsolete. TDLib always sends `version = 2` (`td/mtproto/RawConnection.cpp` sets `PacketInfo::version = 2`). The v1 code paths are retained only for secret chats with old clients. |

MTProto 1.0's SHA-1-based `msg_key` construction is described briefly in chapter
[4.2](../04-encrypted-messages/02-msg-key-and-kdf.md) for completeness only — **do not
implement it**.

---

## 5. Ground rules for your implementation

Four rules that will save you the most time:

1. **Build bottom-up and test each layer.** Do not attempt the login flow until you can
   reliably complete a handshake and get a `pong` back from a `ping`. The single best
   milestone is: *encrypted `help.getConfig` returns a `config` object*.
2. **Log raw hex at every layer boundary.** You will need to see the exact bytes going in
   and out of the encryption. See
   [10.4](../10-implementation/04-testing-and-debugging.md).
3. **Never reuse randomness.** `nonce`, `new_nonce`, `b`, `a` (SRP), `session_id`,
   `random_id`, padding — all must come from a CSPRNG, fresh, every time.
4. **Treat big integers as big-endian byte strings and TL integers as little-endian.**
   Write two clearly-named helpers and never mix them up.

---

## 6. The reference implementation map

Whenever this guide says "TDLib does X", these are the files it means:

| Concern | Files |
|---------|-------|
| Transport framing, obfuscation | `td/mtproto/TcpTransport.{h,cpp}`, `td/mtproto/IStreamTransport.{h,cpp}` |
| Message envelope, encryption | `td/mtproto/Transport.{h,cpp}`, `td/mtproto/KDF.{h,cpp}` |
| DH handshake | `td/mtproto/Handshake.{h,cpp}`, `td/mtproto/DhHandshake.{h,cpp}`, `td/mtproto/RSA.{h,cpp}` |
| Session state | `td/mtproto/AuthData.{h,cpp}`, `td/mtproto/SessionConnection.{h,cpp}` |
| Message assembly | `td/mtproto/CryptoStorer.h`, `td/mtproto/NoCryptoStorer.h`, `td/mtproto/PacketStorer.h` |
| MTProto schema | `td/generate/scheme/mtproto_api.tl` |
| API schema | `td/generate/scheme/telegram_api.tl` |
| Login state machine | `td/telegram/AuthManager.{h,cpp}` |
| 2FA / SRP | `td/telegram/PasswordManager.cpp` |
| Bootstrap DCs, RSA keys | `td/telegram/net/ConnectionCreator.cpp`, `td/telegram/net/PublicRsaKeySharedMain.cpp` |
| `initConnection` header | `td/telegram/net/MtprotoHeader.cpp` |
| Crypto primitives | `tdutils/td/utils/crypto.{h,cpp}` |

A fuller index is in [appendix D](../appendix/D-tdlib-source-map.md).

---

[← Back to index](../README.md) · [Next: Glossary →](02-glossary.md)
