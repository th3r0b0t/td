# 7.3 — `auth.sendCode`

[← Previous](02-api-credentials.md) · [Index](../README.md) · [Next: auth.signIn →](04-sign-in.md)

---

The first real API call of a login. It asks Telegram to deliver a confirmation code to the
phone number, and returns the `phone_code_hash` you will need in the next step.

---

## 1. The call

```
auth.sendCode#a677244f phone_number:string api_id:int api_hash:string
    settings:CodeSettings = auth.SentCode;
```
(`td/generate/scheme/telegram_api.tl:2316`)

| Field | Notes |
|-------|-------|
| `phone_number` | International format. `+` optional; digits only is safest |
| `api_id` | Yours ([7.2](02-api-credentials.md)) |
| `api_hash` | Yours |
| `settings` | A `CodeSettings`; see below |

> **⚠ Phone number format.** Use the full international number: country code then
> subscriber number, e.g. `447700900000`. Do **not** include spaces, dashes, or brackets.
> A leading `+` is accepted but not required. Getting this wrong yields
> `400 PHONE_NUMBER_INVALID`.

---

## 2. `CodeSettings`

```
codeSettings#ad253d78 flags:# allow_flashcall:flags.0?true current_number:flags.1?true
    allow_app_hash:flags.4?true allow_missed_call:flags.5?true allow_firebase:flags.7?true
    unknown_number:flags.9?true logout_tokens:flags.6?Vector<bytes>
    token:flags.8?string app_sandbox:flags.8?Bool = CodeSettings;
```
(`telegram_api.tl:1319`)

Every field is optional, and the flag-only bits (`allow_flashcall`, `current_number`,
`allow_app_hash`, `allow_missed_call`, `allow_firebase`, `unknown_number`) consume **no
bytes** — they are `flags.N?true` ([1.2 §5](../01-serialization/02-wire-encoding.md)).

**For a custom client, send `flags = 0` and nothing else.** That is five bytes on the wire:

```
78 3d 25 ad      codeSettings#ad253d78
00 00 00 00      flags = 0
```

The flags you might set, and why you probably should not:

| Flag | Meaning | Set it? |
|------|---------|---------|
| `allow_flashcall` | Deliver by missed call; you read the code from the caller id | Only on Android with call-log permission |
| `current_number` | The number is the device's own SIM | No |
| `allow_app_hash` | Include an Android SMS Retriever hash | No |
| `allow_missed_call` | Allow `sentCodeTypeMissedCall` | No |
| `allow_firebase` | Allow Firebase-attested SMS | No, requires app attestation |
| `unknown_number` | Number is not the device's own | Optional; harmless |
| `logout_tokens` | Tokens from previous sessions, for faster re-login | No |
| `token` + `app_sandbox` | APNs/Firebase push token for app-based verification | No |

Setting a flag you cannot honour just makes delivery worse. `flags = 0` gets you the
default behaviour, which is what you want.

---

## 3. The reply

Three possible results (`telegram_api.tl:260-262`):

```
auth.sentCode#5e002502 flags:# type:auth.SentCodeType phone_code_hash:string
    next_type:flags.1?auth.CodeType timeout:flags.2?int = auth.SentCode;
auth.sentCodeSuccess#2390fe44 authorization:auth.Authorization = auth.SentCode;
auth.sentCodePaymentRequired#f8827ebf store_product:string phone_code_hash:string
    support_email_address:string support_email_subject:string premium_days:int
    currency:string amount:long = auth.SentCode;
```

### `auth.sentCode` — the normal case

| Field | Meaning |
|-------|---------|
| `type` | How the code was delivered — see [7.1 §5](01-flow-overview.md) |
| `phone_code_hash` | **Save this.** Required by `auth.signIn` |
| `next_type` | What `auth.resendCode` would use next |
| `timeout` | Seconds before `resendCode` is allowed |

**`phone_code_hash` is the critical output.** It is an opaque string tying your `signIn` to
this particular `sendCode`. Lose it and you must start over.

`type` also carries a `length` field on most variants — the number of digits to expect.
Use it to size your input prompt, and remember `SmsWord`/`SmsPhrase` are not numeric.

### `auth.sentCodeSuccess` — already done

The server logged you in without a code, using a previously-issued token. The embedded
`auth.Authorization` is exactly what `auth.signIn` would have returned. Skip to
[7.6](06-after-login.md).

Rare for a custom client, but handle it — a client that assumes `auth.sentCode` will crash.

### `auth.sentCodePaymentRequired` — Fragment numbers

Applies to anonymous numbers requiring a purchase. Not something a simple client handles;
report the error and stop.

---

## 4. Errors

| Error | Meaning | Action |
|-------|---------|--------|
| `303 PHONE_MIGRATE_N` | Number belongs to DC N | Migrate ([6.3](../06-api-layer/03-migration-and-multi-dc.md)) and retry |
| `400 PHONE_NUMBER_INVALID` | Malformed number | Fix the format |
| `400 PHONE_NUMBER_BANNED` | Number is banned | Nothing you can do |
| `400 PHONE_NUMBER_FLOOD` | Too many codes to this number | Wait — hours to days |
| `400 API_ID_INVALID` | Bad credentials | Check `api_id`/`api_hash` |
| `400 API_ID_PUBLISHED_FLOOD` | Public `api_id` | Get your own |
| `406 PHONE_PASSWORD_FLOOD` | Too many attempts | Wait |
| `420 FLOOD_WAIT_X` | Rate limited | Sleep `X` seconds |

`PHONE_MIGRATE_N` is by far the most common and the one you *must* implement — most
accounts do not live on whichever DC you guessed.

---

## 5. Resending

```
auth.resendCode#cae47523 flags:# phone_number:string phone_code_hash:string
    reason:flags.0?string = auth.SentCode;
auth.cancelCode#1f040578 phone_number:string phone_code_hash:string = Bool;
```
(`telegram_api.tl:2328, 2329`)

`resendCode` reuses the **same** `phone_code_hash` and returns a new `auth.sentCode`,
typically with a different `type` (app → SMS → call). Respect the `timeout` from the
previous reply before calling it; TDLib gates this in
`AuthManager::resend_authentication_code` (`td/telegram/AuthManager.cpp:795-803`).

`cancelCode` invalidates the code and lets the user start over with a different number.

---

## 6. Worked call

Wrapped in the connection header as your first query on a fresh connection:

```
invokeWithLayer(229,
  initConnection(api_id=94575, "PC", "Linux", "1.0", "en", "", "", params={},
    auth.sendCode(
      phone_number     = "9996636437",
      api_id           = 94575,
      api_hash         = "a3406de8d171bb422bb6ddf3bbd800e2",
      settings         = codeSettings(flags=0)
    )))
```

The inner `auth.sendCode` alone:

```
4f 24 77 a6                          auth.sendCode#a677244f
0a 39 39 39 36 36 33 36 34 33 37 00  "9996636437"  (1+10 = 11, +1 pad = 12)
ef 71 01 00                          api_id = 94575
20 61 33 34 30 36 64 65 38 …         "a3406de8…" (1+32 = 33, +3 pad = 36)
78 3d 25 ad                          codeSettings#ad253d78
00 00 00 00                          flags = 0
```

---

## 7. Checklist

- [ ] Phone number in international format, digits only
- [ ] `codeSettings` with `flags = 0` unless you can honour a flag
- [ ] `phone_code_hash` from the reply **persisted**
- [ ] All three `auth.SentCode` result types handled
- [ ] `303 PHONE_MIGRATE_N` triggers a DC switch and a retry
- [ ] `timeout` respected before calling `auth.resendCode`
- [ ] `FLOOD_WAIT_X` honoured
- [ ] `type.length` used to size the prompt; non-numeric types not assumed to be digits

---

[← Previous](02-api-credentials.md) · [Index](../README.md) · [Next: auth.signIn →](04-sign-in.md)
