# 9.2 — Connection Management

[← Previous](01-perfect-forward-secrecy.md) · [Index](../README.md) · [Next: Project skeleton →](../10-implementation/01-project-skeleton.md)

---

Everything about keeping a connection alive that does not belong to the protocol proper.

---

## 1. One connection is enough

TDLib opens several connections per DC and classifies them (`Session::Mode::Tcp`,
download/upload sessions, media-only DCs). **You do not need any of that.** A single TCP
connection carries login, message sending, and updates without difficulty.

Add connections only when you have a concrete reason: parallel file transfer, or isolating
long-poll updates from interactive queries.

---

## 2. Address selection and rotation

A DC has several addresses ([2.1](../02-transport/01-datacenters.md)). TDLib shuffles them
before use (`td/telegram/net/ConnectionCreator.cpp:1226`):

```cpp
Random::shuffle(ip_address_strings);
```

and tries ports `{443, 80, 5222}` (`ConnectionCreator.cpp:1244`).

Port 443 is the most likely to survive a restrictive firewall, since it looks like HTTPS.
Port 80 is the fallback. Try in that order, and remember which one worked.

---

## 3. Reconnect strategy

Exponential backoff with jitter:

```
delay = min(60, 1 * 2^attempt) * (0.5 + random(0..1) * 0.5)
```

Reset `attempt` to 0 after a connection stays up for, say, 30 seconds. Without that reset,
a flaky link degrades to one attempt per minute permanently.

Do **not** reconnect instantly in a loop. You will be rate-limited, and on mobile you will
drain the battery.

What survives a reconnect is covered in
[5.5 §2](../05-session/05-reliability.md): the auth key, session id, sequence number, salt,
and time difference all persist; only the transport state and in-flight messages are lost.

---

## 4. Timeouts

| What | Reasonable value | Source |
|------|------------------|--------|
| TCP connect | 10-20 s | — |
| Handshake completion | 10 s | `Handshake.cpp:87` uses 0.6 × `timeout_in_` |
| Query response | 60 s | `NetQueryCreator.cpp:66` (`total_timeout_limit = 60`) |
| Query response (bots) | 8 s | `NetQueryCreator.cpp:74` |
| Ping interval | 30-60 s | — |
| Ping timeout | interval + 5 s | — |

The handshake uses fractional deadlines — `0.6 * timeout_in_` after `req_pq_multi`
(`Handshake.cpp:87`) and `0.8 * timeout_in_` after `req_DH_params`
(`Handshake.cpp:164`) — so that a slow first step does not consume the whole budget.

---

## 5. Detecting a dead connection

TCP will not tell you promptly. A connection can be silently dead for many minutes before
`write()` fails. Two remedies:

* **Pings.** `ping_delay_disconnect(ping_id, disconnect_delay)`
  ([5.5 §7](../05-session/05-reliability.md)). No `pong` within the timeout ⇒ dead.
* **TCP keepalive.** `SO_KEEPALIVE` plus `TCP_KEEPIDLE`/`TCP_KEEPINTVL` where available.
  Cheaper, but coarser and not portable.

Use pings; they also stop the server from dropping you.

---

## 6. Proxies

Three kinds, in ascending order of effort:

| Kind | Effort | Notes |
|------|--------|-------|
| **HTTP CONNECT** | Low | Standard; tunnel then speak MTProto normally |
| **SOCKS5** | Low | Standard; same |
| **MTProxy** | High | Telegram-specific; needs obfuscation + secret |

For MTProxy, the secret formats are in
[2.3](../02-transport/03-obfuscation.md) (`td/mtproto/ProxySecret.h:34-50`):

| Length | First byte | Meaning |
|--------|-----------|---------|
| 16 | — | Plain secret |
| 17 | `0xDD` | Padded-intermediate transport required |
| ≥17 | `0xEE` | Fake-TLS; the rest is a domain name |

In all the prefixed forms the actual secret is bytes `[1..17)`.

With an MTProxy you must also set `initConnection.flags` bit 0 and include
`inputClientProxy#75588b3f address:string port:int`
(`td/generate/scheme/telegram_api.tl:1186`), so the server knows you came via a proxy.

**Skip proxies entirely for a first implementation.**

---

## 7. Packet size limits

`RawConnection.cpp:165-172`:

```
MAX_PACKET_SIZE = (1 << 22) + 1024        // 4 MiB + 1 KiB
```

Reject anything larger before allocating. A length prefix is attacker-controlled input;
treating it as a trusted allocation size is a denial-of-service waiting to happen.

The Intermediate framing additionally requires `size % 4 == 0` and `size < 1 << 24`
(`TcpTransport.cpp:51-52`).

---

## 8. Threading

A workable single-threaded design:

```
loop:
    poll(socket, timeout = next_deadline())
    if readable:  read → deframe → deobfuscate → decrypt → dispatch
    if writable:  encrypt → obfuscate → frame → write
    run timers:   pings, acks (30 s), resends, salt refresh
```

One thread, one `poll`/`epoll`/`kqueue`. Everything in this guide fits that model. TDLib's
actor system exists to manage hundreds of concurrent operations; a single-purpose client
does not need it.

If you do use threads, the auth key, salt, `seq_no`, and pending-query map are all shared
mutable state and need a lock.

---

## 9. Logging

> **⚠ Never log:** `auth_key`, `api_hash`, the password, SRP intermediates (`x`, `a`, `S`,
> `K`), `phone_code_hash`, or the login code. Not even truncated.

Safe and useful to log: `msg_id`, `seq_no`, constructor ids, error codes and messages,
packet lengths, and connection state transitions. That is enough to diagnose almost
anything.

TDLib's `VLOG(mtproto)` statements throughout `SessionConnection.cpp` are a good model for
what granularity is useful.

---

## 10. Checklist

- [ ] A single connection per DC, unless there is a specific reason for more
- [ ] Addresses shuffled; ports tried in order 443, 80, 5222
- [ ] Exponential backoff with jitter, capped at ~60 s
- [ ] Backoff counter reset after a stable connection
- [ ] Connect, handshake, and query timeouts all set
- [ ] Periodic pings with a response timeout
- [ ] `MAX_PACKET_SIZE` enforced before allocating
- [ ] `size % 4 == 0` enforced for Intermediate framing
- [ ] Secrets never logged
- [ ] Shared state protected if multi-threaded

---

[← Previous](01-perfect-forward-secrecy.md) · [Index](../README.md) · [Next: Project skeleton →](../10-implementation/01-project-skeleton.md)
