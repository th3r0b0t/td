# 1.2 — Wire Encoding, Type by Type

[← Previous](01-tl-language.md) · [Index](../README.md) · [Next: Worked examples →](03-worked-examples.md)

---

This chapter is the complete specification of how each TL type becomes bytes. Implement
these functions once, test them against the examples in
[1.3](03-worked-examples.md), and never think about them again.

The reference is `tdutils/td/utils/tl_storers.h` (writing) and
`tdutils/td/utils/tl_parsers.h` (reading).

---

## 1. Integers: `int`, `long`, `double`

All fixed-size numeric types are stored as a raw little-endian memory image
(`tl_storers.h:28-32`):

```cpp
template <class T>
void store_binary(const T &x) {
  std::memcpy(buf_, &x, sizeof(T));
  buf_ += sizeof(T);
}
```

| Type | Bytes | Encoding |
|------|-------|----------|
| `int` | 4 | two's-complement, little-endian |
| `long` | 8 | two's-complement, little-endian |
| `double` | 8 | IEEE 754 binary64, little-endian |
| `#` (nat) | 4 | unsigned, little-endian |
| `Bool` | 4 | `boolTrue` = `0x997275b5`, `boolFalse` = `0xbc799737` |

Examples:

```
int    1          →  01 00 00 00
int   -1          →  FF FF FF FF
int    229        →  E5 00 00 00
long   1          →  01 00 00 00 00 00 00 00
```

> **💡 Implementation note.** On a big-endian machine this `memcpy` would produce
> big-endian output, which would be a bug. Telegram's wire format is little-endian
> regardless of host. Write explicit `to_le32` / `to_le64` helpers rather than casting
> structs. Practically every deployment target is little-endian, which is why TDLib gets
> away with `memcpy`, but do not rely on it.

---

## 2. Fixed-size opaque values: `int128`, `int256`

Stored as raw bytes, **in the order you produced them**, with no length prefix, no padding,
and no byte swapping.

| Type | Bytes |
|------|-------|
| `int128` | 16 |
| `int256` | 32 |

These are used for nonces (`nonce`, `server_nonce`, `new_nonce`, `new_nonce_hash1`) and
for the `msg_key`. Generate them with a CSPRNG, store them in a `uint8[16]`/`uint8[32]`,
and copy them verbatim.

---

## 3. Strings and bytes

`string` and `bytes` share one encoding, with three length forms
(`tl_storers.h:52-88`):

### Short form — length `< 254`

```
┌────────┬─────────────┬────────────┐
│ len    │ data        │ 0–3 zeros  │
│ 1 byte │ len bytes   │ padding    │
└────────┴─────────────┴────────────┘
```

Total length (`1 + len`) is then zero-padded up to a multiple of 4.

### Medium form — `254 ≤ length < 2²⁴`

```
┌────────┬──────────────────┬───────────┬───────────┐
│ 0xFE   │ len (3 bytes LE) │ data      │ 0–3 zeros │
└────────┴──────────────────┴───────────┴───────────┘
```

Total (`4 + len`) padded to a multiple of 4.

### Long form — `2²⁴ ≤ length < 2³²`

```
┌────────┬──────────────────┬─────────┬───────────┬───────────┐
│ 0xFF   │ len (4 bytes LE) │ 00 00 00│ data      │ 0–3 zeros │
└────────┴──────────────────┴─────────┴───────────┴───────────┘
```

Total (`8 + len`) padded to a multiple of 4.

The exact padding logic (`tl_storers.h:78-87`) is: after writing prefix+data, let
`len` be the *prefix-inclusive* length in the short case (note `len++` at
`tl_storers.h:57`); write `(4 − len mod 4) mod 4` zero bytes.

The length calculation mirror (`tl_storers.h:124-136`) states it plainly:

```cpp
size_t add = str.size();
if (add < 254)            add += 1;
else if (add < (1 << 24)) add += 4;
else                      add += 8;
add = (add + 3) & -4;      // round up to a multiple of 4
```

### Examples

```
""            →  00 00 00 00
"a"           →  01 61 00 00
"abc"         →  03 61 62 63
"abcd"        →  04 61 62 63 64  00 00 00
253 bytes     →  FD <253 bytes> 00 00                (1+253 = 254 → pad 2)
254 bytes     →  FE FE 00 00 <254 bytes> 00 00       (4+254 = 258 → pad 2)
```

> **💡 Implementation note.** The `0xFF` long form is essentially unreachable in practice
> (Telegram caps message payloads far below 16 MB) but costs three lines to support.
> Implement it anyway; the *parser* side must at least not crash on it.

### Reading

`tl_parsers.h:126-162`: read the first byte; if `< 254` it is the length; if `== 254` read
3 more bytes as the length; if `== 255` read 4 more (plus 3 ignored). Then skip forward to
the next multiple of 4:

```cpp
result_aligned_len = ((result_len + 3) >> 2) << 2;
```

> **⚠ Security note.** Validate the decoded length against the number of bytes actually
> remaining in the buffer **before** allocating or copying. A malicious or corrupt packet
> can claim a 4 GB string. TDLib's parser tracks `left_len_` and sets an error status
> instead of reading out of bounds; do the same.

---

## 4. Vectors

The boxed form `Vector<T>`:

```
┌───────────────┬────────────┬──────────────────────┐
│ 0x1cb5c415    │ count:int  │ count × element      │
└───────────────┴────────────┴──────────────────────┘
```

The bare form `vector<T>` (lowercase, as in `future_salts.salts`) omits the `0x1cb5c415`
tag and starts directly with the count.

Elements are serialized according to `T`'s own rules — boxed if `T` is an uppercase type,
bare if `T` is a primitive.

`td/tl/tl_object_store.h:67-75`:

```cpp
storer.store_binary(narrow_cast<int32>(vec.size()));
for (auto &e : vec) { /* store each element */ }
```

Example — `Vector<long>` containing `{1, 2}`:

```
15 C4 B5 1C   02 00 00 00   01 00 00 00 00 00 00 00   02 00 00 00 00 00 00 00
└─ 0x1cb5c415┘└── count ──┘ └────── elem[0] ───────┘ └────── elem[1] ───────┘
```

> **⚠ Security note.** Bound-check `count` before allocating. `count` is a signed 32-bit
> value read straight off the wire; a hostile server (or a corrupted decryption) can send
> `0x7fffffff`. Reject any count that exceeds the bytes remaining divided by the minimum
> element size.

---

## 5. Flags

A `flags:#` field is a plain 4-byte little-endian unsigned integer. What matters is the
discipline around it:

**Writing:**
1. Start with `flags = 0`.
2. For each optional field you intend to include, set its bit.
3. For each `flags.N?true` field that is true, set bit `N`.
4. Write `flags` as an `int`.
5. Write each present field, in **schema declaration order**, skipping absent ones and
   skipping all `?true` fields entirely.

**Reading:**
1. Read `flags`.
2. For each field in declaration order, if its bit is set, parse it; otherwise skip.
3. `?true` fields consume no bytes; their value is the bit.
4. Reject unknown bits (see [1.1 §5](01-tl-language.md)).

Multi-word flags (`flags2:#`) behave identically, with fields written `flags2.N?Type`.

---

## 6. Boxed objects

Write the 4-byte constructor id, then the fields in declaration order. That is the entire
rule (`td/tl/tl_object_store.h:19-26`):

```cpp
storer.store_binary(constructor_id);
Func::store(x, storer);
```

Nested objects are simply written inline where the field appears. There is no length
prefix around an object — the reader knows how long it is because it knows the schema.

This is why a mis-serialized nested object corrupts everything after it: there is no
framing to resynchronize on.

---

## 7. The complete encoder, in pseudocode

```
fn write_int(buf, v:i32)      = buf.extend(v.to_le_bytes())
fn write_long(buf, v:i64)     = buf.extend(v.to_le_bytes())
fn write_double(buf, v:f64)   = buf.extend(v.to_bits().to_le_bytes())
fn write_int128(buf, v:[u8;16]) = buf.extend(v)
fn write_int256(buf, v:[u8;32]) = buf.extend(v)

fn write_bytes(buf, data):
    n = data.len()
    if n < 254:
        buf.push(n as u8)
        prefix = 1
    else if n < 1<<24:
        buf.push(254); buf.extend(n.to_le_bytes()[0..3])
        prefix = 4
    else:
        buf.push(255); buf.extend(n.to_le_bytes()[0..4]); buf.extend([0,0,0])
        prefix = 8
    buf.extend(data)
    pad = (4 - (prefix + n) % 4) % 4
    buf.extend(repeat(0, pad))

fn write_string(buf, s) = write_bytes(buf, s.as_utf8_bytes())

fn write_vector(buf, items, write_elem):
    write_int(buf, 0x1cb5c415)
    write_int(buf, items.len())
    for it in items: write_elem(buf, it)

fn write_bare_vector(buf, items, write_elem):
    write_int(buf, items.len())
    for it in items: write_elem(buf, it)
```

And the decoder:

```
fn read_bytes(cur) -> Vec<u8>:
    first = cur.read_u8()
    if first < 254:
        n = first; prefix = 1
    else if first == 254:
        n = cur.read_u24_le(); prefix = 4
    else:
        n = cur.read_u32_le(); cur.skip(3); prefix = 8
    require(cur.remaining() >= n)          // MANDATORY
    data = cur.read(n)
    cur.skip((4 - (prefix + n) % 4) % 4)
    return data
```

---

## 8. Big-endian exceptions — memorize these

TL is little-endian. These values are **big-endian byte strings** carried inside TL
`string`/`bytes` fields, and must not be byte-swapped:

| Field | Where | Notes |
|-------|-------|-------|
| `pq` | `resPQ` | Big-endian integer, ≤ 8 bytes in practice |
| `p`, `q` | `req_DH_params`, `p_q_inner_data*` | Big-endian, minimal length (no leading zeros) |
| `dh_prime` | `server_DH_inner_data` | Big-endian, exactly 256 bytes |
| `g_a` | `server_DH_inner_data` | Big-endian |
| `g_b` | `client_DH_inner_data` | Big-endian |
| `encrypted_data` | `req_DH_params` | Opaque, 256 bytes |
| `A`, `M1` | `inputCheckPasswordSRP` | Big-endian / raw digest |
| `srp_B`, `p` (SRP) | `account.password` | Big-endian |

Everything else — `msg_id`, `seq_no`, `salt`, `session_id`, `flags`, all `int`/`long`
fields — is little-endian.

`td/mtproto/RSA.cpp:144` (`y.to_binary(256)`) and
`td/mtproto/DhHandshake.cpp:220` (`get_g_ab().to_binary(2048 / 8)`) show TDLib producing
these fixed-width big-endian buffers.

> **💡 Implementation note.** `to_binary()` with no argument produces the *minimal*
> big-endian encoding (no leading zero bytes); `to_binary(N)` left-pads with zeros to
> exactly `N` bytes. TDLib uses the minimal form for `p`, `q` and `g_b`
> (`DhHandshake.cpp:173-176`) and the fixed form for the auth key
> (`DhHandshake.cpp:220`) and the SRP `A` value. Getting this wrong for the auth key gives
> you a key of the wrong length and every subsequent decryption fails.

---

[← Previous](01-tl-language.md) · [Index](../README.md) · [Next: Worked examples →](03-worked-examples.md)
