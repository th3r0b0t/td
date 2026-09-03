# 3.5 — Security Checks

[← Previous](04-step3-set-client-dh-params.md) · [Index](../README.md) · [Next: Worked example →](06-worked-example.md)

---

Every check in this chapter is performed by TDLib and every one exists to stop a specific
attack. Implementations that skip them "because it works anyway" are not implementing
MTProto; they are implementing an unauthenticated key exchange that happens to interoperate.

---

## 1. Nonce checks

Performed at four points (`Handshake.cpp:92, 171-176, 214-219, 261-266`):

| Where | Check |
|-------|-------|
| `resPQ` | `nonce == our nonce` |
| `server_DH_params_ok` | `nonce == ours` and `server_nonce == from resPQ` |
| `server_DH_inner_data` (inside the encrypted blob) | both again |
| `dh_gen_ok` | both again |

**Attack prevented:** replay of a recorded handshake, and cross-session mixing where an
attacker splices a reply from one handshake into another.

The check inside `server_DH_inner_data` is the important one — the outer ones are on
unauthenticated fields, but the inner ones are covered by the SHA1 and the temporary AES
key, so matching them proves the peer knew `new_nonce`.

---

## 2. RSA key selection

```cpp
auto r_rsa_key = public_rsa_key->get_rsa_key(res_pq->server_public_key_fingerprints_);
if (r_rsa_key.is_error()) { public_rsa_key->drop_keys(); return r_rsa_key.move_as_error(); }
```
(`Handshake.cpp:98-102`)

**You must fail** if none of your hardcoded keys is in the server's list. Never accept a
key supplied over the network, never derive one from the fingerprint.

**Attack prevented:** a man-in-the-middle offering its own public key would be able to
decrypt `new_nonce` and hence the whole DH exchange.

---

## 3. Encrypted-answer integrity

```cpp
if (dh_params->encrypted_answer_.size() & 15) return Status::Error("Bad padding for encrypted part");
…
size_t pad = answer_parser.get_left_len();
if (pad >= 16) return Status::Error("Too much pad");
…
if (answer_sha1 != answer_real_sha1) return Status::Error("SHA1 mismatch");
```
(`Handshake.cpp:177-212`)

Three separate conditions:

1. Ciphertext length is a multiple of 16 (AES block size).
2. After parsing `server_DH_inner_data`, fewer than 16 bytes remain. More would mean the
   decryption produced something that only *looks* parseable.
3. `SHA1(inner_data)` matches the 20-byte prefix, computed over the parsed extent only.

**Attack prevented:** tampering with `g`, `p` or `g_a` in transit.

---

## 4. Diffie–Hellman parameter validation

This is the deepest set of checks, in `DhHandshake::check_config`
(`DhHandshake.cpp:21-93`).

### 4.1 `p` must be exactly 2048 bits

```cpp
if (prime.get_num_bits() != 2048) return Status::Error("p is not 2048-bit number");
```

i.e. `2²⁰⁴⁷ ≤ p < 2²⁰⁴⁸`. Rejects both undersized primes (weak) and malformed input.

### 4.2 `g` must generate a large subgroup

```cpp
switch (g_int) {
  case 2: mod_ok = prime % 8 == 7u; break;
  case 3: mod_ok = prime % 3 == 2u; break;
  case 4: mod_ok = true; break;
  case 5: mod_ok = (mod_r = prime % 5) == 1u || mod_r == 4u; break;
  case 6: mod_ok = (mod_r = prime % 24) == 19u || mod_r == 23u; break;
  case 7: mod_ok = (mod_r = prime % 7) == 3u || mod_r == 5u || mod_r == 6u; break;
  default: mod_ok = false;
}
```

| `g` | Required condition on `p` |
|-----|---------------------------|
| 2 | `p mod 8 == 7` |
| 3 | `p mod 3 == 2` |
| 4 | *(none)* |
| 5 | `p mod 5 ∈ {1, 4}` |
| 6 | `p mod 24 ∈ {19, 23}` |
| 7 | `p mod 7 ∈ {3, 5, 6}` |
| anything else | **reject** |

TDLib's own comment (`DhHandshake.cpp:28-35`) explains why: these conditions are exactly
the quadratic-reciprocity criteria for `g` being a quadratic residue mod `p`, which makes
`g` generate the subgroup of prime order `(p−1)/2` rather than a small one.

**Attack prevented:** a small-subgroup attack. If `g` generated a subgroup of order 2, then
`g^b mod p` would take only two values and the shared secret would be guessable.

### 4.3 `p` must be a safe prime

```cpp
if (!prime.is_prime(ctx)) return Status::Error("p is not a prime number");
BigNum half_prime = prime; half_prime -= 1; half_prime /= 2;
if (!half_prime.is_prime(ctx)) return Status::Error("(p - 1) / 2 is not a prime number");
```

Both `p` and `(p−1)/2` must be prime (Miller–Rabin is what OpenSSL's `BN_is_prime_ex`
does).

**Attack prevented:** if `p−1` had many small factors, Pohlig–Hellman would make the
discrete log tractable and the server could compute your `b`.

> **💡 Implementation note — this is slow.** Two 2048-bit primality tests take on the order
> of a second. TDLib caches results via `DhCallback::is_good_prime` /
> `add_good_prime` / `add_bad_prime` (`DhHandshake.cpp:66-91`,
> implemented in `td/telegram/net/DcOptions.cpp` and persisted). In practice Telegram uses
> a single fixed `p`, so:
>
> 1. Verify it once, offline, thoroughly.
> 2. Hardcode its hash.
> 3. At runtime, if `SHA256(dh_prime)` matches the known-good value, skip the primality
>    tests; otherwise run the full check.
>
> **Do not** skip the check for unknown primes. Do not simply trust whatever the server
> sends.

### 4.4 `g`, `g_a` and `g_b` must be in range

`DhHandshake::dh_check` (`DhHandshake.cpp:95-126`):

```cpp
BigNum left;  left.set_value(0); left.set_bit(2048 - 64);   // left = 2^1984
BigNum right; BigNum::sub(right, prime, left);              // right = p - 2^1984
if (compare(left, g_a) > 0 || compare(g_a, right) > 0 ||
    compare(left, g_b) > 0 || compare(g_b, right) > 0) {
  return Status::Error("g^a or g^b is not between 2^{2048-64} and dh_prime - 2^{2048-64}");
}
```

Required: `2¹⁹⁸⁴ ≤ g_a ≤ p − 2¹⁹⁸⁴` and the same for `g_b`.

The minimum requirement from the protocol is `1 < g_a < p−1`; the `2²⁰⁴⁸⁻⁶⁴` bound is the
*recommended* stronger version, and it is what TDLib enforces. Use it.

**Attack prevented:** `g_a ∈ {0, 1, p−1}` would force the shared secret to a known constant.
The wider margin also rejects values with long runs of leading or trailing zero bits, which
would leak structure.

Note that TDLib checks **its own `g_b` too** — a defensive measure against a broken RNG.

---

## 5. `new_nonce_hash1`

```cpp
sha1(auth_key_.key(), auth_key_sha1.raw);
auto new_nonce_hash = sha1(PSLICE() << new_nonce_.as_slice() << '\x01'
                                    << auth_key_sha1.as_slice().substr(0, 8));
if (dh_gen_ok->new_nonce_hash1_.as_slice() != Slice(new_nonce_hash).substr(4)) {
  return Status::Error("New nonce hash mismatch");
}
```
(`Handshake.cpp:268-273`)

**Attack prevented:** installing an auth key the server does not have. Without this you
would only discover the mismatch later, as an opaque `-404`.

---

## 6. Randomness quality

| Value | Size | Must be |
|-------|------|---------|
| `nonce` | 16 B | CSPRNG |
| `new_nonce` | 32 B | **CSPRNG — critical** |
| `b` | 256 B | **CSPRNG — critical** |
| `aes_key` (RSA_PAD) | 32 B | **CSPRNG — critical** |
| padding bytes | varies | CSPRNG (TDLib uses `Random::secure_bytes` throughout) |
| `session_id` | 8 B | CSPRNG |

TDLib uses `Random::secure_bytes` for **all** of these, including padding
(`Handshake.cpp:111, 134, 138, 237`).

**Attack prevented:** predicting `new_nonce` breaks the temporary AES key and therefore the
whole DH exchange; predicting `b` yields the auth key directly.

> **⚠ Security note.** Do not use `rand()`, `random()`, `std::mt19937`, `Math.random()`,
> a time-seeded PRNG, or a "fast" RNG. Use `getrandom(2)`, `/dev/urandom`,
> `BCryptGenRandom`, `arc4random_buf`, or Rust's `OsRng`. TDLib deliberately has two
> separate generators — `Random::fast*` for non-security use and `Random::secure_*` for
> everything above.

---

## 7. The 256-byte auth key

```cpp
string key = g_ab_.to_binary(2048 / 8);
```
(`DhHandshake.cpp:207`)

Exactly 256 bytes, **left-padded with zeros**. Contrast with every other `to_binary()` in
the handshake, which is minimal.

**Failure mode if you get this wrong:** roughly 1 handshake in 256 produces a `g_a^b mod p`
with a zero high byte. If you store the minimal 255-byte form, `SHA1(auth_key)` differs,
`auth_key_id` differs, and the server answers `-404` — intermittently, unreproducibly, and
only after everything else appeared to work.

The same applies to `to_binary(256)` calls in SRP ([7.5](../07-login/05-two-factor-srp.md)).

---

## 8. Timeouts

```cpp
if (Time::now() >= start_time_ + timeout_in_ * 0.6) return Status::Error("Handshake ResPQ timeout expired");
…
if (Time::now() >= start_time_ + timeout_in_ * 0.8) return Status::Error("Handshake DH params timeout expired");
```
(`Handshake.cpp:87, 164`)

A stalled handshake is indistinguishable from an attack that withholds a reply. Bound it
and restart with fresh randomness.

---

## 9. Complete validation checklist

Copy this into your test suite.

**Step 1 — resPQ**
- [ ] Constructor id `0x05162463`
- [ ] `nonce` matches
- [ ] At least one fingerprint matches a hardcoded key
- [ ] `pq` factorizes; `p < q`; both minimal big-endian

**Step 2 — server_DH_params_ok**
- [ ] Constructor id `0xd0e8075c` (not `server_DH_params_fail`)
- [ ] `nonce` and `server_nonce` match
- [ ] `encrypted_answer.size() % 16 == 0`
- [ ] Inner constructor id `0xb5890dba`
- [ ] Trailing padding < 16 bytes
- [ ] `SHA1(inner) == answer[0..20)`
- [ ] Inner `nonce` and `server_nonce` match
- [ ] `p` is exactly 2048 bits
- [ ] `g ∈ {2,3,4,5,6,7}` and the `p mod 4g` condition holds
- [ ] `p` is prime
- [ ] `(p−1)/2` is prime
- [ ] `2¹⁹⁸⁴ ≤ g_a ≤ p − 2¹⁹⁸⁴`
- [ ] `2¹⁹⁸⁴ ≤ g_b ≤ p − 2¹⁹⁸⁴`

**Step 3 — dh_gen_ok**
- [ ] Constructor id `0x3bcbf734` (not retry `0x46dc1fb9` or fail `0xa69dae02`)
- [ ] `nonce` and `server_nonce` match
- [ ] `new_nonce_hash1 == SHA1(new_nonce ‖ 0x01 ‖ SHA1(auth_key)[0..8))[4..20)`
- [ ] `auth_key` is exactly 256 bytes

**Throughout**
- [ ] All secrets from a CSPRNG
- [ ] Handshake bounded by a timeout
- [ ] `b`, `new_nonce`, `tmp_aes_*` wiped after use

---

[← Previous](04-step3-set-client-dh-params.md) · [Index](../README.md) · [Next: Worked example →](06-worked-example.md)
