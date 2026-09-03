# 2.4 — Transport Errors

[← Previous](03-obfuscation.md) · [Index](../README.md) · [Next: Auth key overview →](../03-auth-key/01-overview.md)

---

Not every packet you receive is an MTProto message. The transport layer can deliver very
short frames that carry an error code or a control signal instead. If you feed these to
your decryptor you will get confusing failures, so handle them first.

---

## 1. Short frames

`Transport::read` (`td/mtproto/Transport.cpp:418-437`) is the dispatcher:

```cpp
if (error_code >= 300) {
  return ReadResult::make_error(-error_code);
}
if (message.size() < 16) {
  if (message.size() < 4) {
    return Status::Error("Invalid MTProto message: smaller than 4 bytes");
  }
  int32 code = as<int32>(message.begin());
  if (code == 0) {
    return ReadResult::make_nop();
  } else if (code == -1 && message.size() >= 8) {
    return ReadResult::make_quick_ack(as<uint32>(message.begin() + 4));
  } else {
    return ReadResult::make_error(code);
  }
}
```

Decision table for a received frame:

| Frame size | First `int32` LE | Meaning | Action |
|------------|------------------|---------|--------|
| < 4 bytes | — | Malformed | Close the connection |
| 4–15 bytes | `0` | No-op / keepalive | Ignore |
| ≥ 8, < 16 bytes | `-1` | Quick-ack; token is the next `uint32` | Mark the query acknowledged |
| 4–15 bytes | anything else | **Transport error**, value is negative | See §2 |
| ≥ 16 bytes | — | A real MTProto message | Decrypt / parse |

> **💡 Implementation note.** The 16-byte threshold is not arbitrary: the smallest legal
> MTProto frame is an unencrypted message header — `auth_key_id(8) ‖ msg_id(8)` — which is
> exactly 16 bytes. Anything shorter cannot be a message.

---

## 2. Transport error codes

A transport error arrives as a **bare, negative, little-endian `int32`** in a 4-byte frame.
There is no TL constructor, no `msg_id`, no encryption. The absolute value is an HTTP-like
status code.

| Wire value | Code | Meaning | What to do |
|------------|------|---------|------------|
| `-404` (`9C FE FF FF`) | 404 | **`auth_key_id` not found on the server.** The key has been forgotten, was never completed, or you are talking to the wrong DC. | Discard the auth key, redo the handshake |
| `-429` (`53 FE FF FF`) | 429 | Too many connections from this IP | Back off, reduce concurrent connections |
| `-444` | 444 | Invalid DC id in the obfuscation header | Fix the `dc_id` field ([2.3 §2](03-obfuscation.md)) |
| `-400` | 400 | Bad request at the transport level | Usually a framing bug |

`-404` is by far the most common and the most misdiagnosed. TDLib treats it specially in
`td/mtproto/SessionConnection.cpp:596-604`:

```cpp
case Transport::ReadResult::Error:
  on_read_mtproto_error(read_result.error());
  break;
```

and the handler drops the key. **If you get `-404` immediately after a successful
handshake**, the usual causes are, in order of likelihood:

1. You computed `auth_key_id` from the wrong slice. It is `SHA1(auth_key)[12..20)`, not
   `[0..8)` — see [3.4](../03-auth-key/04-step3-set-client-dh-params.md).
2. You are sending the key to a different DC than the one you negotiated it with.
3. Your `auth_key` is not exactly 256 bytes (a leading-zero truncation bug —
   see [3.5 §4](../03-auth-key/05-security-checks.md)).

> **⚠ Security note.** A transport error is unauthenticated — anyone able to inject TCP
> data can send you a fake `-404` and make you throw away a perfectly good auth key,
> forcing a fresh handshake. That is a nuisance, not a compromise, but do not treat
> transport errors as trustworthy signals about anything security-relevant.

---

## 3. The `error_code >= 300` branch

The first check in `Transport::read` handles errors reported by the *transport
implementation itself* rather than by a frame. It exists for HTTP transport, where the
error is an HTTP status line rather than a payload
(`td/mtproto/HttpTransport.cpp` sets `error_code`). For plain TCP you can pass `0`.

---

## 4. Packet-size limits

`td/mtproto/RawConnection.cpp:165-172`:

```cpp
constexpr size_t MAX_PACKET_SIZE = (1 << 22) + 1024;   // 4 MiB + 1 KiB
if (wait_size > MAX_PACKET_SIZE) {
  return Status::Error(PSLICE() << "Expected packet size is too big: " << wait_size);
}
```

Enforce this on **read**, before allocating. On **write**, the Intermediate framing already
caps you at 16 MiB (`CHECK(size < (1 << 24))`, `TcpTransport.cpp:53`), but in practice
your own messages should be far smaller — see the container limits in
[5.2](../05-session/02-containers-and-acks.md).

---

## 5. Connection-level failures

Beyond frames, ordinary network things happen. TDLib's policy
(`td/mtproto/RawConnection.cpp`, `td/telegram/net/ConnectionCreator.cpp`):

| Event | TDLib's response |
|-------|------------------|
| TCP connect timeout | Try the next address for this DC |
| Connection closed by peer | Reconnect with backoff; keep the auth key |
| No data for `ping_disconnect_delay` | Assume dead, reconnect |
| Repeated failures on one address | Deprioritize that address, shuffle |

An important invariant: **the auth key survives reconnection.** Losing a TCP connection
does not invalidate anything cryptographic. What you *do* lose is the obfuscation CTR
state (regenerate the header) and, by convention, you should generate a new `session_id`
only if you want to abandon in-flight queries — otherwise keep it and re-send unacknowledged
messages. See [5.5](../05-session/05-reliability.md).

---

## 6. Debugging checklist for transport problems

| Symptom | Likely cause |
|---------|--------------|
| Server closes the connection instantly, no bytes back | Wrong or missing framing magic |
| First reply is 4 bytes `9C FE FF FF` | `-404`: bad `auth_key_id` or wrong DC |
| Garbage after the first packet, with obfuscation | You did not encrypt all 64 header bytes (CTR desync) |
| Lengths look enormous / negative | You read the length as big-endian, or bit 31 (quick-ack) is set |
| Works on 443 but not on 80 | Some middlebox; unrelated to your code |
| `CHECK(size % 4 == 0)` fires | Your padding computation is wrong at the message layer |

---

[← Previous](03-obfuscation.md) · [Index](../README.md) · [Next: Auth key overview →](../03-auth-key/01-overview.md)
