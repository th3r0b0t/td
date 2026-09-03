# 3.6 — Worked Example: A Complete Handshake

[← Previous](05-security-checks.md) · [Index](../README.md) · [Next: The encrypted envelope →](../04-encrypted-messages/01-envelope.md)

---

A byte-level trace of one handshake. Values are illustrative but structurally exact —
lengths, offsets and orderings match what the reference implementation produces. Use this
as a template for your own hexdump comparisons.

Notation: `‖` is concatenation, `x[i..j)` is a half-open byte range.

---

## Round trip 1

### → `req_pq_multi`

Chosen randomly:

```
nonce = 3E 05 49 82 8C CA 27 E9  66 B3 01 A4 8F EC E2 FC
```

TL body (20 bytes):

```
f1 8e 7e be                                       constructor 0xbe7e8ef1
3e 05 49 82 8c ca 27 e9 66 b3 01 a4 8f ec e2 fc   nonce
```

Unencrypted MTProto message:

```
00 00 00 00 00 00 00 00                           auth_key_id = 0
51 e5 7a c4 2c 6c 0d 68                           msg_id
14 00 00 00                                       message_data_length = 20
f1 8e 7e be 3e 05 ... e2 fc                       data (20 bytes)
<12 random bytes>                                 pad → 32 (multiple of 16)
```

Framed (Intermediate):

```
38 00 00 00                                       length = 56
<the 56 bytes above>
```

Total on the wire: 60 bytes.

### ← `resPQ`

```
00 00 00 00 00 00 00 00                           auth_key_id = 0
01 c8 83 1e c9 7a e5 51                           msg_id (server)
64 00 00 00                                       message_data_length = 100
63 24 16 05                                       constructor 0x05162463
3e 05 49 82 8c ca 27 e9 66 b3 01 a4 8f ec e2 fc   nonce  ← MUST match
a5 cf 4d 33 f4 a1 1e a8 77 ba 4a a5 73 90 73 30   server_nonce
08 17 ed 48 94 1a 08 f9 81 00 00 00               pq: len=8, 17ED48941A08F981, 3 pad
15 c4 b5 1c 01 00 00 00                           Vector<long>, count = 1
21 6b e8 6c 02 2b b4 c3                           fingerprint (LE int64)
```

`pq = 0x17ED48941A08F981` = 1724114033281923457.

### Client work

Factorize: `p = 0x494C553B` (1229739835), `q = 0x53911073` (1394193523).
Check `p < q` ✓.

Encoded minimally, big-endian:

```
p = 49 4C 55 3B         (4 bytes, no leading zeros)
q = 53 91 10 73         (4 bytes)
```

Look up fingerprint `0xc3b42b026ce86b21` among your hardcoded keys → the production key
([appendix B](../appendix/B-datacenters-and-keys.md)).

Generate:

```
new_nonce = 31 1C 85 DB 23 4A A2 64 0A FC 4A 76 A7 35 CF 5B
            1F 0F D6 8B D1 7F A1 81 E1 F5 B0 43 F9 19 CD 7F
```

---

## Round trip 2

### Build `p_q_inner_data_dc`

```
95 5f f5 a9                                       constructor 0xa9f55f95
08 17 ed 48 94 1a 08 f9 81 00 00 00               pq
04 49 4c 55 3b 00 00 00                           p   (len 4, 3 pad)
04 53 91 10 73 00 00 00                           q
3e 05 ... e2 fc                                   nonce        (16)
a5 cf ... 73 30                                   server_nonce (16)
31 1c ... cd 7f                                   new_nonce    (32)
02 00 00 00                                       dc = 2
```

Total: 4 + 12 + 8 + 8 + 16 + 16 + 32 + 4 = **100 bytes**. Under the 144 limit ✓.

### RSA_PAD

```
1.  data = <100 bytes above> ‖ random(92)                    → 192 bytes

2.  aes_key = <32 random bytes>

3.  h = SHA256(aes_key ‖ data)                               → 32 bytes
    data_with_hash = reverse(data) ‖ h                       → 224 bytes
       ^^^^^^^^^^^^ reverse exactly the 192-byte data part

4.  ct = AES256_IGE_encrypt(data_with_hash,
                            key = aes_key,
                            iv  = 00 × 32)                   → 224 bytes
    out[32..256) = ct

5.  out[0..32) = aes_key XOR SHA256(ct)

6.  x = big_endian_int(out)                                  → 2048-bit
    if x >= n: goto 2                                        ← retry loop
    encrypted_data = (x^e mod n) as 256 bytes
```

### → `req_DH_params`

```
be e4 12 d7                                       constructor 0xd712e4be
3e 05 ... e2 fc                                   nonce
a5 cf ... 73 30                                   server_nonce
04 49 4c 55 3b 00 00 00                           p
04 53 91 10 73 00 00 00                           q
21 6b e8 6c 02 2b b4 c3                           fingerprint
fe 00 01 00 <256 bytes>                           encrypted_data:
                                                    0xFE, len 256 as 3-byte LE,
                                                    then 256 bytes, 0 pad
```

Body size: 4 + 16 + 16 + 8 + 8 + 8 + 4 + 256 = 320 bytes.

> Note the `0xFE` long-string form: 256 ≥ 254, so the length uses the 4-byte prefix
> (`FE 00 01 00` = 0xFE + 0x000100 LE = 256). `4 + 256 = 260`, and `260 mod 4 == 0`, so
> there is no trailing padding. See [1.2 §3](../01-serialization/02-wire-encoding.md).

### ← `server_DH_params_ok`

```
5c 07 e8 d0                                       constructor 0xd0e8075c
3e 05 ... e2 fc                                   nonce         ← check
a5 cf ... 73 30                                   server_nonce  ← check
fe 40 02 00 <592 bytes>                           encrypted_answer (multiple of 16 ✓)
```

### Decrypt

```
A = SHA1(new_nonce ‖ server_nonce)          20 bytes
B = SHA1(server_nonce ‖ new_nonce)          20 bytes
C = SHA1(new_nonce ‖ new_nonce)             20 bytes

tmp_aes_key = A[0..20) ‖ B[0..12)                        32 bytes
tmp_aes_iv  = B[12..20) ‖ C[0..20) ‖ new_nonce[0..4)     32 bytes

answer = AES256_IGE_decrypt(encrypted_answer, tmp_aes_key, tmp_aes_iv)
```

Plaintext:

```
[  0.. 20)   SHA1 of the inner data
[ 20.. 24)   ba 90 89 b5                    constructor 0xb5890dba
[ 24.. 40)   nonce                          ← check
[ 40.. 56)   server_nonce                   ← check
[ 56.. 60)   03 00 00 00                    g = 3
[ 60..320)   fe 00 01 00 <256 bytes>        dh_prime
[320..580)   fe 00 01 00 <256 bytes>        g_a
[580..584)   <int32>                        server_time
[584..592)   <0..15 random>                 padding, must be < 16 bytes
```

Verify `SHA1(answer[20 .. 20+564))` — the parsed extent, excluding padding — equals
`answer[0..20)`.

### Validate DH parameters

```
p is 2048 bits                                    ✓
g == 3  →  p mod 3 == 2                           ✓
p is prime                                        ✓  (cached)
(p-1)/2 is prime                                  ✓  (cached)
2^1984 <= g_a <= p - 2^1984                       ✓
```

`server_time_diff = server_time − now()`. Store it.

---

## Round trip 3

### Compute `g_b`

```
b   = <256 random bytes>                          2048-bit
g_b = 3^b mod p                                   minimal big-endian, ~256 bytes
check 2^1984 <= g_b <= p - 2^1984                 ✓
```

### Build and encrypt `client_DH_inner_data`

```
54 b6 43 66                                       constructor 0x6643b654
3e 05 ... e2 fc                                   nonce
a5 cf ... 73 30                                   server_nonce
00 00 00 00 00 00 00 00                           retry_id = 0
fe 00 01 00 <256 bytes>                           g_b
```

Body: 4 + 16 + 16 + 8 + 260 = **304 bytes**.

```
plain = SHA1(body) ‖ body ‖ random_pad
        └── 20 ──┘ └─ 304 ┘
        20 + 304 = 324 → pad to 336 (next multiple of 16) → 12 random bytes

encrypted_data = AES256_IGE_encrypt(plain, tmp_aes_key, tmp_aes_iv)
                                                  ↑ FRESH values, not the mutated IV
```

### → `set_client_DH_params`

```
1f 5f 04 f5                                       constructor 0xf5045f1f
3e 05 ... e2 fc                                   nonce
a5 cf ... 73 30                                   server_nonce
fe 50 01 00 <336 bytes>                           encrypted_data
```

### ← `dh_gen_ok`

```
34 f7 cb 3b                                       constructor 0x3bcbf734
3e 05 ... e2 fc                                   nonce         ← check
a5 cf ... 73 30                                   server_nonce  ← check
<16 bytes>                                        new_nonce_hash1
```

### Final computation

```
auth_key    = g_a^b mod p, as EXACTLY 256 bytes (left-zero-padded)
sha         = SHA1(auth_key)                                  20 bytes
auth_key_id = sha[12..20)                                      8 bytes
aux_hash    = sha[0..8)                                        8 bytes

expected = SHA1(new_nonce ‖ 01 ‖ aux_hash)[4..20)             16 bytes
assert expected == new_nonce_hash1

server_salt = new_nonce[0..8) XOR server_nonce[0..8)
            = 31 1C 85 DB 23 4A A2 64
            ⊕ A5 CF 4D 33 F4 A1 1E A8
            = 94 D3 C8 E8 D7 EB BC CC
```

Handshake complete.

---

## Byte-count reference

| Message | TL body | Padded | Framed |
|---------|---------|--------|--------|
| `req_pq_multi` | 20 | 32 | 60 |
| `resPQ` | 100 | 112 | 140 |
| `req_DH_params` | 320 | 320 | 348 |
| `server_DH_params_ok` | 632 | 640 | 668 |
| `set_client_DH_params` | 376 | 384 | 412 |
| `dh_gen_ok` | 52 | 64 | 92 |

("Padded" includes the 20-byte unencrypted header; "framed" adds the 4-byte Intermediate
length. Exact numbers vary with the random padding TDLib adds — see
[3.1 §5](01-overview.md).)

---

## Debugging table

| Symptom | Most likely cause |
|---------|-------------------|
| No reply to `req_pq_multi` | Framing magic missing or wrong |
| `resPQ` nonce mismatch | You byte-swapped `int128`; it is an opaque array |
| `server_DH_params_fail` | `p`/`q` wrong, or `p ≥ q`, or leading zeros in `p`/`q` |
| No reply to `req_DH_params` | `encrypted_data` not exactly 256 bytes |
| Decrypted answer is garbage | `tmp_KDF` operand order swapped |
| "Too much pad" (≥ 16 leftover) | Same as above — decryption failed silently |
| SHA1 mismatch | Hashing the padding too, or hashing from offset 0 instead of 20 |
| `dh_gen_retry` / `dh_gen_fail` | Wrong `g_b`, or you reused the mutated `tmp_aes_iv` |
| `new_nonce_hash1` mismatch | Using `SHA1(auth_key)[12..20)` instead of `[0..8)` for `aux_hash`, or taking `[0..16)` instead of `[4..20)` of the outer hash |
| Everything works, then `-404` on ~1 in 256 runs | `auth_key` not left-padded to 256 bytes |

---

## Test vectors

The repository ships handshake tests you can crib from:

* `test/mtproto.cpp` — end-to-end handshake against a real DC, plus
  `TEST(Mtproto, PublicRsaKey)` and `TEST(Mtproto, RSA)`.
* `test/crypto.cpp` — `pq_factorize`, AES-IGE and hash vectors.

Run them with `./test/run_all_tests` after a CMake build (see
[10.4](../10-implementation/04-testing-and-debugging.md)).

---

[← Previous](05-security-checks.md) · [Index](../README.md) · [Next: The encrypted envelope →](../04-encrypted-messages/01-envelope.md)
