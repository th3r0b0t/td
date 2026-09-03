# 2.2 — TCP Framings

[← Previous](01-datacenters.md) · [Index](../README.md) · [Next: Obfuscation →](03-obfuscation.md)

---

TCP gives you a byte stream; MTProto needs discrete messages. The **transport framing**
bridges the two. You pick one framing at connection time by sending a magic prefix, and
then every packet uses that framing for the life of the connection.

Telegram defines four framings. **This repository implements two of them**
(`td/mtproto/TcpTransport.cpp`): Intermediate and Padded Intermediate. The other two are
documented here for completeness because you will meet them in other implementations and
in the published protocol description.

**Recommendation: implement Intermediate. It is the simplest correct choice.**

---

## 1. Summary table

| Framing | Init magic | Per-packet header | Length unit | Implemented in TDLib? |
|---------|-----------|-------------------|-------------|------------------------|
| Abridged | `0xEF` (1 byte) | 1 or 4 bytes | 4-byte words | ❌ (only excluded as a forbidden obfuscation prefix) |
| **Intermediate** | `EE EE EE EE` | 4 bytes | **bytes** | ✅ `TcpTransport.cpp:75-78` |
| **Padded Intermediate** | `DD DD DD DD` | 4 bytes + 0–15 trailing random | bytes | ✅ same class, `with_padding_ = true` |
| Full | *(none)* | 8 bytes + 4-byte CRC32 | bytes | ❌ |

---

## 2. Intermediate — the one you should implement

### Choosing it

Immediately after `connect()`, send exactly four bytes:

```
EE EE EE EE
```

(`td/mtproto/TcpTransport.cpp:75-78`)

```cpp
void IntermediateTransport::init_output_stream(ChainBufferWriter *stream) {
  const uint32 magic = with_padding() ? 0xdddddddd : 0xeeeeeeee;
  stream->append(Slice(reinterpret_cast<const char *>(&magic), 4));
}
```

Since all four bytes are identical, endianness is irrelevant. The server does not reply to
this; just start sending packets.

### Sending a packet

```
┌────────────────────┬──────────────────────────┐
│ length (u32 LE)    │ payload                  │
│ 4 bytes            │ `length` bytes           │
└────────────────────┴──────────────────────────┘
```

`length` is the payload size **in bytes**, not words, and does **not** include itself.

Constraints enforced by `IntermediateTransport::write_prepare_inplace`
(`TcpTransport.cpp:50-73`):

```cpp
CHECK(size % 4 == 0);       // payload must be a multiple of 4 bytes
CHECK(size < (1 << 24));    // payload < 16 MiB
```

The 4-byte-alignment requirement is automatically satisfied: an MTProto encrypted message
is always `8 + 16 + 16k` bytes, and an unencrypted one is padded to a multiple of 16
(see [3.2](../03-auth-key/02-step1-req-pq.md)).

### Receiving a packet

`IntermediateTransport::read_from_stream` (`TcpTransport.cpp:20-48`):

```cpp
size_t header_size = 4;
if (stream->size() < header_size) return header_size;      // need more bytes
uint32 data_size;
it.advance(header_size, MutableSlice(&data_size, sizeof(data_size)));
if (data_size & (1u << 31)) {                              // quick-ack, not a length
  if (quick_ack) *quick_ack = data_size;
  stream->advance(header_size);
  return 0;
}
size_t total_size = data_size + header_size;
if (stream_size < total_size) return total_size;           // need more bytes
stream->advance(header_size);
*message = stream->cut_head(data_size).move_as_buffer_slice();
```

Two things to take from this:

1. **Bit 31 of the length field is a flag, not part of the length.** If it is set, the
   whole 4-byte value is a *quick-ack token* and there is no payload following. See
   [5.5](../05-session/05-reliability.md). If you never request quick acks you will never
   receive one, but you should still handle the case defensively.
2. **Cap the length before allocating.** TDLib does this one level up in
   `td/mtproto/RawConnection.cpp:169`:
   ```cpp
   constexpr size_t MAX_PACKET_SIZE = (1 << 22) + 1024;   // 4 MiB + 1 KiB
   if (wait_size > MAX_PACKET_SIZE) return Status::Error("Expected packet size is too big");
   ```

> **⚠ Security note.** Never `malloc(length)` on an unvalidated 32-bit length read from a
> socket. Reject anything above ~4 MiB and close the connection.

### Reference reader loop

```
buffer = []
loop:
    buffer += socket.read()
    while buffer.len() >= 4:
        n = u32_le(buffer[0..4])
        if n & 0x8000_0000:
            handle_quick_ack(n)
            buffer = buffer[4..]
            continue
        if n > MAX_PACKET_SIZE: fail("packet too large")
        if buffer.len() < 4 + n: break        // wait for more
        deliver(buffer[4 .. 4+n])
        buffer = buffer[4+n ..]
```

---

## 3. Padded Intermediate

Identical to Intermediate except:

* the init magic is `DD DD DD DD`;
* 0–15 random bytes are appended to every packet;
* the length field **includes** those padding bytes.

`TcpTransport.cpp:63-72`:

```cpp
size_t append_size = 0;
if (with_padding()) {
  append_size = Random::secure_uint32() % 16;
  …Random::secure_bytes(append);…
}
as<uint32>(message->as_mutable_slice().begin()) = static_cast<uint32>(size + append_size);
```

**Why it exists:** with plain Intermediate, MTProto packet lengths are always multiples of
4 and follow a recognizable distribution, which is a traffic-analysis fingerprint. Random
padding breaks the multiple-of-4 property and blurs the length distribution.

**Consequence for the receiver:** the payload you get is no longer necessarily a multiple
of 16, and it contains trailing garbage. That is fine — the MTProto layer knows the real
message length from `message_data_length` inside the decrypted payload and ignores the
tail. TDLib truncates to a 16-byte multiple before decrypting
(`td/mtproto/Transport.cpp:225`):

```cpp
to_decrypt.remove_suffix(to_decrypt.size() & 15);
```

TDLib selects this framing only when an MTProxy secret requests it
(`td/mtproto/ProxySecret.h:44-46`). Without a proxy, use plain Intermediate.

---

## 4. Abridged (reference only)

Not implemented in this repository. Included because it is the most compact framing and
you will see it elsewhere.

* Init: a **single byte** `0xEF`.
* Per packet:
  * if `payload_len / 4 < 0x7F`: one byte containing `payload_len / 4`;
  * else: byte `0x7F` followed by a 3-byte little-endian word count.
* Payload length is always a multiple of 4, so it is expressed in 4-byte words.

The only trace of it in TDLib is the rule that an obfuscation header must not begin with
`0xEF` (`TcpTransport.cpp:95`), precisely so an obfuscated stream is never mistaken for an
abridged one.

---

## 5. Full (reference only)

Not implemented in this repository. It is the original framing and adds a transport-level
sequence number and checksum:

```
┌───────────────┬────────────────┬───────────┬───────────────┐
│ length (u32)  │ seq_no (u32)   │ payload   │ CRC32 (u32)   │
└───────────────┴────────────────┴───────────┴───────────────┘
```

* `length` counts the **entire frame** including itself and the CRC.
* `seq_no` starts at 0 and increments per frame, separately in each direction. This is a
  *transport* counter and has nothing to do with MTProto's message `seq_no`.
* `CRC32` covers `length ‖ seq_no ‖ payload`.

There is no init magic — Full is what you get if you send no magic at all. That makes it
easy to detect, which is why it has fallen out of use.

> **💡 Implementation note.** Do not confuse this `seq_no` with the message `seq_no` from
> [chapter 5.1](../05-session/01-msg-id-and-seq-no.md). They are unrelated. TDLib has no
> transport `seq_no` at all, and the only CRC32 in the whole repository is in
> `td/generate/tl-parser/crc32.c`, used to compute TL constructor ids.

---

## 6. Which to choose — decision guide

| Situation | Framing |
|-----------|---------|
| You are writing your first implementation | **Intermediate** |
| You need minimum bytes on the wire | Abridged |
| You are connecting through an MTProxy with a secret | Padded Intermediate (usually forced by the secret) |
| Your network censors by packet-length fingerprint | Padded Intermediate + [obfuscation](03-obfuscation.md) |
| A `dcOption` has the `tcpo_only` flag | You **must** use obfuscation |
| Never | Full |

---

## 7. Complete transport-layer pseudocode

```
struct Transport {
    socket: TcpStream,
    inbuf: Vec<u8>,
}

fn connect(addr) -> Transport {
    let s = tcp_connect(addr);
    s.write_all(&[0xEE, 0xEE, 0xEE, 0xEE]);      // Intermediate
    Transport { socket: s, inbuf: vec![] }
}

fn send_packet(t, payload: &[u8]) {
    assert!(payload.len() % 4 == 0);
    assert!(payload.len() < 1 << 24);
    let mut frame = Vec::with_capacity(4 + payload.len());
    frame.extend((payload.len() as u32).to_le_bytes());
    frame.extend(payload);
    t.socket.write_all(&frame);
}

fn recv_packet(t) -> Option<Vec<u8>> {
    loop {
        if t.inbuf.len() >= 4 {
            let n = u32::from_le_bytes(t.inbuf[0..4]);
            if n & 0x8000_0000 != 0 {
                on_quick_ack(n);
                t.inbuf.drain(0..4);
                continue;
            }
            if n as usize > MAX_PACKET_SIZE { return Err("packet too large"); }
            if t.inbuf.len() >= 4 + n as usize {
                let msg = t.inbuf[4 .. 4 + n as usize].to_vec();
                t.inbuf.drain(0 .. 4 + n as usize);
                return Some(msg);
            }
        }
        let chunk = t.socket.read()?;
        if chunk.is_empty() { return None; }      // connection closed
        t.inbuf.extend(chunk);
    }
}
```

If you add obfuscation ([2.3](03-obfuscation.md)), it slots in as a byte-stream filter
below this: encrypt everything written, decrypt everything read, and prepend the 64-byte
header instead of the 4-byte magic.

---

[← Previous](01-datacenters.md) · [Index](../README.md) · [Next: Obfuscation →](03-obfuscation.md)
