# 5.4 — Server Salts and Time Synchronization

[← Previous](03-service-messages.md) · [Index](../README.md) · [Next: Reliability →](05-reliability.md)

---

Two pieces of shared state that are easy to overlook and cause maddening bugs when wrong.

---

## 1. Server salts

### What they are

A `server_salt` is a 64-bit value included in the encrypted header of every message
([4.1](../04-encrypted-messages/01-envelope.md)). It has a validity period. Its purpose is
replay protection: a message captured today cannot be replayed next week, because the salt
it was encrypted with will no longer be accepted.

### Where the first one comes from

From the handshake (`td/mtproto/Handshake.cpp:250`):

```cpp
server_salt_ = as<int64>(new_nonce_.raw) ^ as<int64>(server_nonce_.raw);
```

```
server_salt = new_nonce[0..8) XOR server_nonce[0..8)
```

### Validity

`AuthData::set_server_salt` (`td/mtproto/AuthData.h:225-231`):

```cpp
void set_server_salt(uint64 salt, double now) {
  server_salt_.salt = salt;
  double server_time = get_server_time(now);
  server_salt_.valid_since = server_time;
  server_salt_.valid_until = server_time + 60 * 10;      // 10 minutes
  future_salts_.clear();
}
```

and `is_server_salt_valid` (`AuthData.h:233-235`):

```cpp
bool is_server_salt_valid(double now) const {
  return server_salt_.valid_until > get_server_time(now) + 60;
}
```

So a salt learned from `bad_server_salt` is assumed good for 10 minutes, and considered
"needs replacing" once fewer than 60 seconds remain. The one-minute margin is there so you
switch *before* the server starts rejecting.

Note that `set_server_salt` **clears the future-salt cache** — the server has just told you
its idea of the current salt, which supersedes anything you had queued.

### Getting new ones

Two mechanisms:

**Reactive — `bad_server_salt`** ([5.3 §4](03-service-messages.md)). You use a stale salt,
the server tells you the right one and rejects that one message, you adopt it and re-send.
Costs one round trip, requires no proactive logic, and is **sufficient on its own**.

**Proactive — `get_future_salts`**:

```
get_future_salts#b921bd04 num:int = FutureSalts;
future_salt#0949d9dc valid_since:int valid_until:int salt:long = FutureSalt;
future_salts#ae500895 req_msg_id:long now:int salts:vector<future_salt> = FutureSalts;
```
(`mtproto_api.tl:80, 37, 38`)

TDLib asks for **64** salts, at most once per 60 seconds, and only when
`need_future_salts()` is true (`SessionConnection.cpp:949-956`). The reply is stored sorted
by `valid_since` descending (`AuthData.cpp:91-99`) and the newest applicable one is picked
as time advances.

> **💡 Implementation note.** For a client that logs in and sends one message, implement
> **only** the reactive path. Adopt whatever `bad_server_salt` gives you and re-send. Add
> `get_future_salts` when you build a long-running client and want to avoid the occasional
> extra round trip.

### Persisting

Persist the current salt alongside the auth key. On restart, if it has expired, your first
message triggers `bad_server_salt` and you recover in one round trip — cheap and correct.

---

## 2. Time synchronization

### Why it matters

`msg_id ≈ unix_time × 2³²` ([5.1](01-msg-id-and-seq-no.md)), and the server rejects any
`msg_id` more than 30 seconds in the future or 300 seconds in the past. If your clock is
wrong, **every message you send is silently discarded**.

This is not hypothetical: virtual machines, embedded devices, and freshly booted systems
routinely have clocks minutes or hours off.

### The rule

**Always use server time, never local time, when generating `msg_id`.**

```
server_time = local_time + server_time_difference
```

`AuthData::next_message_id` (`AuthData.cpp:107-109`) opens with exactly that:

```cpp
double server_time = get_server_time(now);
auto t = static_cast<uint64>(server_time * (static_cast<uint64>(1) << 32));
```

### Learning the difference

Three sources, in order of when you encounter them:

**1. `server_DH_inner_data.server_time` during the handshake**
(`Handshake.cpp:221`):

```cpp
server_time_diff_ = dh_inner_data.server_time_ - Time::now();
```

This is your initial value, available before you send a single encrypted message.

**2. Every incoming message's `msg_id`** (`AuthData.cpp:156`):

```cpp
time_difference_was_updated = update_server_time_difference(
    static_cast<uint32>(message_id.get() >> 32) - now);
```

The high 32 bits of any server `msg_id` are the server's unix time. Free continuous
synchronization on every packet.

**3. `future_salts.now`** — the server's current time, if you use that call.

### The update rule is asymmetric

`AuthData::update_server_time_difference` (`AuthData.cpp:70-83`):

```cpp
bool AuthData::update_server_time_difference(double diff) {
  if (!server_time_difference_was_updated_) {
    server_time_difference_was_updated_ = true;
    server_time_difference_ = diff;
  } else if (server_time_difference_ + 1e-4 < diff) {
    server_time_difference_ = diff;
  } else {
    return false;
  }
  return true;
}
```

After the first value, the difference **only ever increases**. TDLib's own comment
(`AuthData.h:214-215`) explains why:

```
// diff == msg_id / 2^32 - now == old_server_now - now <= server_now - now
// server_time_difference >= max{diff}
```

A `msg_id` was generated at some point *before* it reached you, so the time it encodes is a
*lower bound* on the server's current time. Taking the maximum over all observations gives
the tightest safe estimate. Taking the latest observation instead would let network jitter
pull your clock backwards, producing `msg_id`s the server considers too old.

### Resetting

`AuthData::reset_server_time_difference` (`AuthData.cpp:85-89`):

```cpp
void AuthData::reset_server_time_difference(double diff) {
  server_time_difference_was_updated_ = false;
  server_time_difference_ = diff;
}
```

This clears the "monotonic increase" latch, allowing the difference to move **downwards**.
It is called in exactly two situations:

* `bad_msg_notification` code 17 (`msg_id` too high) — `SessionConnection.cpp:354`. Your
  estimate is too far ahead; start over from the server's own message id.
* An `rpc_result` arriving more than 15 seconds "before" the query it answers
  (`SessionConnection.cpp:251-253`):
  ```cpp
  if (info.message_id.get() < req_msg_id - (static_cast<uint64>(15) << 32)) {
    reset_server_time_difference(info.message_id);
  }
  ```
  An answer cannot precede its question by 15 seconds; the estimate must be wrong.

### Persisting

**Persist `server_time_difference`.** On restart, your first `msg_id` will then be correct
immediately, rather than requiring a `bad_msg_notification` round trip. TDLib does this via
`AuthDataShared`.

---

## 3. Failure modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Handshake succeeds, then total silence | `msg_id` from local time on a skewed clock | Use `server_time_diff` |
| Occasional `bad_msg_notification` 16 | Clock drift, or a long pause between messages | Normal; resend |
| Repeated `bad_msg_notification` 17 | You are not calling `reset_server_time_difference` | Implement the reset path |
| `bad_server_salt` on every message | Not adopting the new salt, or not persisting it | Store it before resending |
| Works, then breaks after ~1 hour idle | Salt expired and you never handled `bad_server_salt` | Implement it |
| Client on a VM works, on a laptop does not | Laptop clock is off | Same root cause; use server time |

---

## 4. Minimal correct implementation

```
struct TimeState {
    server_time_diff: f64,
    diff_was_updated: bool,
}

fn server_time(t: &TimeState) -> f64 {
    now() + t.server_time_diff
}

fn observe_server_msg_id(t: &mut TimeState, msg_id: u64) {
    let diff = (msg_id >> 32) as f64 - now();
    if !t.diff_was_updated {
        t.diff_was_updated = true;
        t.server_time_diff = diff;
    } else if t.server_time_diff + 1e-4 < diff {
        t.server_time_diff = diff;          // monotonic increase only
    }
}

fn reset_time(t: &mut TimeState, msg_id: u64) {
    t.diff_was_updated = false;
    t.server_time_diff = (msg_id >> 32) as f64 - now();
}
```

Call `observe_server_msg_id` on **every** decrypted message. Call `reset_time` on
`bad_msg_notification` code 17.

---

## 5. Checklist

- [ ] Initial `server_time_diff` from `server_DH_inner_data.server_time`
- [ ] Updated from the `msg_id` of every incoming message
- [ ] Updates are **monotonically increasing** after the first
- [ ] Reset on `bad_msg_notification` code 17
- [ ] Reset when an `rpc_result` predates its query by > 15 s
- [ ] `msg_id` generated from `local_time + server_time_diff`
- [ ] Both `server_time_diff` and `server_salt` persisted
- [ ] Initial salt = `new_nonce[0..8) ⊕ server_nonce[0..8)`
- [ ] `bad_server_salt` adopts the new salt **and** resends the rejected message

---

[← Previous](03-service-messages.md) · [Index](../README.md) · [Next: Reliability →](05-reliability.md)
