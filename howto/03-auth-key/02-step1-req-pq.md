# 3.2 — Step 1: `req_pq_multi` → `resPQ`

[← Previous](01-overview.md) · [Index](../README.md) · [Next: Step 2, req_DH_params →](03-step2-req-dh-params.md)

---

## 1. The request

```
req_pq_multi#be7e8ef1 nonce:int128 = ResPQ;
```
(`td/generate/scheme/mtproto_api.tl:8`)

Generate 16 cryptographically random bytes for `nonce` and remember them; you will compare
them against every server reply for the rest of the handshake.

`AuthKeyHandshake::on_start` (`Handshake.cpp:315-322`):

```cpp
Random::secure_bytes(nonce_.raw, sizeof(nonce_));
send(connection, create_storer(mtproto_api::req_pq_multi(nonce_)));
state_ = State::ResPQ;
```

### Bytes

The TL body is 20 bytes:

```
f1 8e 7e be                          # 0xbe7e8ef1  req_pq_multi
<16 random bytes>                    # nonce (int128, raw bytes, no endianness)
```

Wrapped as an unencrypted MTProto message (see [3.1 §5](01-overview.md)):

```
00 00 00 00 00 00 00 00              # auth_key_id = 0
<8 bytes>                            # msg_id
14 00 00 00                          # message_data_length = 20
f1 8e 7e be <16 bytes nonce>         # data
<12 bytes>                           # padding to a multiple of 16 (20 + 12 = 32)
```

Then framed with the 4-byte Intermediate length prefix (`38 00 00 00` = 56).

> **💡 Implementation note.** `int128` and `int256` are **opaque byte arrays**, not
> integers. Never byte-swap them. Copy them verbatim between request and reply.

### Why `_multi`?

There is also `req_pq#60469778 nonce:int128 = ResPQ;`
(`mtproto_api.tl:7`). The difference is server-side: `req_pq_multi` permits the server to
return multiple RSA fingerprints and is what all current clients use. TDLib only ever sends
`req_pq_multi`. Note the amusing consequence in `Handshake.cpp:91`:

```cpp
TRY_RESULT(res_pq, fetch_result<mtproto_api::req_pq_multi>(message, false));
```

it parses the *result type* of `req_pq_multi`, which is `ResPQ`.

---

## 2. The reply

```
resPQ#05162463 nonce:int128 server_nonce:int128 pq:string
    server_public_key_fingerprints:Vector<long> = ResPQ;
```
(`mtproto_api.tl:5`)

| Field | Type | Meaning |
|-------|------|---------|
| `nonce` | int128 | **Must equal** the nonce you sent |
| `server_nonce` | int128 | Server's nonce; feeds the temporary AES key |
| `pq` | string | A semiprime, big-endian, usually 8 bytes (< 2⁶³) |
| `server_public_key_fingerprints` | Vector\<long\> | Which RSA keys the server accepts |

### Validation

```cpp
if (res_pq->nonce_ != nonce_) {
  return Status::Error("Nonce mismatch");
}
server_nonce_ = res_pq->server_nonce_;
```
(`Handshake.cpp:92-96`)

Comparing the nonce is mandatory. Store `server_nonce` — it is used in the temporary key
derivation and in every subsequent message.

---

## 3. Choosing an RSA key

```cpp
auto r_rsa_key = public_rsa_key->get_rsa_key(res_pq->server_public_key_fingerprints_);
if (r_rsa_key.is_error()) {
  public_rsa_key->drop_keys();
  return r_rsa_key.move_as_error();
}
```
(`Handshake.cpp:98-103`)

You hold one or more hardcoded RSA public keys ([appendix B](../appendix/B-datacenters-and-keys.md)).
Compute each one's fingerprint and find the one present in the server's list. If none
matches, **abort** — do not proceed with an unknown key.

### Computing a fingerprint

`RSA::get_fingerprint` (`td/mtproto/RSA.cpp:111-123`):

```cpp
int64 RSA::get_fingerprint() const {
  mtproto_api::rsa_public_key public_key(n_.to_binary(), e_.to_binary());
  size_t size = tl_calc_length(public_key);
  std::string tmp(size, '\0');
  size = tl_store_unsafe(public_key, MutableSlice(tmp).ubegin());
  tmp.resize(size);
  return static_cast<int64>(td::as<uint64>(sha1(tmp).substr(12, 8).c_str()));
}
```

Broken down:

1. `n` and `e` are the RSA modulus and exponent as **minimal big-endian** byte strings
   (`to_binary()` strips leading zeros; a 2048-bit modulus is 256 bytes, `e = 65537` is
   3 bytes `01 00 01`).
2. Serialize `rsa_public_key n:string e:string = RSAPublicKey;`
   (`mtproto_api.tl:65`) — this is a **bare** constructor, so **no 4-byte constructor id
   is written**. The result is just `TL_string(n) ‖ TL_string(e)`, with the usual length
   prefixes and 4-byte padding from [1.2](../01-serialization/02-wire-encoding.md).
3. Take `SHA1` of that.
4. The fingerprint is bytes `[12..20)` of the digest, read as a **little-endian**
   `int64`.

> **⚠ Security note.** The fingerprint is not a security check by itself — it is a key
> *selector*. The security comes from you having the correct public key hardcoded in your
> binary. Never fetch a key from the network.

For the exact bytes of the two keys and their fingerprints, see
[appendix B](../appendix/B-datacenters-and-keys.md).

---

## 4. Factorizing `pq`

```cpp
string p, q;
if (pq_factorize(res_pq->pq_, &p, &q) == -1) {
  return Status::Error("Failed to factorize");
}
```
(`Handshake.cpp:105-109`)

`pq` is a product of two primes. You must find them, and the protocol requires
**`p < q`**.

### The numbers involved

`pq` is normally 8 bytes with the top bit clear, i.e. `pq < 2⁶³`, and `p`, `q` are each
around 2³¹. That is small enough for Pollard's rho in a few milliseconds.

`pq_factorize` (`tdutils/td/utils/crypto.cpp:233-257`) dispatches on size:

```cpp
size_t size = pq_str.size();
if (size > 8 || (size == 8 && (pq_str.begin()[0] & 128) != 0)) {
  return pq_factorize_big(pq_str, p_str, q_str);       // BIGNUM path
}
// else: parse as uint64 big-endian, use the 64-bit path
```

### The 64-bit algorithm

`crypto.cpp:104-141` — Pollard's rho with Brent's cycle detection:

```cpp
uint64 pq_factorize(uint64 pq) {
  if (pq <= 2 || pq > (1ULL << 63)) return 1;
  if ((pq & 1) == 0) return 2;
  uint64 g = 0;
  for (int i = 0, iter = 0; i < 3 || iter < 1000; i++) {
    uint64 q = Random::fast(17, 32) % (pq - 1);        // the additive constant
    uint64 x = Random::fast_uint64() % (pq - 1) + 1;   // random start
    uint64 y = x;
    int lim = 1 << (min(5, i) + 18);
    for (int j = 1; j < lim; j++) {
      iter++;
      x = pq_add_mul(q, x, x, pq);                     // x = (x*x + q) mod pq
      uint64 z = x < y ? pq + x - y : x - y;
      g = pq_gcd(z, pq);
      if (g != 1) break;
      if (!(j & (j - 1))) y = x;                       // Brent: y = x at powers of 2
    }
    if (g > 1 && g < pq) break;
  }
  if (g != 0) {
    uint64 other = pq / g;
    if (other < g) g = other;                          // ensure the SMALLER factor
  }
  return g;
}
```

The final `if (other < g) g = other;` is what guarantees `p < q`.

`pq_add_mul` is a mulmod that avoids 128-bit overflow
(`crypto.cpp:78-92`); on a modern 64-bit target you can just use `__int128`.

### Output format

```cpp
*p_str = as_big_endian_string(p);
*q_str = as_big_endian_string(pq / p);
```
(`crypto.cpp:249-250`)

`as_big_endian_string` (`crypto.cpp:158-170`) strips trailing zero bytes of the
little-endian representation before reversing, producing a **minimal big-endian** string
with **no leading zeros**. So a `p` of `0x494C553B` becomes exactly `49 4C 55 3B`, four
bytes — not eight.

> **💡 Implementation note.** Getting this wrong is a common bug. If you send `p` as
> 8 bytes with leading zeros the server will reject `req_DH_params`. Always emit the
> minimal big-endian encoding. The same applies to `q` and to `pq` if you echo it.

### A simpler implementation

You do not have to copy TDLib's algorithm. Any correct factorization works:

```
# Trial division is adequate: p and q are ~2^31, so you would need ~46000 primes
# up to 2^16 only if p were small — which it is not. Prefer Pollard's rho.

pollard_rho(n):
    if n % 2 == 0: return 2
    loop:
        c = random(1, n-1)
        f = |x| (x*x + c) mod n
        x = y = random(0, n-1); d = 1
        while d == 1:
            x = f(x)
            y = f(f(y))
            d = gcd(|x - y|, n)
        if d != n: return d
```

Use 128-bit intermediate products (`unsigned __int128` in C, `u128` in Rust) so
`x*x` does not overflow.

---

## 5. Generating `new_nonce`

```cpp
Random::secure_bytes(new_nonce_.raw, sizeof(new_nonce_));
```
(`Handshake.cpp:111`)

32 bytes from a **cryptographically secure** RNG. This is the single most important secret
in the handshake: it protects the DH exchange, and later it produces the `server_salt`.

> **⚠ Security note.** Do not use `rand()`, `std::mt19937`, or a time seed. If an attacker
> can predict `new_nonce`, they can derive the temporary AES key from the (public)
> `server_nonce` and read the DH parameters — enabling a man-in-the-middle. Use
> `getrandom(2)` / `/dev/urandom` / `BCryptGenRandom` / Rust's `rand::rngs::OsRng`.

---

## 6. Checklist for step 1

- [ ] 16-byte `nonce` from a CSPRNG
- [ ] Sent `0xbe7e8ef1 ‖ nonce` as an unencrypted message with `auth_key_id = 0`
- [ ] Reply constructor id is `0x05162463`
- [ ] `resPQ.nonce == nonce`  ← **abort if not**
- [ ] Saved `server_nonce`
- [ ] Found a fingerprint match among your hardcoded RSA keys  ← **abort if not**
- [ ] Factorized `pq` into `p < q`
- [ ] `p` and `q` encoded as minimal big-endian strings (no leading zeros)
- [ ] 32-byte `new_nonce` from a CSPRNG

---

[← Previous](01-overview.md) · [Index](../README.md) · [Next: Step 2, req_DH_params →](03-step2-req-dh-params.md)
