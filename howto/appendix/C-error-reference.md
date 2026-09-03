# Appendix C — Error Reference

[← Previous](B-datacenters-and-keys.md) · [Index](../README.md) · [Next: Appendix D →](D-tdlib-source-map.md)

---

Every error you can meet, grouped by the layer that produces it. The three layers need
genuinely different handling — see [10.1 §7](../10-implementation/01-project-skeleton.md).

---

## C.1 — Transport errors

Bare 4-byte frames carrying a negative `int32`. No auth key, no `msg_id`, no message —
just a number, usually followed by the server closing the connection.

`Transport::read` (`td/mtproto/Transport.cpp:418-437`):

```cpp
if (error_code >= 300) {
  return ReadResult::make_error(-error_code);
}
if (message.size() < 16) {
  if (message.size() < 4) { return Status::Error("Invalid MTProto message: smaller than 4 bytes"); }
  int32 code = as<int32>(message.begin());
  if (code == 0) {
    return ReadResult::make_nop();
  } else if (code == -1 && message.size() >= 8) {
    return ReadResult::make_quick_ack(as<uint32>(message.begin() + 4));
  } else {
    return ReadResult::make_error(code);
  }
}
```

| Code | Meaning | What to do |
|------|---------|------------|
| `0` | No-op / keepalive | Ignore |
| `-1` + `uint32` | Quick acknowledgement token | Match against sent messages ([5.5 §3](../05-session/05-reliability.md)) |
| `-404` | Auth key not found | Discard the stored key, redo the handshake |
| `-429` | Too many connections from this IP | Back off hard |
| `-444` | Invalid DC / bad request to this DC | Check you are talking to the right DC |
| `-400` | Bad request at the transport layer | Framing or padding is wrong |
| `-388` | Bad auth key | Key is wrong or byte order is reversed |
| `-100` | Malformed packet | First packet is wrong; check framing magic |

Any 4-byte frame under 16 bytes that is not `0` and not `-1` is an error code. Do not try
to decrypt it.

> **⚠ `-404` is ambiguous.** It means "I do not recognize this `auth_key_id`", which can be
> a revoked key **or** an `auth_key_id` you computed or serialized wrongly. If it happens on
> your very first encrypted message, suspect your `SHA1(auth_key)[12..20)` byte order
> before you suspect the server.

Codes ≥ 300 arriving as an HTTP-ish status are negated (`Transport.cpp:420-422`), so a
`404` becomes `-404`. Both spellings mean the same thing.

---

## C.2 — Protocol errors: `bad_msg_notification`

```
bad_msg_notification#a7eff811 bad_msg_id:long bad_msg_seqno:int error_code:int
```
(`td/generate/scheme/mtproto_api.tl:55`)

Full handling at `td/mtproto/SessionConnection.cpp:324-387`.

| Code | Name | Meaning | Recovery |
|------|------|---------|----------|
| 16 | `MsgIdTooLow` | `msg_id` too far in the past | **Resend.** Time updates automatically |
| 17 | `MsgIdTooHigh` | `msg_id` too far in the future | Clear queue, `reset_server_time_difference`, close session |
| 18 | `MsgIdMod4` | `msg_id` not divisible by 4 | **Bug in your code** — fatal |
| 19 | `MsgIdCollision` | Container/message `msg_id` collision | **Bug** — fatal |
| 20 | `MsgIdTooOld` | Message too old | **Resend** |
| 32 | `SeqNoTooLow` | `seq_no` below expected | **Bug** — fatal |
| 33 | `SeqNoTooHigh` | `seq_no` above expected | **Bug** — fatal |
| 34 | `SeqNoNotEven` | Even `seq_no` required (irrelevant message) | **Bug** — fatal |
| 35 | `SeqNoNotOdd` | Odd `seq_no` required (relevant message) | **Bug** — fatal |
| 64 | `InvalidContainer` | Malformed `msg_container` | **Bug** — fatal |

Only **16 and 20 are recoverable** — resend with a fresh `msg_id`. Everything else is a
client bug; TDLib logs `"BUG! CALL FOR A DEVELOPER! Session will be closed"`
(`SessionConnection.cpp:343`) and closes the session.

Code 17 is special: it clears `to_send_`, calls `reset_server_time_difference(info.message_id)`,
and fails the session (`SessionConnection.cpp:350-356`). This is the **only** path that
lets `server_time_difference` decrease — see
[5.4 §2](../05-session/04-salts-and-time.md).

### Diagnosing the fatal ones

| Code | Almost certainly |
|------|------------------|
| 18 | Not masking `msg_id` with `& ~3` |
| 19 | Reusing a `msg_id` on resend instead of allocating a new one |
| 32/33 | `seq_no` counter not shared across the session, or reset on reconnect |
| 34 | Sending an odd `seq_no` for a service message (acks, pings) |
| 35 | Sending an even `seq_no` for a content message (API calls) |
| 64 | Container inner messages not bare, or `bytes` field wrong |

---

## C.3 — `bad_server_salt`

```
bad_server_salt#edab447b bad_msg_id:long bad_msg_seqno:int error_code:int new_server_salt:long
```
(`mtproto_api.tl:56`)

Not really an error — the routine salt-rotation mechanism. `SessionConnection.cpp:389-397`:

```cpp
auth_data_->set_server_salt(bad_server_salt.new_server_salt_, Time::now_cached());
callback_->on_server_salt_updated();
on_message_failed(bad_info.message_id, Status::Error("Bad server salt"));
```

Three steps, all required:

1. Adopt `new_server_salt`.
2. Persist it.
3. **Resend** the message named by `bad_msg_id`, with a new `msg_id` and `seq_no`.

Expect this roughly every 10 minutes on a long-lived connection, and on the first message
after a restart with a stale stored salt.

---

## C.4 — API errors: `rpc_error`

```
rpc_error#2144ca19 error_code:int error_message:string
```
(`mtproto_api.tl:31`)

### Code families

| Code | Family | Retry? |
|------|--------|--------|
| 303 | `SEE_OTHER` — wrong datacenter | Yes, after migrating |
| 400 | `BAD_REQUEST` — your fault | No, not without changing the request |
| 401 | `UNAUTHORIZED` | Only after re-authorizing |
| 403 | `FORBIDDEN` | No |
| 406 | `NOT_ACCEPTABLE` | No |
| 420 | `FLOOD` | Yes, after the stated delay |
| 500 | `INTERNAL` | Yes, with backoff |

### 303 — Migration

Handled at `td/telegram/net/NetQueryDispatcher.cpp:391-418`.

| Message | Effect |
|---------|--------|
| `PHONE_MIGRATE_N` | Change **home** DC to N, resend |
| `NETWORK_MIGRATE_N` | Change **home** DC to N, resend |
| `USER_MIGRATE_N` | Change **home** DC to N, resend |
| `FILE_MIGRATE_N` | Resend **this query only** to N |

See [6.3](../06-api-layer/03-migration-and-multi-dc.md). Validate `N` before using it.

### 420 — Flood control

`NetQueryDelayer::delay` (`td/telegram/net/NetQueryDelayer.cpp:35-71`) recognizes **five**
prefixes, all followed by a number of seconds:

```cpp
for (auto prefix : {Slice("FLOOD_WAIT_"), Slice("SLOWMODE_WAIT_"), Slice("2FA_CONFIRM_WAIT_"),
                    Slice("TAKEOUT_INIT_DELAY_"), Slice("FLOOD_PREMIUM_WAIT_")}) {
  if (begins_with(error_message, prefix)) {
    if (error_message.substr(prefix.size()).find('_') != CSlice::npos) {
      // an unsupported error
      query->set_error(Status::Error(400, error_message));
      …
    }
    timeout = clamp(to_integer<int>(error_message.substr(prefix.size())), 1, 14 * 24 * 60 * 60);
    …
  }
}
```

| Prefix | Meaning |
|--------|---------|
| `FLOOD_WAIT_X` | General rate limit |
| `SLOWMODE_WAIT_X` | Group slow mode |
| `2FA_CONFIRM_WAIT_X` | Waiting period before a 2FA change takes effect |
| `TAKEOUT_INIT_DELAY_X` | Data export delay |
| `FLOOD_PREMIUM_WAIT_X` | Rate limit that would be shorter with Premium |

Three implementation details worth copying:

* **Clamp to 1 second … 14 days** (`14 * 24 * 60 * 60`). A malformed or hostile value must
  not put you to sleep forever.
* **If the suffix contains another `_`**, it is not a number — TDLib rewrites the error as
  a plain `400` rather than trying to parse it (`NetQueryDelayer.cpp:40-45`).
* `FLOOD_SKIP_FAILED_WAIT` gets a 1-second timeout (`:68-70`), as does
  `WORKER_BUSY_TOO_LONG_RETRY` on a 500 (`:32-34`) — retrying instantly is dangerous.

> **⚠ Honour the full delay.** Retrying early extends the wait and can get the `api_id`
> banned.

### 400 — Bad request

**Connection setup:**

| Message | Meaning |
|---------|---------|
| `CONNECTION_NOT_INITED` | `initConnection` missing on this connection |
| `CONNECTION_LAYER_INVALID` | `invokeWithLayer` missing or layer unusable |
| `CONNECTION_API_ID_INVALID` | Bad `api_id` in `initConnection` |
| `API_ID_INVALID` | Bad `api_id`/`api_hash` |
| `API_ID_PUBLISHED_FLOOD` | Publicly-known `api_id` |
| `LANG_PACK_INVALID` | Unregistered `lang_pack` — send an empty string |

The first two are recoverable: TDLib rewrites them as `500` so the query is retried with
the header (`td/telegram/net/Session.cpp:970-974`).

**Phone and code:**

| Message | Meaning |
|---------|---------|
| `PHONE_NUMBER_INVALID` | Malformed number |
| `PHONE_NUMBER_BANNED` | Number is banned |
| `PHONE_NUMBER_FLOOD` | Too many codes sent to this number |
| `PHONE_NUMBER_UNOCCUPIED` | No account — sign up |
| `PHONE_CODE_INVALID` | Wrong code; `phone_code_hash` still valid |
| `PHONE_CODE_EXPIRED` | `phone_code_hash` dead; call `sendCode` again |
| `PHONE_CODE_EMPTY` | Flag bit 0 not set, or empty string |
| `PHONE_PASSWORD_FLOOD` | Too many attempts (often reported as 406) |

**Password / SRP:**

| Message | Meaning |
|---------|---------|
| `PASSWORD_HASH_INVALID` | Wrong password, **or** a bug in your SRP |
| `SRP_ID_INVALID` | Stale `srp_id` — re-fetch `account.getPassword` |
| `SRP_PASSWORD_CHANGED` | Password changed mid-flow — re-fetch |
| `PASSWORD_REQUIRED` | 2FA needed but not supplied |

**Peers and messages:**

| Message | Meaning |
|---------|---------|
| `PEER_ID_INVALID` | Bad/stale `access_hash`, or wrong `InputPeer` kind |
| `USERNAME_NOT_OCCUPIED` | No such username |
| `USERNAME_INVALID` | Malformed username |
| `MESSAGE_EMPTY` | Empty message text |
| `MESSAGE_TOO_LONG` | Over `config.message_length_max` **UTF-16 code units** |
| `RANDOM_ID_DUPLICATE` | `random_id` reused with different content |
| `CHAT_WRITE_FORBIDDEN` | No permission to post |
| `USER_IS_BLOCKED` | Recipient blocked you |
| `YOU_BLOCKED_USER` | You blocked the recipient |
| `ENTITY_BOUND_TO_WRONG_MESSAGE` | Entity offsets out of range |
| `MSG_ID_INVALID` | Bad message id in `reply_to` |

### 401 — Unauthorized

| Message | Meaning |
|---------|---------|
| `SESSION_PASSWORD_NEEDED` | **Not a failure** — supply 2FA ([7.5](../07-login/05-two-factor-srp.md)) |
| `AUTH_KEY_UNREGISTERED` | Server does not know this key — redo the handshake |
| `AUTH_KEY_INVALID` | Key is invalid |
| `SESSION_EXPIRED` | Session no longer valid |
| `SESSION_REVOKED` | User revoked this session from another device |
| `USER_DEACTIVATED` | Account deactivated |
| `AUTH_KEY_PERM_EMPTY` | Temporary key used without binding ([9.1](../09-advanced/01-perfect-forward-secrecy.md)) |

> **⚠ Match on the message, not the code.** Treating every `401` as "start login over"
> turns `SESSION_PASSWORD_NEEDED` into an infinite loop. TDLib compares the string
> explicitly (`td/telegram/AuthManager.cpp:1577`).

### 403 — Forbidden

| Message | Meaning |
|---------|---------|
| `USER_DEACTIVATED_BAN` | Account banned |
| `CHAT_SEND_PLAIN_FORBIDDEN` | Text messages disabled in that chat |
| `CHAT_WRITE_FORBIDDEN` | No write permission |
| `RIGHT_FORBIDDEN` | Insufficient rights |

### 500 — Internal

| Message | Meaning |
|---------|---------|
| `AUTH_RESTART` | Restart the login flow |
| `RANDOM_ID_DUPLICATE` | Retry with a new `random_id` |
| `WORKER_BUSY_TOO_LONG_RETRY` | Retry after **1 second** (`NetQueryDelayer.cpp:32-34`) |

Retry 500s with exponential backoff. They are usually transient.

---

## C.5 — Local errors

Failures your own code should detect, with no server involvement:

| Condition | Where | Chapter |
|-----------|-------|---------|
| `p` is not exactly 2048 bits | DH validation | [3.5](../03-auth-key/05-security-checks.md) |
| `p` or `(p−1)/2` not prime | DH validation | [3.5](../03-auth-key/05-security-checks.md) |
| `g_a` outside `[2^1984, p − 2^1984]` | DH validation | [3.5](../03-auth-key/05-security-checks.md) |
| `nonce`/`server_nonce` mismatch | Handshake | [3.2](../03-auth-key/02-step1-req-pq.md) |
| SHA-1 mismatch on `server_DH_inner_data` | Handshake | [3.3](../03-auth-key/03-step2-req-dh-params.md) |
| `new_nonce_hash1` mismatch | Handshake | [3.4](../03-auth-key/04-step3-set-client-dh-params.md) |
| `msg_key` mismatch on decrypt | Envelope | [4.2](../04-encrypted-messages/02-msg-key-and-kdf.md) |
| `pad_size` outside `[12, 1024]` | Envelope | [4.1](../04-encrypted-messages/01-envelope.md) |
| `message_data_length % 4 != 0` | Envelope | [4.1](../04-encrypted-messages/01-envelope.md) |
| Wrong `session_id` on an inbound message | Session | [5.1](../05-session/01-msg-id-and-seq-no.md) |
| Inbound `msg_id` is even | Session | [5.1](../05-session/01-msg-id-and-seq-no.md) |
| Duplicate inbound `msg_id` | Session | [5.1](../05-session/01-msg-id-and-seq-no.md) |
| Packet larger than `MAX_PACKET_SIZE` | Transport | [9.2 §7](../09-advanced/02-connection-management.md) |
| `0 < B < p` violated | SRP | [7.5 §3](../07-login/05-two-factor-srp.md) |

> **⚠ Every one of these is a security check, not a sanity check.** Skipping the DH
> validation or the `msg_key` comparison makes you vulnerable to an active attacker. Fail
> closed.

---

## C.6 — Triage flowchart

```
Got something back
      │
      ├─ 4 bytes, < 16 total? ────────────► Transport error (C.1)
      │                                     0 → nop, -1 → quick ack
      │                                     else → reconnect
      │
      ├─ bad_msg_notification? ───────────► Protocol error (C.2)
      │                                     16/20 → resend
      │                                     else  → fix your code
      │
      ├─ bad_server_salt? ────────────────► Adopt salt, resend (C.3)
      │
      └─ rpc_error? ──────────────────────► API error (C.4)
                │
                ├─ 303 → migrate, retry
                ├─ 420 → sleep N (clamped 1s..14d), retry
                ├─ 500 → backoff, retry
                ├─ 401 SESSION_PASSWORD_NEEDED → SRP
                ├─ 401 other → re-authenticate
                └─ 400/403 → surface to caller, do NOT retry
```

---

## C.7 — Checklist

- [ ] Transport, protocol, and API errors handled as three distinct classes
- [ ] Frames under 16 bytes parsed as transport codes, never decrypted
- [ ] `0` (nop) and `-1` (quick ack) not treated as errors
- [ ] `-404` clears the stored key and restarts the handshake
- [ ] `bad_msg_notification` 16/20 resend; others logged loudly as client bugs
- [ ] Code 17 resets `server_time_difference` and closes the session
- [ ] `bad_server_salt` adopts, persists, **and** resends
- [ ] All five `420` prefixes recognized
- [ ] `420` delays clamped to 1 s … 14 days
- [ ] Non-numeric `420` suffixes rewritten as `400`, not parsed
- [ ] `401` matched on **message**, not just code
- [ ] `303` prefixes distinguished: `FILE_MIGRATE_` is per-query
- [ ] All local security checks implemented and failing closed

---

[← Previous](B-datacenters-and-keys.md) · [Index](../README.md) · [Next: Appendix D →](D-tdlib-source-map.md)
