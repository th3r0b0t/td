# Appendix B — Datacenters and RSA Public Keys

[← Previous](A-constructor-ids.md) · [Index](../README.md) · [Next: Appendix C →](C-error-reference.md)

---

Bootstrap data: the addresses you connect to before you know anything, and the RSA keys
that authenticate the handshake.

---

## B.1 — Bootstrap addresses

From `ConnectionCreator::get_default_dc_options` (`td/telegram/net/ConnectionCreator.cpp:1244-1279`).

### Production, IPv4

| DC | Address | Nickname |
|----|---------|----------|
| 1 | `149.154.175.50` | pluto |
| 2 | `149.154.167.51` | venus |
| 2 | `95.161.76.100` | venus (alternate) |
| 3 | `149.154.175.100` | aurora |
| 4 | `149.154.167.91` | vesta |
| 5 | `149.154.171.5` | flora |

### Production, IPv6

| DC | Address |
|----|---------|
| 1 | `2001:b28:f23d:f001::a` |
| 2 | `2001:67c:4e8:f002::a` |
| 3 | `2001:b28:f23d:f003::a` |
| 4 | `2001:67c:4e8:f004::a` |
| 5 | `2001:b28:f23f:f005::a` |

Note DC 5's IPv6 prefix is `f23f`, not `f23d` — this is not a typo in this document, it is
what the source says at `ConnectionCreator.cpp:1277`.

### Test, IPv4

| DC | Address |
|----|---------|
| 1 | `149.154.175.10` |
| 2 | `149.154.167.40` |
| 3 | `149.154.175.117` |

### Test, IPv6

| DC | Address |
|----|---------|
| 1 | `2001:b28:f23d:f001::e` |
| 2 | `2001:67c:4e8:f002::e` |
| 3 | `2001:b28:f23d:f003::e` |

The production addresses end in `::a`, the test ones in `::e`. Only DCs 1-3 exist on test.

### Ports

```cpp
vector<int> ports = {443, 80, 5222};
```
(`ConnectionCreator.cpp:1244`)

Try 443 first — it looks like HTTPS and survives most firewalls. Then 80, then 5222.

### Address rotation

TDLib shuffles the address list before use (`ConnectionCreator.cpp:1226`):

```cpp
Random::shuffle(ip_address_strings);
```

This spreads load and avoids every client hammering the first address. Do the same, and
remember which address/port combination worked so you can prefer it next time.

### Web addresses

For completeness, the Emscripten build uses WebSocket URLs instead
(`ConnectionCreator.cpp:1245-1256`): `pluto`, `venus`, `aurora`, `vesta`, `flora`
`.web.telegram.org/apiws` (with `_test` suffix for the test DCs). Irrelevant for a native
client.

> **💡 These are bootstrap values only.** Call `help.getConfig`
> ([6.3](../06-api-layer/03-migration-and-multi-dc.md)) after connecting and use the
> `dc_options` from the reply. Hard-coded addresses do change.

---

## B.2 — RSA public keys

From `PublicRsaKeySharedMain::create` (`td/telegram/net/PublicRsaKeySharedMain.cpp:14-52`).

TDLib ships **exactly one key per environment**. Older documentation lists four production
keys; the current implementation uses one.

### Production

```
-----BEGIN RSA PUBLIC KEY-----
MIIBCgKCAQEA6LszBcC1LGzyr992NzE0ieY+BSaOW622Aa9Bd4ZHLl+TuFQ4lo4g
5nKaMBwK/BIb9xUfg0Q29/2mgIR6Zr9krM7HjuIcCzFvDtr+L0GQjae9H0pRB2OO
62cECs5HKhT5DZ98K33vmWiLowc621dQuwKWSQKjWf50XYFw42h21P2KXUGyp2y/
+aEyZ+uVgLLQbRA1dEjSDZ2iGRy12Mk5gpYc397aYp438fsJoHIgJ2lgMv5h7WY9
t6N/byY9Nw9p21Og3AoXSL2q/2IJ1WRUhebgAdGVMlV1fkuOQoEzR7EdpqtQD9Cs
5+bfo3Nhmcyvk5ftB0WkJ9z6bNZ7yxrP8wIDAQAB
-----END RSA PUBLIC KEY-----
```
(`PublicRsaKeySharedMain.cpp:40-47`)

### Test

```
-----BEGIN RSA PUBLIC KEY-----
MIIBCgKCAQEAyMEdY1aR+sCR3ZSJrtztKTKqigvO/vBfqACJLZtS7QMgCGXJ6XIR
yy7mx66W0/sOFa7/1mAZtEoIokDP3ShoqF4fVNb6XeqgQfaUHd8wJpDWHcR2OFwv
plUUI1PLTktZ9uW2WE23b+ixNwJjJGwBDJPQEQFBE+vfmH0JP503wr5INS1poWg/
j25sIWeYPHYeOrFp/eXaqhISP6G+q2IeTaWTXpwZj4LzXq5YOpk4bYEQ6mvRq7D1
aHWfYmlEGepfaYR8Q0YqvvhYtMte3ITnuSJs171+GDqpdKcSwHnd6FudwGO4pcCO
j4WcDuXc2CTHgH8gFTNhp/Y8/SpDOhvn9QIDAQAB
-----END RSA PUBLIC KEY-----
```
(`PublicRsaKeySharedMain.cpp:25-32`)

> **⚠ The test DCs use the test key.** Using the production key against a test DC gives you
> a fingerprint that appears in no `resPQ`, and your handshake stalls at
> `req_DH_params`. This is one of the most common early failures.

Both are `BEGIN RSA PUBLIC KEY` (PKCS#1), not `BEGIN PUBLIC KEY` (SubjectPublicKeyInfo).
OpenSSL's `PEM_read_bio_RSAPublicKey` reads PKCS#1; `PEM_read_bio_PUBKEY` does not.

Both keys are 2048-bit, so `RSA::size()` returns 256 (`td/mtproto/RSA.cpp:125-128`).

---

## B.3 — Computing a fingerprint

`resPQ.server_public_key_fingerprints` is a `Vector<long>`. You must match one of those
values against your keys to know which to encrypt with.

`RSA::get_fingerprint` (`td/mtproto/RSA.cpp:111-123`):

```cpp
string n_str = n_.to_binary();
string e_str = e_.to_binary();
mtproto_api::rsa_public_key public_key(n_str, e_str);
size_t size = tl_calc_length(public_key);
std::vector<unsigned char> tmp(size);
size = tl_store_unsafe(public_key, tmp.data());
unsigned char key_sha1[20];
sha1(Slice(tmp.data(), tmp.size()), key_sha1);
return as<int64>(key_sha1 + 12);
```

In words:

```
1. n = modulus,  as MINIMAL big-endian bytes  (to_binary(), no fixed width)
2. e = exponent, as MINIMAL big-endian bytes  (usually 3 bytes: 01 00 01)
3. Serialize rsa_public_key{n, e} as TL — BARE, no constructor id
4. fingerprint = SHA1(serialized)[12..20)  read as a LITTLE-ENDIAN int64
```

Three details that trip people up:

**The constructor is bare.** `mtproto_api.tl:65` declares it without an id:

```
rsa_public_key n:string e:string;
```

So the serialization is just the two TL strings, with **no** 4-byte constructor prefix. See
[1.1](../01-serialization/01-tl-language.md) on bare versus boxed.

**`n` and `e` are minimal.** `BigNum::to_binary()` with no argument emits the shortest
big-endian representation — no leading zero byte. For a 2048-bit modulus with the top bit
set, that is 256 bytes; `e = 65537` is 3 bytes.

**The last 8 bytes are read as little-endian.** `as<int64>(key_sha1 + 12)` is a raw
reinterpretation of `SHA1(...)[12..20)`, which on any supported platform is a
little-endian load. Compare it against the `long` values in
`server_public_key_fingerprints` as-is.

### Worked shape

For the production key, the serialized `rsa_public_key` is:

```
fe 00 01 00                  string length 256 (0xFE + 3-byte LE length)
6c bb 33 05 c1 b5 2c 6c …    n, 256 bytes big-endian
                             (256 % 4 == 0, so no padding)
03 01 00 01                  string length 3, then e = 010001
                             (3 data bytes + 1 length byte = 4, no padding)
```

Total 264 bytes. SHA-1 that, take bytes 12-19, read as little-endian `int64`.

> **💡 Compute the fingerprint at startup**, don't hard-code it. If Telegram rotates keys
> you only need to swap the PEM, and computing it exercises your TL serializer as a bonus
> self-test.

---

## B.4 — Choosing a key

`PublicRsaKeySharedMain::get_rsa_key` (`PublicRsaKeySharedMain.cpp:54+`) walks the
fingerprint list from `resPQ` and returns the first key it recognizes.

```
for fp in resPQ.server_public_key_fingerprints:
    if fp in my_keys:
        use my_keys[fp]
        break
else:
    fatal — no known key
```

If no fingerprint matches, stop. Do not guess, and do not encrypt with an unmatched key —
the server will reject it and you will have spent a round trip learning nothing.

---

## B.5 — Test accounts

Test datacenters accept synthetic phone numbers:

```
99966XYYYY
```

where `X` is the DC number and `YYYY` is arbitrary. TDLib's tests use `9996636437` and
`9996636438` (`test/tdclient.cpp:676-677`).

The login code is the DC number repeated to the required length — `22222` for DC 2.

No SMS is sent, and rate limits are relaxed. Do all early development here. See
[10.4](../10-implementation/04-testing-and-debugging.md).

---

## B.6 — Quick reference

```
Production DC 2:  149.154.167.51:443
Test DC 2:        149.154.167.40:443
Test phone:       9996626666        (DC 2)
Test code:        22222
Layer:            229               (td/telegram/Version.h:13)
```

---

[← Previous](A-constructor-ids.md) · [Index](../README.md) · [Next: Appendix C →](C-error-reference.md)
