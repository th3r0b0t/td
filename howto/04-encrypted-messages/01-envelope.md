# 4.1 — The Encrypted Message Envelope

[← Previous](../03-auth-key/06-worked-example.md) · [Index](../README.md) · [Next: msg_key and the KDF →](02-msg-key-and-kdf.md)

---

You have an auth key. Every message from now on travels inside this envelope.

---

## 1. Layout

From `struct CryptoHeader` and `struct CryptoPrefix`
(`td/mtproto/Transport.cpp:36-75`):

```
┌─────────────────────────────────────────────────────────────────────────┐
│ PLAINTEXT (24 bytes)                                                    │
├──────────────────────────┬──────────────────────────────────────────────┤
│ auth_key_id    8 bytes   │ msg_key            16 bytes                  │
└──────────────────────────┴──────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────────────────┐
│ AES-256-IGE ENCRYPTED (multiple of 16 bytes)                            │
├───────────┬───────────┬──────────┬─────────┬─────────┬──────┬───────────┤
│ salt      │ session_id│ msg_id   │ seq_no  │ length  │ body │ padding   │
│ 8 bytes   │ 8 bytes   │ 8 bytes  │ 4 bytes │ 4 bytes │ N    │ 12..1024  │
└───────────┴───────────┴──────────┴─────────┴─────────┴──────┴───────────┘
 └──────── encrypted_header ───────┘└────── CryptoPrefix ─────┘
```

In C terms:

```c
struct CryptoHeader {
  uint64  auth_key_id;
  uint8   message_key[16];
  /* --- encryption starts here --- */
  uint64  salt;
  uint64  session_id;
};
struct CryptoPrefix {
  uint64  msg_id;
  uint32  seq_no;
  uint32  message_data_length;
};
```

`encrypt_begin()` (`Transport.cpp:55-57`) returns `&salt` — encryption covers everything
from `salt` to the end of the padding.

---

## 2. Field reference

| Field | Size | Endianness | Meaning |
|-------|------|------------|---------|
| `auth_key_id` | 8 | LE (raw bytes) | `SHA1(auth_key)[12..20)`. Tells the server which key to use. |
| `msg_key` | 16 | raw | `SHA256(key_fragment ‖ plaintext)[8..24)`. See [4.2](02-msg-key-and-kdf.md). |
| `salt` | 8 | LE int64 | Current server salt. See [5.4](../05-session/04-salts-and-time.md). |
| `session_id` | 8 | LE int64 | Random non-zero, constant for the session. See [5.1](../05-session/01-msg-id-and-seq-no.md). |
| `msg_id` | 8 | LE uint64 | Time-based, unique, ordered. See [5.1](../05-session/01-msg-id-and-seq-no.md). |
| `seq_no` | 4 | LE uint32 | Content-related message counter. See [5.1](../05-session/01-msg-id-and-seq-no.md). |
| `message_data_length` | 4 | LE uint32 | Byte length of `body`, **excluding** padding. |
| `body` | N | — | TL-serialized object |
| `padding` | 12..1024 | — | Random. **Minimum 12 bytes** in MTProto 2.0. |

> **💡 Implementation note.** The plaintext part is 24 bytes and the encrypted part starts
> at offset 24. All the multi-byte integers are little-endian, matching the rest of TL.
> `int128`-style fields (`msg_key`) are opaque byte arrays.

---

## 3. Padding

`do_calc_crypto_size2_basic` (`Transport.cpp:167-179`):

```cpp
size_t do_calc_crypto_size2_basic(size_t data_size, size_t enc_size, size_t raw_size) {
  size_t encrypted_size = (enc_size + data_size + 12 + 15) & ~15;

  std::array<size_t, 10> sizes{{64, 128, 192, 256, 384, 512, 768, 1024, 1280}};
  for (auto size : sizes) {
    if (encrypted_size <= size) {
      return raw_size + size;
    }
  }
  encrypted_size = (encrypted_size - 1280 + 447) / 448 * 448 + 1280;
  return raw_size + encrypted_size;
}
```

With `enc_size = 16` (salt + session_id) and `raw_size = 24` (auth_key_id + msg_key), where
`data_size` is `msg_id ‖ seq_no ‖ length ‖ body` = `16 + body_len`:

1. **Minimum:** at least 12 bytes of padding, then round the encrypted part up to a
   multiple of 16.
2. **Bucketing:** round the encrypted size up to the next value in
   `{64, 128, 192, 256, 384, 512, 768, 1024, 1280}`.
3. **Above 1280:** round up to `1280 + 448·k`.

The final packet size is `24 + encrypted_size`.

### Why bucket?

Without bucketing, the ciphertext length reveals the plaintext length to within 16 bytes.
Bucketing coarsens that: any message whose encrypted content is between 193 and 256 bytes
looks identical on the wire. This is traffic analysis resistance, not confidentiality.

### The random-padding variant

`do_calc_crypto_size2_rand` (`Transport.cpp:181-185`), used when
`packet_info->use_random_padding` is set:

```cpp
size_t rand_data_size = Random::secure_uint32() & 0xff;      // 0..255
size_t encrypted_size = (enc_size + data_size + rand_data_size + 12 + 15) & ~15;
```

Instead of deterministic buckets, add 0–255 random bytes then round to 16. Either scheme is
acceptable to the server.

> **💡 Implementation note.** The *simplest correct* implementation is:
> `pad_len = 12 + ((-(16 + body_len + 12)) mod 16)`, i.e. 12 to 27 bytes.
> That satisfies the server's `12 ≤ pad ≤ 1024` check. Add bucketing later if you care
> about traffic analysis.

The padding must be **random** (`Random::secure_bytes(pad)`, `Transport.cpp:352`), not
zeros — in MTProto 2.0 the padding is covered by `msg_key`, so predictable padding weakens
the MAC's effective randomization.

---

## 4. Building an outgoing message

`Transport::write_crypto` (`Transport.cpp:368-388`) plus
`write_crypto_impl` (`Transport.cpp:343-366`):

```
1.  data = msg_id ‖ seq_no ‖ message_data_length ‖ body
2.  padded_size = calc_crypto_size2(len(data))
3.  Lay out the buffer:
        [0..8)    auth_key_id
        [8..24)   msg_key            (filled in at step 6)
        [24..32)  salt
        [32..40)  session_id
        [40..)    data
        then      random padding to padded_size
4.  to_encrypt = buffer[24 .. padded_size]        # salt onwards
5.  (quick_ack_token, msg_key) = calc_message_key2(auth_key, X = 0, to_encrypt)
6.  buffer[8..24) = msg_key
7.  (aes_key, aes_iv) = KDF2(auth_key, msg_key, X = 0)
8.  buffer[24 .. padded_size] = AES256_IGE_encrypt(to_encrypt, aes_key, aes_iv)
9.  send(buffer)
```

Note step 5 happens **before** step 8 — the `msg_key` is computed over the *plaintext*.
This is MTProto's "MAC-then-encrypt" arrangement.

**`X = 0` for client→server.** `write_crypto` calls
`write_crypto_impl(0, …)` (`Transport.cpp:385`). See [4.2 §3](02-msg-key-and-kdf.md).

---

## 5. Parsing an incoming message

`Transport::read_crypto_impl` (`Transport.cpp:214-300`), condensed:

```
1.  if len(message) < 24 + 16: error "too small"
2.  if message[0..8) != auth_key_id: error "auth key id mismatch"
3.  msg_key = message[8..24)
4.  (aes_key, aes_iv) = KDF2(auth_key, msg_key, X = 8)
5.  to_decrypt = message[24..]
    to_decrypt = to_decrypt[0 .. len - (len & 15)]      # drop transport padding
6.  plaintext = AES256_IGE_decrypt(to_decrypt, aes_key, aes_iv)
7.  (_, real_msg_key) = calc_message_key2(auth_key, X = 8, plaintext)
8.  if real_msg_key != msg_key: error "msg_key mismatch"        ← constant time!
9.  read salt, session_id, msg_id, seq_no, message_data_length
10. if message_data_length % 4 != 0: error
11. if message_data_length > available: error
12. pad_size = available - message_data_length
    if pad_size < 12 or pad_size > 1024: error
13. body = plaintext[32 .. 32 + message_data_length)
```

The exact source of the length checks (`Transport.cpp:258-299`):

```cpp
if (info->message_data_length % sizeof(int32) != 0) {
  return Status::Error("Invalid MTProto message: math_data_length is not divisible by 4");
}
…
size_t pad_size = data_size - info->message_data_length;
if (pad_size < 12 || pad_size > 1024) {
  return Status::Error("Invalid MTProto message: invalid padding length");
}
```

**`X = 8` for server→client.** `read_crypto` calls
`read_crypto_impl(8, …)` (`Transport.cpp:306`).

---

## 6. Mandatory receive-side validation

| # | Check | Why |
|---|-------|-----|
| 1 | `auth_key_id` matches yours | The message is not for you / wrong key |
| 2 | **Recomputed `msg_key` equals the received one — all 16 bytes** | This is the *only* integrity check. Skipping it accepts arbitrary ciphertext. |
| 3 | `message_data_length % 4 == 0` | TL objects are word-aligned |
| 4 | `message_data_length` fits in the decrypted buffer | Prevents out-of-bounds reads |
| 5 | `12 ≤ pad_size ≤ 1024` | Rejects malformed / truncated packets |
| 6 | `session_id` matches yours | [5.1](../05-session/01-msg-id-and-seq-no.md) |
| 7 | `msg_id` is **odd** (server-originated) | [5.1](../05-session/01-msg-id-and-seq-no.md) |
| 8 | `msg_id` within the time window and not a duplicate | [5.4](../05-session/04-salts-and-time.md) |

Checks 1–5 live in `Transport.cpp`; checks 6–8 in
`AuthData::check_packet` (`td/mtproto/AuthData.cpp:139-167`).

> **⚠ Security note — compare `msg_key` in constant time.** A byte-at-a-time comparison
> that returns early leaks, via timing, how many leading bytes an attacker guessed
> correctly, enabling a forgery oracle. Use `CRYPTO_memcmp`, `sodium_memcmp`, or Rust's
> `subtle::ConstantTimeEq`. TDLib compares `UInt128` values, whose `operator==` is a fixed
> 16-byte comparison.

> **⚠ Security note — reject, do not repair.** If any check fails, discard the packet.
> Never try to parse a partially valid message.

---

## 7. Complete size example

Sending `messages.sendMessage` with a 12-character text, body = 68 bytes:

```
data       = msg_id(8) + seq_no(4) + length(4) + body(68)      = 84
enc_size   = salt(8) + session_id(8)                           = 16
raw_size   = auth_key_id(8) + msg_key(16)                      = 24

encrypted_size = (16 + 84 + 12 + 15) & ~15 = 112
                 → bucket up to 128
packet_size    = 24 + 128 = 152 bytes
padding        = 128 - 16 - 84 = 28 bytes
```

Sanity: `12 ≤ 28 ≤ 1024` ✓, `152 - 24 = 128` is a multiple of 16 ✓, and `152 % 4 == 0` so
the Intermediate framing accepts it ✓.

---

## 8. MTProto 1.0 (do not implement)

TDLib still contains the v1 code path (`packet_info->version == 1`) for **secret chats**
only. Its differences:

* `msg_key = SHA1(plaintext)[4..20)` — SHA-1, and computed over the *unpadded* data only.
* Padding 0–15 bytes, not covered by `msg_key`.
* A different KDF (`KDF()` rather than `KDF2()`, `td/mtproto/KDF.cpp:17-50`).

For client–server MTProto, always use version 2. `PacketInfo::version` defaults to 2
(`td/mtproto/PacketInfo.h:38`).

---

[← Previous](../03-auth-key/06-worked-example.md) · [Index](../README.md) · [Next: msg_key and the KDF →](02-msg-key-and-kdf.md)
