# 1.3 — Worked Serialization Examples

[← Previous](02-wire-encoding.md) · [Index](../README.md) · [Next: Datacenters →](../02-transport/01-datacenters.md)

---

Use these as unit-test vectors for your serializer. Every byte is accounted for.

---

## Example 1 — `req_pq_multi`

Schema (`td/generate/scheme/mtproto_api.tl:73`):

```
req_pq_multi#be7e8ef1 nonce:int128 = ResPQ;
```

With `nonce = 3E0549828CCA27E966B301A48FECE2FC` (16 bytes, generated randomly):

```
offset  bytes                                             meaning
------  ------------------------------------------------  -------------------------
  0     F1 8E 7E BE                                       0xbe7e8ef1, LE
  4     3E 05 49 82 8C CA 27 E9 66 B3 01 A4 8F EC E2 FC   nonce, raw, no swapping
------
total 20 bytes
```

Note that the constructor id `0xbe7e8ef1` appears reversed on the wire (`F1 8E 7E BE`)
because it is an `int` and therefore little-endian, while the nonce is **not** reversed
because it is an `int128`.

---

## Example 2 — parsing `resPQ`

Schema (`td/generate/scheme/mtproto_api.tl:13`):

```
resPQ#05162463 nonce:int128 server_nonce:int128 pq:string
               server_public_key_fingerprints:Vector<long> = ResPQ;
```

A real response, annotated:

```
offset  bytes                                             meaning
------  ------------------------------------------------  -------------------------
  0     63 24 16 05                                       0x05162463 → resPQ
  4     3E 05 49 82 8C CA 27 E9 66 B3 01 A4 8F EC E2 FC   nonce (echoes yours)
 20     A5 CF 4D 33 F4 A1 1E A8 77 BA 4A A5 73 90 73 30   server_nonce
 36     08                                                pq: length 8
 37     17 ED 48 94 1A 08 F9 81                           pq = 0x17ED48941A08F981
 45     00 00 00                                          padding to (1+8)→12
 48     15 C4 B5 1C                                       0x1cb5c415 → Vector
 52     01 00 00 00                                       count = 1
 56     A5 B7 F7 09 35 5F C3 0B                           fingerprint[0]
------
total 64 bytes
```

Things to check in your parser:

* The `nonce` must equal the one you sent (`td/mtproto/Handshake.cpp:92-94`).
* `pq` is a **big-endian** integer: `0x17ED48941A08F981` = 1724114033281923457.
* The padding after `pq` brings `1 + 8 = 9` up to `12`, i.e. 3 zero bytes.
* Fingerprints are `long`s and therefore little-endian. `A5 B7 F7 09 35 5F C3 0B` is
  `0x0BC35F3509F7B7A5`.

---

## Example 3 — `p_q_inner_data_dc`

Schema (`td/generate/scheme/mtproto_api.tl:15`):

```
p_q_inner_data_dc#a9f55f95 pq:string p:string q:string nonce:int128
                           server_nonce:int128 new_nonce:int256 dc:int = P_Q_inner_data;
```

With `pq = 0x17ED48941A08F981`, `p = 0x494C553B`, `q = 0x53911073`, and DC 2:

```
offset  bytes                                             meaning
------  ------------------------------------------------  -------------------------
  0     95 5F F5 A9                                       0xa9f55f95
  4     08 17 ED 48 94 1A 08 F9 81 00 00 00               pq (8 bytes + pad)
 16     04 49 4C 55 3B 00 00 00                           p  (4 bytes + pad)
 24     04 53 91 10 73 00 00 00                           q  (4 bytes + pad)
 32     <16 bytes>                                        nonce
 48     <16 bytes>                                        server_nonce
 64     <32 bytes>                                        new_nonce
 96     02 00 00 00                                       dc = 2
------
total 100 bytes
```

Notes:

* `p` and `q` are **minimal-length big-endian** encodings, produced by
  `BigNum::to_binary()` with no width argument
  (`tdutils/td/utils/BigNum.cpp`, used at `td/mtproto/Handshake.cpp:107`). Do not
  zero-pad them to 8 bytes.
* `p < q` — TDLib's factorizer guarantees this by swapping if needed
  (`tdutils/td/utils/crypto.cpp:220-222`).
* `dc` is the datacenter you are connected to. For a *temporary* key (PFS) use
  `p_q_inner_data_temp_dc#56fddf88` instead, which appends `expires_in:int`
  (`td/generate/scheme/mtproto_api.tl:16`, built at
  `td/mtproto/Handshake.cpp:119-120`).
* TDLib checks that this serialization is at most 144 bytes before RSA-padding it
  (`td/mtproto/Handshake.cpp:129-131`).

---

## Example 4 — `msgs_ack` with two ids

Schema (`td/generate/scheme/mtproto_api.tl:53`):

```
msgs_ack#62d6b459 msg_ids:Vector<long> = MsgsAck;
```

```
offset  bytes                                             meaning
------  ------------------------------------------------  -------------------------
  0     59 B4 D6 62                                       0x62d6b459
  4     15 C4 B5 1C                                       Vector
  8     02 00 00 00                                       count = 2
 12     00 00 00 A4 F1 D9 67 00                           msg_ids[0] (LE long)
 20     04 00 00 A8 F1 D9 67 00                           msg_ids[1]
------
total 28 bytes
```

`msg_id`s are `long`s, hence little-endian; the "timestamp" part is therefore in the
*high* bytes, which appear last.

---

## Example 5 — `invokeWithLayer` + `initConnection` + `help.getConfig`

This is the exact structure TDLib builds in
`td/telegram/net/MtprotoHeader.cpp:28-101`. Schemas:

```
invokeWithLayer#da9b0d0d {X:Type} layer:int query:!X = X;
initConnection#c1cd5ea9 {X:Type} flags:# api_id:int device_model:string
    system_version:string app_version:string system_lang_code:string
    lang_pack:string lang_code:string proxy:flags.0?InputClientProxy
    params:flags.1?JSONValue query:!X = X;
help.getConfig#c4f9186b = Config;
```

With `api_id = 12345`, no proxy, and no `params`:

```
offset  bytes                                             meaning
------  ------------------------------------------------  -------------------------
  0     0D 0D 9B DA                                       invokeWithLayer
  4     E5 00 00 00                                       layer = 229
  8     A9 5E CD C1                                       initConnection
 12     00 00 00 00                                       flags = 0 (no proxy/params)
 16     39 30 00 00                                       api_id = 12345
 20     07 44 65 73 6B 74 6F 70                           "Desktop"       (7 + pad 0)
 28     05 4C 69 6E 75 78 00 00                           "Linux"         (5 + pad 2)
 36     05 31 2E 30 2E 30 00 00                           "1.0.0"         (5 + pad 2)
 44     02 65 6E 00                                       "en"            (2 + pad 1)
 48     00 00 00 00                                       lang_pack = ""
 52     02 65 6E 00                                       lang_code = "en"
 56     6B 18 F9 C4                                       help.getConfig
------
total 60 bytes
```

Points worth noting:

* `initConnection` is not a data structure — it is a *function* that takes another
  function as its last argument. That nested query is written inline with no length
  prefix.
* Every string is padded independently to a 4-byte boundary.
* When you *do* include `params` (a `JSONValue`), TDLib sets bit 1 and always injects a
  `tz_offset` number (`MtprotoHeader.cpp:87-98`). It is optional for you.
* Empty strings still cost 4 bytes (`00 00 00 00`).

---

## Example 6 — flags with `?true` fields

`messages.sendMessage` (`td/generate/scheme/telegram_api.tl:2521`) begins:

```
messages.sendMessage#fef48f62 flags:# no_webpage:flags.1?true silent:flags.5?true
    background:flags.6?true clear_draft:flags.7?true … peer:InputPeer
    reply_to:flags.0?InputReplyTo message:string random_id:long …
```

To send a plain message with link previews disabled, to `inputPeerUser(id, hash)`:

```
flags = 1 << 1 = 0x00000002        // no_webpage = true; nothing else set

offset  bytes                                             meaning
------  ------------------------------------------------  -------------------------
  0     62 8F F4 FE                                       messages.sendMessage
  4     02 00 00 00                                       flags: bit 1 set
  8     4C A5 E8 DD                                       inputPeerUser#dde8a54c
 12     <8 bytes>                                         user_id (long)
 20     <8 bytes>                                         access_hash (long)
 28     0C 48 65 6C 6C 6F 2C 20 77 6F 72 6C 64 00 00 00   "Hello, world" (12+pad 3)
 44     <8 bytes>                                         random_id (long)
------
total 52 bytes
```

Observe:

* `no_webpage` set bit 1 but contributed **zero bytes** to the payload.
* `reply_to` uses bit 0, which is clear, so it is absent entirely — the `peer` is followed
  immediately by `message`.
* `peer` is a boxed `InputPeer`, so it carries its own constructor id.

---

## Example 7 — a complete `msg_container`

Container format (`td/generate/scheme/mtproto_api.tl:47-48`, built at
`td/mtproto/CryptoStorer.h:193-198`):

```
msg_container#73f1f8dc messages:vector<%Message> = MessageContainer;
message msg_id:long seqno:int bytes:int body:Object;      // BARE, no constructor id
```

Two messages — a `msgs_ack` and a query:

```
offset  bytes                                             meaning
------  ------------------------------------------------  -------------------------
  0     DC F8 F1 73                                       msg_container
  4     02 00 00 00                                       count = 2  (BARE vector:
                                                            no 0x1cb5c415 tag!)
  8     <8 bytes>                                         messages[0].msg_id
 16     04 00 00 00                                       messages[0].seqno = 4 (even)
 20     1C 00 00 00                                       messages[0].bytes = 28
 24     59 B4 D6 62 …                                     messages[0].body = msgs_ack
 52     <8 bytes>                                         messages[1].msg_id
 60     05 00 00 00                                       messages[1].seqno = 5 (odd)
 64     3C 00 00 00                                       messages[1].bytes = 60
 68     0D 0D 9B DA …                                     messages[1].body
------
```

Critical details:

* The count is a **bare** vector length — there is no `0x1cb5c415` tag. This is one of the
  few places in MTProto where that happens.
* `bytes` is the exact length of `body`, and `body` is **not** padded.
* Each inner message has its own `msg_id` and `seq_no`, allocated from the same counters as
  a standalone message would use.
* The container itself is a message and gets its own `msg_id` and a **non-content-related**
  (even) `seq_no` (`td/mtproto/CryptoStorer.h:252-253`).

---

## Example 8 — string padding edge cases

The single most common serialization bug. Verify all of these:

| Input length | Prefix bytes | Data | Padding | Total |
|--------------|--------------|------|---------|-------|
| 0 | 1 (`00`) | 0 | 3 | 4 |
| 1 | 1 | 1 | 2 | 4 |
| 2 | 1 | 2 | 1 | 4 |
| 3 | 1 | 3 | 0 | 4 |
| 4 | 1 | 4 | 3 | 8 |
| 7 | 1 | 7 | 0 | 8 |
| 8 | 1 | 8 | 3 | 12 |
| 253 | 1 | 253 | 2 | 256 |
| 254 | 4 (`FE FE 00 00`) | 254 | 2 | 260 |
| 255 | 4 | 255 | 1 | 260 |
| 256 | 4 | 256 | 0 | 260 |
| 257 | 4 | 257 | 3 | 264 |

Formula: `total = round_up_to_4(prefix_len + data_len)` where `prefix_len` is 1, 4, or 8.

---

## Test harness suggestion

```
assert_eq(ser_bytes(b""),        hex("00000000"))
assert_eq(ser_bytes(b"a"),       hex("01610000"))
assert_eq(ser_bytes(b"abc"),     hex("03616263"))
assert_eq(ser_bytes(b"abcd"),    hex("0461626364000000"))
assert_eq(ser_int(229),          hex("e5000000"))
assert_eq(ser_long(1),           hex("0100000000000000"))
assert_eq(ser_vector_long([1,2]),
          hex("15c4b51c02000000" "0100000000000000" "0200000000000000"))
assert_eq(len(ser_bytes(random_bytes(253))), 256)
assert_eq(len(ser_bytes(random_bytes(254))), 260)
for n in 0..1000: assert_eq(deser_bytes(ser_bytes(random_bytes(n))), that_input)
```

---

[← Previous](02-wire-encoding.md) · [Index](../README.md) · [Next: Datacenters →](../02-transport/01-datacenters.md)
