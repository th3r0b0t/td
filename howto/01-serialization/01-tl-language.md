# 1.1 — Reading a TL Schema

[← Previous](../00-introduction/03-architecture.md) · [Index](../README.md) · [Next: Wire encoding →](02-wire-encoding.md)

---

TL ("Type Language") is Telegram's IDL. Everything that crosses the wire — from
`req_pq_multi` to `messages.sendMessage` — is described by a line in a `.tl` schema file.
If you can read those lines, you can serialize and parse anything.

The two schemas you need are both in this repository:

| Schema | Path | Contents |
|--------|------|----------|
| MTProto | `td/generate/scheme/mtproto_api.tl` | 87 lines. The handshake and service messages. |
| API | `td/generate/scheme/telegram_api.tl` | ~3000 lines. Everything else. |

---

## 1. Anatomy of a schema line

```
resPQ#05162463 nonce:int128 server_nonce:int128 pq:string server_public_key_fingerprints:Vector<long> = ResPQ;
└─┬─┘ └───┬──┘ └──────────────────────────┬───────────────────────────────────────────────────────┘   └─┬─┘
  │       │                               │                                                            │
  │       │                               │                                                            └── result TYPE
  │       │                               └── FIELDS: name:type, in wire order
  │       └── CONSTRUCTOR ID (4 bytes, hex)
  └── CONSTRUCTOR NAME
```

Read it as: *"the constructor `resPQ`, identified on the wire by the four bytes
`0x05162463`, carries four fields in this order, and produces a value of the abstract type
`ResPQ`."*

Two conventions make the whole schema readable:

* **lowercase-initial names are constructors** (concrete values you can serialize),
* **Uppercase-initial names are types** (abstract; one of several constructors).

So `ResPQ` is a type, and `resPQ` happens to be its only constructor. Compare:

```
dh_gen_ok#3bcbf734    nonce:int128 server_nonce:int128 new_nonce_hash1:int128 = Set_client_DH_params_answer;
dh_gen_retry#46dc1fb9 nonce:int128 server_nonce:int128 new_nonce_hash2:int128 = Set_client_DH_params_answer;
dh_gen_fail#a69dae02  nonce:int128 server_nonce:int128 new_nonce_hash3:int128 = Set_client_DH_params_answer;
```
(`td/generate/scheme/mtproto_api.tl:24-26`)

Three constructors, one type. When you receive a `Set_client_DH_params_answer`, you read
the first 4 bytes and switch on them to decide which of the three you got.

---

## 2. Functions vs constructors

Every `.tl` file has a marker line:

```
---functions---
```

Everything **before** it declares data constructors. Everything **after** it declares
*methods* you can call. `td/generate/scheme/mtproto_api.tl:71` is where the split happens.

A function line looks identical to a constructor line, but the result type is what you get
back:

```
req_pq_multi#be7e8ef1 nonce:int128 = ResPQ;
```
(`td/generate/scheme/mtproto_api.tl:73`)

*"Serialize `0xbe7e8ef1` followed by a 16-byte nonce; you will receive a `ResPQ` back."*

On the wire there is no distinction whatsoever between a function call and a data
structure — a function call **is** a serialized object. This is why methods can be nested
inside other methods (`invokeWithLayer(layer, initConnection(…, help.getConfig()))`).

---

## 3. Built-in types

These are declared at the top of `mtproto_api.tl:1-11` and are the primitives every other
type is built from:

```
int ? = Int;
long ? = Long;
double ? = Double;
string ? = String;
vector {t:Type} # [ t ] = Vector t;
int128 4*[ int ] = Int128;
int256 8*[ int ] = Int256;
```

| TL type | Size | Wire representation |
|---------|------|---------------------|
| `int` | 4 bytes | signed little-endian |
| `long` | 8 bytes | signed little-endian |
| `double` | 8 bytes | IEEE 754 binary64, little-endian |
| `string` | variable | length-prefixed, padded to 4 bytes |
| `bytes` | variable | identical to `string` on the wire |
| `int128` | 16 bytes | **raw bytes, no reordering** |
| `int256` | 32 bytes | **raw bytes, no reordering** |
| `Vector<T>` | variable | `0x1cb5c415`, count, then elements |
| `Bool` | 4 bytes | `boolTrue#997275b5` / `boolFalse#bc799737` |
| `#` | 4 bytes | a "nat" — in practice the `flags` word |

> **💡 Implementation note.** `int128` and `int256` are **not integers** despite the
> names. They are fixed-length opaque byte strings. Nonces are `int128`/`int256`; if you
> byte-swap them your handshake will fail with a nonce mismatch and no useful diagnostic.
> TDLib stores them as `UInt128`/`UInt256` structs holding a raw `uint8 raw[N]` array
> (`tdutils/td/utils/UInt.h`).

`string` and `bytes` are the same on the wire. The distinction is purely documentation:
`string` means "UTF-8 text", `bytes` means "arbitrary octets". Big numbers such as `pq`,
`dh_prime`, `g_a` and `g_b` are declared `string` in the MTProto schema but are
**big-endian byte strings**, not text.

---

## 4. Boxed and bare types

This is the single concept people get wrong most often.

* **Boxed**: the value is prefixed with its 4-byte constructor id, so the reader can tell
  which constructor it is.
* **Bare**: the value is written with no constructor id, because the reader already knows
  what to expect.

The rule as implemented in TDLib
(`td/tl/tl_object_store.h:19-35`):

```cpp
// TlStoreBoxed writes the constructor id first, then the fields:
storer.store_binary(constructor_id);
// TlStoreBoxedUnknown writes the object's own dynamic id:
storer.store_binary(x->get_id());
// bare types write only the fields.
```

**In practice, for a client implementer, the rules reduce to:**

1. A field whose declared type starts with an **uppercase** letter (`InputPeer`,
   `CodeSettings`, `Vector<long>`) is **boxed** — write the constructor id of whichever
   concrete constructor you chose.
2. A field whose declared type is a **built-in primitive** (`int`, `long`, `string`,
   `bytes`, `int128`, `int256`, `double`, `#`) is **bare** — just the raw value.
3. `Vector<T>` is boxed (`0x1cb5c415`); the lowercase `vector<T>` used inside
   `future_salts` is bare (no id). Compare `mtproto_api.tl:38` (`salts:vector<future_salt>`,
   bare) with `mtproto_api.tl:53` (`msg_ids:Vector<long>`, boxed).
4. The elements *inside* a `Vector<T>` are boxed iff `T` is an uppercase type.
5. Messages inside a `msg_container` are bare (a hand-rolled format, see
   [5.2](../05-session/02-containers-and-acks.md)).
6. A `%Type` prefix in the schema means "the bare version of this type".

> **💡 Implementation note.** When in doubt, look at what TDLib generates. The rule of
> thumb that works 99% of the time: *if the type name in the schema begins with a capital
> letter, emit a constructor id.*

---

## 5. Optional fields and the `flags` word

Modern API constructors use conditional fields:

```
auth.signIn#8d52a951 flags:# phone_number:string phone_code_hash:string
                     phone_code:flags.0?string email_verification:flags.1?EmailVerification
                     = auth.Authorization;
```
(`td/generate/scheme/telegram_api.tl:2318`)

Reading rules:

* `flags:#` declares a 32-bit little-endian word. It is **always present**.
* `field:flags.N?Type` means: *this field is present in the byte stream if and only if bit
  `N` of `flags` is set*. If the bit is clear, the field occupies **zero bytes**.
* `field:flags.N?true` is special: the field carries **no bytes at all, ever**. The flag
  bit *is* the value. `no_webpage:flags.1?true` in `messages.sendMessage` is a pure boolean
  encoded as a bit.
* Some constructors have a second flags word, written `flags2:#`, with fields
  `flags2.N?Type`. `user#b1b8cc83` in `telegram_api.tl:115` is one such.

TDLib's macros make the pattern explicit
(`tdutils/td/utils/tl_helpers.h:30-58`): flags are accumulated bit by bit, then a single
`int32` is written, then the present fields follow in declaration order.

> **⚠ Security note.** When *parsing*, reject a message whose `flags` word contains bits
> you do not recognize, rather than silently continuing. TDLib does exactly this
> (`END_PARSE_FLAGS` validates that no unknown high bits remain,
> `tdutils/td/utils/tl_helpers.h:47-58`). Continuing after an unknown flag means you are
> parsing at the wrong offset and everything after it is garbage.

Worked example: to call `auth.signIn` with a phone code and no email verification:

```
flags = 1 << 0 = 0x00000001
wire:  51 A9 52 8D  01 00 00 00  <phone_number>  <phone_code_hash>  <phone_code>
       └ ctor id ┘  └─ flags ──┘
```

---

## 6. Where constructor ids come from

The `#xxxxxxxx` in a schema line is the **CRC-32** (IEEE 802.3, polynomial `0xEDB88320`,
init `0xFFFFFFFF`, final XOR `0xFFFFFFFF` — i.e. the same CRC-32 as zlib/gzip) of the
normalized schema line with the `#id` removed.

The normalization is performed in `td/generate/tl-parser/tl-parser.c:1468-1480`
(`tl_count_combinator_name`):

```c
tl_buf_reset ();
tl_buf_add_string_nospace (c->real_id ? c->real_id : c->id, -1);  /* constructor name    */
tl_buf_add_tree (c->left, 1);                                     /* the argument list   */
tl_buf_add_string ("=", -1);                                      /* " ="                */
tl_buf_add_tree (c->right, 1);                                    /* the result type     */
c->name = compute_crc32 (buf, buf_pos);
```

with the CRC implementation at `td/generate/tl-parser/crc32.c`
(`compute_crc32(data,len) = crc32_partial(data,len,-1) ^ -1`).

Normalization rules that matter:

* the `#id` annotation is stripped;
* tokens are separated by exactly one space;
* `name:Type` is emitted with no space around the colon;
* `<`/`>` become spaces: `Vector<long>` is hashed as `Vector long`;
* conditional fields are hashed as `name.N?Type`;
* the leading `{X:Type}` template parameter and the `!` in `query:!X` are dropped.

So `resPQ`'s id is `CRC32("resPQ nonce:int128 server_nonce:int128 pq:string server_public_key_fingerprints:Vector long = ResPQ")` = `0x05162463`, matching the annotation at
`td/generate/scheme/mtproto_api.tl:13`.

> **💡 Implementation note.** You almost never need to compute these. The `.tl` files
> already carry every id. Copy them into a constants file (see
> [appendix A](../appendix/A-constructor-ids.md)) and move on. Computing them yourself is
> only useful if you want to auto-generate bindings from a schema file, in which case
> reproduce the normalization above exactly — a single extra space changes the id.

**Watch out for short ids.** Ids in the schema are written without leading zeros:
`users.getUsers#d91a548` is `0x0d91a548`, and `auth.sentCodeTypeFirebaseSms#9fd736` is
`0x009fd736`. Always pad to 8 hex digits.

---

## 7. Comment lines in `mtproto_api.tl`

Several important constructors are *commented out* in TDLib's copy of the MTProto schema
because the C++ code handles them by hand rather than through generated parsers. **They are
still real, and you must implement them.** They are, with their real ids:

| Line | Constructor | Id |
|------|-------------|-----|
| `:30` | `rpc_result#f35c6d01 req_msg_id:long result:Object` | `0xf35c6d01` |
| `:47` | `msg_container#73f1f8dc messages:vector<%Message>` | `0x73f1f8dc` |
| `:48` | `message msg_id:long seqno:int bytes:int body:Object` | *(bare)* |
| `:49` | `msg_copy#e06046b2 orig_message:Message` | `0xe06046b2` |
| `:81` | `ping#7abe77ec ping_id:long = Pong` | `0x7abe77ec` |
| `:83` | `destroy_session#e7512126 session_id:long` | `0xe7512126` |
| `:42-43` | `destroy_session_ok#e22045fc` / `destroy_session_none#62d350c9` | — |

`msg_container` and `rpc_result` are the two you cannot live without; TDLib's handling of
them is in `td/mtproto/SessionConnection.cpp:214-278`.

---

## 8. Practical advice on schema handling

You have three options, in increasing order of effort and payoff:

1. **Hand-write the ~40 constructors you need.** Perfectly reasonable for a
   "log in and send a message" client. This guide gives you all of them.
2. **Write a small `.tl` parser and code generator.** A few hundred lines; pays for itself
   the moment you need more of the API.
3. **Use an existing generator** for your language.

If you hand-write, keep serialization and parsing in one file per constructor group and
add a unit test that round-trips each one. The commonest bug in hand-written TL code is a
field written in the wrong order, and it is invisible until the server rejects the query.

---

[← Previous](../00-introduction/03-architecture.md) · [Index](../README.md) · [Next: Wire encoding →](02-wire-encoding.md)
