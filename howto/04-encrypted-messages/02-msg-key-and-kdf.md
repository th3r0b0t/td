# 4.2 — `msg_key` and the Key Derivation Function

[← Previous](01-envelope.md) · [Index](../README.md) · [Next: AES-IGE →](03-aes-ige.md)

---

Two derivations sit at the heart of MTProto 2.0 encryption. Both are pure functions of the
auth key and the plaintext; get either one wrong and nothing works.

---

## 1. `msg_key`

`Transport::calc_message_key2` (`td/mtproto/Transport.cpp:148-164`):

```cpp
// MTProto v2.0
std::pair<uint32, UInt128> Transport::calc_message_key2(const AuthKey &auth_key, int X, Slice to_encrypt) {
  // msg_key_large = SHA256 (substr (auth_key, 88+x, 32) + plaintext + random_padding);
  Sha256State state;
  state.init();
  state.feed(Slice(auth_key.key()).substr(88 + X, 32));
  state.feed(to_encrypt);

  uint8 msg_key_large_raw[32];
  MutableSlice msg_key_large(msg_key_large_raw, sizeof(msg_key_large_raw));
  state.extract(msg_key_large, true);

  // msg_key = substr (msg_key_large, 8, 16);
  UInt128 res;
  as_mutable_slice(res).copy_from(msg_key_large.substr(8, 16));

  return std::make_pair(as<uint32>(msg_key_large_raw) | (1u << 31), res);
}
```

Formally:

```
msg_key_large = SHA256( auth_key[88+X .. 120+X)  ‖  plaintext )
msg_key       = msg_key_large[8..24)
```

where `plaintext` is `salt ‖ session_id ‖ msg_id ‖ seq_no ‖ length ‖ body ‖ padding` —
**everything that will be encrypted, including the padding**.

| Symbol | Value |
|--------|-------|
| `X` | `0` when you are sending, `8` when you are receiving |
| `auth_key[88+X .. 120+X)` | 32 bytes of the auth key: `[88..120)` for send, `[96..128)` for receive |
| `msg_key` | Bytes 8–23 of the 32-byte digest — the **middle** 16 bytes |

The first four bytes of `msg_key_large`, with bit 31 forced on, are the **quick-ack token**
for this message ([5.5](../05-session/05-reliability.md)).

> **⚠ Common bug.** `msg_key` is `msg_key_large[8..24)`, not `[0..16)`. Taking the first
> 16 bytes gives a valid-looking value that the server will reject.

### Why this construction

`msg_key` is simultaneously:

* an **integrity tag** — it is a keyed hash of the plaintext, so a modified ciphertext
  decrypts to something whose recomputed `msg_key` will not match;
* an **IV source** — the AES key and IV are derived from it, so identical plaintexts under
  the same key still encrypt differently (as long as `msg_id` differs, which it always
  does).

---

## 2. KDF2

`KDF2` (`td/mtproto/KDF.cpp:74-104`):

```cpp
// sha256_a = SHA256 (msg_key + substr(auth_key, x, 36));
buf.copy_from(msg_key_slice);
buf.substr(16, 36).copy_from(auth_key.substr(X, 36));
sha256(buf, sha256_a);

// sha256_b = SHA256 (substr(auth_key, 40+x, 36) + msg_key);
buf.copy_from(auth_key.substr(40 + X, 36));
buf.substr(36).copy_from(msg_key_slice);
sha256(buf, sha256_b);

// aes_key = substr(sha256_a, 0, 8) + substr(sha256_b, 8, 16) + substr(sha256_a, 24, 8);
// aes_iv  = substr(sha256_b, 0, 8) + substr(sha256_a, 8, 16) + substr(sha256_b, 24, 8);
```

Formally:

```
A = SHA256( msg_key ‖ auth_key[X .. X+36) )
B = SHA256( auth_key[40+X .. 76+X) ‖ msg_key )

aes_key = A[0..8)  ‖ B[8..24) ‖ A[24..32)        32 bytes
aes_iv  = B[0..8)  ‖ A[8..24) ‖ B[24..32)        32 bytes
```

Note the elegant symmetry: the key and the IV use the *same* three slices, with `A` and `B`
swapped in every position. It is easy to write this correctly if you keep that in mind, and
easy to get subtly wrong if you copy one line and edit it.

The IV is **32 bytes** because AES-IGE needs two blocks of IV — see
[4.3](03-aes-ige.md).

---

## 3. The `X` parameter

`X` selects which halves of the auth key are used, giving the two directions independent
key material.

| Direction | `X` | Source |
|-----------|-----|--------|
| Client → Server | **0** | `write_crypto` → `write_crypto_impl(0, …)`, `Transport.cpp:385` |
| Server → Client | **8** | `read_crypto` → `read_crypto_impl(8, …)`, `Transport.cpp:306` |

Concretely, the byte ranges of the 256-byte auth key:

| Purpose | Sending (`X=0`) | Receiving (`X=8`) |
|---------|-----------------|-------------------|
| `msg_key` prefix | `[88..120)` | `[96..128)` |
| `A` suffix | `[0..36)` | `[8..44)` |
| `B` prefix | `[40..76)` | `[48..84)` |

> **⚠ Common bug.** Swapping `X` is the single most frequent MTProto implementation
> mistake. The symptom is that your messages are accepted by the server but you cannot
> decrypt anything it sends back (or vice versa). If your `msg_key` verification fails on
> *every* incoming packet while sending works, you are using `X = 0` on receive.

Note that this asymmetry means **you cannot decrypt your own outgoing messages** with the
receive parameters — useful to remember when writing round-trip tests: encrypt with `X=0`,
decrypt with `X=0`.

---

## 4. Putting it together

### Encrypt

```
plaintext = salt ‖ session_id ‖ msg_id ‖ seq_no ‖ length ‖ body ‖ random_padding
            (total length must be a multiple of 16)

msg_key_large = SHA256(auth_key[88..120) ‖ plaintext)
msg_key       = msg_key_large[8..24)

A = SHA256(msg_key ‖ auth_key[0..36))
B = SHA256(auth_key[40..76) ‖ msg_key)
aes_key = A[0..8) ‖ B[8..24) ‖ A[24..32)
aes_iv  = B[0..8) ‖ A[8..24) ‖ B[24..32)

ciphertext = AES256_IGE_encrypt(plaintext, aes_key, aes_iv)
packet     = auth_key_id ‖ msg_key ‖ ciphertext
```

### Decrypt

```
assert packet[0..8) == auth_key_id
msg_key = packet[8..24)

A = SHA256(msg_key ‖ auth_key[8..44))
B = SHA256(auth_key[48..84) ‖ msg_key)
aes_key = A[0..8) ‖ B[8..24) ‖ A[24..32)
aes_iv  = B[0..8) ‖ A[8..24) ‖ B[24..32)

plaintext = AES256_IGE_decrypt(packet[24..], aes_key, aes_iv)

expected = SHA256(auth_key[96..128) ‖ plaintext)[8..24)
assert constant_time_eq(expected, msg_key)          ← MANDATORY
```

---

## 5. Test vector construction

Since the auth key is per-installation, there are no universal published vectors. Build
your own:

1. Fix a 256-byte `auth_key` (e.g. bytes `0x00, 0x01, …, 0xFF` repeated).
2. Fix a plaintext of a known length.
3. Print `msg_key`, `aes_key`, `aes_iv`, and the ciphertext.
4. Assert that decrypting with the same `X` reproduces the plaintext.
5. Assert that `X = 0` and `X = 8` produce **different** `aes_key`s.

TDLib's own `test/mtproto.cpp` exercises the full path against a live datacenter; its
`test/crypto.cpp` covers AES-IGE and the hash primitives.

Steps 4 and 5 catch almost every KDF bug on their own.

---

## 6. Checklist

- [ ] `msg_key_large = SHA256(auth_key[88+X .. 120+X) ‖ plaintext)`
- [ ] `msg_key = msg_key_large[8..24)` — the **middle** 16 bytes
- [ ] Plaintext for the hash **includes the padding**
- [ ] `A = SHA256(msg_key ‖ auth_key[X .. X+36))` — `msg_key` **first**
- [ ] `B = SHA256(auth_key[40+X .. 76+X) ‖ msg_key)` — `msg_key` **last**
- [ ] `aes_key = A[0..8) ‖ B[8..24) ‖ A[24..32)`
- [ ] `aes_iv = B[0..8) ‖ A[8..24) ‖ B[24..32)`
- [ ] `X = 0` for sending, `X = 8` for receiving
- [ ] `aes_iv` is 32 bytes, not 16
- [ ] `msg_key` verified on receive, in constant time

---

[← Previous](01-envelope.md) · [Index](../README.md) · [Next: AES-IGE →](03-aes-ige.md)
