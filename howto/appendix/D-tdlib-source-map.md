# Appendix D — TDLib Source Map

[← Previous](C-error-reference.md) · [Index](../README.md) · [Next: Appendix E →](E-implementation-checklist.md)

---

Where each part of this guide came from. Use this when you want to check a claim, or when
you hit behaviour this guide does not cover and need to read the reference implementation
yourself.

Line numbers are for the state of this repository at layer **229**
(`td/telegram/Version.h:13`). They drift; the function names are stable.

---

## D.1 — Directory layout

| Directory | Contains |
|-----------|----------|
| `td/mtproto/` | **The protocol itself.** Handshake, crypto, transport, session framing. Independent of the Telegram API |
| `td/telegram/` | The API layer: login, messages, updates, business logic |
| `td/telegram/net/` | Network plumbing: DC selection, query dispatch, retries |
| `td/generate/scheme/` | `.tl` schema files — the authoritative wire format |
| `td/generate/tl-parser/` | The TL parser, including constructor-id computation |
| `tdutils/td/utils/` | Crypto primitives, big numbers, TL storers/parsers |
| `tdnet/td/net/` | TCP, TLS, proxies |

If you read only one directory, read `td/mtproto/`. It is ~6000 lines and is the whole
protocol.

---

## D.2 — Chapter → source

### Chapter 1 — Serialization

| Topic | Source |
|-------|--------|
| Constructor-id computation | `td/generate/tl-parser/tl-parser.c:1468-1480` |
| CRC-32 implementation | `td/generate/tl-parser/crc32.c` |
| int/long/double storing | `tdutils/td/utils/tl_storers.h:28-32` |
| String/bytes storing | `tdutils/td/utils/tl_storers.h:52-88` |
| Length calculation | `tdutils/td/utils/tl_storers.h:124-136` |
| Parsing | `tdutils/td/utils/tl_parsers.h:126-162` |
| Flag validation | `tdutils/td/utils/tl_helpers.h:47-58` |
| MTProto schema | `td/generate/scheme/mtproto_api.tl` (87 lines) |
| API schema | `td/generate/scheme/telegram_api.tl` |

### Chapter 2 — Transport

| Topic | Source |
|-------|--------|
| Intermediate/padded framing | `td/mtproto/TcpTransport.cpp:20-78` |
| Read path, quick-ack bit | `td/mtproto/TcpTransport.cpp:20-48` |
| Write path, size checks | `td/mtproto/TcpTransport.cpp:50-73` |
| Obfuscation header | `td/mtproto/TcpTransport.cpp:80-143` |
| Max packet size | `td/mtproto/RawConnection.cpp:165-172` |
| Fake-TLS | `td/mtproto/TlsInit.cpp`, `tdnet/td/net/TlsReaderByteFlow.cpp:15-37` |
| Proxy secret formats | `td/mtproto/ProxySecret.h:34-50` |
| Bootstrap DC addresses | `td/telegram/net/ConnectionCreator.cpp:1226, 1244-1279` |

> **Note.** TDLib implements **only** Intermediate and PaddedIntermediate. Abridged and
> Full appear in the public docs but not in this codebase.

### Chapter 3 — Auth key

| Topic | Source |
|-------|--------|
| Handshake driver | `td/mtproto/Handshake.cpp:315-322` (`on_start`) |
| `req_pq_multi` → `resPQ` | `td/mtproto/Handshake.cpp:86-161` (`on_res_pq`) |
| RSA_PAD | `td/mtproto/Handshake.cpp:127-155` |
| `server_DH_params` | `td/mtproto/Handshake.cpp:163-254` |
| `dh_gen_*` | `td/mtproto/Handshake.cpp:256-285` |
| DH parameter validation | `td/mtproto/DhHandshake.cpp:21-126` |
| Key derivation, `auth_key_id` | `td/mtproto/DhHandshake.cpp:207, 225-229` |
| PQ factorization | `tdutils/td/utils/crypto.cpp:104-141, 172-231, 233-257` |
| Big-endian encoding | `tdutils/td/utils/crypto.cpp:158-170` |
| RSA key loading and use | `td/mtproto/RSA.cpp:111-155` |
| RSA public keys | `td/telegram/net/PublicRsaKeySharedMain.cpp:14-52` |
| `tmp_KDF` (SHA-1) | `td/mtproto/KDF.cpp:52-72` |

### Chapter 4 — Encrypted messages

| Topic | Source |
|-------|--------|
| Header layouts | `td/mtproto/Transport.cpp:36-75` |
| `msg_key` computation | `td/mtproto/Transport.cpp:148-164` |
| `X` = 0 (client), 8 (server) | `td/mtproto/Transport.cpp:385, 306` |
| Padding size buckets | `td/mtproto/Transport.cpp:167-185` |
| Receive-side validation | `td/mtproto/Transport.cpp:214-300` |
| Short-frame errors | `td/mtproto/Transport.cpp:418-437` |
| KDF2 (SHA-256) | `td/mtproto/KDF.cpp:74-104` |
| KDF v1 (SHA-1) | `td/mtproto/KDF.cpp:17-50` |
| AES-IGE | `tdutils/td/utils/crypto.cpp:467-557` |
| AES-CTR | `tdutils/td/utils/crypto.cpp:667-723` |
| Plaintext padding | `td/mtproto/NoCryptoStorer.h:20-25` |

### Chapter 5 — Session

| Topic | Source |
|-------|--------|
| `msg_id` generation | `td/mtproto/AuthData.cpp:107-125` |
| `msg_id` validity windows | `td/mtproto/AuthData.cpp:127-137` |
| Inbound packet checks | `td/mtproto/AuthData.cpp:139-167` |
| Time-difference latch | `td/mtproto/AuthData.cpp:70-83`; rationale at `AuthData.h:214-215` |
| Time-difference reset | `td/mtproto/AuthData.cpp:85-89` |
| `seq_no` | `td/mtproto/AuthData.h:267-274` |
| Salt handling | `td/mtproto/AuthData.h:225-235`, `AuthData.cpp:91-99` |
| Future-salt requests | `td/mtproto/SessionConnection.cpp:949-956` |
| `session_id` generation | `td/telegram/net/Session.cpp:261-265` |
| Container construction | `td/mtproto/CryptoStorer.h:188-203` |
| Ack scheduling | `td/mtproto/SessionConnection.cpp:512-514` |
| `rpc_result` dispatch | `td/mtproto/SessionConnection.cpp:239-278` |
| `new_session_created` | `td/mtproto/SessionConnection.cpp:309-322` |
| `bad_msg_notification` | `td/mtproto/SessionConnection.cpp:324-387` |
| `bad_server_salt` | `td/mtproto/SessionConnection.cpp:389-397` |

### Chapter 6 — API layer

| Topic | Source |
|-------|--------|
| `initConnection` header | `td/telegram/MtprotoHeader.cpp:28-101` |
| Anonymous mode | `td/telegram/MtprotoHeader.cpp:49-55, 76` |
| `CONNECTION_NOT_INITED` recovery | `td/telegram/net/Session.cpp:970-974` |
| Outgoing gzip | `td/telegram/net/NetQueryCreator.cpp:64-102` |
| Pre-auth allowed queries | `td/telegram/net/NetQueryCreator.cpp:78-79` |
| Migration | `td/telegram/net/NetQueryDispatcher.cpp:391-418` |
| Retry delays | `td/telegram/net/NetQueryDelayer.cpp:30-75` |
| Cross-DC authorization | `td/telegram/DcAuthManager.cpp:145-195` |
| Current layer | `td/telegram/Version.h:13` |

### Chapter 7 — Login

| Topic | Source |
|-------|--------|
| Login state machine | `td/telegram/AuthManager.cpp` |
| `auth.sendCode` | `td/telegram/AuthManager.cpp:802` |
| `auth.signIn` flags | `td/telegram/AuthManager.cpp:805-813` |
| `SESSION_PASSWORD_NEEDED` | `td/telegram/AuthManager.cpp:1577` |
| `auth.checkPassword` | `td/telegram/AuthManager.cpp:872` |
| SRP hashing | `td/telegram/PasswordManager.cpp:37-51` |
| SRP proof | `td/telegram/PasswordManager.cpp:73-142` |
| Test accounts | `test/tdclient.cpp:676-677` |

### Chapter 8 — Sending a message

| Topic | Source |
|-------|--------|
| Peer handling | `td/telegram/DialogId.cpp`, `td/telegram/UserId.h` |
| Username resolution | `td/telegram/MessagesManager.cpp` |
| Message sending | `td/telegram/MessagesManager.cpp` |
| Update processing | `td/telegram/UpdatesManager.cpp` |
| Schema | `td/generate/scheme/telegram_api.tl:2513-2521, 2772-2773` |

### Chapter 9 — Advanced

| Topic | Source |
|-------|--------|
| Temp-key validity | `td/telegram/net/Session.cpp:1445` |
| Bind blob construction | `td/mtproto/SessionConnection.cpp:867-889` |
| `auth.bindTempAuthKey` dispatch | `td/telegram/net/Session.cpp:1364-1390` |
| Connection creation | `td/telegram/net/ConnectionCreator.cpp` |
| Ping/timeout constants | `td/mtproto/SessionConnection.h` |

---

## D.3 — The twenty files that matter

If you are porting the protocol, read these in order:

1. `td/generate/scheme/mtproto_api.tl` — 87 lines, the whole protocol schema
2. `tdutils/td/utils/tl_storers.h` — serialization
3. `tdutils/td/utils/tl_parsers.h` — deserialization
4. `td/mtproto/TcpTransport.cpp` — framing and obfuscation
5. `td/mtproto/Handshake.cpp` — auth key creation
6. `td/mtproto/DhHandshake.cpp` — DH validation
7. `td/mtproto/RSA.cpp` — RSA and fingerprints
8. `tdutils/td/utils/crypto.cpp` — AES-IGE, AES-CTR, factorization
9. `td/mtproto/KDF.cpp` — key derivation
10. `td/mtproto/Transport.cpp` — the encrypted envelope
11. `td/mtproto/AuthData.cpp` / `.h` — `msg_id`, `seq_no`, salts, time
12. `td/mtproto/CryptoStorer.h` — containers
13. `td/mtproto/SessionConnection.cpp` — service messages
14. `td/telegram/MtprotoHeader.cpp` — `initConnection`
15. `td/telegram/net/Session.cpp` — session lifecycle
16. `td/telegram/net/NetQueryDispatcher.cpp` — routing and migration
17. `td/telegram/net/NetQueryDelayer.cpp` — retry policy
18. `td/telegram/AuthManager.cpp` — login
19. `td/telegram/PasswordManager.cpp` — SRP
20. `td/telegram/net/ConnectionCreator.cpp` — DC addresses

---

## D.4 — Reading the schema

`td/generate/scheme/mtproto_api.tl` is 87 lines and defines everything below the API
layer. Worth reading in full — it is shorter than this appendix.

`td/generate/scheme/telegram_api.tl` is enormous. Navigate it with grep:

```bash
grep -n "^auth\." td/generate/scheme/telegram_api.tl
grep -n "^messages.sendMessage" td/generate/scheme/telegram_api.tl
grep -n -- "---functions---" td/generate/scheme/telegram_api.tl
```

Everything before `---functions---` is a type; everything after is a callable method.

> **⚠ Always verify a line number before relying on it.** The schema changes every layer.
> Grep for the constructor name rather than trusting a number — including the numbers in
> this document.

---

## D.5 — Things TDLib does that the docs do not mention

Collected differences between this codebase and the public protocol description. Each is
covered in the relevant chapter.

| Behaviour | Where |
|-----------|-------|
| Only Intermediate and PaddedIntermediate framings exist | [2.2](../02-transport/02-tcp-framings.md) |
| `req_pq` (non-multi) is absent from the schema | [3.2](../03-auth-key/02-step1-req-pq.md) |
| `server_DH_params_fail` is absent from the schema | [3.3](../03-auth-key/03-step2-req-dh-params.md) |
| Padding is bucketed, not merely 12-1024 bytes | [4.1](../04-encrypted-messages/01-envelope.md) |
| `server_time_difference` only ever increases | [5.4](../05-session/04-salts-and-time.md) |
| Container inner `body` is `string`, not `Object` | [5.2](../05-session/02-containers-and-acks.md) |
| `new_session_created.server_salt` is ignored | [5.3](../05-session/03-service-messages.md) |
| `gzip_packed` can appear inside `rpc_result` | [5.3](../05-session/03-service-messages.md) |
| `initConnection` has an undocumented flag bit 10 | [6.1](../06-api-layer/01-init-connection.md) |
| `tz_offset` is injected into the params JSON | [6.1](../06-api-layer/01-init-connection.md) |
| The bind blob uses MTProto **v1** (SHA-1) KDF | [9.1](../09-advanced/01-perfect-forward-secrecy.md) |
| SRP `k` uses unpadded `p` but padded `g` | [7.5](../07-login/05-two-factor-srp.md) |

---

## D.6 — Building TDLib as a reference

You do not need to build TDLib to use this guide, but a working build gives you a
known-good implementation to compare against:

```bash
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build . --target tdjson -j$(nproc)
```

Then run with verbose MTProto logging to see the exact packets a correct client sends:

```
td_set_log_verbosity_level(5)
```

The `mtproto` log category (`VLOG(mtproto)` throughout `td/mtproto/`) prints message ids,
sequence numbers, and constructor names. Comparing your trace against TDLib's is the
fastest way to find a framing bug. See
[10.4](../10-implementation/04-testing-and-debugging.md).

---

## D.7 — Checklist

- [ ] You know which directory owns each layer of the protocol
- [ ] You can find a constructor in the schema by name, not line number
- [ ] You have read `mtproto_api.tl` in full (87 lines)
- [ ] You know where to look when this guide is silent
- [ ] You have a TDLib build available for comparison, or know how to get one

---

[← Previous](C-error-reference.md) · [Index](../README.md) · [Next: Appendix E →](E-implementation-checklist.md)
