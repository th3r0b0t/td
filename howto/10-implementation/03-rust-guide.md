# 10.3 — Implementation Guide: Rust

[← Previous](02-c-guide.md) · [Index](../README.md) · [Next: Testing and debugging →](04-testing-and-debugging.md)

---

Rust-specific concerns. The protocol is in chapters 1-8.

---

## 1. Crates

| Need | Crate | Notes |
|------|-------|-------|
| Big integers | `num-bigint` + `num-traits` | Pure Rust; `rug` (GMP) is faster |
| AES | `aes` | Block cipher only — you build IGE on top |
| AES-CTR | `ctr` | For obfuscation |
| SHA-1 | `sha1` | |
| SHA-256/512 | `sha2` | |
| HMAC | `hmac` | For PBKDF2 |
| PBKDF2 | `pbkdf2` | Configure with `Hmac<Sha512>` |
| RSA | `rsa`, or `num-bigint` directly | See §5 |
| CSPRNG | `rand` (`OsRng`) or `getrandom` | |
| gzip | `flate2` | |
| Zeroing | `zeroize` | |

The RustCrypto crates (`aes`, `sha1`, `sha2`, `hmac`, `pbkdf2`, `ctr`) share a consistent
trait interface and are the natural default.

Pin exact versions and check them against the advisory database before adding.

---

## 2. Types

```rust
type MsgId  = u64;      // unsigned — see td/mtproto/AuthData.cpp:107-125
type SeqNo  = u32;
type CtorId = u32;      // constructor ids are unsigned 32-bit

const MTPROTO_LAYER: i32 = 229;   // td/telegram/Version.h:13
```

`msg_id` **must** be `u64`. Signed comparison breaks once the top bit is set, which happens
for real timestamps.

Use newtypes for the things that are easy to swap by accident:

```rust
struct AuthKeyId(u64);
struct SessionId(u64);
struct RandomId(i64);
```

The compiler will then catch you passing a `session_id` where an `auth_key_id` belongs —
a bug that otherwise surfaces as an inexplicable `-404`.

---

## 3. Serialization

`byteorder`, or plain `to_le_bytes`:

```rust
fn write_i32(buf: &mut Vec<u8>, v: i32)  { buf.extend_from_slice(&v.to_le_bytes()); }
fn write_i64(buf: &mut Vec<u8>, v: i64)  { buf.extend_from_slice(&v.to_le_bytes()); }

fn write_bytes(buf: &mut Vec<u8>, b: &[u8]) {
    let start = buf.len();
    if b.len() < 254 {
        buf.push(b.len() as u8);
    } else {
        assert!(b.len() < (1 << 24));
        buf.push(0xFE);
        buf.extend_from_slice(&(b.len() as u32).to_le_bytes()[..3]);
    }
    buf.extend_from_slice(b);
    while (buf.len() - start) % 4 != 0 { buf.push(0); }
}
```

The padding is relative to the **start of this field**, not the start of the buffer — see
`tdutils/td/utils/tl_storers.h:52-88` and [1.2 §3](../01-serialization/02-wire-encoding.md).
Writing `while buf.len() % 4 != 0` happens to work only when the field starts at a
multiple of 4, which is true in practice but not something to rely on.

Reading:

```rust
struct Reader<'a> { buf: &'a [u8], pos: usize }

impl<'a> Reader<'a> {
    fn take(&mut self, n: usize) -> Result<&'a [u8], Error> {
        let end = self.pos.checked_add(n).ok_or(Error::Overflow)?;
        let s = self.buf.get(self.pos..end).ok_or(Error::Truncated)?;
        self.pos = end;
        Ok(s)
    }
    fn i32(&mut self) -> Result<i32, Error> {
        Ok(i32::from_le_bytes(self.take(4)?.try_into().unwrap()))
    }
}
```

`get(..)` returns `Option`, so truncated input is a clean error rather than a panic. Never
index with `[a..b]` on untrusted input.

---

## 4. AES-IGE

```rust
use aes::Aes256;
use aes::cipher::{BlockEncrypt, BlockDecrypt, KeyInit, generic_array::GenericArray};

pub fn ige_encrypt(key: &[u8; 32], iv: &[u8; 32], data: &mut [u8]) {
    assert_eq!(data.len() % 16, 0);
    let cipher = Aes256::new(GenericArray::from_slice(key));
    let mut c_prev: [u8; 16] = iv[0..16].try_into().unwrap();   // encrypted_iv
    let mut p_prev: [u8; 16] = iv[16..32].try_into().unwrap();  // plaintext_iv
    for block in data.chunks_exact_mut(16) {
        let plain: [u8; 16] = block.try_into().unwrap();
        let mut b = [0u8; 16];
        for i in 0..16 { b[i] = plain[i] ^ c_prev[i]; }
        let mut ga = GenericArray::clone_from_slice(&b);
        cipher.encrypt_block(&mut ga);
        for i in 0..16 { block[i] = ga[i] ^ p_prev[i]; }
        c_prev = block.try_into().unwrap();
        p_prev = plain;
    }
}
```

`chunks_exact_mut(16)` makes the block-size requirement structural. Capturing `plain`
before overwriting `block` handles the in-place aliasing that bites C implementations.

The 32-byte IV split matches `tdutils/td/utils/crypto.cpp:478-479`.

---

## 5. Big integers and RSA

```rust
use num_bigint::BigUint;

// Fixed-width big-endian, left-zero-padded — equivalent to to_binary(256)
fn to_binary(n: &BigUint, len: usize) -> Vec<u8> {
    let mut v = n.to_bytes_be();
    assert!(v.len() <= len);
    let mut out = vec![0u8; len - v.len()];
    out.append(&mut v);
    out
}

let g_b = g.modpow(&b, &p);
let g_b_bytes = to_binary(&g_b, 256);       // exactly 256 bytes
```

`BigUint::to_bytes_be` gives the **minimal** representation. Use it directly for `p` and
`q` in `p_q_inner_data` (`crypto.cpp:158-170` emits minimal form), and wrap it in
`to_binary(256)` everywhere a fixed width is required: `g_a`, `g_b`, `auth_key`, `A`, `S`,
`g_pad`.

RSA in the handshake is **raw** `x^e mod n` with no PKCS padding — MTProto's `RSA_PAD` does
its own ([3.2](../03-auth-key/02-step1-req-pq.md)):

```rust
let x = BigUint::from_bytes_be(&padded_256_bytes);
if x >= n { return Err(Retry); }             // Handshake.cpp:149-152
let encrypted = to_binary(&x.modpow(&e, &n), 256);
```

The `x >= n` check is required — TDLib retries the whole `RSA_PAD` loop when it fails.

> **⚠ `num-bigint`'s `modpow` is not constant-time.** For a client this is a low risk (the
> attacker would need local timing access), but be aware. `rug` with GMP's
> `powm_sec` is the alternative.

---

## 6. Zeroing

```rust
use zeroize::{Zeroize, ZeroizeOnDrop};

#[derive(ZeroizeOnDrop)]
struct AuthKey([u8; 256]);
```

`zeroize` uses volatile writes and compiler fences, so it is not optimized away. Apply it
to `auth_key`, `new_nonce`, DH exponents, the password, and SRP intermediates.

Note that `Vec<u8>` reallocation leaves copies behind. For long-lived secrets, use a
fixed-size array from the start.

---

## 7. Async or not

Both work. Sync is simpler and entirely adequate:

```rust
let mut stream = TcpStream::connect(addr)?;
stream.set_nodelay(true)?;
stream.write_all(&frame)?;
stream.read_exact(&mut header)?;
```

`read_exact` handles short reads for you — one of the more common C bugs is simply absent.

With `tokio`, the same code with `.await`. Choose async only if you need concurrent
connections or want to integrate with an existing async application.

---

## 8. Error handling

```rust
#[derive(Debug, thiserror::Error)]
enum Error {
    #[error("io: {0}")]              Io(#[from] std::io::Error),
    #[error("truncated packet")]     Truncated,
    #[error("length overflow")]      Overflow,
    #[error("unexpected constructor 0x{0:08x}")] BadCtor(u32),
    #[error("transport error {0}")]  Transport(i32),
    #[error("rpc {code}: {message}")] Rpc { code: i32, message: String },
    #[error("bad msg notification {0}")] BadMsg(i32),
}
```

Keeping `Transport`, `BadMsg`, and `Rpc` as distinct variants forces the caller to handle
them differently, which is exactly what
[10.1 §7](01-project-skeleton.md) argues for.

Add a helper for the one error that must never be swallowed:

```rust
impl Error {
    fn flood_wait_secs(&self) -> Option<u32> {
        match self {
            Error::Rpc { code: 420, message } => message
                .strip_prefix("FLOOD_WAIT_")
                .and_then(|s| s.parse().ok())
                .map(|s: u32| s.clamp(1, 14 * 24 * 60 * 60)),
            _ => None,
        }
    }
}
```

The clamp matches `td/telegram/net/NetQueryDelayer.cpp:35-71`.

---

## 9. Avoid `unsafe`

Nothing in this guide requires it. If you find yourself reaching for
`std::mem::transmute` to parse a packet, use the byte-wise reader from §3 instead — it is
both safe and correct on every architecture.

---

## 10. Checklist

- [ ] `msg_id` is `u64`; constructor ids are `u32`
- [ ] Newtypes for `AuthKeyId`, `SessionId`, `RandomId`
- [ ] All parser reads via `get(..)`/`checked_add`, never `[a..b]` on untrusted input
- [ ] String padding computed from the field start
- [ ] AES-IGE uses `chunks_exact_mut(16)` and captures the plaintext before overwriting
- [ ] `to_binary(n, 256)` helper used for every fixed-width big integer
- [ ] Minimal big-endian form for `p` and `q`
- [ ] `x >= n` checked before raw RSA, with a retry
- [ ] `zeroize` on all secret types
- [ ] Distinct error variants for transport, protocol, and RPC failures
- [ ] `FLOOD_WAIT_X` parsed and clamped
- [ ] No `unsafe`

---

[← Previous](02-c-guide.md) · [Index](../README.md) · [Next: Testing and debugging →](04-testing-and-debugging.md)
