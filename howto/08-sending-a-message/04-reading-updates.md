# 8.4 — Reading Updates

[← Previous](03-send-message.md) · [Index](../README.md) · [Next: Perfect forward secrecy →](../09-advanced/01-perfect-forward-secrecy.md)

---

Updates are how the server pushes news to you: new messages, edits, typing indicators, read
receipts. If your goal is "send one message and exit", **you can skip this chapter
entirely**. If you want a client that stays connected, read on.

---

## 1. Updates arrive unsolicited

An update is a message with `req_msg_id = 0` — that is, not an `rpc_result` at all. It just
appears in your decrypted stream ([6.2 §4](../06-api-layer/02-rpc-results-and-errors.md)).

So your dispatcher needs a default branch: any constructor id that is not a service message
and not `rpc_result` is an update.

Two properties to internalize:

* Updates arrive at any time, including between a query and its answer, and inside
  containers.
* Updates may be **duplicated** and may arrive **out of order**.

---

## 2. The `Updates` container types

```
updatesTooLong#e317af7e = Updates;
updateShortMessage#313bc7f8 flags:# … id:int user_id:long message:string pts:int pts_count:int date:int … = Updates;
updateShortChatMessage#4d6deea5 flags:# … id:int from_id:long chat_id:long message:string pts:int pts_count:int date:int … = Updates;
updateShort#78d4dec1 update:Update date:int = Updates;
updatesCombined#725b04c3 updates:Vector<Update> users:Vector<User> chats:Vector<Chat> date:int seq_start:int seq:int = Updates;
updates#74ae4240 updates:Vector<Update> users:Vector<User> chats:Vector<Chat> date:int seq:int = Updates;
updateShortSentMessage#9015e101 flags:# out:flags.1?true id:int pts:int pts_count:int date:int … = Updates;
```
(`td/generate/scheme/telegram_api.tl:520-526`)

| Constructor | Contents |
|-------------|----------|
| `updateShort` | Exactly one `Update`, no peer objects |
| `updates` | A vector of `Update`s **plus** `users` and `chats` |
| `updatesCombined` | Same, spanning a `seq` range |
| `updateShortMessage` | A compact new private message — **no** `users` vector |
| `updateShortChatMessage` | A compact new basic-group message |
| `updateShortSentMessage` | Confirmation of your own send |
| `updatesTooLong` | "Too much happened; call `updates.getDifference`" |

> **⚠ The `short` forms omit the peer objects.** `updateShortMessage` gives you a
> `user_id` with no `User` object and therefore no access hash. If you do not already have
> that user cached, you cannot reply without a lookup. This is the whole reason the
> `min`/cache discipline of [8.1](01-peers-and-access-hashes.md) matters.

---

## 3. The `Update` types

There are hundreds (`telegram_api.tl:347` onwards). The ones a minimal client cares about:

| Update | Constructor | Meaning |
|--------|-------------|---------|
| `updateNewMessage` | `0x1f2b0afd` | New message in a private chat or basic group |
| `updateNewChannelMessage` | `0x62ba04d9` | New message in a channel or supergroup |
| `updateMessageID` | `0x4e90bfd6` | Maps your `random_id` to the server's message id |

**Ignore everything else.** New update types ship with every layer; an unknown constructor
id must never be fatal.

---

## 4. `pts`, `qts`, `date`, `seq`

Four counters that let you detect gaps.

| Counter | Scope | Carried by |
|---------|-------|-----------|
| `pts` | Messages in private chats and basic groups | Most message updates |
| `pts` (per-channel) | Messages in one channel | Channel updates |
| `qts` | Secret chats | Encrypted updates |
| `seq` | The update stream itself | `updates`, `updatesCombined` |
| `date` | Server time | All container types |

The rule for `pts`:

```
if update.pts == local_pts + update.pts_count:
    apply; local_pts = update.pts
elif update.pts <= local_pts:
    ignore   # already applied
else:
    gap — call updates.getDifference
```

`updatesCombined` carries `seq_start` and `seq`, defining a range; `updates` carries a
single `seq`. A `seq` of 0 means "not sequenced, apply immediately".

> **💡 Implementation note.** Getting this fully right is genuinely hard — it is a
> significant fraction of TDLib's complexity. For a simple client, **do not track state at
> all**: apply every update you receive, tolerate duplicates, and accept that you may miss
> some. That is a perfectly reasonable design for a send-only or best-effort tool.

---

## 5. Catching up

```
updates.getState#edd4882a = updates.State;
updates.state#a56c2a3e pts:int qts:int date:int seq:int unread_count:int = updates.State;

updates.getDifference#19c2f763 flags:# pts:int pts_limit:flags.1?int
    pts_total_limit:flags.0?int date:int qts:int qts_limit:flags.2?int = updates.Difference;

updates.differenceEmpty#5d75a138 date:int seq:int = updates.Difference;
updates.difference#00f49ca0 new_messages:Vector<Message> new_encrypted_messages:Vector<EncryptedMessage>
    other_updates:Vector<Update> chats:Vector<Chat> users:Vector<User> state:updates.State = updates.Difference;
updates.differenceSlice#a8fb1981 … intermediate_state:updates.State = updates.Difference;
updates.differenceTooLong#4afe8f6d pts:int = updates.Difference;
```
(`telegram_api.tl:2772, 513, 2773, 515-518`)

The catch-up loop:

1. On first login, call `updates.getState` and store `pts`, `qts`, `date`, `seq`.
2. On a detected gap or `updatesTooLong`, call `updates.getDifference(pts, date, qts)`.
3. `updates.differenceEmpty` → you are current.
4. `updates.difference` → apply, adopt `state`, done.
5. `updates.differenceSlice` → apply, adopt `intermediate_state`, **call again**.
6. `updates.differenceTooLong` → you are too far behind; reset `pts` and reload from
   scratch.

Note `updates.difference#00f49ca0` — the schema writes it as `#f49ca0`, missing the leading
zero byte. Constructor ids are always 32 bits
([1.1](../01-serialization/01-tl-language.md)).

---

## 6. Minimal update handling

For a client that sends a message and wants confirmation:

```
on_update(body):
    id = read_u32_le(body)
    match id:
        0x9015e101  updateShortSentMessage -> message sent, id = <field>
        0x74ae4240  updates                 -> absorb users/chats
                                              scan for updateMessageID
        0x725b04c3  updatesCombined         -> same
        0x78d4dec1  updateShort             -> inspect the single update
        _                                    -> log and ignore
```

Twenty lines. No `pts`, no `getDifference`, no state.

---

## 7. Keeping the connection open

Updates only arrive while you have a connection. To receive them:

* Keep the TCP connection open and keep reading.
* Send `ping_delay_disconnect` periodically ([5.5 §7](../05-session/05-reliability.md)) so
  the server knows you are alive.
* Reconnect on failure and, if you track state, call `updates.getDifference` afterwards.

There is no polling mode over TCP — updates are pushed.

---

## 8. Checklist

- [ ] Unknown constructor ids logged and ignored, never fatal
- [ ] All seven `Updates` container types recognized
- [ ] `users`/`chats` from `updates`/`updatesCombined` absorbed into the cache
- [ ] Short forms understood to omit peer objects
- [ ] Duplicate and out-of-order updates tolerated
- [ ] If tracking state: `pts`/`qts`/`date`/`seq` stored and gaps trigger `getDifference`
- [ ] If tracking state: `differenceSlice` loops until `difference` or `differenceEmpty`
- [ ] `updatesTooLong` triggers a catch-up
- [ ] Connection kept open with periodic pings for a long-running client

---

[← Previous](03-send-message.md) · [Index](../README.md) · [Next: Perfect forward secrecy →](../09-advanced/01-perfect-forward-secrecy.md)
