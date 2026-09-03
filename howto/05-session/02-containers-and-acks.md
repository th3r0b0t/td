# 5.2 — Containers and Acknowledgements

[← Previous](01-msg-id-and-seq-no.md) · [Index](../README.md) · [Next: Service messages →](03-service-messages.md)

---

## 1. Why containers exist

One encrypted packet carries one message. If you want to send a query *and* acknowledge
three server messages *and* a ping, you would need four packets — four AES operations, four
TCP writes, four round trips of framing overhead.

`msg_container` packs several messages into one. The server unpacks it and processes each
message as if it had arrived separately.

**You can ignore containers on the send side entirely** and send one message per packet.
The protocol permits it. But **you must handle them on the receive side** — the server
containerizes aggressively and you will receive containers from the very first reply.

---

## 2. Format

```
msg_container#73f1f8dc messages:vector<%Message> = MessageContainer;
```
(`td/generate/scheme/mtproto_api.tl:47`, commented out because TDLib hand-codes it)

```
message msg_id:long seqno:int bytes:int body:string = Message;
```
(`mtproto_api.tl:48`)

Wire layout:

```
dc f8 f1 73                       msg_container#73f1f8dc
NN NN NN NN                       count (int32)
┌─ repeated `count` times ─────────────────────────────┐
│ <8 bytes>   msg_id                                   │
│ <4 bytes>   seqno                                    │
│ <4 bytes>   bytes  (length of body)                  │
│ <bytes>     body   (a complete TL object)            │
└──────────────────────────────────────────────────────┘
```

Two things to notice:

* The inner messages form a **bare** vector — there is **no** `0x1cb5c415` tag and no
  per-element boxing beyond the four fields. See
  [1.2 §4](../01-serialization/02-wire-encoding.md).
* Each inner message carries its **own** `msg_id` and `seqno`. They are independent
  messages that merely share a packet.

`ContainerImpl::do_store` (`td/mtproto/CryptoStorer.h:193-198`):

```cpp
storer.store_binary(mtproto_api::msg_container::ID);
storer.store_binary(cnt_);
storer.store_storer(storer_);
```

and each inner message, `QueryImpl::do_store` (`CryptoStorer.h:141-161`):

```cpp
storer.store_binary(query_.message_id);
storer.store_binary(query_.seq_no);
…
storer.store_binary(static_cast<uint32>(all_storer.size()));
storer.store_storer(all_storer);
```

---

## 3. The container's own header

The container is itself a message, so the outer envelope has its own `msg_id` and `seq_no`:

| Field | Value |
|-------|-------|
| Outer `msg_id` | A fresh id, generated **after** all the inner ids (so it is the largest) |
| Outer `seq_no` | `next_seq_no(is_content_related = false)` — **even** |

`SessionConnection.cpp` allocates the container's `message_id` last, precisely so that it
exceeds every id inside it.

---

## 4. Parsing a received container

```
fn handle_message(body, msg_id, seq_no):
    if read_u32(body) == 0x73f1f8dc:
        count = read_i32(body[4..])
        offset = 8
        for i in 0..count:
            inner_msg_id = read_u64(body[offset..])
            inner_seq_no = read_u32(body[offset+8..])
            inner_len    = read_u32(body[offset+12..])
            inner_body   = body[offset+16 .. offset+16+inner_len]
            handle_message(inner_body, inner_msg_id, inner_seq_no)   // recurse once
            offset += 16 + inner_len
    else:
        dispatch(body, msg_id, seq_no)
```

> **⚠ Security note.** Validate `count` and every `inner_len` against the remaining buffer
> before indexing. A malformed container is the easiest place to introduce a buffer
> over-read. Also **bound the recursion** — containers are not supposed to nest, so refuse
> a container inside a container rather than recursing arbitrarily.

Note that the *outer* container's `msg_id` is not acknowledged; you acknowledge the inner
messages individually, based on their own `seq_no` parity.

---

## 5. Limits

From `td/mtproto/SessionConnection.cpp` and `SessionConnection.h`:

| Limit | Value | Source |
|-------|-------|--------|
| Maximum queries per flush | `MAX_QUERY_COUNT = 1000` | `SessionConnection.cpp` |
| Maximum total query payload | `1 << 15` = 32768 bytes | `SessionConnection.cpp` |
| Maximum message ids in one `msgs_ack` / state request | 8192 | `SessionConnection.cpp` |

If you exceed the payload limit, split into several containers. In practice, for a "log in
and send a message" client, you will never approach any of these.

---

## 6. Acknowledgements

### When to acknowledge

`SessionConnection.cpp:512-514`:

```cpp
if ((seq_no & 1) != 0) {
  send_ack(message_id);
}
```

Acknowledge a received message **iff its `seq_no` is odd** (i.e. it was content-related).
Service messages from the server — acks, `new_session_created`, `bad_server_salt`, `pong` —
have even `seq_no` and need no acknowledgement.

### The message

```
msgs_ack#62d6b459 msg_ids:Vector<long> = MsgsAck;
```
(`mtproto_api.tl:53`)

Sent with `is_content_related = false`, so an **even** `seq_no`.

### When to flush

TDLib batches acks and sends them with the next outgoing packet, subject to two triggers
(`SessionConnection.h`):

| Trigger | Value |
|---------|-------|
| `ACK_DELAY` | 30 seconds — flush pending acks at most this long after the first |
| `MAX_UNACKED_PACKETS` | 100 — flush immediately once this many are pending |

So: collect acks in a list; piggyback them on whatever you send next; and if nothing is
being sent, flush after 30 seconds or 100 pending, whichever comes first.

### What happens if you do not acknowledge

The server keeps the message in its outgoing queue and **resends it periodically, forever**.
Your `rpc_result` for a query will arrive again and again. Since you deduplicate by
`msg_id` ([5.1](01-msg-id-and-seq-no.md)) you will not process it twice, but you will burn
bandwidth and the server will eventually consider the session unhealthy.

This is the single most common "my client mostly works but keeps getting the same reply"
bug.

### Receiving acks

```
msgs_ack#62d6b459 msg_ids:Vector<long> = MsgsAck;
```

The server also sends `msgs_ack` to confirm receipt of *your* messages. On receiving one,
remove those ids from your "sent but unacknowledged" set — you no longer need to be able to
resend them.

Note that an ack means "received", **not** "processed". The actual answer arrives separately
as `rpc_result`.

---

## 7. Building an outgoing packet — TDLib's assembly order

`CryptoImpl` (`td/mtproto/CryptoStorer.h:205-368`) assembles, in this order, whichever
parts are present:

```
1.  queries        (the actual API calls, each content-related)
2.  msgs_ack       (pending acknowledgements)
3.  http_wait      (only for the HTTP transport)
4.  get_future_salts
5.  msgs_state_req (asking about messages you have not heard about)
6.  msg_resend_req
7.  rpc_drop_answer (cancellations)
8.  ping / ping_delay_disconnect
9.  destroy_auth_key
```

Then (`CryptoStorer.h:188-203`):

```
if (cnt_ > 1) wrap everything in msg_container
else          send the single message directly
```

The `cnt_ > 1` condition matters: a container holding one message is legal but wasteful, and
TDLib avoids it.

---

## 8. A minimal send strategy

For a first implementation:

```
send_query(q):
    msg_id  = next_msg_id()
    seq_no  = next_seq_no(content_related = true)
    parts   = [(msg_id, seq_no, serialize(q))]

    if pending_acks not empty:
        parts.append((next_msg_id(), next_seq_no(false), msgs_ack(pending_acks)))
        pending_acks.clear()

    if len(parts) == 1:
        encrypt_and_send(parts[0])
    else:
        body = msg_container(parts)
        encrypt_and_send((next_msg_id(), next_seq_no(false), body))

    remember(msg_id -> q)          # for retries and rpc_result matching
```

That is enough to log in and send a message. Batching, quick acks, and the state-request
machinery are optimizations.

---

## 9. Checklist

- [ ] Container id `0x73f1f8dc`, then `int32` count, then bare inner messages
- [ ] Inner message = `msg_id(8) ‖ seqno(4) ‖ bytes(4) ‖ body`
- [ ] **No** `0x1cb5c415` vector tag inside the container
- [ ] Container's own `seq_no` is even (not content-related)
- [ ] Container's own `msg_id` is greater than every inner `msg_id`
- [ ] Only build a container when there is more than one message
- [ ] Received containers unpacked and each inner message dispatched
- [ ] Inner lengths validated against the remaining buffer
- [ ] Nested containers rejected
- [ ] Messages with odd `seq_no` acknowledged via `msgs_ack`
- [ ] `msgs_ack` sent with even `seq_no`
- [ ] Acks flushed within 30 s or after 100 pending

---

[← Previous](01-msg-id-and-seq-no.md) · [Index](../README.md) · [Next: Service messages →](03-service-messages.md)
