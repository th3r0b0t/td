# 4.4 — Encrypt / Decrypt Recipes

[← Previous](03-aes-ige.md) · [Index](../README.md) · [Next: msg_id and seq_no →](../05-session/01-msg-id-and-seq-no.md)

---

Everything from chapter 4 as two self-contained procedures you can transcribe directly.

---

## 1. Encrypt

**Inputs:** `auth_key` (256 B), `auth_key_id` (8 B), `salt` (8 B), `session_id` (8 B),
`msg_id` (u64), `seq_no` (u32), `body` (TL-serialized, length a multiple of 4).

```
 1.  data = LE64(msg_id) ‖ LE32(seq_no) ‖ LE32(len(body)) ‖ body
     # len(data) == 16 + len(body)

 2.  # padding: at least 12 bytes, total encrypted part a multiple of 16
     min_total = 16 + len(data) + 12                # salt + session_id + data + min pad
     total     = (min_total + 15) & ~15
     # optional bucketing for traffic analysis resistance:
     #   for b in [64,128,192,256,384,512,768,1024,1280]:
     #       if total <= b: total = b; break
     #   else: total = 1280 + ceil((total - 1280) / 448) * 448
     pad_len = total - 16 - len(data)                # 12..27 without bucketing
     padding = csprng(pad_len)

 3.  plaintext = salt ‖ session_id ‖ data ‖ padding
     assert len(plaintext) % 16 == 0

 4.  msg_key_large = SHA256(auth_key[88..120) ‖ plaintext)
     msg_key       = msg_key_large[8..24)

 5.  A = SHA256(msg_key ‖ auth_key[0..36))
     B = SHA256(auth_key[40..76) ‖ msg_key)
     aes_key = A[0..8) ‖ B[8..24) ‖ A[24..32)
     aes_iv  = B[0..8) ‖ A[8..24) ‖ B[24..32)

 6.  ciphertext = AES256_IGE_encrypt(plaintext, aes_key, aes_iv)

 7.  packet = auth_key_id ‖ msg_key ‖ ciphertext
     assert len(packet) % 4 == 0

 8.  send_framed(packet)                             # 4-byte LE length prefix
```

Reference: `Transport::write_crypto` (`td/mtproto/Transport.cpp:368-388`) and
`write_crypto_impl` (`:343-366`).

---

## 2. Decrypt

**Inputs:** `packet` (from the transport), `auth_key`, `auth_key_id`, expected
`session_id`.

```
 1.  if len(packet) < 40:                            # 24 header + 16 minimum block
         reject "too small"

 2.  if packet[0..8) != auth_key_id:
         reject "auth key id mismatch"

 3.  msg_key = packet[8..24)

 4.  A = SHA256(msg_key ‖ auth_key[8..44))
     B = SHA256(auth_key[48..84) ‖ msg_key)
     aes_key = A[0..8) ‖ B[8..24) ‖ A[24..32)
     aes_iv  = B[0..8) ‖ A[8..24) ‖ B[24..32)

 5.  to_decrypt = packet[24..]
     to_decrypt = to_decrypt[0 .. len(to_decrypt) & ~15]     # drop transport padding
     if len(to_decrypt) == 0: reject

 6.  plaintext = AES256_IGE_decrypt(to_decrypt, aes_key, aes_iv)

 7.  expected = SHA256(auth_key[96..128) ‖ plaintext)[8..24)
     if not constant_time_eq(expected, msg_key):
         reject "msg_key mismatch"                   # ← the ONLY integrity check

 8.  salt        = LE64(plaintext[0..8))
     session_id  = LE64(plaintext[8..16))
     msg_id      = LE64(plaintext[16..24))
     seq_no      = LE32(plaintext[24..28))
     length      = LE32(plaintext[28..32))

 9.  if length % 4 != 0:                    reject
     if 32 + length > len(plaintext):       reject
     pad = len(plaintext) - 32 - length
     if pad < 12 or pad > 1024:             reject

10.  if session_id != our_session_id:       reject
     if msg_id % 2 != 1:                    reject      # server messages are odd
     if not time_window_ok(msg_id):         reject
     if is_duplicate(msg_id):               reject

11.  body = plaintext[32 .. 32 + length)
     dispatch(body, msg_id, seq_no)
```

Reference: `Transport::read_crypto_impl` (`Transport.cpp:214-300`) and
`AuthData::check_packet` (`td/mtproto/AuthData.cpp:139-167`).

Steps 10's checks are covered in detail in
[5.1](../05-session/01-msg-id-and-seq-no.md) and
[5.4](../05-session/04-salts-and-time.md).

---

## 3. Byte-offset cheat sheet

### Outgoing packet

| Offset | Size | Content | Encrypted? |
|--------|------|---------|-----------|
| 0 | 8 | `auth_key_id` | no |
| 8 | 16 | `msg_key` | no |
| 24 | 8 | `salt` | yes |
| 32 | 8 | `session_id` | yes |
| 40 | 8 | `msg_id` | yes |
| 48 | 4 | `seq_no` | yes |
| 52 | 4 | `message_data_length` | yes |
| 56 | N | `body` | yes |
| 56+N | 12+ | padding | yes |

### Auth key slices

| Use | Send (`X=0`) | Receive (`X=8`) |
|-----|--------------|-----------------|
| `msg_key` prefix | `auth_key[88..120)` | `auth_key[96..128)` |
| `A` suffix | `auth_key[0..36)` | `auth_key[8..44)` |
| `B` prefix | `auth_key[40..76)` | `auth_key[48..84)` |

### Hash slices

| Value | Slice |
|-------|-------|
| `msg_key` | `SHA256(...)[8..24)` |
| `auth_key_id` | `SHA1(auth_key)[12..20)` |
| `aux_hash` (handshake) | `SHA1(auth_key)[0..8)` |
| `new_nonce_hash1` | `SHA1(new_nonce ‖ 0x01 ‖ aux_hash)[4..20)` |
| RSA fingerprint | `SHA1(TL(rsa_public_key))[12..20)` |
| quick-ack token | `SHA256(...)[0..4) \| 0x80000000` |

---

## 4. Minimal-message worked example

Sending `msgs_ack` with one message id (the smallest realistic message).

```
body = 59 b4 d6 62                       # msgs_ack#62d6b459
       15 c4 b5 1c                       # Vector
       01 00 00 00                       # count = 1
       <8 bytes msg_id>
     = 20 bytes

data = <8: msg_id> <4: seq_no> 14 00 00 00 <20: body>
     = 36 bytes

min_total = 16 + 36 + 12 = 64
total     = (64 + 15) & ~15 = 64
            bucket: 64 <= 64  → 64
pad_len   = 64 - 16 - 36 = 12          # exactly the minimum

plaintext = <8: salt> <8: session_id> <36: data> <12: random>
          = 64 bytes                    ✓ multiple of 16

packet    = <8: auth_key_id> <16: msg_key> <64: ciphertext>
          = 88 bytes                    ✓ multiple of 4

wire      = 58 00 00 00 <88 bytes>      # 88 = 0x58
          = 92 bytes total
```

---

## 5. Failure modes

| Symptom | Cause |
|---------|-------|
| Server replies `-404` | `auth_key_id` computed from the wrong SHA1 slice, or the wrong DC |
| Server silently drops your messages | `msg_id` outside the time window; check `server_time_diff` |
| `msg_key` mismatch on **every** incoming packet | Using `X = 0` on receive instead of `X = 8` |
| `msg_key` mismatch on some packets | Not truncating `to_decrypt` to a multiple of 16 (transport padding) |
| Decrypt yields plausible header but garbage body | `aes_key`/`aes_iv` slices swapped (`A`↔`B`) |
| `length % 4 != 0` after decrypt | The whole decryption is wrong; the check just caught it |
| `pad_size` out of range | Same; or you forgot the 12-byte minimum when encrypting |
| Works for small messages, fails for large | Missing the `>1280` padding branch, or your `msg_key` hash omits the padding |
| Intermittent failures ~1 time in 256 | `auth_key` not left-padded to exactly 256 bytes |

---

## 6. Debugging strategy

Work bottom-up; each layer has a cheap self-check.

1. **Transport.** Hexdump every frame in and out. Lengths must match, and every outgoing
   payload must be a multiple of 4.
2. **Crypto round trip, offline.** Encrypt a packet, decrypt it yourself with `X = 0`.
   If that fails, the bug is in AES-IGE or KDF2 — no network needed.
3. **First encrypted message.** Send `ping` and expect `pong`. If you get `-404`, the
   problem is `auth_key_id` or the auth key itself; if you get nothing, it is `msg_id` or
   `salt`.
4. **Decryption.** Once the server replies, check `msg_key` verification with `X = 8`. Log
   the first 32 bytes of the decrypted plaintext — `salt`, `session_id` and `msg_id` should
   all look plausible immediately.
5. **Only then** move on to the session layer.

TDLib's own logging is useful here: `VLOG(raw_mtproto)` in
`Transport.cpp:348-349` dumps every outgoing packet body in hex. Enable it with
`td::VERBOSITY_NAME(raw_mtproto)` if you build the reference implementation to compare
against.

---

[← Previous](03-aes-ige.md) · [Index](../README.md) · [Next: msg_id and seq_no →](../05-session/01-msg-id-and-seq-no.md)
