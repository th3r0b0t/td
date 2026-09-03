# 8.1 — Peers and Access Hashes

[← Previous](../07-login/06-after-login.md) · [Index](../README.md) · [Next: Resolving a recipient →](02-resolving-a-recipient.md)

---

To send a message you need a recipient. In Telegram a recipient is an `InputPeer`, and
constructing one correctly is the step most people trip over.

---

## 1. The three kinds of peer

| Kind | What it is |
|------|------------|
| **User** | A person or bot — a one-to-one chat |
| **Chat** | A basic group (legacy, small, no username) |
| **Channel** | A supergroup or broadcast channel |

Telegram uses three parallel families of constructors for these, and they are easy to
confuse:

| Family | Purpose | Example |
|--------|---------|---------|
| `Peer` | **Identifies** a peer in server output | `peerUser#59511722 user_id:long` |
| `InputPeer` | **Addresses** a peer in your requests | `inputPeerUser#dde8a54c user_id:long access_hash:long` |
| `InputUser` | Addresses a *user* specifically | `inputUser#f21158c6 user_id:long access_hash:long` |

(`td/generate/scheme/telegram_api.tl:99-101, 38-44, 46-49`)

Note the asymmetry: `Peer` carries **only an id**, `InputPeer` carries **id and access
hash**. That is the whole difficulty.

---

## 2. The `InputPeer` constructors

```
inputPeerEmpty#7f3b18ea = InputPeer;
inputPeerSelf#7da07ec9 = InputPeer;
inputPeerChat#35a95cb9 chat_id:long = InputPeer;
inputPeerUser#dde8a54c user_id:long access_hash:long = InputPeer;
inputPeerChannel#27bcbbfc channel_id:long access_hash:long = InputPeer;
inputPeerUserFromMessage#a87b0a1c peer:InputPeer msg_id:int user_id:long = InputPeer;
inputPeerChannelFromMessage#bd2a0840 peer:InputPeer msg_id:int channel_id:long = InputPeer;
```
(`telegram_api.tl:38-44`)

Two are free — they need no access hash at all:

* **`inputPeerSelf#7da07ec9`** — you. Four bytes, no fields. **Use this for your first
  test.** Sending a message to yourself ("Saved Messages") requires no lookup, no contact,
  and no access hash.
* **`inputPeerChat#35a95cb9`** — a basic group, identified by `chat_id` alone. Basic groups
  have no access hash.

The rest need one.

---

## 3. What an access hash is

An `access_hash` is a 64-bit token issued **by the server, to you** that proves you are
entitled to interact with that peer. It is:

* **Per-account** — your access hash for a user is different from mine.
* **Per-datacenter** in effect — do not carry one across DCs.
* **Not derivable** — you cannot compute it from the user id.
* **Obtainable only from the server**, in the `User` or `Channel` object of some reply.

The design intent is to stop id enumeration: knowing that user ids are sequential gains you
nothing, because you cannot address a user without a hash you were given.

> **⚠ `PEER_ID_INVALID`.** Nine times out of ten this means a missing, wrong, or stale
> access hash — not a wrong id.

---

## 4. Where access hashes come from

Any reply that carries `User` or `Channel` objects. In practice:

| Source | Gives you |
|--------|-----------|
| `auth.authorization.user` | Your own |
| `contacts.resolveUsername` | The peer behind a username |
| `contacts.getContacts` | All your contacts |
| `messages.getDialogs` | Everyone in your chat list |
| `users.getUsers` | Specific users you already have hashes for |
| Any `Updates` with a `users:Vector<User>` field | Everyone mentioned |

The general pattern: **replies come bundled with the `User` and `Chat` objects they refer
to.** A `messages.Messages` reply, for instance, has `users` and `chats` vectors alongside
the messages, precisely so you can resolve every `peerUser` id you see.

> **💡 Implementation note.** Maintain a `HashMap<user_id, access_hash>` (and one for
> channels) and populate it from **every** reply that contains `User` or `Chat` objects.
> This single habit eliminates most `PEER_ID_INVALID` errors.

---

## 5. The `min` flag

`user#b1b8cc83` has a flag `min:flags.20?true` (`telegram_api.tl:115`). A "min" user object
is a partial one, and its `access_hash` — if present at all — is **not usable for general
API calls**. You may only use it in the context of the message it arrived with, via
`inputPeerUserFromMessage`.

```
inputPeerUserFromMessage#a87b0a1c peer:InputPeer msg_id:int user_id:long = InputPeer;
```

This says "the user with this id, as seen in message `msg_id` of chat `peer`" — the server
looks the hash up on your behalf.

**Rule:** never overwrite a full (non-`min`) cached access hash with one from a `min`
object. Doing so silently corrupts your cache and produces intermittent
`PEER_ID_INVALID`s that are very hard to debug.

---

## 6. `InputUser` vs `InputPeer`

```
inputUserEmpty#b98886cf = InputUser;
inputUserSelf#f7c1b13f = InputUser;
inputUser#f21158c6 user_id:long access_hash:long = InputUser;
inputUserFromMessage#1da448e2 peer:InputPeer msg_id:int user_id:long = InputUser;
```
(`telegram_api.tl:46-49`)

Structurally identical to the user variants of `InputPeer`, but a **different boxed type**
with **different constructor ids**. `users.getUsers` takes `Vector<InputUser>`;
`messages.sendMessage` takes `InputPeer`. Substituting one for the other produces a parse
error on the server.

Check the schema line for the call you are making, every time.

---

## 7. Id ranges and the "bare" id

You may see peer ids written as large negative numbers in other clients — that is a
*client-side* encoding used to squeeze the three peer kinds into one integer space
(TDLib does this in `td/telegram/DialogId.h`). **The wire protocol does not use it.**

On the wire:

* `user_id`, `chat_id`, `channel_id` are all plain positive `long`s.
* The *kind* is carried by the constructor, not by the sign or magnitude of the id.

Do not import another client's marker scheme into your protocol code.

---

## 8. Recommended approach for a first client

```
1. Send to yourself:      inputPeerSelf#7da07ec9        (no lookup at all)
2. Send to a username:    contacts.resolveUsername → cache → inputPeerUser
3. Send to a group:       messages.getDialogs → cache → inputPeerChat / inputPeerChannel
```

Start at 1. It removes every variable except the message-sending call itself.

---

## 9. Checklist

- [ ] `Peer` (output) and `InputPeer` (input) not confused
- [ ] `InputPeer` and `InputUser` not confused — different ids for identical shapes
- [ ] `inputPeerSelf#7da07ec9` used for the first end-to-end test
- [ ] An access-hash cache populated from every reply carrying `User`/`Chat` objects
- [ ] `min` users never overwrite full cached hashes
- [ ] `inputPeerUserFromMessage` used when only a `min` object is available
- [ ] Ids kept as plain positive `long`s on the wire
- [ ] `PEER_ID_INVALID` treated as "bad access hash" first

---

[← Previous](../07-login/06-after-login.md) · [Index](../README.md) · [Next: Resolving a recipient →](02-resolving-a-recipient.md)
