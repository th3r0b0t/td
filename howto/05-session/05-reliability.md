# 5.5 — Reliability: Resending, Cancellation, Quick Acks

[← Previous](04-salts-and-time.md) · [Index](../README.md) · [Next: initConnection →](../06-api-layer/01-init-connection.md)

---

MTProto assumes the connection can drop at any moment. This chapter covers what you must do
to survive that, and a few optional mechanisms.

---

## 1. The resend obligation

Keep every sent query in a map `msg_id → query` until you receive **either** an `rpc_result`
for it **or** an `msgs_ack` covering it. You will need to resend it in five situations:

| Trigger | What to resend |
|---------|----------------|
| Connection dropped | Everything unacknowledged |
| `bad_server_salt` | The message named by `bad_msg_id` |
| `bad_msg_notification` code 16 or 20 | The message named by `bad_msg_id` |
| `new_session_created` | Everything with `msg_id < first_msg_id` |
| `msg_resend_req` from the server | The listed ids |

`SessionConnection::on_message_failed` is TDLib's single entry point for all of these.

> **⚠ Important.** A resend is a **new message**: allocate a fresh `msg_id` and a fresh
> `seq_no`. Do not reuse the old ones — that triggers `bad_msg_notification` code 16 or 19.
> The *query* is the same; the *message* is new.

### Idempotency

Resending an API call means the server may execute it twice. For most calls that is
harmless. For `messages.sendMessage` it is not — you would post the message twice.

The protection is `random_id`, a field on every message-sending method
(`telegram_api.tl`, `messages.sendMessage`). The server deduplicates on it. So:

**Generate `random_id` once, when you first construct the query, and keep it identical
across every resend.** See [8.3](../08-sending-a-message/03-send-message.md).

---

## 2. Reconnection

Losing the TCP connection costs you:

| Lost | Kept |
|------|------|
| Obfuscation CTR state (regenerate the 64-byte header) | `auth_key` |
| Transport framing state | `auth_key_id` |
| In-flight, unacknowledged messages | `session_id` |
| | `seq_no` counter |
| | `server_salt` |
| | `server_time_difference` |

So on reconnect: open a new socket, send the framing magic (or a new obfuscation header),
and resend everything unacknowledged with fresh `msg_id`s. Do **not** generate a new
`session_id` and do **not** reset `seq_no` — the server still has the session.

TDLib backs off between attempts and rotates through the DC's addresses
(`td/telegram/net/ConnectionCreator.cpp`). A simple exponential backoff from 1 s to 60 s
with jitter is adequate.

---

## 3. Quick acknowledgements

An optimization: get told "received" one round trip earlier than a full `msgs_ack`.

### Requesting one

Set bit 31 of the Intermediate length prefix when sending
(`td/mtproto/TcpTransport.cpp:54-56`):

```cpp
if (quick_ack) {
  size |= static_cast<size_t>(1) << 31;
}
```

### The token

`Transport::calc_message_key2` returns it alongside `msg_key`
(`td/mtproto/Transport.cpp:163`):

```cpp
return std::make_pair(as<uint32>(msg_key_large_raw) | (1u << 31), res);
```

```
quick_ack_token = msg_key_large[0..4) as u32 LE, with bit 31 forced on
```

Remember `token → msg_id` for messages you sent with the quick-ack flag.

### Receiving one

Two forms, both already covered:

* A bare 4-byte frame whose value has bit 31 set — recognized by
  `IntermediateTransport::read_from_stream` (`TcpTransport.cpp:30-36`).
* An 8-byte message body starting with `int32 -1` followed by the token — recognized by
  `Transport::read` (`Transport.cpp:432-434`).

Look up the token, mark that message received, drop it from the resend queue.

> **💡 Implementation note.** Skip this entirely for a first implementation. It saves
> latency on file uploads; it does nothing for a login flow. But **do** handle the
> *receipt* of an unexpected quick ack gracefully — see
> [2.4](../02-transport/04-transport-errors.md).

---

## 4. Cancelling a query

```
rpc_drop_answer#58e4a740 req_msg_id:long = RpcDropAnswer;
rpc_answer_unknown#5e2ad36e = RpcDropAnswer;
rpc_answer_dropped_running#cd78e586 = RpcDropAnswer;
rpc_answer_dropped#a43ad8b7 msg_id:long seq_no:int bytes:int = RpcDropAnswer;
```
(`mtproto_api.tl:79, 33-35`)

Tells the server you no longer want the answer. Three possible replies:

| Reply | Meaning |
|-------|---------|
| `rpc_answer_unknown` | The server has no record of the query |
| `rpc_answer_dropped_running` | The query is executing; it will complete but the answer is discarded |
| `rpc_answer_dropped` | The answer existed and has been dropped |

Note the last two do **not** mean the operation was undone — a cancelled
`messages.sendMessage` may still have sent the message.

Not needed for a simple client.

---

## 5. Ordering with `invokeAfterMsg`

```
invokeAfterMsg#cb9f372d {X:Type} msg_id:long query:!X = X;
invokeAfterMsgs#3dc4b4f0 {X:Type} msg_ids:Vector<long> query:!X = X;
```
(`td/generate/scheme/telegram_api.tl`)

Wraps a query so the server executes it only after the named message(s) complete. TDLib
emits it via `InvokeAfter` in `td/mtproto/CryptoStorer.h:146-147` and uses it to keep
message-sending in order.

Without it, two `messages.sendMessage` calls sent back-to-back may be processed in either
order. If you send exactly one message, you do not need it. If you send several and care
about ordering, chain them.

---

## 6. `msgs_state_req` and `msg_resend_req`

```
msgs_state_req#da69fb52 msg_ids:Vector<long> = MsgsStateReq;
msg_resend_req#7d861a08 msg_ids:Vector<long> = MsgResendReq;
```
(`mtproto_api.tl:59, 58`)

Use `msgs_state_req` to ask "what happened to these messages?" when you have heard nothing
for a long time, and `msg_resend_req` to ask the server to resend answers you missed. Both
are limited to 8192 ids per request in TDLib.

For a simple client, blind resending on reconnect is simpler and works.

---

## 7. Pings as a liveness check

```
ping_delay_disconnect#f3427b8c ping_id:long disconnect_delay:int = Pong;
```
(`mtproto_api.tl:82`)

Send one every N seconds with `disconnect_delay = N + 2`. Two benefits:

* If you stop pinging, the server closes the connection rather than leaving it half-open.
* If you do not get a `pong` within a timeout, *you* know the connection is dead — TCP
  alone can take many minutes to notice.

TDLib's HTTP-related constants for comparison (`SessionConnection.h`):
`HTTP_MAX_DELAY = 30`, `HTTP_MAX_AFTER = 10`, `http_max_wait = 25`.

---

## 8. Recommended minimal reliability layer

```
struct Pending {
    query:     Vec<u8>,     // serialized, WITHOUT msg_id/seq_no
    msg_id:    u64,         // current msg_id (changes on resend)
    random_id: i64,         // stable across resends, for idempotency
    sent_at:   Instant,
}

on_send(q):
    p = Pending { query: serialize(q), msg_id: next_msg_id(), random_id: q.random_id, … }
    pending.insert(p.msg_id, p)
    encrypt_and_send(p)

on_ack(ids):                        // msgs_ack, or a quick ack
    for id in ids: pending[id].acked = true

on_rpc_result(req_msg_id, result):
    pending.remove(req_msg_id)
    complete(req_msg_id, result)

resend(old_msg_id):
    p = pending.remove(old_msg_id)
    p.msg_id = next_msg_id()        // NEW id
    p.seq_no = next_seq_no(true)    // NEW seq_no
    pending.insert(p.msg_id, p)     // same query bytes, same random_id
    encrypt_and_send(p)

on_reconnect():
    for p in pending.values(): resend(p.msg_id)
```

Everything else in this chapter is optional.

---

## 9. Checklist

- [ ] Sent queries retained until `rpc_result` or `msgs_ack`
- [ ] Resends allocate a **new** `msg_id` and `seq_no`
- [ ] Resends reuse the **same** `random_id`
- [ ] Resend on: disconnect, `bad_server_salt`, `bad_msg_notification` 16/20,
      `new_session_created`, `msg_resend_req`
- [ ] `session_id` and `seq_no` preserved across reconnects
- [ ] Obfuscation header regenerated on each new connection
- [ ] Unexpected quick acks handled without crashing
- [ ] Reconnect uses exponential backoff with jitter
- [ ] Optional: periodic `ping_delay_disconnect` as a liveness check

---

[← Previous](04-salts-and-time.md) · [Index](../README.md) · [Next: initConnection →](../06-api-layer/01-init-connection.md)
