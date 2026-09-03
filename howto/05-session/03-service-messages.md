# 5.3 — Service Messages

[← Previous](02-containers-and-acks.md) · [Index](../README.md) · [Next: Salts and time →](04-salts-and-time.md)

---

Service messages are the MTProto-level control plane: acknowledgements, errors, salt
updates, pings. They arrive interleaved with your API results and you must handle them, or
your client will slowly desynchronize.

All of them are defined in `td/generate/scheme/mtproto_api.tl`; the handlers are the
overloads of `SessionConnection::on_packet` in `td/mtproto/SessionConnection.cpp`.

---

## 1. Dispatch table

After decrypting and unpacking containers ([5.2](02-containers-and-acks.md)), switch on the
first `int32` of each message body:

| Constructor id | Name | Priority |
|----------------|------|----------|
| `0x73f1f8dc` | `msg_container` | **must** — unpack recursively |
| `0xf35c6d01` | `rpc_result` | **must** — your answers arrive here |
| `0x62d6b459` | `msgs_ack` | **must** — clear your resend queue |
| `0xedab447b` | `bad_server_salt` | **must** — update salt and resend |
| `0xa7eff811` | `bad_msg_notification` | **must** — otherwise you never recover |
| `0x9ec20908` | `new_session_created` | **must** — resend in-flight queries |
| `0x3072cfa1` | `gzip_packed` | **must** — decompress and re-dispatch |
| `0x347773c5` | `pong` | should |
| `0xae500895` | `future_salts` | should |
| `0x276d3ec6` | `msg_detailed_info` | optional |
| `0x809db6df` | `msg_new_detailed_info` | optional |
| `0x04deb57d` | `msgs_state_info` | optional |
| `0x8cc0d131` | `msgs_all_info` | optional |
| *anything else* | an **update** | should — see [8.4](../08-sending-a-message/04-reading-updates.md) |

Anything not in this table is a `telegram_api` object pushed by the server — an update. Do
not treat it as an error.

---

## 2. `rpc_result`

```
rpc_result#f35c6d01 req_msg_id:long result:Object = RpcResult;
```
(`mtproto_api.tl:30`, commented out; TDLib parses it by hand)

This is how every API answer arrives. `req_msg_id` is the `msg_id` of the query you sent —
that is the only link between question and answer, which is why you must remember it.

`on_packet_rpc_result` (`SessionConnection.cpp:239-278`):

```cpp
auto req_msg_id = static_cast<uint64>(parser.fetch_long());
…
if (req_msg_id == 0) {
  LOG(ERROR) << "Receive an update in rpc_result " << info;
  return Status::Error("Receive an update in rpc_result");
}
…
switch (parser.fetch_int()) {
  case mtproto_api::rpc_error::ID: { … on_message_result_error(…); }
  case mtproto_api::gzip_packed::ID: {
    BufferSlice object = gzdecode(gzip.packed_data_);
    return callback_->on_message_result_ok(MessageId(req_msg_id), std::move(object), info.size);
  }
  default:
    packet.remove_prefix(sizeof(req_msg_id));
    return callback_->on_message_result_ok(MessageId(req_msg_id), as_buffer_slice(packet), info.size);
}
```

So the algorithm is:

```
req_msg_id = read_u64()
if req_msg_id == 0: error
next = peek_u32()
if next == 0x2144ca19:   parse rpc_error, fail the query
if next == 0x3072cfa1:   gunzip, then parse the result normally
otherwise:               the remaining bytes are the result object
```

Note the `default` branch does **not** consume the `int32` it peeked — it rewinds and hands
the whole remainder (constructor id included) to the caller.

### `rpc_error`

```
rpc_error#2144ca19 error_code:int error_message:string = RpcError;
```
(`mtproto_api.tl:31`)

`error_code` is HTTP-like (400, 401, 403, 420, 303, 500) and `error_message` is a short
uppercase token, sometimes with a numeric suffix (`FLOOD_WAIT_42`, `PHONE_MIGRATE_4`).
Full reference in [appendix C](../appendix/C-error-reference.md); handling in
[6.2](../06-api-layer/02-rpc-results-and-errors.md).

---

## 3. `bad_msg_notification`

```
bad_msg_notification#a7eff811 bad_msg_id:long bad_msg_seqno:int error_code:int = BadMsgNotification;
```
(`mtproto_api.tl:55`)

The complete code list, transcribed from `SessionConnection.cpp:328-385`:

| Code | Name | Meaning | TDLib's action |
|------|------|---------|----------------|
| 16 | `MsgIdTooLow` | `msg_id` too far in the past | Re-send the message (time is updated automatically) |
| 17 | `MsgIdTooHigh` | `msg_id` too far in the future | **Reset `server_time_difference`, close the session** |
| 18 | `MsgIdMod4` | `msg_id` not divisible by 4 | Fatal — close |
| 19 | `MsgIdCollision` | Container and an older message share a `msg_id` | Fatal — close |
| 20 | `MsgIdTooOld` | Message too old / server no longer has the salt | Re-send |
| 32 | `SeqNoTooLow` | `seq_no` too low | Fatal — close |
| 33 | `SeqNoTooHigh` | `seq_no` too high | Fatal — close |
| 34 | `SeqNoNotEven` | Even `seq_no` expected for an irrelevant message | Fatal — close |
| 35 | `SeqNoNotOdd` | Odd `seq_no` expected for a relevant message | Fatal — close |
| 64 | `InvalidContainer` | Malformed container | Fatal — close |

Codes 16, 17 and 20 are **recoverable** — they mean your clock is off. The important one:

```cpp
case MsgIdTooHigh:
  to_send_.clear();
  reset_server_time_difference(info.message_id);
  callback_->on_session_failed(Status::Error("MessageId is too high"));
```
(`SessionConnection.cpp:350-356`)

`reset_server_time_difference` recomputes the offset from the `msg_id` of the
`bad_msg_notification` itself — the server's own message id encodes its current time. That
is how you resynchronize without an extra round trip. See
[5.4](04-salts-and-time.md).

Everything else means a bug in *your* implementation. TDLib logs
`". BUG! CALL FOR A DEVELOPER! Session will be closed"` for those
(`SessionConnection.cpp:343`), which is a fair summary.

---

## 4. `bad_server_salt`

```
bad_server_salt#edab447b bad_msg_id:long bad_msg_seqno:int error_code:int
    new_server_salt:long = BadMsgNotification;
```
(`mtproto_api.tl:56`)

`SessionConnection.cpp:389-397`:

```cpp
auth_data_->set_server_salt(bad_server_salt.new_server_salt_, Time::now_cached());
callback_->on_server_salt_updated();
on_message_failed(bad_info.message_id, Status::Error("Bad server salt"));
```

Three actions, all required:

1. Adopt `new_server_salt` immediately.
2. Persist it.
3. **Re-send** the message identified by `bad_msg_id` — it was not processed.

This is completely routine. Salts rotate roughly hourly, and the server hands you the new
one the moment you use a stale one. A client that handles `bad_server_salt` correctly never
needs `get_future_salts` at all.

---

## 5. `new_session_created`

```
new_session_created#9ec20908 first_msg_id:long unique_id:long server_salt:long = NewSession;
```
(`mtproto_api.tl:45`)

Sent when the server creates fresh state for your `session_id` — after a long gap, a server
restart, or your first message on a new session.

`SessionConnection.cpp:309-322`:

```cpp
auto first_message_id = MessageId(static_cast<uint64>(new_session_created.first_msg_id_));
auto it = service_queries_.find(first_message_id);
if (it != service_queries_.end()) {
  first_message_id = it->second.container_message_id_;
}
callback_->on_new_session_created(new_session_created.unique_id_, first_message_id);
```

**What it means for you:** every message you sent with a `msg_id` **less than**
`first_msg_id` was discarded. Re-send those queries.

> **💡 Implementation note.** TDLib **ignores** `new_session_created.server_salt`. It relies
> on `bad_server_salt` and `get_future_salts` instead. You may adopt the salt if you like;
> nothing breaks either way. What you must not ignore is `first_msg_id`.

---

## 6. `msgs_ack`

```
msgs_ack#62d6b459 msg_ids:Vector<long> = MsgsAck;
```
(`mtproto_api.tl:53`)

Server → client, it confirms receipt of your messages. Remove the listed ids from your
"unacknowledged" set. Client → server, see [5.2](02-containers-and-acks.md).

Received acks mean **received**, not **processed**. Keep waiting for `rpc_result`.

---

## 7. `pong` and pings

```
ping_delay_disconnect#f3427b8c ping_id:long disconnect_delay:int = Pong;
pong#347773c5 msg_id:long ping_id:long = Pong;
```
(`mtproto_api.tl:82, 40`)

Note that plain `ping#7abe77ec` is **commented out** in TDLib's schema
(`mtproto_api.tl:81`) — TDLib always uses the delay-disconnect variant, which doubles as a
server-side watchdog: if the server hears nothing from you for `disconnect_delay` seconds,
it closes the connection.

TDLib's parameters (`SessionConnection.cpp`, `SessionConnection.h`):

```cpp
ping_id = static_cast<int64>(auth_data_->next_message_id(Time::now_cached()).get());
…mtproto_api::ping_delay_disconnect(ping_id, ping_disconnect_delay() + 2)…
```

* `ping_id` is a fresh `msg_id` value, used purely as a nonce.
* `disconnect_delay` is your ping interval plus a small margin.

`pong.msg_id` is the `msg_id` of your ping message; `pong.ping_id` echoes your `ping_id`.
Use either to measure round-trip time.

Pings are **not content-related** (even `seq_no`) and need not be acknowledged.

---

## 8. `future_salts` and `get_future_salts`

```
get_future_salts#b921bd04 num:int = FutureSalts;
future_salt#0949d9dc valid_since:int valid_until:int salt:long = FutureSalt;
future_salts#ae500895 req_msg_id:long now:int salts:vector<future_salt> = FutureSalts;
```
(`mtproto_api.tl:80, 37, 38`)

TDLib requests **64** salts at a time, at most once per 60 seconds
(`SessionConnection.cpp:949-956`).

Note that `salts:vector<future_salt>` is **bare** — no `0x1cb5c415` tag, just an `int32`
count followed by the elements ([1.2 §4](../01-serialization/02-wire-encoding.md)).

`future_salts.now` is the server's current unix time and is another opportunity to
resynchronize.

Optional for a simple client. `bad_server_salt` alone is sufficient.

---

## 9. `gzip_packed`

```
gzip_packed#3072cfa1 packed_data:string = GzipPacked;
```
(`mtproto_api.tl:51`)

A transparent wrapper. It can appear:

* as the `result` inside `rpc_result` (handled at `SessionConnection.cpp:264-273`);
* as a whole message body (handled at `SessionConnection.cpp:408-423`).

Decompress with raw gzip (`gzdecode`) and re-dispatch the result as if it had arrived
uncompressed.

### Compressing outgoing messages

TDLib's policy (`td/telegram/net/NetQueryCreator.cpp`):

| Rule | Value |
|------|-------|
| Minimum size to attempt compression | 128 bytes (1024 for bots) |
| Required compression ratio | 0.9 |
| For payloads ≥ 16384 | Test-compress the middle 1024 bytes first, to avoid wasting CPU |

**You do not need to compress anything.** The server accepts uncompressed queries. But you
**must** be able to decompress, because the server compresses its replies aggressively.

> **⚠ Security note.** Bound the decompressed size. A malicious or buggy peer could send a
> small `gzip_packed` that expands to gigabytes. TDLib's `gzdecode` takes a
> `max_size` parameter for exactly this reason
> (`tdutils/td/utils/Gzip.cpp`).

---

## 10. Informational messages

These four tell you about message state. TDLib handles them minimally
(`SessionConnection.cpp:474-520`); you can ignore them entirely in a first
implementation.

```
msgs_state_req#da69fb52 msg_ids:Vector<long> = MsgsStateReq;
msgs_state_info#04deb57d req_msg_id:long info:string = MsgsStateInfo;
msgs_all_info#8cc0d131 msg_ids:Vector<long> info:string = MsgsAllInfo;
msg_detailed_info#276d3ec6 msg_id:long answer_msg_id:long bytes:int status:int = MsgDetailedInfo;
msg_new_detailed_info#809db6df answer_msg_id:long bytes:int status:int = MsgDetailedInfo;
msg_resend_req#7d861a08 msg_ids:Vector<long> = MsgResendReq;
```
(`mtproto_api.tl:58-63`)

The `info` field is a byte string with one status byte per message id:

| Value | Meaning |
|-------|---------|
| 1 | Nothing is known about the message |
| 2 | Message not received (id too low) |
| 3 | Message not received (id too high) |
| 4 | Message received, not yet processed |
| 5 | Message received and processed |
| 6 | Message received and processed, answer already acknowledged |
| 7 | Message received and processed, answer not acknowledged |

`msg_detailed_info` and `msg_new_detailed_info` tell you an answer exists for a query. The
practical response is to acknowledge `answer_msg_id` so the server stops resending it.

---

## 11. `destroy_auth_key`

```
destroy_auth_key#d1435160 = DestroyAuthKeyRes;
destroy_auth_key_ok#f660e1d4 = DestroyAuthKeyRes;
destroy_auth_key_none#0a9f2259 = DestroyAuthKeyRes;
destroy_auth_key_fail#ea109b13 = DestroyAuthKeyRes;
```
(`mtproto_api.tl:87, 67-69`)

Asks the server to forget the auth key. Used on logout. Handlers at
`SessionConnection.cpp:286-307`. Note the connection becomes unusable afterwards.

---

## 12. Minimal viable handler

```
fn dispatch(body, msg_id, seq_no):
    if seq_no is odd: pending_acks.push(msg_id)

    match read_u32(body):
        0x73f1f8dc => unpack container, dispatch each inner message
        0x3072cfa1 => dispatch(gunzip(body), msg_id, seq_no)
        0xf35c6d01 => handle_rpc_result(body)
        0x62d6b459 => for id in msg_ids: unacked.remove(id)
        0xedab447b => salt = new_server_salt; resend(bad_msg_id)
        0xa7eff811 => match error_code:
                          16 | 20 => resend(bad_msg_id)
                          17      => resync_time_from(msg_id); reset session; resend all
                          _       => fatal("implementation bug", error_code)
        0x9ec20908 => resend every query with msg_id < first_msg_id
        0x347773c5 => record round-trip time
        _          => handle_update(body)
```

That is roughly 60 lines and covers everything you need to log in and send a message.

---

## 13. Checklist

- [ ] Containers unpacked before dispatch
- [ ] `gzip_packed` handled both as a message body and inside `rpc_result`
- [ ] Decompressed size bounded
- [ ] `rpc_result` matched to the originating query by `req_msg_id`
- [ ] `rpc_error` inside `rpc_result` detected by peeking at `0x2144ca19`
- [ ] `bad_server_salt` updates the salt **and** resends
- [ ] `bad_msg_notification` codes 16/20 resend; code 17 resynchronizes time
- [ ] `new_session_created` resends everything below `first_msg_id`
- [ ] `msgs_ack` clears the resend queue
- [ ] Unknown constructors treated as updates, not errors

---

[← Previous](02-containers-and-acks.md) · [Index](../README.md) · [Next: Salts and time →](04-salts-and-time.md)
