# 10.2 — Implementation Guide: C

[← Previous](01-project-skeleton.md) · [Index](../README.md) · [Next: Rust guide →](03-rust-guide.md)

---

C-specific concerns. The protocol is in chapters 1-8; this chapter is about the pitfalls
that C in particular will hand you.

---

## 1. Toolchain

```
cc -std=c11 -Wall -Wextra -Wconversion -O2 \
   $(pkg-config --cflags openssl zlib) \
   *.c -o tgclient \
   $(pkg-config --libs openssl zlib)
```

`-Wconversion` is worth the noise here: most MTProto bugs in C are silent integer
conversions.

For development, add:

```
-fsanitize=address,undefined -g
```

ASan and UBSan will catch buffer overruns in the framing code and misaligned reads in the
parser — the two most likely failure modes.

---

## 2. Endianness

**Everything in MTProto is little-endian**, except big integers (`p`, `q`, `g_a`, `g_b`,
`A`, `S`), which are **big-endian**.

Do not `memcpy` a struct and hope. Write explicit accessors:

```c
static void put_u32le(uint8_t *p, uint32_t v) {
    p[0] = (uint8_t)(v);        p[1] = (uint8_t)(v >> 8);
    p[2] = (uint8_t)(v >> 16);  p[3] = (uint8_t)(v >> 24);
}
static uint32_t get_u32le(const uint8_t *p) {
    return (uint32_t)p[0] | ((uint32_t)p[1] << 8)
         | ((uint32_t)p[2] << 16) | ((uint32_t)p[3] << 24);
}
```

These are correct on both endiannesses and compile to a single instruction on x86 and ARM.
`*(uint32_t *)p` is **undefined behaviour** for unaligned `p` and wrong on big-endian
machines.

---

## 3. Alignment

Message bodies inside a container are not necessarily 4-byte-aligned relative to your
buffer's start. Always go through byte accessors. This is exactly what
`-fsanitize=undefined` will catch if you do not.

---

## 4. Buffers and bounds

Every length in an incoming packet is attacker-controlled. Before every read:

```c
typedef struct { const uint8_t *p; size_t len, pos; int err; } parser_t;

static int need(parser_t *ps, size_t n) {
    if (ps->err || ps->pos + n > ps->len) { ps->err = 1; return 0; }
    return 1;
}
static uint32_t fetch_u32(parser_t *ps) {
    if (!need(ps, 4)) return 0;
    uint32_t v = get_u32le(ps->p + ps->pos);
    ps->pos += 4;
    return v;
}
```

Note `ps->pos + n > ps->len` can overflow if `n` is huge. Prefer
`n > ps->len - ps->pos` (safe because `pos <= len` is an invariant), or check `n` against
`MAX_PACKET_SIZE` first.

Check the "sticky error" flag once at the end rather than after every field — that is what
TDLib's `TlParser` does (`tdutils/td/utils/tl_parsers.h`).

---

## 5. Integer types

| Wire | Use |
|------|-----|
| `int` | `int32_t` |
| `long` | `int64_t` |
| `double` | `double` (IEEE 754, LE) |
| Lengths | `size_t`, checked before narrowing |
| Constructor ids | `uint32_t` |

`msg_id` is `uint64_t` — TDLib treats it as unsigned throughout
(`td/mtproto/AuthData.cpp:107-125`). Comparing `msg_id`s as signed will misbehave once the
top bit is set.

Constructor ids are naturally `uint32_t`. Writing them as `int32_t` means constants like
`0xfef48f62` need casts and produce `-Wconversion` warnings; use `uint32_t`.

---

## 6. Secrets in memory

```c
static void secure_zero(void *p, size_t n) {
#if defined(__STDC_LIB_EXT1__)
    memset_s(p, n, 0, n);
#elif defined(__OpenBSD__)
    explicit_bzero(p, n);
#else
    volatile unsigned char *v = p;
    while (n--) *v++ = 0;
#endif
}
```

A plain `memset` before `free` is routinely optimized away — the compiler can see the
memory is dead. Use `OPENSSL_cleanse` if you are already linking OpenSSL; it is the
simplest correct option.

Zero: `auth_key`, `new_nonce`, DH exponents `b` and `a`, the password, and every SRP
intermediate.

---

## 7. Random numbers

```c
#include <openssl/rand.h>
if (RAND_bytes(buf, (int)len) != 1) { /* FATAL — do not continue */ }
```

or, without OpenSSL:

```c
#include <sys/random.h>
if (getrandom(buf, len, 0) != (ssize_t)len) { /* FATAL */ }
```

> **⚠ `rand()`, `random()`, and `srand(time(NULL))` are not acceptable** for `nonce`,
> `new_nonce`, `b`, `a`, `session_id`, or `random_id`. A predictable `new_nonce` destroys
> the handshake's security entirely.
>
> A failed CSPRNG call must be **fatal**. Continuing with a partially filled buffer is
> worse than crashing.

---

## 8. Big integers with OpenSSL

```c
BN_CTX *ctx = BN_CTX_new();
BIGNUM *g = BN_new(), *b = BN_new(), *p = BN_new(), *gb = BN_new();

BN_bin2bn(p_bytes, 256, p);          /* big-endian in  */
BN_set_word(g, 3);
BN_bin2bn(b_bytes, 256, b);

BN_mod_exp(gb, g, b, p, ctx);        /* g_b = g^b mod p */

unsigned char out[256];
BN_bn2binpad(gb, out, 256);          /* big-endian out, left-zero-padded */
```

`BN_bn2binpad` is the important one: it produces **exactly** 256 bytes with leading zeros,
which is what `to_binary(256)` does in TDLib (`DhHandshake.cpp:207`,
`PasswordManager.cpp:107`). `BN_bn2bin` writes the *minimal* representation and will
silently give you 255 bytes about once in 256 — a bug that reproduces rarely and is
miserable to find.

Conversely, `p` and `q` in `p_q_inner_data` are the **minimal** big-endian form with no
leading zeros (`crypto.cpp:158-170`).

Prefer `BN_mod_exp_mont_consttime` for secret exponents if you want constant-time
behaviour.

---

## 9. AES-IGE

Not in OpenSSL's API. Implement it directly over the block cipher
([4.3](../04-encrypted-messages/03-aes-ige.md)):

```c
/* iv is 32 bytes: iv[0..16) = encrypted_iv, iv[16..32) = plaintext_iv */
void aes_ige_encrypt(const uint8_t key[32], uint8_t iv[32],
                     const uint8_t *in, uint8_t *out, size_t len)
{
    AES_KEY k;
    AES_set_encrypt_key(key, 256, &k);
    uint8_t *c_prev = iv;        /* previous ciphertext block */
    uint8_t *p_prev = iv + 16;   /* previous plaintext  block */
    uint8_t tmp[16];
    for (size_t i = 0; i < len; i += 16) {
        for (int j = 0; j < 16; j++) tmp[j] = in[i + j] ^ c_prev[j];
        AES_encrypt(tmp, out + i, &k);
        for (int j = 0; j < 16; j++) out[i + j] ^= p_prev[j];
        c_prev = out + i;
        p_prev = (uint8_t *)in + i;
    }
}
```

`len` must be a multiple of 16 — assert it. Decryption swaps the roles and uses
`AES_decrypt` with an **ECB-initialized** key
(`tdutils/td/utils/crypto.cpp:472-476`).

Watch the in-place case: if `in == out`, `p_prev` points into memory you have already
overwritten. Either forbid in-place operation or save the plaintext block first.

---

## 10. Sockets

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
int one = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &one, sizeof one);
```

`TCP_NODELAY` matters: without it, Nagle's algorithm delays your small handshake packets by
up to 40 ms each.

Handle **short reads and short writes**. `read()` returning fewer bytes than you asked for
is normal, not an error. Loop until you have the full frame:

```c
static int read_full(int fd, uint8_t *buf, size_t n) {
    size_t got = 0;
    while (got < n) {
        ssize_t r = read(fd, buf + got, n - got);
        if (r == 0) return -1;                       /* peer closed */
        if (r < 0) { if (errno == EINTR) continue; return -1; }
        got += (size_t)r;
    }
    return 0;
}
```

Also ignore `SIGPIPE` (`signal(SIGPIPE, SIG_IGN)`) or use `MSG_NOSIGNAL`, or a write to a
closed socket will kill your process.

---

## 11. Checklist

- [ ] Built with `-Wall -Wextra -Wconversion` and, in development, ASan + UBSan
- [ ] All multi-byte reads and writes via explicit byte accessors
- [ ] No unaligned pointer casts
- [ ] Every parser read bounds-checked, with no length arithmetic overflow
- [ ] `msg_id` handled as `uint64_t`
- [ ] `RAND_bytes`/`getrandom` for all secrets; failure is fatal
- [ ] `BN_bn2binpad` for fixed-width big-integer output
- [ ] Minimal big-endian form for `p` and `q`
- [ ] AES-IGE asserts a 16-byte multiple; in-place aliasing handled
- [ ] Secrets zeroed with `OPENSSL_cleanse` or equivalent
- [ ] `TCP_NODELAY` set; short reads/writes looped; `SIGPIPE` ignored

---

[← Previous](01-project-skeleton.md) · [Index](../README.md) · [Next: Rust guide →](03-rust-guide.md)
