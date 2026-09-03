# 0.2 — Glossary

[← Previous](01-what-you-are-building.md) · [Index](../README.md) · [Next: Architecture →](03-architecture.md)

---

Every term used in this guide, in alphabetical order. Come back here whenever a chapter
uses a word you have not met yet.

---

### `a` (SRP)
A fresh 2048-bit random exponent used only in the two-factor password check. Unrelated to
the DH handshake's `b`. See [7.5](../07-login/05-two-factor-srp.md).

### access hash
A 64-bit token that accompanies a user/channel id and proves you are allowed to reference
that peer. You cannot invent one; you must obtain it from a previous API response.
Access hashes are **per-account** — one user's access hash for Durov is not valid for
another account. See [8.1](../08-sending-a-message/01-peers-and-access-hashes.md).

### auth key / authorization key
The 2048-bit (256-byte) shared secret produced by the DH handshake. It is the root of all
subsequent encryption on a connection to a given datacenter. It is *not* tied to a logged-in
user until you actually log in — an auth key can exist and work perfectly while being
completely anonymous.
Produced in `td/mtproto/Handshake.cpp:228` (`handshake.gen_key()`).

### `auth_key_id`
The 64-bit identifier of an auth key, transmitted in clear at the start of every encrypted
message so the server knows which key to use.
`auth_key_id = SHA1(auth_key)[12..20)`, read as a little-endian `int64`.
(`td/mtproto/DhHandshake.cpp:225-229`)

### bare type
A TL value serialized **without** a leading 4-byte constructor id, because the type is
unambiguous from context. `int128`, `int256`, and the inner messages of a `msg_container`
are bare. Contrast **boxed**. See [1.1](../01-serialization/01-tl-language.md).

### boxed type
A TL value serialized **with** a leading 4-byte constructor id. Almost everything on the
wire is boxed. See [1.1](../01-serialization/01-tl-language.md).

### constructor id
A 4-byte tag identifying a TL constructor, equal to the CRC-32 of the normalized schema
line. E.g. `resPQ` is `0x05162463` (`td/generate/scheme/mtproto_api.tl:13`). Also called
"combinator number" or just "the magic".

### container
See **`msg_container`**.

### content-related message
A message that "counts" for sequence numbering and requires an acknowledgement. All RPC
queries are content-related; `msgs_ack`, `ping`, `http_wait`, and containers are not.
The distinction drives `seq_no` (`td/mtproto/AuthData.h:267-274`). See
[5.1](../05-session/01-msg-id-and-seq-no.md).

### datacenter (DC)
One of Telegram's server clusters, numbered 1–5 for production. Your account "lives" in
one DC (your *home* DC). Auth keys are per-DC: an auth key created with DC 2 is
meaningless to DC 4. See [2.1](../02-transport/01-datacenters.md).

### `dh_prime`
The 2048-bit safe prime `p` sent by the server in `server_DH_inner_data`. You **must**
validate it (see [3.5](../03-auth-key/05-security-checks.md)); an unvalidated `dh_prime`
lets an attacker choose a group in which discrete log is easy.

### fingerprint (RSA)
A 64-bit identifier of a server RSA public key:
`SHA1(TL_serialize(rsa_public_key(n, e)))[12..20)` as a little-endian `int64`.
(`td/mtproto/RSA.cpp:111-123`)

### flags
A 32-bit word in a TL constructor that says which optional fields follow. Written as
`flags:#` in the schema, with dependent fields written `field:flags.N?Type`.
See [1.2](../01-serialization/02-wire-encoding.md).

### `g`, `g_a`, `g_b`, `g_ab`
Diffie–Hellman values. `g` is a small generator (2–7), `g_a` is the server's public value,
`g_b` is yours, and `g_ab = g_a^b mod dh_prime` is the shared secret that becomes the auth
key.

### gzip_packed
A TL wrapper (`0x3072cfa1`) whose payload is a gzip stream containing another TL object.
The server uses it for large responses; you must transparently decompress it. See
[6.2](../06-api-layer/02-rpc-results-and-errors.md).

### IGE
"Infinite Garble Extension" — the block cipher mode MTProto uses for AES. Rarely seen
outside Telegram. Needs a 32-byte IV (two AES blocks). See
[4.3](../04-encrypted-messages/03-aes-ige.md).

### `initConnection`
The TL wrapper carrying your `api_id` and client metadata. Must be sent (inside
`invokeWithLayer`) as the outermost wrapper of the first query on every new connection, or
you get `CONNECTION_NOT_INITED`. See [6.1](../06-api-layer/01-init-connection.md).

### `invokeWithLayer`
The TL wrapper carrying the API layer number, e.g. 229. Outermost wrapper of the first
query. `invokeWithLayer#da9b0d0d {X:Type} layer:int query:!X = X;`

### layer
A version number for `telegram_api.tl`. This repository targets layer **229**
(`td/telegram/Version.h:13`). The MTProto layer beneath it is not versioned this way.

### `msg_id`
A 64-bit message identifier that doubles as a timestamp:
approximately `unix_time × 2³²`. Must be strictly increasing within a session, must have
the right parity, and must be within the server's acceptance window. See
[5.1](../05-session/01-msg-id-and-seq-no.md).

### `msg_key`
A 16-byte value transmitted in clear that both authenticates and (via the KDF) keys each
encrypted message. In MTProto 2.0 it is
`SHA256(auth_key[88+x .. 120+x) ‖ plaintext)[8..24)`. (`td/mtproto/Transport.cpp:148-164`)

### `msg_container`
A pseudo-message (`0x73f1f8dc`) holding several messages in one encrypted envelope,
to reduce round trips. See [5.2](../05-session/02-containers-and-acks.md).

### `new_nonce`
A 256-bit client secret generated during the handshake. It never appears on the wire in
clear — it is only ever sent inside the RSA-encrypted block. It seeds the temporary AES
key and the initial server salt.

### nonce / `server_nonce`
128-bit values exchanged in clear during the handshake, used to bind the three round trips
together and to prevent replay.

### PFS (perfect forward secrecy)
Optional MTProto feature where a short-lived *temporary* auth key is used for actual
traffic and bound to the permanent key via `auth.bindTempAuthKey`. See
[9.1](../09-advanced/01-perfect-forward-secrecy.md).

### `pq`
A semiprime (product of two primes) sent by the server in `resPQ` that you must factor.
It is small (≤ 63 bits in practice), and factoring it is a cheap proof-of-work, not a
security measure. See [3.2](../03-auth-key/02-step1-req-pq.md).

### quick ack
An optimization where the server confirms receipt of a message with a 4-byte token instead
of a full `msgs_ack`. Requested by setting the high bit of the transport length field.
See [5.5](../05-session/05-reliability.md).

### RPC
Remote procedure call. In MTProto, a *query* is a TL function object sent as a message
body; the *response* comes back as `rpc_result(req_msg_id, result)` or
`rpc_error(code, message)`.

### safe prime
A prime `p` such that `(p−1)/2` is also prime. `dh_prime` must be one. Verifying this
requires two Miller–Rabin tests on 2048-bit numbers.

### salt / server salt
A 64-bit value included (encrypted) in every message, rotated by the server every ~30
minutes. Sending a stale salt gets you a `bad_server_salt` with the correct value. See
[5.4](../05-session/04-salts-and-time.md).

### `seq_no`
A 32-bit per-session counter. Even values for non-content-related messages, odd for
content-related. See [5.1](../05-session/01-msg-id-and-seq-no.md).

### session
A 64-bit random `session_id` plus the associated `seq_no` counter and message history.
Distinct from a *login session* (the account-level "active session" a user sees in
Settings). Multiple MTProto sessions can share one auth key.

### TL (Type Language)
Telegram's schema language and serialization format. See [chapter 1](../01-serialization/).

### transport framing
The outermost layer that turns a byte stream into discrete packets. Telegram defines four
(Abridged, Intermediate, Padded Intermediate, Full); this repository implements two. See
[2.2](../02-transport/02-tcp-framings.md).

### `x` (the KDF offset)
`0` for client→server messages, `8` for server→client messages. Selects which half of the
auth key material is used, so the two directions never share key material.
(`td/mtproto/Transport.cpp:306` and `:385`)

---

[← Previous](01-what-you-are-building.md) · [Index](../README.md) · [Next: Architecture →](03-architecture.md)
