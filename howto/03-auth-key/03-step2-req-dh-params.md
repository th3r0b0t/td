# 3.3 — Step 2: `req_DH_params` → `server_DH_params_ok`

[← Previous](02-step1-req-pq.md) · [Index](../README.md) · [Next: Step 3, set_client_DH_params →](04-step3-set-client-dh-params.md)

---

This is the hardest step. It has two halves: building an RSA-encrypted blob with a
non-standard padding scheme, and decrypting the server's reply with a temporary AES key.

---

## 1. Building the inner data

```
p_q_inner_data_dc#a9f55f95 pq:string p:string q:string nonce:int128
    server_nonce:int128 new_nonce:int256 dc:int = P_Q_inner_data;
```
(`mtproto_api.tl:11`)

```cpp
data = store_object(mtproto_api::p_q_inner_data_dc(
    res_pq->pq_, p, q, nonce_, server_nonce_, new_nonce_, dc_id_));
```
(`Handshake.cpp:116`)

| Field | Value |
|-------|-------|
| `pq` | Echo `resPQ.pq` verbatim |
| `p` | Smaller factor, minimal big-endian |
| `q` | Larger factor, minimal big-endian |
| `nonce` | Your nonce |
| `server_nonce` | From `resPQ` |
| `new_nonce` | Your 32 fresh random bytes |
| `dc` | The datacenter id you are talking to |

For a temporary (PFS) key, use instead:

```
p_q_inner_data_temp_dc#56fddf88 pq:string p:string q:string nonce:int128
    server_nonce:int128 new_nonce:int256 dc:int expires_in:int = P_Q_inner_data;
```
(`mtproto_api.tl:12`)

### Size

With an 8-byte `pq` and 4-byte `p`/`q`:

```
  4  constructor id
 12  pq      (1 len byte + 8 data + 3 pad)
  8  p       (1 + 4 + 3)
  8  q       (1 + 4 + 3)
 16  nonce
 16  server_nonce
 32  new_nonce
  4  dc
───
100 bytes
```

TDLib enforces `data_size <= 144` (`Handshake.cpp:129-131`) because of the padding scheme
below.

---

## 2. RSA_PAD — the encryption scheme

`Handshake.cpp:127-155`:

```cpp
string encrypted_data(256, '\0');
auto data_size = data.size();
if (data_size > 144) return Status::Error("Too big data");

data.resize(192);
Random::secure_bytes(MutableSlice(data).substr(data_size));

while (true) {
  string aes_key(32, '\0');
  Random::secure_bytes(MutableSlice(aes_key));

  string data_with_hash = PSTRING() << data << sha256(aes_key + data);
  std::reverse(data_with_hash.begin(), data_with_hash.begin() + data.size());

  string decrypted_data(256, '\0');
  string aes_iv(32, '\0');
  aes_ige_encrypt(aes_key, aes_iv, data_with_hash, MutableSlice(decrypted_data).substr(32));

  auto hash = sha256(MutableSlice(decrypted_data).substr(32));
  for (size_t i = 0; i < 32; i++) {
    decrypted_data[i] = static_cast<char>(aes_key[i] ^ hash[i]);
  }

  if (rsa_key.rsa.encrypt(decrypted_data, encrypted_data)) break;
}
```

Step by step:

### 2.1 Pad the data to 192 bytes

```
data = p_q_inner_data_dc_bytes ‖ random(192 − len)
```

Note that `data` is now exactly 192 bytes and `data.size()` in the following line refers to
**192**, not the original length (`resize` happened first).

### 2.2 Generate a fresh 32-byte `aes_key`

Regenerated on every loop iteration.

### 2.3 Build `data_with_hash` (224 bytes)

```
data_with_hash = data ‖ SHA256(aes_key ‖ data)
                 └192┘   └────── 32 ──────┘
```

then **reverse the first 192 bytes in place** (the `data` part only; the hash is left
alone):

```
data_with_hash = reverse(data) ‖ SHA256(aes_key ‖ data)
```

> **⚠ Common bug.** The reversal is over `data.size()` bytes — and by this point
> `data.size() == 192`, because of the `resize(192)` two lines earlier. It is **not** the
> original TL length. Reverse exactly the first 192 bytes.

### 2.4 AES-256-IGE encrypt with a **zero IV**

```
key = aes_key                (32 bytes)
iv  = 00 × 32                (32 bytes — IGE uses two blocks of IV)
ciphertext = AES_IGE_encrypt(data_with_hash)     # 224 bytes in, 224 out
```

Store it at offset 32 of a 256-byte buffer:

```
out[32..256) = ciphertext
```

### 2.5 Compute the key mask

```
out[0..32) = aes_key XOR SHA256(out[32..256))
```

This makes the whole 256-byte block indistinguishable from random and lets the server
recover `aes_key` by computing the same SHA256.

### 2.6 RSA-encrypt with **no further padding**

`RSA::encrypt` (`td/mtproto/RSA.cpp:125-155`) is textbook RSA:

```cpp
BigNum x = BigNum::from_binary(from);
if (BigNum::compare(x, n_) >= 0) {
  return false;                       // caller retries with a new aes_key
}
BigNum y;
BigNum::mod_exp(y, x, e_, n_, ctx);   // y = x^e mod n
…y.to_binary(256)…
```

Because the input is a raw 256-byte big-endian integer and the modulus is also 256 bytes,
about half the time the value is ≥ *n* and RSA fails. **That is why the whole construction
is inside a `while (true)` loop** — on failure you regenerate `aes_key` and start over from
2.2. Expect one or two iterations on average.

### 2.7 Result

`encrypted_data` is exactly 256 bytes.

### Visual summary

```
     ┌───────────────── 192 bytes ─────────────────┬──── 32 ────┐
     │ reverse( TL(p_q_inner_data) ‖ random pad )  │ SHA256(    │
     │                                             │  aes_key ‖ │
     │                                             │  data )    │
     └─────────────────────────────────────────────┴────────────┘
                              │ AES-256-IGE, key = aes_key, IV = 0×32
                              ▼
     ┌──── 32 ────┬───────────────── 224 bytes ─────────────────┐
     │ aes_key ⊕  │              ciphertext                     │
     │ SHA256(ct) │                                             │
     └────────────┴─────────────────────────────────────────────┘
                              │ RSA: x^e mod n   (retry if x ≥ n)
                              ▼
                        256 bytes → encrypted_data
```

> **💡 Implementation note.** There is an older scheme (plain `SHA1(data) ‖ data ‖ random`
> with PKCS-free RSA) still described in some documents. TDLib implements **only**
> RSA_PAD as above. Use it.

---

## 3. The request

```
req_DH_params#d712e4be nonce:int128 server_nonce:int128 p:string q:string
    public_key_fingerprint:long encrypted_data:string = Server_DH_Params;
```
(`mtproto_api.tl:14`)

```cpp
mtproto_api::req_DH_params req_dh_params(nonce_, server_nonce_, p, q,
                                         rsa_key.fingerprint, encrypted_data);
send(connection, create_function_storer(req_dh_params));
```
(`Handshake.cpp:157-158`)

`p` and `q` appear **twice** in effect — once here in clear, once inside the RSA blob. The
server compares them; that is the proof-of-work check.

---

## 4. The reply

```
server_DH_params_ok#d0e8075c nonce:int128 server_nonce:int128
    encrypted_answer:string = Server_DH_Params;
```
(`mtproto_api.tl:16`)

There is also `server_DH_params_fail#79cb045d ... new_nonce_hash:int128`
(`mtproto_api.tl:15`, commented out in TDLib's schema since it is never handled). If you
receive it, your `p`/`q` or RSA blob was wrong; abort.

### Validation

```cpp
if (dh_params->nonce_ != nonce_) return Status::Error("Nonce mismatch");
if (dh_params->server_nonce_ != server_nonce_) return Status::Error("Server nonce mismatch");
if (dh_params->encrypted_answer_.size() & 15) return Status::Error("Bad padding for encrypted part");
```
(`Handshake.cpp:171-179`)

---

## 5. Deriving the temporary AES key

`tmp_KDF` (`td/mtproto/KDF.cpp:52-72`):

```cpp
void tmp_KDF(const UInt128 &server_nonce, const UInt256 &new_nonce,
             UInt256 *tmp_aes_key, UInt256 *tmp_aes_iv) {
  // tmp_aes_key := SHA1(new_nonce + server_nonce) + substr (SHA1(server_nonce + new_nonce), 0, 12);
  sha1(new_nonce ‖ server_nonce)  → tmp_aes_key[0..20)
  sha1(server_nonce ‖ new_nonce)  → tmp_aes_key[20..32)   (first 12 bytes of the digest)

  // tmp_aes_iv := substr (SHA1(server_nonce + new_nonce), 12, 8) +
  //               SHA1(new_nonce + new_nonce) + substr (new_nonce, 0, 4);
  sha1(server_nonce ‖ new_nonce)[12..20) → tmp_aes_iv[0..8)
  sha1(new_nonce ‖ new_nonce)            → tmp_aes_iv[8..28)
  new_nonce[0..4)                        → tmp_aes_iv[28..32)
}
```

Written out explicitly, with `A = SHA1(new_nonce ‖ server_nonce)`,
`B = SHA1(server_nonce ‖ new_nonce)`, `C = SHA1(new_nonce ‖ new_nonce)`:

```
tmp_aes_key = A[0..20)  ‖ B[0..12)                    (32 bytes)
tmp_aes_iv  = B[12..20) ‖ C[0..20) ‖ new_nonce[0..4)  (32 bytes)
```

Note the ordering asymmetry: `new_nonce` comes **first** in `A`, `server_nonce` comes
**first** in `B`. Getting this backwards is a classic bug that produces a SHA1 mismatch in
the next step.

---

## 6. Decrypting `encrypted_answer`

```cpp
tmp_KDF(server_nonce_, new_nonce_, &tmp_aes_key, &tmp_aes_iv);
auto save_tmp_aes_iv = tmp_aes_iv;
aes_ige_decrypt(as_slice(tmp_aes_key), as_mutable_slice(tmp_aes_iv), answer, answer);
tmp_aes_iv = save_tmp_aes_iv;
```
(`Handshake.cpp:181-188`)

The IV is saved and restored because `aes_ige_decrypt` **mutates** it — IGE carries state
forward. You need the original IV again in §8. If your AES-IGE routine also mutates its IV
argument, make a copy.

The plaintext has this layout:

```
┌──── 20 ────┬──────── variable ────────┬─── 0..15 ───┐
│ SHA1(inner)│ server_DH_inner_data     │ random pad  │
└────────────┴──────────────────────────┴─────────────┘
```

Parsing (`Handshake.cpp:191-212`):

```cpp
TlParser answer_parser(answer);
UInt<160> answer_sha1 = answer_parser.fetch_binary<UInt<160>>();   // 20 bytes
int32 id = answer_parser.fetch_int();
if (id != mtproto_api::server_DH_inner_data::ID) return Status::Error(…);
mtproto_api::server_DH_inner_data dh_inner_data(answer_parser);
if (answer_parser.get_error() != nullptr) return Status::Error(…);

size_t pad = answer_parser.get_left_len();
if (pad >= 16) return Status::Error("Too much pad");

size_t dh_inner_data_size = answer.size() - pad - 20;
UInt<160> answer_real_sha1;
sha1(answer.substr(20, dh_inner_data_size), answer_real_sha1.raw);
if (answer_sha1 != answer_real_sha1) return Status::Error("SHA1 mismatch");
```

Three checks here, all mandatory:

1. The constructor id must be `server_DH_inner_data` (`0xb5890dba`).
2. Leftover bytes after parsing must be **< 16** — anything more means the decryption
   produced garbage.
3. `SHA1` over the *parsed extent only* (from offset 20, length `dh_inner_data_size`) must
   equal the stored digest. The padding is **not** hashed.

> **⚠ Security note.** This SHA1 is the only integrity check on the DH parameters. Skipping
> it means accepting arbitrary attacker-chosen `g`, `p` and `g_a`.

---

## 7. `server_DH_inner_data`

```
server_DH_inner_data#b5890dba nonce:int128 server_nonce:int128 g:int
    dh_prime:string g_a:string server_time:int = Server_DH_inner_data;
```
(`mtproto_api.tl:18`)

```cpp
if (dh_inner_data.nonce_ != nonce_) return Status::Error("Nonce mismatch");
if (dh_inner_data.server_nonce_ != server_nonce_) return Status::Error("Server nonce mismatch");
server_time_diff_ = dh_inner_data.server_time_ - Time::now();
```
(`Handshake.cpp:214-221`)

**Record `server_time_diff`.** Every `msg_id` you generate from now on is based on the
server's clock, not yours — see [5.4](../05-session/04-salts-and-time.md). If your machine
is 5 minutes fast and you ignore this, every message you send will be rejected with
`bad_msg_notification` code 17.

The `g`, `dh_prime` and `g_a` values then go through the validation described in
[3.5](05-security-checks.md). **Do not skip it.**

---

## 8. Preparing your side

```cpp
DhHandshake handshake;
handshake.set_config(dh_inner_data.g_, dh_inner_data.dh_prime_);
handshake.set_g_a(dh_inner_data.g_a_);
TRY_STATUS(handshake.run_checks(false, dh_callback));
string g_b = handshake.get_g_b();
auto auth_key_params = handshake.gen_key();
```
(`Handshake.cpp:223-228`)

`get_g_b` (`td/mtproto/DhHandshake.cpp:128-140`) generates a 2048-bit random exponent `b`
and computes `g_b = g^b mod p`, serialized as a **minimal big-endian** string.

The `auth_key` is computed here already (`gen_key()` → `g_a^b mod p`), but it only becomes
valid once the server confirms in step 3.

---

## 9. Checklist for step 2

- [ ] Built `p_q_inner_data_dc` with `dc` set to the DC you are connected to
- [ ] Padded `data` to exactly 192 bytes with random
- [ ] Fresh 32-byte `aes_key` per attempt
- [ ] `data_with_hash = data ‖ SHA256(aes_key ‖ data)`, then **first 192 bytes reversed**
- [ ] AES-256-IGE with a **32-byte zero IV**, output at `out[32..256)`
- [ ] `out[0..32) = aes_key ⊕ SHA256(out[32..256))`
- [ ] Raw RSA `x^e mod n`, **retry the whole loop** if `x ≥ n`
- [ ] `encrypted_data` is exactly 256 bytes
- [ ] Verified both nonces in the reply
- [ ] `encrypted_answer.size() % 16 == 0`
- [ ] `tmp_KDF` argument order correct (`new_nonce` first for the key, `server_nonce` first
      for the second hash)
- [ ] Saved a copy of `tmp_aes_iv` before decrypting
- [ ] Constructor id is `0xb5890dba`
- [ ] Leftover padding < 16 bytes
- [ ] SHA1 over the inner data (excluding padding) matches
- [ ] Both nonces inside `server_DH_inner_data` verified
- [ ] `server_time_diff` recorded

---

[← Previous](02-step1-req-pq.md) · [Index](../README.md) · [Next: Step 3, set_client_DH_params →](04-step3-set-client-dh-params.md)
