# 4.3 — AES-256-IGE

[← Previous](02-msg-key-and-kdf.md) · [Index](../README.md) · [Next: Encrypt/decrypt recipes →](04-encrypt-decrypt-recipes.md)

---

IGE — Infinite Garble Extension — is the block cipher mode MTProto uses. It is not in
OpenSSL's EVP interface, not in `libsodium`, and not in most language standard libraries.
You will implement it yourself on top of raw AES-256 ECB. It is about 20 lines.

---

## 1. Parameters

| Item | Size |
|------|------|
| Key | **32 bytes** (AES-256) |
| IV | **32 bytes** = two AES blocks |
| Block | 16 bytes |
| Input length | Must be a multiple of 16 |

The 32-byte IV is the thing that surprises people. `AesIgeStateImpl::init`
(`tdutils/td/utils/crypto.cpp:469-480`):

```cpp
void init(Slice key, Slice iv, bool encrypt) {
  CHECK(key.size() == 32);
  CHECK(iv.size() == 32);
  …
  encrypted_iv_.load(iv.ubegin());                    // iv[0..16)
  plaintext_iv_.load(iv.ubegin() + AES_BLOCK_SIZE);   // iv[16..32)
}
```

So:

```
iv[0..16)   → "previous ciphertext block"  (c₀)
iv[16..32)  → "previous plaintext block"   (p₀)
```

---

## 2. The algorithm

For blocks numbered from 1, with `c₀` and `p₀` taken from the IV:

```
Encryption:   cᵢ = AES_encrypt(pᵢ ⊕ cᵢ₋₁) ⊕ pᵢ₋₁
Decryption:   pᵢ = AES_decrypt(cᵢ ⊕ pᵢ₋₁) ⊕ cᵢ₋₁
```

Compare with CBC (`cᵢ = AES_encrypt(pᵢ ⊕ cᵢ₋₁)`): IGE adds the extra `⊕ pᵢ₋₁` on the
output. The consequence is the "infinite garble" property — a single bit flipped in the
ciphertext corrupts that block *and every subsequent block*, rather than just two blocks as
in CBC. That is exactly what you want when a MAC covers the whole message.

TDLib's decrypt loop (`crypto.cpp:528-551`) reads almost as the formula:

```cpp
while (len) {
  encrypted.load(in);

  plaintext_iv_ ^= encrypted;                                     // pᵢ₋₁ ⊕ cᵢ
  evp_.decrypt(plaintext_iv_.raw(), plaintext_iv_.raw(), AES_BLOCK_SIZE);
  plaintext_iv_ ^= encrypted_iv_;                                 // ⊕ cᵢ₋₁

  plaintext_iv_.store(out);                                       // this is pᵢ
  encrypted_iv_ = encrypted;                                      // cᵢ₋₁ ← cᵢ
  …
}
```

The encrypt loop (`crypto.cpp:488-526`) is batched over 31 blocks at a time to exploit
OpenSSL's CBC implementation — an optimization, not a semantic difference. The batching
works because `AES_encrypt(pᵢ ⊕ cᵢ₋₁)` is exactly a CBC step; TDLib pre-XORs the `pᵢ₋₁`
terms, runs one CBC pass, then post-XORs. **Do not copy this trick.** Write the naive
block-at-a-time loop; it is clearer and fast enough.

---

## 3. Reference implementation

```
fn aes_ige_encrypt(key: [u8; 32], iv: [u8; 32], data: &[u8]) -> Vec<u8> {
    assert!(data.len() % 16 == 0);
    let cipher = Aes256::new(key);          // raw ECB block encryption
    let mut prev_c = iv[0..16].to_vec();
    let mut prev_p = iv[16..32].to_vec();
    let mut out = Vec::with_capacity(data.len());

    for block in data.chunks(16) {
        let mut t = xor(block, &prev_c);
        cipher.encrypt_block(&mut t);       // AES-256 ECB, single block
        let c = xor(&t, &prev_p);
        out.extend_from_slice(&c);
        prev_c = c;
        prev_p = block.to_vec();
    }
    out
}

fn aes_ige_decrypt(key: [u8; 32], iv: [u8; 32], data: &[u8]) -> Vec<u8> {
    assert!(data.len() % 16 == 0);
    let cipher = Aes256::new(key);
    let mut prev_c = iv[0..16].to_vec();
    let mut prev_p = iv[16..32].to_vec();
    let mut out = Vec::with_capacity(data.len());

    for block in data.chunks(16) {
        let mut t = xor(block, &prev_p);
        cipher.decrypt_block(&mut t);       // AES-256 ECB DECRYPT
        let p = xor(&t, &prev_c);
        out.extend_from_slice(&p);
        prev_c = block.to_vec();
        prev_p = p;
    }
    out
}
```

> **⚠ Common bug.** Decryption uses the AES **decryption** primitive
> (`AES_decrypt` / `EVP_aes_256_ecb` in decrypt mode). TDLib makes this explicit at
> `crypto.cpp:472-476`: `init_encrypt_cbc` for encryption, `init_decrypt_ecb` for
> decryption. Some modes (CTR, OFB) use the encryption primitive in both directions; IGE
> does not.

### With OpenSSL in C

```c
AES_KEY aes;
/* encrypt */ AES_set_encrypt_key(key, 256, &aes);   AES_encrypt(in, out, &aes);
/* decrypt */ AES_set_decrypt_key(key, 256, &aes);   AES_decrypt(in, out, &aes);
```

OpenSSL also ships `AES_ige_encrypt()` in `<openssl/aes.h>` for older versions — it
implements exactly this mode with the same 32-byte IV convention. It is deprecated and
absent from some builds, so writing the loop yourself is more portable.

---

## 4. The IV is mutated

`get_iv` (`crypto.cpp:482-486`) exists because the state advances:

```cpp
void get_iv(MutableSlice iv) {
  encrypted_iv_.store(iv.ubegin());
  plaintext_iv_.store(iv.ubegin() + AES_BLOCK_SIZE);
}
```

After processing a message, `encrypted_iv_` holds the last ciphertext block and
`plaintext_iv_` the last plaintext block. TDLib exploits this during the handshake, where it
must **save and restore** the IV (`td/mtproto/Handshake.cpp:184-188`):

```cpp
auto save_tmp_aes_iv = tmp_aes_iv;
aes_ige_decrypt(as_slice(tmp_aes_key), as_mutable_slice(tmp_aes_iv), answer, answer);
tmp_aes_iv = save_tmp_aes_iv;
```

For ordinary encrypted messages this does not matter — every message derives a fresh
`aes_key`/`aes_iv` from its own `msg_key`, and the IV is never carried across messages.

> **💡 Implementation note.** If you write your IGE function to take the IV **by value**
> (Rust: `iv: [u8; 32]`; C: copy into a local), you cannot make the "reused a mutated IV"
> mistake at all. Recommended.

---

## 5. In-place operation

TDLib routinely passes the same buffer as input and output
(`Transport.cpp:365`, `Handshake.cpp:187`):

```cpp
aes_ige_encrypt(as_slice(aes_key), as_mutable_slice(aes_iv), to_encrypt, to_encrypt);
```

The naive loop above supports this naturally if you read `block` into a local before
writing the output — which the chunk-based code does. If you write output over the input
buffer with pointer arithmetic, save `prev_p` (encryption) or `prev_c` (decryption)
*before* overwriting.

---

## 6. Verifying your implementation

Since IGE has no widely published test vectors, verify by construction:

1. **Round trip.** `decrypt(encrypt(P, K, IV), K, IV) == P` for random `P` of length
   16, 32, 48, 1024.
2. **Avalanche.** Flip one bit in the ciphertext of block *i*; every plaintext block from
   *i* onwards must change. (If only blocks *i* and *i+1* change, you implemented CBC.)
3. **IV sensitivity.** Changing any byte of the 32-byte IV must change the first output
   block.
4. **Against TDLib.** Build this repository and compare with
   `td::aes_ige_encrypt` from `tdutils/td/utils/crypto.h`; `test/crypto.cpp` has ready
   harnesses.
5. **Against OpenSSL.** If your OpenSSL has `AES_ige_encrypt`, compare directly.

Test 2 is the one that catches the most implementations.

---

## 7. Where AES-IGE is used

| Context | Key | IV |
|---------|-----|-----|
| RSA_PAD inner encryption ([3.3](../03-auth-key/03-step2-req-dh-params.md)) | random 32 bytes | **32 zero bytes** |
| Handshake DH blobs ([3.3](../03-auth-key/03-step2-req-dh-params.md)) | `tmp_aes_key` | `tmp_aes_iv` |
| Encrypted messages ([4.1](01-envelope.md)) | `KDF2` output | `KDF2` output |

The obfuscation layer ([2.3](../02-transport/03-obfuscation.md)) uses AES-**CTR**, not
IGE — different mode, 16-byte IV, and the counter persists across the whole connection.

---

## 8. Checklist

- [ ] Key is 32 bytes, IV is **32 bytes**
- [ ] `c₀ = iv[0..16)`, `p₀ = iv[16..32)`
- [ ] Encrypt: `cᵢ = AES_enc(pᵢ ⊕ cᵢ₋₁) ⊕ pᵢ₋₁`
- [ ] Decrypt: `pᵢ = AES_dec(cᵢ ⊕ pᵢ₋₁) ⊕ cᵢ₋₁`
- [ ] Decrypt uses the AES **decryption** primitive
- [ ] Input length is a multiple of 16 — assert it
- [ ] IV taken by value, or explicitly copied before use
- [ ] Round-trip test passes
- [ ] Avalanche test passes (all later blocks garbled)

---

[← Previous](02-msg-key-and-kdf.md) · [Index](../README.md) · [Next: Encrypt/decrypt recipes →](04-encrypt-decrypt-recipes.md)
