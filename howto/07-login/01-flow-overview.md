# 7.1 — Login Flow Overview

[← Previous](../06-api-layer/03-migration-and-multi-dc.md) · [Index](../README.md) · [Next: API credentials →](02-api-credentials.md)

---

Everything up to now was protocol plumbing. From here on you are talking to the Telegram
API proper. This chapter is the map; the following five are the detail.

---

## 1. The whole flow on one page

```
                     ┌──────────────────────────┐
                     │  auth_key established    │   ← chapter 3
                     │  initConnection sent     │   ← chapter 6.1
                     └────────────┬─────────────┘
                                  │
                                  ▼
                  auth.sendCode(phone, api_id, api_hash, settings)
                                  │
              ┌───────────────────┼────────────────────┐
              │                   │                    │
       303 PHONE_MIGRATE_N   auth.sentCode      auth.sentCodeSuccess
              │              {phone_code_hash}   {authorization}
              ▼                   │                    │
    switch DC, redo handshake,    │                    ▼
    resend  ─────────────────►    │              ✅ logged in
                                  │                (rare: QR/token)
                                  ▼
                          user types the code
                                  │
                                  ▼
        auth.signIn(flags=1, phone, phone_code_hash, phone_code)
                                  │
        ┌─────────────────────────┼──────────────────────┬─────────────────┐
        │                         │                      │                 │
 auth.authorization    401 SESSION_PASSWORD_NEEDED   400 PHONE_CODE_*   auth.authorization
   {user}                        │                    INVALID/EXPIRED   SignUpRequired
        │                        ▼                        │                 │
        ▼               account.getPassword               ▼                 ▼
   ✅ logged in                  │                  retry / resend      auth.signUp
                                 ▼                                     (new account)
                        compute SRP A and M1
                                 │
                                 ▼
              auth.checkPassword(inputCheckPasswordSRP{srp_id, A, M1})
                                 │
                     ┌───────────┴────────────┐
                     ▼                        ▼
            auth.authorization      400 PASSWORD_HASH_INVALID
                     │                        │
                     ▼                        ▼
              ✅ logged in                retry
```

---

## 2. The calls

| # | Call | Constructor | Purpose |
|---|------|-------------|---------|
| 1 | `auth.sendCode` | `0xa677244f` | Request a login code |
| 2 | `auth.signIn` | `0x8d52a951` | Submit the code |
| 3 | `account.getPassword` | `0x548a30f5` | Get 2FA parameters (only if needed) |
| 4 | `auth.checkPassword` | `0xd18b4d16` | Submit the 2FA proof (only if needed) |

Plus two you may need:

| Call | Constructor | Purpose |
|------|-------------|---------|
| `auth.resendCode` | `0xcae47523` | Ask for the code again, by another route |
| `auth.signUp` | `0xaac7b717` | Register a brand-new account |

All of these are sent **without** the "requires authorization" marker — TDLib uses
`create_unauth` for every one of them (`td/telegram/AuthManager.cpp:802, 810, 872, 1364`).

---

## 3. State machine

Model login explicitly. TDLib's `AuthManager::State` enum is the reference; a minimal
version:

```
                  ┌─────────────┐
                  │   Start     │
                  └──────┬──────┘
                         │ user gives phone number
                         ▼
                  ┌─────────────┐
                  │ WaitCode    │◄──── resendCode
                  └──────┬──────┘
                         │ user gives code
                         ▼
              ┌──────────┴───────────┐
              ▼                      ▼
      ┌──────────────┐       ┌──────────────┐
      │ WaitPassword │       │  WaitSignUp  │
      └──────┬───────┘       └──────┬───────┘
             │ user gives password  │ user gives name
             ▼                      ▼
                  ┌─────────────┐
                  │     Ok      │
                  └─────────────┘
```

You must persist `phone_number` and `phone_code_hash` between `sendCode` and `signIn` —
both are required parameters of `signIn`, and `phone_code_hash` comes only from the
`sendCode` reply.

---

## 4. What you need before you start

| Thing | Where from |
|-------|------------|
| A working `auth_key` for the DC | [Chapter 3](../03-auth-key/01-overview.md) |
| Working encryption and framing | Chapters [2](../02-transport/02-tcp-framings.md) and [4](../04-encrypted-messages/01-envelope.md) |
| `initConnection` sent on the connection | [6.1](../06-api-layer/01-init-connection.md) |
| `api_id` and `api_hash` | [7.2](02-api-credentials.md) |
| A phone number that can receive a code | Yours |
| Handling for `303`, `420`, gzip | [6.2](../06-api-layer/02-rpc-results-and-errors.md), [6.3](../06-api-layer/03-migration-and-multi-dc.md) |

---

## 5. Where the code arrives

Telegram will not always SMS you. `auth.sentCode.type` says how it was delivered
(`telegram_api.tl:854-864`):

| Type | Constructor | Delivery |
|------|-------------|----------|
| `auth.sentCodeTypeApp` | `0x3dbb5986` | To another logged-in Telegram client |
| `auth.sentCodeTypeSms` | `0xc000bba2` | SMS |
| `auth.sentCodeTypeCall` | `0x5353e5a7` | Voice call reading digits |
| `auth.sentCodeTypeFlashCall` | `0xab03c6d9` | Missed call; the code is in the caller number |
| `auth.sentCodeTypeMissedCall` | `0x82006484` | Missed call, `prefix` + last `length` digits |
| `auth.sentCodeTypeEmailCode` | `0xf450f59b` | Email |
| `auth.sentCodeTypeFragmentSms` | `0xd9565c39` | Fragment.com anonymous number |
| `auth.sentCodeTypeFirebaseSms` | `0x009fd736` | SMS gated by app attestation |
| `auth.sentCodeTypeSmsWord` | `0xa416ac81` | A **word**, not digits |
| `auth.sentCodeTypeSmsPhrase` | `0xb37794af` | A **phrase**, not digits |
| `auth.sentCodeTypeSetUpEmailRequired` | `0xa5491dea` | You must configure email login first |

> **⚠ The very first time you log in with a new `api_id`, the code goes to the *Telegram
> app*, not SMS** — `auth.sentCodeTypeApp`. If you have Telegram installed on your phone,
> look there. This surprises nearly everyone.
>
> Note also that `SmsWord` and `SmsPhrase` are not numeric. A client that assumes digits
> will break.

---

## 6. Rate limits, honestly

Login is the most aggressively rate-limited part of the API.

* Repeated `auth.sendCode` for the same number → `420 FLOOD_WAIT_X`, growing quickly.
* Wrong codes → the code is invalidated; you must `sendCode` again.
* Automated login attempts → the `api_id` can be banned.

**Test with the test datacenters** ([Appendix B](../appendix/B-datacenters-and-keys.md))
while developing. They accept synthetic phone numbers such as `9996636437` and take any
code of the form `dcdcdcdc...`. TDLib's own tests use `9996636437` and `9996636438`
(`test/tdclient.cpp:676-677`).

---

## 7. Checklist

- [ ] Login modelled as an explicit state machine
- [ ] `phone_number` and `phone_code_hash` retained between `sendCode` and `signIn`
- [ ] `303 PHONE_MIGRATE_N` handled before anything else
- [ ] All `auth.*` calls sent **without** the authorization requirement
- [ ] All `auth.sentCodeType*` variants at least recognized, non-numeric ones not assumed to be digits
- [ ] `SESSION_PASSWORD_NEEDED` routed to the SRP path
- [ ] `FLOOD_WAIT_X` honoured
- [ ] Developed against the test DCs first

---

[← Previous](../06-api-layer/03-migration-and-multi-dc.md) · [Index](../README.md) · [Next: API credentials →](02-api-credentials.md)
