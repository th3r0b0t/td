# 8.3 — `messages.sendMessage`

[← Previous](02-resolving-a-recipient.md) · [Index](../README.md) · [Next: Reading updates →](04-reading-updates.md)

---

The destination of this entire guide.

---

## 1. The call

```
messages.sendMessage#fef48f62 flags:# no_webpage:flags.1?true silent:flags.5?true
    background:flags.6?true clear_draft:flags.7?true noforwards:flags.14?true
    update_stickersets_order:flags.15?true invert_media:flags.16?true
    allow_paid_floodskip:flags.19?true peer:InputPeer reply_to:flags.0?InputReplyTo
    message:string random_id:long reply_markup:flags.2?ReplyMarkup
    entities:flags.3?Vector<MessageEntity> schedule_date:flags.10?int
    schedule_repeat_period:flags.24?int send_as:flags.13?InputPeer
    quick_reply_shortcut:flags.17?InputQuickReplyShortcut effect:flags.18?long
    allow_paid_stars:flags.21?long suggested_post:flags.22?SuggestedPost
    rich_message:flags.23?InputRichMessage = Updates;
```
(`td/generate/scheme/telegram_api.tl:2521`)

Long, but only **three** fields are mandatory:

| Field | Type | Value |
|-------|------|-------|
| `peer` | `InputPeer` | Where to send — [8.1](01-peers-and-access-hashes.md) |
| `message` | `string` | The text, UTF-8 |
| `random_id` | `long` | A unique 64-bit value — see §3 |

Everything else is flag-gated. **Send `flags = 0`** and skip them all.

---

## 2. The minimal call

To yourself, with text `Hello`:

```
62 8f f4 fe                  messages.sendMessage#fef48f62
00 00 00 00                  flags = 0
c9 7e a0 7d                  inputPeerSelf#7da07ec9
05 48 65 6c 6c 6f 00 00      "Hello"  (1 len + 5 data + 2 pad = 8)
?? ?? ?? ?? ?? ?? ?? ??      random_id (8 random bytes)
```

28 bytes. That is the whole thing.

Note the field **order on the wire**: `flags`, then `peer`, then `message`, then
`random_id`. TL serializes fields in declaration order, and `reply_to` (flag bit 0) would
sit between `peer` and `message` if it were present. Get the order wrong and the server
sees garbage.

---

## 3. `random_id` — the deduplication key

**This is not optional and it is not cosmetic.**

The server remembers `random_id` values for a period and silently discards a second
`sendMessage` with the same one. That is what stops a resend
([5.5](../05-session/05-reliability.md)) from posting your message twice.

Rules:

1. Generate it from a **cryptographically secure** random source. 64 bits, non-zero.
2. Generate it **once**, when the user hits send.
3. Keep it **identical** across every retry of that message — including retries after a
   reconnect, after `bad_server_salt`, and after `FLOOD_WAIT`.
4. Never reuse it for a different message.

> **⚠ The classic bug.** Regenerating `random_id` inside the retry loop. The message posts
> once per retry, and the user sees duplicates. Store `random_id` with the pending query,
> not in the send function.

You also get it back: `updateMessageID#4e90bfd6 id:int random_id:long`
(`telegram_api.tl:348`) maps your `random_id` to the server-assigned message id.

---

## 4. Flags worth knowing

| Bit | Field | Effect |
|-----|-------|--------|
| 0 | `reply_to` | Reply to a specific message |
| 1 | `no_webpage` | Suppress link preview |
| 2 | `reply_markup` | Inline keyboard (bots) |
| 3 | `entities` | Formatting — bold, links, code |
| 5 | `silent` | No notification sound |
| 7 | `clear_draft` | Clear the saved draft for this chat |
| 10 | `schedule_date` | Send later (unix time) |

For plain text, all zero.

### Formatting

Telegram does **not** parse Markdown or HTML on the wire. `message` is plain text; styling
is carried separately in `entities:flags.3?Vector<MessageEntity>` as
(offset, length, style) triples.

> **⚠ Entity offsets are in UTF-16 code units**, not bytes and not Unicode code points.
> The text itself is UTF-8. This mismatch is a notorious source of off-by-N bugs with
> emoji and non-Latin scripts. Avoid entities entirely in a first implementation.

---

## 5. The reply — `Updates`

`messages.sendMessage` returns `Updates`, not a message object. Six constructors
(`telegram_api.tl:520-526`):

```
updatesTooLong#e317af7e = Updates;
updateShortMessage#313bc7f8 … = Updates;
updateShortChatMessage#4d6deea5 … = Updates;
updateShort#78d4dec1 update:Update date:int = Updates;
updatesCombined#725b04c3 updates:Vector<Update> users:Vector<User> chats:Vector<Chat> date:int seq_start:int seq:int = Updates;
updates#74ae4240 updates:Vector<Update> users:Vector<User> chats:Vector<Chat> date:int seq:int = Updates;
```

For a successful `sendMessage` you will usually get one of:

**`updateShortSentMessage#9015e101`** (`telegram_api.tl:526`) — the compact confirmation:

```
updateShortSentMessage#9015e101 flags:# out:flags.1?true id:int pts:int pts_count:int
    date:int media:flags.9?MessageMedia entities:flags.7?Vector<MessageEntity>
    ttl_period:flags.25?int = Updates;
```

`id` is the new message's id. Note it does **not** echo `random_id` — the correlation is
that this is the answer to *your* `rpc_result`.

**`updates#74ae4240`** — a full update set, typically containing
`updateMessageID#4e90bfd6` (mapping `random_id` → `id`) and `updateNewMessage#1f2b0afd`
(the complete `message#7600b9d3` object), plus the `users`/`chats` vectors you should
absorb into your cache.

**If the reply parses as any `Updates` variant, the message was sent.** For a
"send one message and exit" program you can stop there. See
[8.4](04-reading-updates.md) for doing more.

---

## 6. Errors

| Error | Meaning |
|-------|---------|
| `400 PEER_ID_INVALID` | Bad or stale access hash — [8.1](01-peers-and-access-hashes.md) |
| `400 MESSAGE_EMPTY` | Empty `message` |
| `400 MESSAGE_TOO_LONG` | Over the limit (`config.message_length_max`, typically 4096) |
| `400 RANDOM_ID_DUPLICATE` | `random_id` reused for different content |
| `400 CHAT_WRITE_FORBIDDEN` | No permission to post |
| `400 USER_IS_BLOCKED` | The recipient blocked you |
| `400 YOU_BLOCKED_USER` | You blocked the recipient |
| `403 CHAT_SEND_PLAIN_FORBIDDEN` | Text messages disabled in that chat |
| `420 FLOOD_WAIT_X` | Rate limited |
| `420 SLOWMODE_WAIT_X` | Group slow mode |

`MESSAGE_TOO_LONG` is measured in **UTF-16 code units**, matching the entity convention. A
4096-character limit is not 4096 bytes.

---

## 7. The complete sequence

Everything in this guide, end to end:

```
 1. Resolve DC address                              → 2.1
 2. TCP connect                                     → 2.2
 3. Send framing magic / obfuscation header         → 2.2, 2.3
 4. req_pq_multi → req_DH_params → set_client_DH_params
                                                    → 3.2, 3.3, 3.4
 5. auth_key, auth_key_id, server_salt in hand      → 3.4
 6. invokeWithLayer(229, initConnection(…, help.getConfig()))
                                                    → 6.1
 7. auth.sendCode(phone, api_id, api_hash, settings)
                                                    → 7.3
     └─ 303 PHONE_MIGRATE_N → back to step 1        → 6.3
 8. auth.signIn(1, phone, phone_code_hash, code)    → 7.4
     └─ 401 SESSION_PASSWORD_NEEDED
          → account.getPassword → SRP → auth.checkPassword
                                                    → 7.5
 9. Persist auth_key + dc_id                        → 7.6
10. messages.sendMessage(0, inputPeerSelf, "Hello", random_id)
                                                    → this chapter
11. Reply parses as Updates                         → done ✅
```

---

## 8. Checklist

- [ ] `flags = 0` for a plain text message
- [ ] Fields in declaration order: `flags`, `peer`, `message`, `random_id`
- [ ] `random_id` from a secure RNG, non-zero
- [ ] `random_id` generated **once** and reused across every retry
- [ ] `random_id` stored with the pending query, not regenerated in the send path
- [ ] `message` is UTF-8, non-empty, within `config.message_length_max` UTF-16 units
- [ ] No `entities` unless UTF-16 offsets are computed correctly
- [ ] All six `Updates` constructors at least recognized
- [ ] `users`/`chats` from the reply absorbed into the access-hash cache
- [ ] `PEER_ID_INVALID` investigated as an access-hash problem
- [ ] `FLOOD_WAIT_X` and `SLOWMODE_WAIT_X` honoured

---

[← Previous](02-resolving-a-recipient.md) · [Index](../README.md) · [Next: Reading updates →](04-reading-updates.md)
