# 3.4 — Step 3: `set_client_DH_params` → `dh_gen_ok`

[← Previous](03-step2-req-dh-params.md) · [Index](../README.md) · [Next: Security checks →](05-security-checks.md)

---

## 1. Choosing `b` and computing `g_b`

`DhHandshake::get_g_b` and `gen_key` (`td/mtproto/DhHandshake.cpp:128-140, 206-223`):

```cpp
void DhHandshake::gen_b() {
  b_ = BigNum::from_binary(Random::secure_string(2048 / 8));   // 256 random bytes
  …
  BigNum::mod_exp(g_b_, g_bn_, b_, prime_, ctx_);              // g_b = g^b mod p
  …
}
```

* `b` is a **2048-bit** random value from a CSPRNG — 256 bytes, interpreted as a
  big-endian integer.
* `g_b = g^b mod p`.
* `g_b` is serialized as a **minimal big-endian** string (`to_binary()`), so it may be
  slightly shorter than 256 bytes if it happens to have leading zero bytes.

> **⚠ Security note.** `b` must be secret, unpredictable, and used exactly once. Wipe it
> from memory when the handshake finishes. If `b` leaks, the auth key is trivially
> recoverable from the public `g_a`.

TDLib also validates its own `g_b` against the same range check applied to `g_a`
(`DhHandshake.cpp:100-126`) — see [3.5](05-security-checks.md).

---

## 2. Building `client_DH_inner_data`

```
client_DH_inner_data#6643b654 nonce:int128 server_nonce:int128
    retry_id:long g_b:string = Client_DH_Inner_Data;
```
(`mtproto_api.tl:20`)

```cpp
auto data = store_object(mtproto_api::client_DH_inner_data(nonce_, server_nonce_, 0, g_b));
```
(`Handshake.cpp:230`)

`retry_id` is **0** on the first attempt. If the server answers `dh_gen_retry`, a
retrying client would set it to the `new_nonce_hash2` from that reply. TDLib does not
retry — it treats `dh_gen_retry` as an error and restarts the whole handshake
(`Handshake.cpp:279-280`). Do the same; it is simpler and equally correct.

---

## 3. Encrypting it

`Handshake.cpp:231-239`:

```cpp
size_t encrypted_data_size = 20 + data.size();
size_t encrypted_data_size_with_pad = (encrypted_data_size + 15) & -16;
string encrypted_data_str(encrypted_data_size_with_pad, '\0');
MutableSlice encrypted_data = encrypted_data_str;
sha1(data, encrypted_data.ubegin());                       // SHA1 at offset 0
encrypted_data.substr(20, data.size()).copy_from(data);    // data at offset 20
Random::secure_bytes(encrypted_data.substr(encrypted_data_size));   // random tail
tmp_KDF(server_nonce_, new_nonce_, &tmp_aes_key, &tmp_aes_iv);
aes_ige_encrypt(as_slice(tmp_aes_key), as_mutable_slice(tmp_aes_iv), encrypted_data, encrypted_data);
```

Layout, mirroring what the server sent you:

```
┌──── 20 ────┬─────── data ───────┬── 0..15 random ──┐
│ SHA1(data) │ client_DH_inner_data│  pad to /16     │
└────────────┴─────────────────────┴─────────────────┘
```

Two details:

* **Regenerate `tmp_aes_key`/`tmp_aes_iv` from scratch** (or use the saved copies). The
  decryption in step 2 advanced the IV; you need the *original* values here.
* The padding is **random**, not zeros, and is not covered by the SHA1.

---

## 4. The request

```
set_client_DH_params#f5045f1f nonce:int128 server_nonce:int128
    encrypted_data:string = Set_client_DH_params_answer;
```
(`mtproto_api.tl:22`)

```cpp
mtproto_api::set_client_DH_params set_client_dh_params(nonce_, server_nonce_, encrypted_data);
send(connection, create_function_storer(set_client_dh_params));
```
(`Handshake.cpp:241-242`)

---

## 5. Computing the auth key locally

Done immediately, before the reply arrives (`Handshake.cpp:228, 244-250`):

```cpp
auto auth_key_params = handshake.gen_key();
…
auth_key_ = AuthKey(auth_key_params.first, std::move(auth_key_params.second));
auth_key_.set_created_at(dh_inner_data.server_time_);
server_salt_ = as<int64>(new_nonce_.raw) ^ as<int64>(server_nonce_.raw);
```

### The key itself

`DhHandshake::gen_key` (`DhHandshake.cpp:206-223`):

```cpp
std::pair<int64, string> DhHandshake::gen_key() {
  string key = g_ab_.to_binary(2048 / 8);
  auto key_id = calc_key_id(key);
  return std::make_pair(key_id, std::move(key));
}
```

with `g_ab_` computed as:

```cpp
BigNum::mod_exp(g_ab_, g_a_, b_, prime_, ctx_);     // auth_key = g_a^b mod p
```

> **⚠ Critical.** `to_binary(2048 / 8)` produces **exactly 256 bytes, zero-padded on the
> left**. This is different from every other `to_binary()` call in the handshake, which
> emits a minimal encoding. If `g_a^b mod p` happens to be less than 2²⁰⁴⁰, its natural
> encoding is shorter and you **must** left-pad with zeros to 256 bytes. Roughly 1 in 256
> handshakes hits this; the symptom is a mysterious `-404` on the very next connection
> that you cannot reproduce.

### The key id

`DhHandshake::calc_key_id` (`DhHandshake.cpp:225-229`):

```cpp
int64 DhHandshake::calc_key_id(Slice auth_key) {
  UInt<160> auth_key_sha1;
  sha1(auth_key, auth_key_sha1.raw);
  return as<int64>(auth_key_sha1.raw + 12);
}
```

```
auth_key_id = SHA1(auth_key)[12..20)   read as a little-endian int64
```

Bytes **12 through 19**, not 0–7. On the wire, `auth_key_id` is simply those 8 bytes in
order, so if you keep it as a byte array you never need to think about endianness.

### The initial server salt

`Handshake.cpp:250`:

```cpp
server_salt_ = as<int64>(new_nonce_.raw) ^ as<int64>(server_nonce_.raw);
```

```
server_salt = new_nonce[0..8) XOR server_nonce[0..8)
```

An 8-byte XOR of the *first eight bytes* of each nonce. Use it as the salt for your first
encrypted messages; the server will hand you better ones via `bad_server_salt` or
`get_future_salts` ([5.4](../05-session/04-salts-and-time.md)).

---

## 6. The reply

Three possible constructors (`mtproto_api.tl:24-26`):

```
dh_gen_ok#3bcbf734    nonce:int128 server_nonce:int128 new_nonce_hash1:int128 = Set_client_DH_params_answer;
dh_gen_retry#46dc1fb9 nonce:int128 server_nonce:int128 new_nonce_hash2:int128 = Set_client_DH_params_answer;
dh_gen_fail#a69dae02  nonce:int128 server_nonce:int128 new_nonce_hash3:int128 = Set_client_DH_params_answer;
```

| Reply | Meaning | TDLib's action (`Handshake.cpp:258-284`) |
|-------|---------|------------------------------------------|
| `dh_gen_ok` | Success | Verify hash, `state_ = Finish` |
| `dh_gen_retry` | Server wants another attempt | `Status::Error("DhGenRetry")` — restart |
| `dh_gen_fail` | Server rejected the exchange | `Status::Error("DhGenFail")` — restart |

### Verifying `new_nonce_hash1`

`Handshake.cpp:268-273`:

```cpp
UInt<160> auth_key_sha1;
sha1(auth_key_.key(), auth_key_sha1.raw);
auto new_nonce_hash = sha1(PSLICE() << new_nonce_.as_slice() << '\x01'
                                    << auth_key_sha1.as_slice().substr(0, 8));
if (dh_gen_ok->new_nonce_hash1_.as_slice() != Slice(new_nonce_hash).substr(4)) {
  return Status::Error("New nonce hash mismatch");
}
```

Written out:

```
aux_hash = SHA1(auth_key)[0..8)                              # note: [0..8), NOT [12..20)
expected = SHA1( new_nonce ‖ 0x01 ‖ aux_hash )[4..20)        # last 16 of 20 bytes
assert expected == dh_gen_ok.new_nonce_hash1
```

Two easy mistakes:

* `aux_hash` is the **first** 8 bytes of `SHA1(auth_key)`, whereas `auth_key_id` is bytes
  12–19 of the same digest. Different slices of the same hash.
* The result is bytes `[4..20)` of the outer SHA1 — the **last 16**, not the first 16.

The `0x01` literal corresponds to `dh_gen_ok`. The retry and fail variants use `0x02` and
`0x03` respectively in the same formula, which is how you would validate them if you chose
to handle them.

> **⚠ Security note.** This check is what proves the server actually derived the same
> `auth_key`. Without it you would happily install a key the server never agreed to, and
> every subsequent message would fail with an unexplained `-404`. Always verify it.

---

## 7. You now have

| Item | Size | Derivation |
|------|------|------------|
| `auth_key` | 256 bytes | `g_a^b mod p`, zero-padded on the left |
| `auth_key_id` | 8 bytes | `SHA1(auth_key)[12..20)` |
| `server_salt` | 8 bytes | `new_nonce[0..8) ⊕ server_nonce[0..8)` |
| `server_time_diff` | int | `server_time − local_time`, from step 2 |

**Persist `auth_key` and the DC it belongs to.** Re-running the handshake on every start is
slow, hammers the server, and — if you already logged in — throws away your authorization.

**Discard now:** `b`, `new_nonce`, `tmp_aes_key`, `tmp_aes_iv`, `nonce`, `server_nonce`
(after computing the salt). Overwrite the buffers.

Next: pick a random non-zero `session_id`, set `seq_no = 0`, and start sending encrypted
messages ([chapter 4](../04-encrypted-messages/01-envelope.md)).

---

## 8. Checklist for step 3

- [ ] `b` is 2048 bits from a CSPRNG
- [ ] `g_b = g^b mod p`, minimal big-endian
- [ ] `g_b` passes the same range check as `g_a` ([3.5](05-security-checks.md))
- [ ] `client_DH_inner_data` with `retry_id = 0`
- [ ] Encrypted as `SHA1(data) ‖ data ‖ random pad to /16`
- [ ] **Fresh** `tmp_aes_key` / `tmp_aes_iv` (not the mutated ones)
- [ ] `auth_key = g_a^b mod p` as **exactly 256 bytes, left-zero-padded**
- [ ] `auth_key_id = SHA1(auth_key)[12..20)`
- [ ] `server_salt = new_nonce[0..8) ⊕ server_nonce[0..8)`
- [ ] Reply is `dh_gen_ok` (`0x3bcbf734`), not retry/fail
- [ ] Both nonces verified
- [ ] `new_nonce_hash1 == SHA1(new_nonce ‖ 0x01 ‖ SHA1(auth_key)[0..8))[4..20)`
- [ ] `auth_key` persisted along with its DC id
- [ ] `b`, `new_nonce` and the temporary keys wiped

---

[← Previous](03-step2-req-dh-params.md) · [Index](../README.md) · [Next: Security checks →](05-security-checks.md)
