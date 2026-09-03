# 2.3 — Obfuscation ("obfuscated2")

[← Previous](02-tcp-framings.md) · [Index](../README.md) · [Next: Transport errors →](04-transport-errors.md)

---

## 1. What it is and when you need it

Plain Intermediate framing is trivially fingerprintable: the connection starts with
`EE EE EE EE` and then every packet begins with a 4-byte length that is a multiple of 4.
Any DPI box can spot it.

**Obfuscation** replaces the fixed magic with a 64-byte pseudo-random header and encrypts
the *entire* subsequent byte stream (framing included) with AES-256-CTR, using keys derived
from that header. To an observer the connection is an unstructured random byte stream from
the first byte.

Note carefully: **this is not additional security**. The obfuscation key travels in clear
in the header — anyone who reads the first 64 bytes can decrypt everything. Its only
purpose is to defeat protocol fingerprinting. Your actual security comes entirely from the
MTProto encryption layer above it.

**You need obfuscation when:**

* connecting through an MTProxy (the `secret` in a `tg://proxy?...` link),
* a `dcOption` carries the `tcpo_only` flag (`telegram_api.tl:536`),
* your network blocks plain MTProto.

**You do not need it** for a first implementation against a normal datacenter. Skip this
chapter, come back later.

All the code below is from `ObfuscatedTransport::init`,
`td/mtproto/TcpTransport.cpp:80-143`.

---

## 2. Building the 64-byte header

### Step 1: generate 64 random bytes, subject to constraints

`TcpTransport.cpp:88-108`:

```cpp
while (true) {
  Random::secure_bytes(header_slice);
  if (secret_.emulate_tls()) break;
  if (as<uint8>(header.data()) == 0xef) continue;
  uint32 first_int = as<uint32>(header.data());
  if (first_int == 0x44414548 || first_int == 0x54534f50 || first_int == 0x20544547 ||
      first_int == 0x4954504f || first_int == 0xdddddddd || first_int == 0xeeeeeeee ||
      first_int == 0x02010316) {
    continue;
  }
  uint32 second_int = as<uint32>(header.data() + sizeof(uint32));
  if (second_int == 0) continue;
  break;
}
```

The forbidden values, and what each would make the stream look like:

| Constraint | Bytes on the wire | Would be mistaken for |
|------------|-------------------|-----------------------|
| `header[0] != 0xEF` | `EF …` | Abridged framing |
| `first_int != 0x44414548` | `48 45 41 44` | HTTP `HEAD` |
| `first_int != 0x54534F50` | `50 4F 53 54` | HTTP `POST` |
| `first_int != 0x20544547` | `47 45 54 20` | HTTP `GET ` |
| `first_int != 0x4954504F` | `4F 50 54 49` | HTTP `OPTI`(ONS) |
| `first_int != 0xDDDDDDDD` | `DD DD DD DD` | Padded Intermediate framing |
| `first_int != 0xEEEEEEEE` | `EE EE EE EE` | Intermediate framing |
| `first_int != 0x02010316` | `16 03 01 02` | A TLS 1.0 handshake record |
| `header[4..8) != 00000000` | — | Reserved / ambiguous |

`first_int` and `second_int` are read as **little-endian** `uint32`s, which is why the
byte sequences above appear reversed relative to the constants.

The chance of hitting any of these randomly is about 2⁻²⁹, so the loop essentially never
iterates — but TDLib still asserts a bound of 10 attempts (`TcpTransport.cpp:90`).

### Step 2: overwrite the protocol and DC fields

`TcpTransport.cpp:109-112`:

```cpp
as<uint32>(header_slice.begin() + 56) = impl_.with_padding() ? 0xdddddddd : 0xeeeeeeee;
if (dc_id_ != 0) {
  as<int16>(header_slice.begin() + 60) = dc_id_;
}
```

Final layout of the 64-byte header:

```
offset  size  content
------  ----  ------------------------------------------------------------
  0     56    random (subject to the constraints above)
 56      4    protocol magic: EE EE EE EE (Intermediate)
                            or DD DD DD DD (Padded Intermediate)
 60      2    dc_id as int16 LE — only if you are talking to a proxy;
              use 0 (i.e. leave random/zero) when connecting directly
 62      2    random
```

> **💡 Implementation note.** The `dc_id` field is how an **MTProxy** knows which
> datacenter to relay you to. When you connect directly to a Telegram server it is
> ignored, and TDLib leaves it as random bytes (`dc_id_ == 0` skips the write). For test
> datacenters, proxies expect `dc_id + 10000`; for media-only, `-dc_id`.

---

## 3. Deriving the AES-CTR keys

`TcpTransport.cpp:114-139`:

```cpp
string rheader = header;
std::reverse(rheader.begin(), rheader.end());          // reverse all 64 bytes

UInt256 key    = as<UInt256>(rheader.data() + 8);      // recv key  = reversed[8..40)
UInt128 recv_iv = as<UInt128>(rheader.data() + 8 + 32);// recv IV   = reversed[40..56)
fix_key(key);
aes_ctr_byte_flow_.init(key, recv_iv);

output_key_ = as<UInt256>(header.data() + 8);          // send key  = header[8..40)
fix_key(output_key_);
output_state_.init(as_slice(output_key_),
                   Slice(header.data() + 8 + 32, 16)); // send IV   = header[40..56)
```

In plain terms:

```
send_key = header[8  .. 40)      (32 bytes)
send_iv  = header[40 .. 56)      (16 bytes)

rev      = reverse(header)       (the whole 64 bytes, byte-reversed)
recv_key = rev[8  .. 40)         (32 bytes)
recv_iv  = rev[40 .. 56)         (16 bytes)
```

Both directions use **AES-256-CTR** with a 128-bit big-endian counter
(`tdutils/td/utils/crypto.cpp:667-723`). CTR is a stream cipher, so encryption and
decryption are the same operation, and the two directions have independent counters that
advance continuously for the life of the connection.

### With an MTProxy secret

`TcpTransport.cpp:118-127`:

```cpp
auto fix_key = [&](UInt256 &key) {
  if (!proxy_secret.empty()) {
    Sha256State state;
    state.init();
    state.feed(as_slice(key));
    state.feed(proxy_secret);
    state.extract(as_mutable_slice(key));
  }
};
```

That is: `key ← SHA256(key ‖ proxy_secret)`, applied to **both** keys, IVs unchanged.

The `proxy_secret` is bytes `[1..17)` of the raw secret when the raw secret is at least 17
bytes long (`td/mtproto/ProxySecret.h:34-40`). Secret formats:

| Raw secret length | Meaning |
|-------------------|---------|
| 16 bytes | Plain secret; used directly |
| 17 bytes, first byte `0xDD` | Use **Padded Intermediate** framing; secret is bytes `[1..17)` |
| ≥ 17 bytes, first byte `0xEE` | **Fake TLS** mode; secret is bytes `[1..17)`, the rest is the SNI domain |

(`td/mtproto/ProxySecret.h:44-50`)

---

## 4. Sending the header

This is the step everybody gets wrong. `TcpTransport.cpp:140-142`:

```cpp
header_ = header;                                                     // plaintext copy
output_state_.encrypt(header_slice, header_slice);                    // encrypt ALL 64 bytes
MutableSlice(header_).substr(56).copy_from(header_slice.substr(56));  // keep only [56..64)
```

So what actually goes on the wire is:

```
┌──────────────────────────────┬──────────────────────────┐
│ header[0..56)  PLAINTEXT     │ enc(header)[56..64)      │
└──────────────────────────────┴──────────────────────────┘
```

The first 56 bytes are sent **as generated** (unencrypted). Only the last 8 bytes are
replaced with their ciphertext.

**Why:** the server reads the first 56 bytes in clear, derives the same keys from them,
then decrypts bytes `[56..64)` to recover the protocol magic and DC id. This proves you
derived the keys correctly and tells the server (or proxy) what framing to use — all
without a round trip.

**Crucially, the encryption of the whole 64 bytes must still happen**, because it advances
the sending CTR counter by exactly 4 blocks. If you only encrypt the last 8 bytes, your
counter will be wrong and every subsequent byte will be garbage.

```
correct:                                    wrong:
  ct = AES_CTR_encrypt(header[0..64])         ct = AES_CTR_encrypt(header[56..64])
  send(header[0..56] ‖ ct[56..64])            send(header[0..56] ‖ ct)
  # counter is now at block 4                 # counter is at block 1  ← desync
```

The header is prepended to the first outgoing packet (`do_write_main`,
`TcpTransport.cpp:164-171`) and sent once only.

---

## 5. Sending and receiving after the header

`ObfuscatedTransport::write` (`TcpTransport.cpp:154-162`):

```cpp
impl_.write_prepare_inplace(&message, quick_ack);   // add the 4-byte Intermediate length
output_state_.encrypt(message.as_slice(), message.as_mutable_slice());
do_write_main(std::move(message));
```

Order matters: **frame first, then encrypt**. The length prefix is inside the encrypted
stream. On receive, decrypt first, then parse the framing
(`TcpTransport.cpp:145-152`).

Conceptually, obfuscation is a transparent byte-stream filter:

```
                    ┌─────────────────────────────────┐
  MTProto bytes ──▶ │ Intermediate framing (§2.2)     │ ──▶ framed bytes
                    └─────────────────────────────────┘
                                    │
                    ┌───────────────▼─────────────────┐
                    │ AES-256-CTR (send_key/send_iv)  │
                    └───────────────┬─────────────────┘
                                    ▼
                          [64-byte header] ‖ ciphertext… ──▶ socket
```

---

## 6. Fake TLS (`emulate_tls`)

When the proxy secret starts with `0xEE`, the obfuscated stream is additionally wrapped in
what looks like a TLS 1.3 session.

* The client sends a full synthetic **ClientHello**, built by
  `td/mtproto/TlsInit.cpp`. It contains GREASE values, a plausible extension list, and an
  HMAC-SHA256 over the hello (keyed with the proxy secret) written at offset `[11..43)`,
  XORed with the current unix time (`TlsInit.cpp:510-517, 578-605`).
* After the handshake, every chunk is wrapped in a TLS application-data record
  (`do_write_tls`, `TcpTransport.cpp:173-213`):
  ```
  17 03 03 <length: 2 bytes BIG-endian> <payload>
  ```
  with `MAX_TLS_PACKET_LENGTH = 2878` (`TcpTransport.h:162`).
* The very first record is preceded by a ChangeCipherSpec: `14 03 03 00 01 01`
  (`TcpTransport.cpp:208-209`).
* The reader requires each record to begin with `17 03 03` plus a 2-byte big-endian length
  (`TlsReaderByteFlow.cpp:15-37`).

> **💡 Implementation note.** The record length here is **big-endian**, unlike every
> other length in MTProto. This is because it is a real TLS record header.

Fake TLS is a substantial amount of work for a niche benefit. Implement it last, if ever.

---

## 7. Checklist

- [ ] 64 random bytes generated with a CSPRNG
- [ ] First byte is not `0xEF`
- [ ] First LE `uint32` is none of the seven forbidden values
- [ ] Second LE `uint32` is non-zero
- [ ] Bytes `[56..60)` set to the framing magic
- [ ] Bytes `[60..62)` set to `dc_id` as `int16` LE (only when using a proxy)
- [ ] `send_key = header[8..40)`, `send_iv = header[40..56)`
- [ ] `recv_key = reverse(header)[8..40)`, `recv_iv = reverse(header)[40..56)`
- [ ] Both keys passed through `SHA256(key ‖ secret)` if a proxy secret is present
- [ ] **All 64 bytes** encrypted (to advance the counter), but only `[56..64)` of the
      ciphertext transmitted
- [ ] Framing applied *before* encryption on send, *after* decryption on receive
- [ ] CTR counters never reset for the life of the connection

---

[← Previous](02-tcp-framings.md) · [Index](../README.md) · [Next: Transport errors →](04-transport-errors.md)
