# 7.2 — API Credentials (`api_id` and `api_hash`)

[← Previous](01-flow-overview.md) · [Index](../README.md) · [Next: auth.sendCode →](03-send-code.md)

---

Short chapter, but skipping it will cost you an afternoon.

---

## 1. What they are

| Credential | Type | Purpose |
|------------|------|---------|
| `api_id` | `int32` | Identifies your *application* |
| `api_hash` | `string` (32 lowercase hex chars) | Proves you own that `api_id` |

They identify the **application**, not the user. The same pair is used by every user of
your program.

They are used in exactly two places:

* `initConnection.api_id` ([6.1](../06-api-layer/01-init-connection.md))
* `auth.sendCode(phone_number, api_id, api_hash, settings)` ([7.3](03-send-code.md))

After login they are not needed again for the lifetime of the authorization.

---

## 2. Getting them

1. Go to <https://my.telegram.org>.
2. Log in with your phone number.
3. Open **API development tools**.
4. Fill in an app name and short name.
5. You are shown `App api_id` and `App api_hash`.

The credentials are issued once per account. Note them down.

---

## 3. Why you cannot borrow someone else's

Telegram detects and blocks publicly-known `api_id` values:

```
400 API_ID_PUBLISHED_FLOOD
```

This error means "this `api_id` appears in public source code and is being abused". It
applies to the `api_id` values embedded in the official clients, in tutorials, and in
TDLib's own examples. It does not go away with time.

TDLib's example `api_id` is `21724`. **Do not use it.** Its documentation
(`example/README.md`) says so explicitly.

Related errors:

| Error | Meaning |
|-------|---------|
| `400 API_ID_INVALID` | No such `api_id`, or the `api_hash` does not match it |
| `400 API_ID_PUBLISHED_FLOOD` | Publicly-known `api_id` |

---

## 4. Storing them

> **⚠ Security note.** `api_hash` is a **secret**. Anyone with your pair can impersonate
> your application. Treat it like a password:
>
> * Never commit it to a repository.
> * Never log it, not even at debug level.
> * Load it from an environment variable, a config file outside the repository, or a
>   keyring.
>
> That said, understand the limits: any client binary that talks to Telegram must contain
> the pair somewhere, so it cannot be kept secret from a determined user of your own
> program. What you are protecting against is casual copying and accidental publication.

A reasonable pattern:

```
api_id   = env("TG_API_ID")   or fail
api_hash = env("TG_API_HASH") or fail
```

with a `.env` file listed in `.gitignore`.

---

## 5. What they do *not* protect

`api_id`/`api_hash` are not user credentials and grant no access on their own. Access comes
from the **authorization** you obtain by logging in, which is bound to the `auth_key`. See
[7.6](06-after-login.md) for how to store *that* — it is far more sensitive.

---

## 6. Documentation examples

Throughout this guide, examples use:

```
api_id   = 94575
api_hash = a3406de8d171bb422bb6ddf3bbd800e2
```

These come from TDLib's own documentation examples and exist so that byte-level worked
examples have concrete values. **They will not work.** Substitute your own.

---

## 7. Checklist

- [ ] Own `api_id`/`api_hash` obtained from <https://my.telegram.org>
- [ ] Loaded from the environment or a config file, not hard-coded
- [ ] Not committed to version control
- [ ] Not written to logs
- [ ] `API_ID_INVALID` and `API_ID_PUBLISHED_FLOOD` produce a clear message to the user

---

[← Previous](01-flow-overview.md) · [Index](../README.md) · [Next: auth.sendCode →](03-send-code.md)
