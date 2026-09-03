# 2.1 — Datacenters: Where to Connect

[← Previous](../01-serialization/03-worked-examples.md) · [Index](../README.md) · [Next: TCP framings →](02-tcp-framings.md)

---

## 1. The datacenter model

Telegram runs several server clusters called **datacenters**, numbered 1 through 5 in
production. Three facts drive everything in this chapter:

1. **Auth keys are per-datacenter.** A key you negotiated with DC 2 is meaningless to DC 4.
   To use another DC you must perform a *complete new handshake* there.
2. **Your account lives in exactly one DC** — the one closest to where you registered.
   That is your *home* or *main* DC. Only your home DC can perform the login flow for your
   phone number.
3. **You do not know which DC is yours until you try.** You connect to a default DC,
   attempt `auth.sendCode`, and if it is the wrong one the server replies
   `rpc_error(303, "PHONE_MIGRATE_4")` telling you where to go.

---

## 2. Bootstrap addresses

You need at least one hardcoded address to reach the first server. TDLib's are in
`td/telegram/net/ConnectionCreator.cpp:1244-1279`.

### Production

| DC | IPv4 | IPv6 |
|----|------|------|
| 1 | `149.154.175.50` | `2001:b28:f23d:f001::a` |
| 2 | `149.154.167.51`, `95.161.76.100` | `2001:67c:4e8:f002::a` |
| 3 | `149.154.175.100` | `2001:b28:f23d:f003::a` |
| 4 | `149.154.167.91` | `2001:67c:4e8:f004::a` |
| 5 | `149.154.171.5` | `2001:b28:f23f:f005::a` |

### Test

| DC | IPv4 | IPv6 |
|----|------|------|
| 1 | `149.154.175.10` | `2001:b28:f23d:f001::e` |
| 2 | `149.154.167.40` | `2001:67c:4e8:f002::e` |
| 3 | `149.154.175.117` | `2001:b28:f23d:f003::e` |

### Ports

`vector<int> ports = {443, 80, 5222};` (`ConnectionCreator.cpp:1244`)

All three carry the same protocol. **Use 443** — it is the least likely to be filtered and
the most likely to be assumed to be TLS by middleboxes. Fall back to 80 then 5222 if 443
fails.

> **💡 Implementation note.** Start with **DC 2** for production. It is the conventional
> default and is where most bootstrap traffic goes. Any DC will do for the handshake; you
> only need the *right* one for the account-specific calls.

### Test datacenters

Test DCs (`use_test_dc`) are a completely separate universe: separate accounts, separate
auth keys, and a **different RSA public key** (see
[appendix B](../appendix/B-datacenters-and-keys.md)). They are ideal for development
because they accept "virtual" phone numbers of the form `99966XYYYY` where `X` is the DC
number, and the login code is the DC number repeated. TDLib's own integration tests use
`9996636437` and `9996636438` (`test/tdclient.cpp:676-677`).

Note that TDLib itself has **no special handling** for these numbers — the convention is
entirely server-side. You just type the code like any other.

---

## 3. Learning the real address list: `help.getConfig`

Hardcoded IPs go stale. As soon as you have an encrypted connection, call:

```
help.getConfig#c4f9186b = Config;
```

The reply (`config#cc1a241e`, `td/generate/scheme/telegram_api.tl:538`) contains, among
~50 other fields:

```
this_dc:int                    ← which DC you are currently talking to
dc_options:Vector<DcOption>    ← the authoritative address list
test_mode:Bool                 ← sanity check against your own expectation
expires:int                    ← when to refetch
```

Each entry:

```
dcOption#18b7a10d flags:# ipv6:flags.0?true media_only:flags.1?true
    tcpo_only:flags.2?true cdn:flags.3?true static:flags.4?true
    this_port_only:flags.5?true id:int ip_address:string port:int
    secret:flags.10?bytes = DcOption;
```
(`td/generate/scheme/telegram_api.tl:536`)

Flag meanings:

| Flag | Bit | Meaning |
|------|-----|---------|
| `ipv6` | 0 | `ip_address` is an IPv6 literal |
| `media_only` | 1 | Only for file up/download — do **not** use for API calls |
| `tcpo_only` | 2 | Only reachable via obfuscated TCP; plain framing will be rejected |
| `cdn` | 3 | A CDN datacenter; different key handling, not usable for normal API calls |
| `static` | 4 | The address is stable and may be cached long-term |
| `this_port_only` | 5 | Do not try the other ports on this address |

**Persist `dc_options` and prefer them over your hardcoded list on subsequent runs.**
TDLib does exactly this in `td/telegram/ConfigManager.cpp` and
`td/telegram/net/DcOptionsSet.cpp`.

> **⚠ Security note.** `help.getConfig` must be called over an **encrypted** connection
> (i.e. after the handshake). Fetching the DC list over an unauthenticated channel would
> let an attacker redirect you. TDLib always sends it as an ordinary encrypted query.

---

## 4. When you get sent to another DC

Three errors mean "you are on the wrong DC":

| Error | Meaning | Action |
|-------|---------|--------|
| `PHONE_MIGRATE_X` | This phone number belongs to DC *X* | Switch main DC to *X* and redo |
| `NETWORK_MIGRATE_X` | Your network is served by DC *X* | Same |
| `USER_MIGRATE_X` | This user belongs to DC *X* | Same |
| `FILE_MIGRATE_X` | This *file* lives on DC *X* | Retry **only this query** on *X*; keep your main DC |

All four arrive as `rpc_error` with code `303`.

TDLib's handler (`td/telegram/net/NetQueryDispatcher.cpp:391-418`):

```cpp
static constexpr CSlice prefixes[] = {"PHONE_MIGRATE_", "NETWORK_MIGRATE_", "USER_MIGRATE_"};
for (auto &prefix : prefixes) {
  if (error_message.substr(0, prefix.size()) == prefix) {
    auto new_main_dc_id = to_integer<int32>(error_message.substr(prefix.size()));
    set_main_dc_id(new_main_dc_id);
    …resend…
  }
}
```

**Concretely, on `PHONE_MIGRATE_4` you must:**

1. Close the current connection.
2. Connect to DC 4's address.
3. Perform a **brand-new DH handshake** — your DC 2 auth key is useless here.
4. Send `invokeWithLayer(initConnection(…))` again.
5. Retry `auth.sendCode`.

This is not an optimization you can skip; there is no way to transfer a key between DCs.
Persist the discovered main DC id so you do not repeat the dance on every start
(TDLib stores it as the `"main_dc_id"` key).

---

## 5. Using more than one DC at once

Once you are logged in on your home DC, you can authorize other DCs *without* repeating
the phone-code flow, using exported authorizations
(`td/telegram/net/DcAuthManager.cpp:160-182`):

```
1. On the HOME DC:    auth.exportAuthorization#e5bfffcd dc_id:int = auth.ExportedAuthorization
                      → auth.exportedAuthorization#b434e2b8 id:long bytes:bytes
2. Handshake with the TARGET DC (new auth key)
3. On the TARGET DC:  auth.importAuthorization#a57a7dad id:long bytes:bytes = auth.Authorization
```

`auth.exportAuthorization` and `auth.importAuthorization` are among the few methods that
may be sent on a connection that is not yet user-authorized
(`td/telegram/net/NetQueryCreator.cpp:78-79`).

For a "log in and send one message" client you never need this. You need it as soon as you
want to download a file that lives elsewhere, or talk to a `media_only` DC.

---

## 6. Choosing an address at runtime — a workable policy

```
1.  If you have a persisted main_dc_id and a persisted auth key for it → use them.
2.  Else main_dc_id = 2.
3.  Candidate addresses = persisted dc_options for main_dc_id
                          ∪ hardcoded bootstrap for main_dc_id
    Filter out: cdn, media_only.
    Prefer: static, then IPv4 (unless you are on IPv6-only), then port 443.
4.  Connect with a short timeout (TDLib uses a few seconds); on failure try the next.
5.  After a successful encrypted connection, call help.getConfig and refresh the list.
```

TDLib randomizes the order of equal-preference addresses
(`ConnectionCreator.cpp:1226`, `Random::shuffle(ip_address_strings)`) to spread load. Doing
the same is polite and helps you notice if one address is broken.

---

## 7. What about HTTP and WebSocket?

MTProto also runs over HTTP (`td/mtproto/HttpTransport.cpp`) and, in Emscripten builds,
over WebSocket to hosts like `pluto.web.telegram.org/apiws`
(`ConnectionCreator.cpp:1245-1256`). The message and session layers are identical; only the
framing differs, and HTTP additionally needs `http_wait` long-polling to receive
server-initiated messages.

**Use plain TCP.** It is simpler, faster, and is what this guide documents.

---

[← Previous](../01-serialization/03-worked-examples.md) · [Index](../README.md) · [Next: TCP framings →](02-tcp-framings.md)
