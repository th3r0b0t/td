# 6.2 — `rpc_result`, `rpc_error`, and gzip

[← Previous](01-init-connection.md) · [Index](../README.md) · [Next: Migration and multi-DC →](03-migration-and-multi-dc.md)

---

Every API call you make comes back as one of two things: an `rpc_result` carrying the
answer, or an `rpc_result` carrying an `rpc_error`. This chapter covers both, plus the
gzip compression that can wrap either direction.

---

## 1. `rpc_result`

```
rpc_result#f35c6d01 req_msg_id:long result:Object = RpcResult;
```
(`td/generate/scheme/mtproto_api.tl:30` — commented out because TDLib parses it by hand)

| Field | Type | Meaning |
|-------|------|---------|
| `req_msg_id` | `long` | The `msg_id` of *your* message that this answers |
| `result` | `Object` | The answer — a boxed object of whatever type your call returns |

Match `req_msg_id` against your pending-query map ([5.5](../05-session/05-reliability.md)) to
find out which call this answers. There is no other correlation mechanism.

### The parse

`SessionConnection::on_packet_rpc_result` (`td/mtproto/SessionConnection.cpp:239-278`):

```cpp
auto req_msg_id = static_cast<uint64>(parser.fetch_long());
if (req_msg_id == 0) {
  LOG(ERROR) << "Receive an update in rpc_result " << info;
  return Status::Error("Receive an update in rpc_result");
}

if (info.message_id.get() < req_msg_id - (static_cast<uint64>(15) << 32)) {
  reset_server_time_difference(info.message_id);
}

switch (parser.fetch_int()) {
  case mtproto_api::rpc_error::ID:   … callback_->on_message_result_error(…);
  case mtproto_api::gzip_packed::ID: … gzdecode(gzip.packed_data_) …
  default:
    packet.remove_prefix(sizeof(req_msg_id));
    return callback_->on_message_result_ok(…, as_buffer_slice(packet), info.size);
}
```

Three things to copy:

1. **`req_msg_id == 0` is an error.** It means the server sent an update inside an
   `rpc_result`, which should not happen.
2. The **15-second sanity check** on time — see
   [5.4 §2](../05-session/04-salts-and-time.md).
3. The parser **peeks** the next `int32`, and in the default branch **rewinds** and passes
   the remaining bytes through unparsed. Your reader must be able to do the same: read the
   constructor id, and if it is neither `rpc_error` nor `gzip_packed`, hand the whole
   remainder (starting at that id) to the type-specific parser for the query in question.

> **💡 Implementation note.** The type of `result` is determined by *your* query, not by
> anything in the message. You must remember, per `msg_id`, what you asked for, so you know
> how to parse the answer. A `HashMap<msg_id, QueryKind>` is the minimum.

---

## 2. `rpc_error`

```
rpc_error#2144ca19 error_code:int error_message:string = RpcError;
```
(`mtproto_api.tl:31`)

| Field | Meaning |
|-------|---------|
| `error_code` | An HTTP-like status: 303, 400, 401, 403, 420, 500 |
| `error_message` | A machine-readable string, e.g. `PHONE_CODE_INVALID` |

### Code families

| Code | Family | Handling |
|------|--------|----------|
| 303 | `SEE_OTHER` — wrong datacenter | Migrate; see [6.3](03-migration-and-multi-dc.md) |
| 400 | `BAD_REQUEST` — your fault | Fix the request; do not retry blindly |
| 401 | `UNAUTHORIZED` | Log in, or handle `SESSION_PASSWORD_NEEDED` |
| 403 | `FORBIDDEN` | Permission denied; do not retry |
| 420 | `FLOOD_WAIT_X` | Wait `X` seconds, then retry |
| 500 | `INTERNAL` | Server-side; retry with backoff |

### `FLOOD_WAIT_X`

The single most important error to handle correctly. The message is literally
`FLOOD_WAIT_` followed by a number of seconds.

`td/telegram/net/NetQueryDelayer.cpp:35-71` parses it and clamps the delay:

* Minimum 1 second.
* Maximum 14 days (`14 * 24 * 60 * 60`).

There are related variants: `FLOOD_PREMIUM_WAIT_X` (the wait would be shorter for a
Premium account) and `SLOWMODE_WAIT_X` (group slow mode).

> **⚠ Do not ignore `FLOOD_WAIT`.** Retrying immediately gets the wait extended, and
> persistent abuse gets the `api_id` banned. Sleep the full duration.

### Errors you will actually hit

See [Appendix C](../appendix/C-error-reference.md) for the full list. The ones that matter
during login and a first message:

| Message | Meaning |
|---------|---------|
| `PHONE_NUMBER_INVALID` | Malformed number |
| `PHONE_CODE_INVALID` | Wrong code |
| `PHONE_CODE_EXPIRED` | Code timed out; call `auth.resendCode` or start over |
| `SESSION_PASSWORD_NEEDED` | 2FA is on; see [7.5](../07-login/05-two-factor-srp.md) |
| `PASSWORD_HASH_INVALID` | Wrong password, or your SRP computation is wrong |
| `AUTH_KEY_UNREGISTERED` | The server does not know your auth key; redo the handshake |
| `PEER_ID_INVALID` | Bad peer, usually a wrong `access_hash` |
| `MESSAGE_EMPTY` | Empty message text |
| `USER_DEACTIVATED_BAN` | Account banned |

---

## 3. gzip

```
gzip_packed#3072cfa1 packed_data:string = Object;
```
(`mtproto_api.tl:51`)

The `packed_data` is a **raw DEFLATE stream with a gzip header** — what `gzip`/zlib produce
in gzip mode, not raw deflate and not zlib-wrapped. Decompressing yields the serialized
object, which you then parse normally.

### Incoming

gzip can appear in **two** places:

1. As the top-level body of a message — handle it in your message dispatcher, decompress,
   and re-dispatch the result.
2. As the `result` of an `rpc_result` — handled explicitly at
   `SessionConnection.cpp:264-273`. The comment in the source is blunt about the
   irregularity:
   ```cpp
   // yep, gzip in rpc_result
   BufferSlice object = gzdecode(gzip.packed_data_);
   ```

Both must be handled. Large replies — `help.getConfig`, contact lists, message histories —
arrive gzipped in practice.

> **⚠ Decompression bomb.** Cap the decompressed size. TDLib's `gzdecode` has a limit
> parameter for exactly this reason. A malicious or buggy server could otherwise exhaust
> your memory.

### Outgoing

Optional. `NetQueryCreator::create` (`td/telegram/net/NetQueryCreator.cpp:64-102`):

```cpp
size_t min_gzipped_size = 128;              // 1024 for bots (line 75)
…
auto gzip_flag = slice.size() < min_gzipped_size ? GzipFlag::Off : GzipFlag::On;
if (slice.size() >= 16384) {
  // test compression ratio for the middle part
  // if it is less than 0.9, then try to compress the whole request
  size_t TESTED_SIZE = 1024;
  BufferSlice compressed_part = gzencode(slice.as_slice().substr((slice.size() - TESTED_SIZE) / 2, TESTED_SIZE), 0.9);
  if (compressed_part.empty()) gzip_flag = GzipFlag::Off;
}
if (gzip_flag == GzipFlag::On) {
  BufferSlice compressed = gzencode(slice.as_slice(), 0.9);
  if (compressed.empty()) gzip_flag = GzipFlag::Off;   // did not reach 0.9 ratio
  else slice = std::move(compressed);
}
```

The rules:

* Below 128 bytes (1024 for bots): never compress.
* At or above 16384 bytes: first test-compress a 1024-byte sample from the middle. If that
  does not reach a 0.9 ratio, do not bother with the whole thing.
* Compress; if the result is not at least 10% smaller, send uncompressed.

The compressed query is then serialized as `gzip_packed` where the plain one would have
gone (`td/mtproto/CryptoStorer.h:150-154`).

**You may skip outgoing gzip entirely.** It is a bandwidth optimization; nothing requires
it. Incoming gzip is *not* optional.

---

## 4. Dispatching a decrypted message

Putting chapters 5 and 6 together, here is the complete dispatch:

```
fn on_message(msg_id, seq_no, body):
    observe_server_msg_id(msg_id)        // 5.4
    if seq_no is odd: queue_ack(msg_id)  // 5.2

    id = read_u32_le(body[0..4))
    match id:
        0x73f1f8dc  msg_container    -> for each inner: on_message(...)   // recurse
        0x3072cfa1  gzip_packed      -> on_message(msg_id, seq_no, gzdecode(...))
        0xf35c6d01  rpc_result       -> on_rpc_result(body[4..])
        0x62d6b459  msgs_ack         -> mark acked
        0xedab447b  bad_server_salt  -> adopt salt, resend
        0xa7eff811  bad_msg_notif    -> see 5.3 §5
        0x9ec20908  new_session_created -> resend older, adopt salt
        0x347773c5  pong             -> liveness
        0xae500895  future_salts     -> store
        0x276d3ec6  msg_detailed_info
        0x809db6df  msg_new_detailed_info
        0x7d861a08  msg_resend_req   -> resend listed ids
        _                            -> an update: pass to your update handler
```

Anything you do not recognize is almost certainly an **update** from `telegram_api.tl`
(`updates`, `updateShort`, `updateShortMessage`, …). See
[8.4](../08-sending-a-message/04-reading-updates.md).

> **💡 Implementation note.** Do not error out on unknown constructor ids. New update types
> appear with every layer. Log and ignore.

---

## 5. Checklist

- [ ] `rpc_result.req_msg_id` matched against a pending-query map
- [ ] `req_msg_id == 0` treated as an error
- [ ] The next `int32` peeked, with a rewind in the default branch
- [ ] `rpc_error` (`0x2144ca19`) recognized inside `rpc_result`
- [ ] `gzip_packed` (`0x3072cfa1`) recognized inside `rpc_result` **and** as a top-level body
- [ ] Decompressed size capped
- [ ] `FLOOD_WAIT_X` parsed, clamped to 1 s … 14 d, and honoured
- [ ] `303` errors routed to the migration handler
- [ ] Unknown constructor ids logged and ignored, not fatal
- [ ] Per-`msg_id` record of what was asked, so the answer can be typed

---

[← Previous](01-init-connection.md) · [Index](../README.md) · [Next: Migration and multi-DC →](03-migration-and-multi-dc.md)
