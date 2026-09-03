# 8.2 — Resolving a Recipient

[← Previous](01-peers-and-access-hashes.md) · [Index](../README.md) · [Next: messages.sendMessage →](03-send-message.md)

---

Three ways to turn "who I want to message" into an `InputPeer`, in increasing order of
effort.

---

## 1. Yourself — no lookup

```
inputPeerSelf#7da07ec9 = InputPeer;
```
(`td/generate/scheme/telegram_api.tl:39`)

Four bytes: `c9 7e a0 7d`. No fields, no lookup, no access hash.

Messages sent here appear in **Saved Messages**. This is the correct target for your first
end-to-end test — it exercises the entire stack with zero dependence on peer resolution.

---

## 2. By username — `contacts.resolveUsername`

```
contacts.resolveUsername#725afbbc flags:# username:string referer:flags.0?string
    = contacts.ResolvedPeer;
contacts.resolvedPeer#7f077ad9 peer:Peer chats:Vector<Chat> users:Vector<User>
    = contacts.ResolvedPeer;
```
(`telegram_api.tl:2493, 778`)

Send `flags = 0` and the username **without** the leading `@`.

The reply gives you:

* `peer` — a `Peer`, telling you the **kind** and **id**.
* `users` and `chats` — the full objects, carrying the **access hashes**.

So the procedure is:

```
1. call contacts.resolveUsername("durov")
2. look at resolved.peer:
     peerUser{user_id}     → find that id in resolved.users    → take access_hash
     peerChannel{channel_id} → find that id in resolved.chats  → take access_hash
     peerChat{chat_id}     → no access hash needed
3. build inputPeerUser{user_id, access_hash}
        or inputPeerChannel{channel_id, access_hash}
        or inputPeerChat{chat_id}
```

> **⚠ The access hash is not in `peer`.** `peerUser#59511722` has a single field,
> `user_id:long`. You *must* cross-reference the `users` vector. This catches everyone once.

Errors:

| Error | Meaning |
|-------|---------|
| `400 USERNAME_NOT_OCCUPIED` | No such username |
| `400 USERNAME_INVALID` | Malformed username |
| `420 FLOOD_WAIT_X` | Too many lookups |

Username lookups are rate-limited more tightly than most calls. Cache the result.

---

## 3. By phone number — contacts

```
contacts.getContacts#5dd69e12 hash:long = contacts.Contacts;
contacts.contacts#eae87e42 contacts:Vector<Contact> saved_count:int users:Vector<User>
    = contacts.Contacts;
```
(`telegram_api.tl:2485, 305`)

Pass `hash = 0` to get the full list; the `users` vector carries the access hashes.

The `hash` parameter is Telegram's list-diffing mechanism: send back a hash computed over
the ids you already have, and if nothing changed the server replies with a "not modified"
constructor instead of the whole list. `hash = 0` always returns everything, which is fine
for a simple client.

To message someone **not** in your contacts and without a username, you would need
`contacts.importContacts` — beyond the scope of a first client, and heavily rate-limited.

---

## 4. From your chat list — `messages.getDialogs`

```
messages.getDialogs#a0f4cb4f flags:# exclude_pinned:flags.0?true folder_id:flags.1?int
    offset_date:int offset_id:int offset_peer:InputPeer limit:int hash:long
    = messages.Dialogs;
messages.dialogs#15ba6c40 dialogs:Vector<Dialog> messages:Vector<Message>
    chats:Vector<Chat> users:Vector<User> = messages.Dialogs;
messages.dialogsSlice#71e094f3 count:int dialogs:Vector<Dialog> messages:Vector<Message>
    chats:Vector<Chat> users:Vector<User> = messages.Dialogs;
```
(`telegram_api.tl:2513, 312, 313`)

For the first page:

```
flags       = 0
offset_date = 0
offset_id   = 0
offset_peer = inputPeerEmpty#7f3b18ea
limit       = 100
hash        = 0
```

`dialog#fc89f7f3` (`telegram_api.tl:243`) carries `peer:Peer` — again, ids only. The
`users` and `chats` vectors alongside carry the hashes. Same cross-referencing as §2.

Two reply constructors: `messages.dialogs` (complete) and `messages.dialogsSlice`
(paginated, with a total `count`). Handle both.

This is the most useful call for populating your access-hash cache in one shot: one request
gives you hashes for everyone you have ever talked to.

---

## 5. Building the cache

```
fn absorb(users: Vec<User>, chats: Vec<Chat>):
    for u in users:
        if u.is_min:                       // flags bit 20
            continue                       // do NOT cache min hashes
        if u.access_hash is present:       // flags bit 0
            user_hashes[u.id] = u.access_hash
    for c in chats:
        if c is channel and not c.is_min and c.access_hash is present:
            channel_hashes[c.id] = c.access_hash
```

Call `absorb` on the `users`/`chats` vectors of **every** reply that has them:
`contacts.resolvedPeer`, `messages.dialogs`, `messages.messages`, `updates`, and so on.

---

## 6. Comparison

| Method | Round trips | Rate limit | Needs |
|--------|-------------|------------|-------|
| `inputPeerSelf` | 0 | — | Nothing |
| `contacts.resolveUsername` | 1 | Tight | A public username |
| `contacts.getContacts` | 1 | Moderate | The person in your contacts |
| `messages.getDialogs` | 1 | Loose | An existing conversation |

For a "send one message" program: use `inputPeerSelf` to prove the pipeline works, then
`contacts.resolveUsername` for a real recipient.

---

## 7. Checklist

- [ ] `inputPeerSelf` used for the first test
- [ ] `contacts.resolveUsername` sent with `flags = 0` and no leading `@`
- [ ] Access hash taken from the `users`/`chats` vectors, **not** from `peer`
- [ ] All three `Peer` kinds handled when interpreting `resolvedPeer.peer`
- [ ] `messages.dialogs` **and** `messages.dialogsSlice` both handled
- [ ] `inputPeerEmpty` used as `offset_peer` for the first dialogs page
- [ ] Every reply's `users`/`chats` absorbed into the cache
- [ ] `min` objects excluded from the cache
- [ ] Username resolutions cached; `FLOOD_WAIT_X` honoured

---

[← Previous](01-peers-and-access-hashes.md) · [Index](../README.md) · [Next: messages.sendMessage →](03-send-message.md)
