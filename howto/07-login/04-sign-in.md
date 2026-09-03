# 7.4 — `auth.signIn`

[← Previous](03-send-code.md) · [Index](../README.md) · [Next: Two-factor SRP →](05-two-factor-srp.md)

---

Submit the code the user received. If the account has no two-step verification, this is the
last step of login.

---

## 1. The call

```
auth.signIn#8d52a951 flags:# phone_number:string phone_code_hash:string
    phone_code:flags.0?string email_verification:flags.1?EmailVerification
    = auth.Authorization;
```
(`td/generate/scheme/telegram_api.tl:2318`)

| Field | Flag | Notes |
|-------|------|-------|
| `phone_number` | — | Exactly what you sent to `auth.sendCode` |
| `phone_code_hash` | — | From the `auth.sentCode` reply |
| `phone_code` | bit 0 | The code the user typed |
| `email_verification` | bit 1 | Only for email-based login |

For an ordinary code login, `flags = 1`.

`AuthManager::send_auth_sign_in_query` (`td/telegram/AuthManager.cpp:805-813`):

```cpp
bool is_email = !email_code_.is_empty();
int32 flags = is_email ? telegram_api::auth_signIn::EMAIL_VERIFICATION_MASK
                       : telegram_api::auth_signIn::PHONE_CODE_MASK;
start_net_query(NetQueryType::SignIn,
                G()->net_query_creator().create_unauth(telegram_api::auth_signIn(
                    flags, send_code_helper_.get_phone_number(), send_code_helper_.get_phone_code_hash(), code_,
                    is_email ? email_code_.get_input_email_verification() : nullptr)));
```

Note the two are **mutually exclusive** — exactly one of bits 0 and 1 is set.

> **⚠ `phone_number` must match byte-for-byte.** If you normalized the number before
> `sendCode` (say, stripping a `+`), send the same normalized form here. A mismatch gives
> `400 PHONE_CODE_INVALID`, which is a confusing way to be told the number is wrong.

---

## 2. The replies

```
auth.authorization#2ea2c0d4 flags:# setup_password_required:flags.1?true
    otherwise_relogin_days:flags.1?int tmp_sessions:flags.0?int
    future_auth_token:flags.2?bytes user:User = auth.Authorization;
auth.authorizationSignUpRequired#44747e9a flags:# terms_of_service:flags.0?help.TermsOfService
    = auth.Authorization;
```
(`telegram_api.tl:264, 265`)

### `auth.authorization` — success

**You are logged in.** The `auth_key` you have been using is now bound to this account, on
this DC, permanently — until the user logs out or revokes the session.

| Field | Meaning |
|-------|---------|
| `user` | Your own `User` object |
| `setup_password_required` | The account has no 2FA and Telegram suggests adding one |
| `otherwise_relogin_days` | Days until re-login is forced if no password is set |
| `tmp_sessions` | Number of temporary sessions allowed (PFS-related) |
| `future_auth_token` | Token for faster re-login later |

`user` is a `user#b1b8cc83` (`telegram_api.tl:115`) — a large flag-heavy constructor. Two
fields matter now:

* `id:long` — your own user id.
* `access_hash:flags.0?long` — your own access hash.

Together they let you construct `inputPeerSelf` alternatives and, more usefully, confirm
who you logged in as. See [8.1](../08-sending-a-message/01-peers-and-access-hashes.md).

> **💡 Implementation note.** `user#b1b8cc83` has **two** flag words (`flags:#` and
> `flags2:#`), the second appearing partway through the field list. Your parser must handle
> multi-flag constructors. If you only need `id` and `access_hash`, you still have to walk
> every preceding conditional field to find them — there is no way to skip ahead in TL.

### `auth.authorizationSignUpRequired` — no such account

The number is valid and the code was correct, but no account exists. To create one:

```
auth.signUp#aac7b717 flags:# no_joined_notifications:flags.0?true phone_number:string
    phone_code_hash:string first_name:string last_name:string = auth.Authorization;
```
(`telegram_api.tl:2317`)

Sent with the same `phone_number` and `phone_code_hash`
(`td/telegram/AuthManager.cpp:872`). If `terms_of_service` was present in the
`SignUpRequired` reply, you are expected to show it and record acceptance
(`help.termsOfService#780a0310`, `telegram_api.tl:810`).

---

## 3. Errors

| Error | Meaning | Action |
|-------|---------|--------|
| `401 SESSION_PASSWORD_NEEDED` | 2FA is on | Go to [7.5](05-two-factor-srp.md) |
| `400 PHONE_CODE_INVALID` | Wrong code | Ask again; the code is still valid |
| `400 PHONE_CODE_EXPIRED` | Code timed out | `auth.resendCode` or start over |
| `400 PHONE_CODE_EMPTY` | Empty `phone_code` | Set flag bit 0 and the field |
| `400 PHONE_NUMBER_UNOCCUPIED` | No account (older form of `SignUpRequired`) | Sign up |
| `400 PHONE_NUMBER_INVALID` | Number mismatch with `sendCode` | Use the identical string |
| `303 PHONE_MIGRATE_N` | Wrong DC | Migrate and retry the **whole** login |
| `420 FLOOD_WAIT_X` | Rate limited | Wait |

### `SESSION_PASSWORD_NEEDED` is not a failure

TDLib checks for it explicitly (`td/telegram/AuthManager.cpp:1577`):

```cpp
net_query->error().code() == 401 && net_query->error().message() == CSlice("SESSION_PASSWORD_NEEDED")
```

It means the code was accepted and the server now wants the second factor. Do **not** treat
a `401` here as "log in again from scratch" — you would loop forever. Match on the message
string.

### Wrong code vs expired code

`PHONE_CODE_INVALID` — the code is wrong but the *session* is fine. Prompt again with the
same `phone_code_hash`.

`PHONE_CODE_EXPIRED` — the `phone_code_hash` is dead. You need a new `sendCode` (or
`resendCode`).

Confusing these leads to a client that either never lets the user retry, or retries
uselessly forever.

---

## 4. Worked call

```
51 a9 52 8d                             auth.signIn#8d52a951
01 00 00 00                             flags = 1  (phone_code present)
0a 39 39 39 36 36 33 36 34 33 37 00     "9996636437"
16 3f 63 34 …                           phone_code_hash (from sentCode)
05 32 32 32 32 32 00 00                 "22222"  (test-DC code)
```

On the test datacenters the code is the DC number repeated to the required length — DC 2
gives `22222`.

---

## 5. After success

The moment you receive `auth.authorization`:

1. **Persist the `auth_key`.** It is now an account credential, not just a session key.
2. Persist the `server_salt` and `server_time_difference`.
3. Record your own `user.id`.
4. Mark the DC as authorized.
5. Discard `phone_code_hash` and the code — they are spent.

See [7.6](06-after-login.md).

---

## 6. Checklist

- [ ] `flags = 1` with `phone_code` present, for a normal code login
- [ ] `phone_number` identical to the `sendCode` value
- [ ] `phone_code_hash` from the `sentCode` reply
- [ ] `401 SESSION_PASSWORD_NEEDED` matched on **message**, routed to SRP
- [ ] `PHONE_CODE_INVALID` and `PHONE_CODE_EXPIRED` handled differently
- [ ] `auth.authorizationSignUpRequired` handled or reported clearly
- [ ] Parser copes with `user#b1b8cc83`'s two flag words
- [ ] `auth_key` persisted immediately on success

---

[← Previous](03-send-code.md) · [Index](../README.md) · [Next: Two-factor SRP →](05-two-factor-srp.md)
