# 5.1 — `msg_id`, `seq_no` and `session_id`

[← Previous](../04-encrypted-messages/04-encrypt-decrypt-recipes.md) · [Index](../README.md) · [Next: Containers and acks →](02-containers-and-acks.md)

---

Three small integers in the message header carry the whole ordering and deduplication
scheme. They are also where implementations most often go quietly wrong — a bad `msg_id`
does not produce an error, it produces silence.

---

## 1. `session_id`

An 8-byte random value identifying one logical stream of messages. It is **not** the TCP
connection: you keep the same `session_id` across reconnects, and the server keeps
per-session state (acknowledged messages, sequence numbers) for it.

`td/telegram/net/Session.cpp:261-265`:

```cpp
uint64 session_id = 0;
do {
  Random::secure_bytes(reinterpret_cast<uint8 *>(&session_id), sizeof(session_id));
} while (session_id == 0);
```

Rules:

* Random, from a CSPRNG.
* **Must not be zero.**
* Constant for the life of the session.
* Every incoming message must carry it; reject those that do not
  (`AuthData::check_packet`, `AuthData.cpp:142-145`).

**Generate a new one when:** you deliberately abandon all in-flight queries, or the server
tells you to via `new_session_created` handling that you cannot reconcile. Otherwise keep
it — a new `session_id` means the server forgets everything it knew about your pending
messages.

Generating a new `session_id` also requires resetting `seq_no` to 0
(`AuthData::clear_seq_no`, `AuthData.h:276-278`).

---

## 2. `msg_id`

A 64-bit value that is simultaneously a timestamp, a unique identifier, and an ordering
key.

### Structure

```
┌────────────────────────────────┬──────────────────────────────┐
│ unix time (seconds)  32 bits   │ fractional + counter  32 bits│
└────────────────────────────────┴──────────────────────────────┘
```

Approximately `unix_time × 2³²`. The low two bits encode the message *kind*:

| `msg_id % 4` | Meaning |
|--------------|---------|
| `0` | Client → server, **content-related** |
| `1` | Client → server, not content-related (service message) |
| `2` | Server → client, content-related |
| `3` | Server → client, not content-related |

In practice you only need two rules:

* **Everything you send must be even** (divisible by 4 in TDLib's implementation).
* **Everything you receive must be odd** — enforce this.

### Generation

`AuthData::next_message_id` (`td/mtproto/AuthData.cpp:107-125`):

```cpp
MessageId AuthData::next_message_id(double now) {
  double server_time = get_server_time(now);
  auto t = static_cast<uint64>(server_time * (static_cast<uint64>(1) << 32));

  // randomize lower bits for clocks with low precision
  auto rx = Random::secure_int32();
  auto to_xor = rx & ((1 << 22) - 1);

  t ^= to_xor;
  auto result = MessageId(t & static_cast<uint64>(-4));
  if (last_message_id_ >= result) {
    auto to_mul = ((rx >> 22) & 1023) + 1;
    result = MessageId(last_message_id_.get() + 8 * to_mul);
  }
  last_message_id_ = result;
  return result;
}
```

Step by step:

1. `t = server_time × 2³²` — note **server** time, i.e. `local_time + server_time_diff`.
2. XOR the low 22 bits with random. This is deliberate: on systems with coarse clocks,
   consecutive calls would otherwise produce identical values.
3. `& ~3` clears the low two bits, making the id divisible by 4 (client, content-related).
4. If the result is not strictly greater than the previous id, bump it by
   `8 × (1..1024)` instead. The multiple of 8 preserves divisibility by 4.
5. Remember it as `last_message_id_`.

> **⚠ Critical.** `msg_id` must be **strictly increasing** within a session. If you send
> two messages with `msg_id` out of order, the server replies
> `bad_msg_notification` with error code 16 or 17, or silently ignores you. Step 4 is what
> guarantees monotonicity when the clock has not advanced.

### Validity windows

`AuthData.cpp:127-137`:

```cpp
bool AuthData::is_valid_outbound_msg_id(MessageId message_id, double now) const {
  double server_time = get_server_time(now);
  auto id_time = static_cast<double>(message_id.get()) / static_cast<double>(1ULL << 32);
  return server_time - 150 < id_time && id_time < server_time + 30;
}

bool AuthData::is_valid_inbound_msg_id(MessageId message_id, double now) const {
  …
  return server_time - 300 < id_time && id_time < server_time + 30;
}
```

| Direction | Window |
|-----------|--------|
| Outbound (yours) | `[server_time − 150 s, server_time + 30 s]` |
| Inbound (server's) | `[server_time − 300 s, server_time + 30 s]` |

The asymmetry is intentional: the server is stricter about what it accepts than you need to
be, and the server may re-send old messages you have not acknowledged.

> **⚠ This is why `server_time_diff` matters.** If your system clock is 10 minutes fast and
> you generate `msg_id` from local time, every message you send falls outside the server's
> window and is dropped. The server never tells you why in a way you will notice unless you
> handle `bad_msg_notification` code 17. Always base `msg_id` on
> `local_time + server_time_diff` — see [5.4](04-salts-and-time.md).

### Duplicate detection

`AuthData::check_packet` calls `duplicate_checker_.check(message_id)`
(`AuthData.cpp:153`). The checker (`td/mtproto/AuthData.h`) keeps a sliding window of the
last **1000** message ids and rejects repeats.

The server retransmits messages you have not acknowledged, so duplicates are normal, not an
attack. Handle them by silently dropping — but still acknowledge them, or the server will
keep resending.

---

## 3. `seq_no`

A 32-bit counter of **content-related** messages.

`AuthData::next_seq_no` (`td/mtproto/AuthData.h:267-274`):

```cpp
int32 next_seq_no(bool is_content_related) {
  int32 res = seq_no_;
  if (is_content_related) {
    res |= 1;
    seq_no_ += 2;
  }
  return res;
}
```

So:

| Message kind | `seq_no` returned | Counter advances? |
|--------------|-------------------|-------------------|
| Content-related | `counter | 1` (**odd**) | yes, by 2 |
| Not content-related | `counter` (**even**) | no |

Since `seq_no_` starts at 0 and only ever increases by 2, it is always even; the `| 1`
makes content-related messages odd.

### What is "content-related"?

TDLib's answer is unambiguous — in `SessionConnection.cpp:829`, when queueing a query:

```cpp
to_send_.push_back(MtprotoQuery{message_id, seq_no, …, /* is_content_related */ true, …});
```

and every service message is sent with `false`. Concretely:

| Content-related (odd `seq_no`) | Not content-related (even `seq_no`) |
|--------------------------------|--------------------------------------|
| Any API query (`help.getConfig`, `auth.sendCode`, `messages.sendMessage`, …) | `msgs_ack` |
| `invokeWithLayer` wrapping a query | `ping`, `ping_delay_disconnect` |
| | `http_wait` |
| | `get_future_salts` |
| | `msg_container` itself |
| | `msgs_state_req`, `msg_resend_req` |
| | `destroy_auth_key`, `rpc_drop_answer` |

The rule of thumb: **if the server must remember it and might resend a reply, it is
content-related.** Acks and pings are fire-and-forget.

> **💡 Implementation note.** The container `msg_container` is *not* content-related, but
> the messages *inside* it each carry their own `msg_id` and `seq_no` and follow the normal
> rules. See [5.2](02-containers-and-acks.md).

### Why it exists

`seq_no` lets each side detect gaps: if the server has processed content-related messages
with `seq_no` 1, 3, 7, it knows it missed 5. It also tells the receiver whether a message
needs acknowledging — TDLib queues an ack exactly when the received `seq_no` is odd
(`SessionConnection.cpp:512-514`):

```cpp
if ((seq_no & 1) != 0) {
  send_ack(message_id);
}
```

Do the same. Acknowledging every message, including even-`seq_no` ones, is harmless but
wasteful; failing to acknowledge odd ones causes endless retransmission.

---

## 4. Server-side validation you will trip over

`bad_msg_notification` codes related to these fields
(`td/mtproto/SessionConnection.cpp`, and see [5.3](03-service-messages.md)):

| Code | Meaning | Cause |
|------|---------|-------|
| 16 | `msg_id` too low | Your clock is behind, or you reused an id |
| 17 | `msg_id` too high | Your clock is ahead; you ignored `server_time_diff` |
| 18 | `msg_id` not divisible by 4 | You forgot `& ~3` |
| 19 | `msg_id` collision | Two messages in one container share an id |
| 20 | `msg_id` too old | The container has been waiting too long |
| 32 | `seq_no` too low | Counter went backwards |
| 33 | `seq_no` too high | You skipped values |
| 34 | Even `seq_no` on a content-related message | You passed `is_content_related = false` |
| 35 | Odd `seq_no` on a non-content-related message | The reverse |

Codes 16 and 17 include the correct server time in the reply, which lets you resynchronize
— see [5.4](04-salts-and-time.md).

---

## 5. Complete state

```
struct Session {
    auth_key:          [u8; 256],
    auth_key_id:       [u8; 8],
    session_id:        u64,        // random non-zero, stable
    server_salt:       u64,        // updated by the server
    salt_valid_until:  i32,
    server_time_diff:  f64,        // server_time - local_time
    last_msg_id:       u64,        // for monotonicity
    seq_no:            u32,        // even; incremented by 2
    pending_acks:      Vec<u64>,
    sent_unacked:      Map<u64, Query>,
    seen_msg_ids:      RingBuffer<u64, 1000>,
}
```

### Generating a header

```
fn next_msg_id(s: &mut Session) -> u64 {
    let server_time = now() + s.server_time_diff;
    let mut t = (server_time * 4294967296.0) as u64;
    let rx = csprng_u32();
    t ^= (rx & 0x3F_FFFF) as u64;              // low 22 bits
    let mut id = t & !3u64;
    if id <= s.last_msg_id {
        id = s.last_msg_id + 8 * (((rx >> 22) & 1023) as u64 + 1);
    }
    s.last_msg_id = id;
    id
}

fn next_seq_no(s: &mut Session, content_related: bool) -> u32 {
    let res = s.seq_no;
    if content_related {
        s.seq_no += 2;
        return res | 1;
    }
    res
}
```

---

## 6. Checklist

- [ ] `session_id` is random, non-zero, stable across reconnects
- [ ] `msg_id` derived from **server** time, not local time
- [ ] `msg_id` low two bits cleared (`& ~3`)
- [ ] `msg_id` strictly increasing; bump by a multiple of 8 on collision
- [ ] `seq_no` starts at 0, `| 1` and `+= 2` only for content-related messages
- [ ] Incoming `session_id` verified
- [ ] Incoming `msg_id` verified odd
- [ ] Incoming `msg_id` checked against the ±300/+30 s window
- [ ] Duplicate incoming `msg_id`s dropped (window of ~1000)
- [ ] Messages with odd incoming `seq_no` acknowledged

---

[← Previous](../04-encrypted-messages/04-encrypt-decrypt-recipes.md) · [Index](../README.md) · [Next: Containers and acks →](02-containers-and-acks.md)
