# 6.3 — Migration and Multi-Datacenter Operation

[← Previous](02-rpc-results-and-errors.md) · [Index](../README.md) · [Next: Login overview →](../07-login/01-flow-overview.md)

---

Telegram runs several datacenters. Your account lives on exactly one of them — its **home
DC**. This chapter covers finding it, moving to it, and working with the others.

---

## 1. Why migration happens

You start by guessing a datacenter (DC 2 is the usual default). Your account may live
elsewhere. The server tells you so with a `303` error whose message names the correct DC.

```
303 PHONE_MIGRATE_4
```

means: *this phone number belongs to DC 4; ask DC 4.*

---

## 2. The four migration errors

`NetQueryDispatcher::try_fix_migrate` (`td/telegram/net/NetQueryDispatcher.cpp:391-418`):

```cpp
static constexpr CSlice file_migrate_prefix = "FILE_MIGRATE_";
if (begins_with(error_message, file_migrate_prefix)) {
  auto new_dc_id = to_integer<int32>(error_message.substr(file_migrate_prefix.size()));
  if (!DcId::is_valid(new_dc_id)) { … return; }
  net_query->resend(DcId::internal(new_dc_id));      // this query only
  return;
}
static constexpr CSlice prefixes[] = {"PHONE_MIGRATE_", "NETWORK_MIGRATE_", "USER_MIGRATE_"};
for (auto &prefix : prefixes) {
  if (error_message.substr(0, prefix.size()) == prefix) {
    auto new_main_dc_id = to_integer<int32>(error_message.substr(prefix.size()));
    set_main_dc_id(new_main_dc_id);                   // change the HOME dc
    if (!net_query->dc_id().is_main()) {
      net_query->resend(DcId::internal(new_main_dc_id));
    } else {
      net_query->resend();
    }
    break;
  }
}
```

| Prefix | When | Effect |
|--------|------|--------|
| `PHONE_MIGRATE_N` | `auth.sendCode` for a number on another DC | **Change home DC** to N, resend |
| `NETWORK_MIGRATE_N` | Your IP is served better by another DC | **Change home DC** to N, resend |
| `USER_MIGRATE_N` | Your account lives on another DC | **Change home DC** to N, resend |
| `FILE_MIGRATE_N` | A file lives on another DC | Resend **this query only** to N; home DC unchanged |

The distinction matters: the first three are permanent relocations, the fourth is a
one-off. Getting it wrong means either repeatedly migrating for every file, or never
migrating at all.

---

## 3. What migrating actually costs

Every datacenter needs its **own auth key**. Auth keys are not portable — they are bound to
the DC that generated them.

So "migrate to DC 4" means:

1. Connect to DC 4's address ([2.1](../02-transport/01-datacenters.md)).
2. Run the **entire** handshake from [chapter 3](../03-auth-key/01-overview.md) against DC 4.
3. Send `initConnection` on the new connection.
4. Resend the query.

If you were already authorized on the old DC, add step 3½ — see §5.

> **💡 Implementation note.** During login this is common and cheap: you have no
> authorization to transfer yet, so a plain handshake is all you need. Just be sure your
> code can run the handshake more than once and keep more than one auth key.

---

## 4. Discovering datacenter addresses

The bootstrap addresses in [Appendix B](../appendix/B-datacenters-and-keys.md) are enough to
get started, but they change. The authoritative list comes from:

```
help.getConfig#c4f9186b = Config;
config#cc1a241e flags:# … this_dc:int dc_options:Vector<DcOption> … = Config;
dcOption#18b7a10d flags:# ipv6:flags.0?true media_only:flags.1?true tcpo_only:flags.2?true
    cdn:flags.3?true static:flags.4?true this_port_only:flags.5?true
    id:int ip_address:string port:int secret:flags.10?bytes = DcOption;
```
(`telegram_api.tl:2791, 538, 536`)

`help.getConfig` works **before** authorization, so call it right after `initConnection`.

### Reading `dcOption`

| Flag | Meaning |
|------|---------|
| bit 0 `ipv6` | `ip_address` is IPv6 |
| bit 1 `media_only` | Only for downloads/uploads, not general API calls |
| bit 2 `tcpo_only` | Only reachable with obfuscated transport |
| bit 3 `cdn` | A CDN node, not a full DC |
| bit 4 `static` | The address is stable; do not treat as a hint |
| bit 5 `this_port_only` | Use exactly `port`, do not try others |
| bit 10 `secret` | MTProxy secret for this address |

Filter for: matching `id`, address family you support, **not** `media_only`, **not** `cdn`,
and if you do not implement obfuscation, **not** `tcpo_only`.

`config.this_dc` tells you which DC answered — useful for confirming a migration took.

### `help.getNearestDc`

```
help.getNearestDc#1fb33026 = NearestDc;
nearestDc#8e1a1775 country:string this_dc:int nearest_dc:int = NearestDc;
```
(`telegram_api.tl:2792, 540`)

Also pre-authorization. `nearest_dc` is a *suggestion* based on your IP, not where your
account lives. Useful for a fresh sign-up; irrelevant for signing in to an existing
account, where `PHONE_MIGRATE_N` is authoritative.

---

## 5. Authorizing on a second datacenter

Once logged in on your home DC, other DCs still do not know you. Rather than logging in
again, you transfer the authorization:

```
auth.exportAuthorization#e5bfffcd dc_id:int = auth.ExportedAuthorization;
auth.exportedAuthorization#b434e2b8 id:long bytes:bytes = auth.ExportedAuthorization;
auth.importAuthorization#a57a7dad id:long bytes:bytes = auth.Authorization;
```
(`telegram_api.tl:2321, 267, 2322`)

The procedure, from `DcAuthManager::dc_loop` (`td/telegram/net/DcAuthManager.cpp:145-195`):

1. On the **home** DC, call `auth.exportAuthorization(dc_id = N)`
   (`DcAuthManager.cpp:164-166`, sent with `DcId::main()` and `AuthFlag::On`).
2. You get back `auth.exportedAuthorization{id, bytes}`.
3. Handshake with DC N to get an auth key for it.
4. On DC N, call `auth.importAuthorization(id, bytes)`
   (`DcAuthManager.cpp:181-183`, sent with `AuthFlag::Off` — DC N does not know you yet, so
   the query must not be marked as requiring authorization).
5. DC N replies `auth.authorization` and is now authorized.

Note `total_timeout_limit_ = 60 * 60 * 24` on both queries
(`DcAuthManager.cpp:167, 184`) — TDLib will keep retrying for a day rather than give up.

`auth.exportAuthorization` is one of two calls explicitly allowed to be sent before
authorization completes (`NetQueryCreator.cpp:78-79`, alongside `auth.bindTempAuthKey`).

> **💡 Implementation note.** **You almost certainly do not need this.** To log in and send
> a text message you need exactly one DC: your home DC. `exportAuthorization` matters for
> file transfer, which lives on media DCs.

---

## 6. Recommended flow for a simple client

```
1. Connect to DC 2 (or any bootstrap address).
2. Handshake → auth_key for DC 2.
3. invokeWithLayer(layer, initConnection(…, help.getConfig()))
4. Store dc_options from the reply.
5. auth.sendCode(phone, …)
     └─ 303 PHONE_MIGRATE_N?
          ├─ yes: set home DC = N, connect to N, handshake, initConnection,
          │       retry auth.sendCode
          └─ no: continue
6. auth.signIn(…)
7. Send your message.
```

Steps 3-4 are optional but recommended: they give you fresh addresses and cost one cheap
round trip.

---

## 7. State to keep per datacenter

| Item | Scope |
|------|-------|
| `auth_key` + `auth_key_id` | Per DC |
| `server_salt` | Per DC |
| `session_id` | Per DC (per session, really) |
| `seq_no` counter | Per session |
| `server_time_difference` | **Global** — all DCs share one clock |
| "initConnection sent" flag | Per connection |
| Authorized? | Per DC |

A `HashMap<dc_id, DcState>` covers it.

---

## 8. Checklist

- [ ] `303` errors parsed for the four `*_MIGRATE_N` prefixes
- [ ] `N` validated as a plausible DC id before use
- [ ] `PHONE_`/`NETWORK_`/`USER_MIGRATE_` change the **home** DC
- [ ] `FILE_MIGRATE_` redirects **only that query**
- [ ] The handshake can run more than once and its results kept per DC
- [ ] `initConnection` sent on every new connection, including post-migration ones
- [ ] `help.getConfig` called before authorization to refresh `dc_options`
- [ ] `dcOption` filtering excludes `cdn` and `media_only` for general API calls
- [ ] Per-DC auth key, salt, and session state kept separate
- [ ] `server_time_difference` shared globally

---

[← Previous](02-rpc-results-and-errors.md) · [Index](../README.md) · [Next: Login overview →](../07-login/01-flow-overview.md)
