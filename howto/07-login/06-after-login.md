# 7.6 — After Login: Persisting and Protecting the Authorization

[← Previous](05-two-factor-srp.md) · [Index](../README.md) · [Next: Peers and access hashes →](../08-sending-a-message/01-peers-and-access-hashes.md)

---

You have an `auth.authorization`. This chapter is about what changes at that moment, and
what you must now be careful with.

---

## 1. What just happened

Your `auth_key` was, until now, an anonymous session key. The server has now **bound it to
an account**. Nothing about the key itself changed — the binding lives on the server.

Practical consequences:

* The same `auth_key` now authorizes API calls as that user.
* It stays valid indefinitely — there is no expiry.
* It survives disconnection, restart, and IP change.
* Logging in again from scratch is unnecessary and creates a second session.
* The user can see it in **Settings → Devices** and revoke it.

**The `auth_key` is now equivalent to the account password in power.** Anyone who obtains
it has full access without needing the phone number, the code, or the 2FA password.

---

## 2. What to persist

| Item | Size | Required? | Why |
|------|------|-----------|-----|
| `auth_key` | 256 bytes | **Yes** | Without it you must log in again |
| `auth_key_id` | 8 bytes | Derived | `SHA1(auth_key)[12..20)` — recompute or store |
| Home `dc_id` | 4 bytes | **Yes** | Which DC the key belongs to |
| DC address/port | — | Recommended | Avoid a `help.getConfig` round trip |
| `server_salt` | 8 bytes | Recommended | Avoids one `bad_server_salt` round trip |
| `server_time_difference` | 8 bytes | Recommended | Avoids one `bad_msg_notification` round trip |
| Own `user_id` | 8 bytes | Recommended | Identify your own messages |
| `session_id` | 8 bytes | Optional | A new one is fine; costs a `new_session_created` |
| `seq_no` | 4 bytes | Only with `session_id` | Must match the session |

The minimum viable persisted state is `auth_key` + `dc_id`. Everything else is an
optimization or a convenience.

> **⚠ Never persist:** the password, the login code, `phone_code_hash`, `srp_id`, `srp_B`,
> or any SRP intermediate. They are spent, and storing them is pure risk.

---

## 3. How to protect it

> **⚠ Security note.** The `auth_key` file is the crown jewel. Minimum precautions:
>
> * File permissions `0600` — owner read/write only.
> * Not in a directory that gets backed up to a third party unencrypted.
> * Not in the repository, not in `/tmp`, not world-readable.
> * Encrypted at rest if the platform provides a keyring or secure enclave. TDLib supports
>   an encryption key for its database for exactly this reason.
> * Zero the buffer in memory when done, if your language lets you.
> * **Never logged**, not even truncated. A partial `auth_key` in a log file is still a
>   serious leak.

A reasonable minimum on Unix:

```
open(path, O_WRONLY | O_CREAT | O_TRUNC, 0600)
```

and check the mode on read; refuse to load a key file that is group- or world-readable.

---

## 4. Reusing the key on restart

```
1. Load auth_key, dc_id, salt, time_difference.
2. Connect to dc_id.
3. Send the transport magic / obfuscation header.       ← new every connection
4. Send invokeWithLayer(layer, initConnection(…, help.getConfig()))
5. Carry on.
```

**No handshake.** Skipping straight to encrypted messages with a stored key is the entire
point of persisting it.

If the key has been revoked you will get:

```
401 AUTH_KEY_UNREGISTERED
```

or the connection will be closed with `-404` ([2.4](../02-transport/04-transport-errors.md)).
Both mean: discard the stored key, run a fresh handshake, and log in again.

---

## 5. Verifying you are logged in

The cheapest check:

```
users.getUsers#0d91a548 id:Vector<InputUser> = Vector<User>;
```

with `inputUserSelf#f7c1b13f`. It returns your own `User`, or `401` if the key is not
authorized. See [8.1](../08-sending-a-message/01-peers-and-access-hashes.md).

---

## 6. Logging out

```
auth.logOut#3e72ba19 = auth.LoggedOut;
auth.loggedOut#c3a2835f flags:# future_auth_token:flags.0?bytes = auth.LoggedOut;
```
(`td/generate/scheme/telegram_api.tl:2319, 1541`)

This **revokes the `auth_key` on the server**. After it returns, delete your stored key —
it is now useless.

`future_auth_token`, if present, can speed up a subsequent login. Storing it is optional
and it is much less sensitive than an `auth_key`, but it is still a credential.

> **💡 Implementation note.** Simply deleting your local key file without calling
> `auth.logOut` leaves the session alive on the server, visible in the user's device list
> forever. Call `auth.logOut` when the user asks to log out; delete the file only for
> "forget this device locally".

---

## 7. One session, not many

Every completed handshake-plus-login creates a **new** entry in the user's device list. A
client that logs in on every start will fill that list with clutter and may trip
anti-abuse limits.

Persist, reuse, and only re-authenticate when the key is actually rejected.

---

## 8. Checklist

- [ ] `auth_key` and `dc_id` persisted immediately on `auth.authorization`
- [ ] Storage is `0600`, outside the repository, never logged
- [ ] `server_salt`, `server_time_difference`, and own `user_id` persisted too
- [ ] Password, code, `phone_code_hash`, and SRP values **not** persisted
- [ ] Startup path loads the key and skips the handshake
- [ ] `AUTH_KEY_UNREGISTERED` / `-404` clears the stored key and restarts login
- [ ] `auth.logOut` used for real logout; local deletion only for "forget device"
- [ ] The client does not re-authenticate on every start

---

[← Previous](05-two-factor-srp.md) · [Index](../README.md) · [Next: Peers and access hashes →](../08-sending-a-message/01-peers-and-access-hashes.md)
