# 3.1 — Auth Key Generation: Overview

[← Previous](../02-transport/04-transport-errors.md) · [Index](../README.md) · [Next: Step 1, req_pq →](02-step1-req-pq.md)

---

## 1. The goal

At the end of this chapter you will possess a **256-byte shared secret** — the *auth key* —
known only to you and one Telegram datacenter, plus its 8-byte `auth_key_id` and an initial
`server_salt`. Everything after this point is encrypted with it.

The auth key is established by an authenticated Diffie–Hellman exchange over three
round trips, all sent as **unencrypted MTProto messages** (`auth_key_id = 0`).

---

## 2. The three round trips at a glance

```
CLIENT                                                         SERVER
  │
  │  ── req_pq_multi(nonce) ────────────────────────────────────▶
  │                                                              │
  │  ◀── resPQ(nonce, server_nonce, pq, fingerprints) ───────────
  │
  │     factorize pq = p·q ; pick RSA key ; generate new_nonce
  │
  │  ── req_DH_params(nonce, server_nonce, p, q,                 │
  │        fingerprint, RSA(p_q_inner_data_dc)) ─────────────────▶
  │                                                              │
  │  ◀── server_DH_params_ok(nonce, server_nonce,                │
  │         encrypted_answer{ g, p, g_a, server_time }) ─────────
  │
  │     verify DH parameters ; pick b ; g_b = g^b mod p
  │
  │  ── set_client_DH_params(nonce, server_nonce,                │
  │        encrypted{ g_b }) ────────────────────────────────────▶
  │                                                              │
  │  ◀── dh_gen_ok(nonce, server_nonce, new_nonce_hash1) ────────
  │
  │     auth_key = g_a^b mod p
  │     auth_key_id = SHA1(auth_key)[12..20)
  │     server_salt = new_nonce[0..8) XOR server_nonce[0..8)
  ▼
```

The state machine is `AuthKeyHandshake` in `td/mtproto/Handshake.cpp`; its states are
`Start → ResPQ → ServerDHParams → DHGenResponse → Finish`
(`td/mtproto/Handshake.h:70-76`).

---

## 3. Why it is shaped like this

Each element of the design answers a specific attack. Understanding this makes the checks
in [3.5](05-security-checks.md) feel necessary rather than arbitrary.

| Element | Purpose |
|---------|---------|
| `nonce` (client, 128-bit) | Ties every server reply to *this* handshake attempt |
| `server_nonce` (128-bit) | Ties client messages to *this* server session; contributes to the temporary AES key |
| `pq` factorization | A small proof-of-work; makes handshake flooding expensive for the client, cheap to verify for the server |
| RSA encryption of the inner data | Only the real Telegram server (holder of the private key) can read `new_nonce`, so only it can derive the temporary AES key |
| `new_nonce` (256-bit) | The client's secret contribution; the temporary AES key that protects the DH exchange is derived from it |
| DH over a 2048-bit safe prime | Produces the actual long-term shared secret with forward secrecy against later RSA key compromise |
| `new_nonce_hash1` | Proves the server derived the same auth key you did |
| Explicit `g`, `p` validation | Prevents a malicious server from choosing weak parameters that let it compute your key |

> **⚠ Security note.** Skipping any of the validation steps in
> [3.5](05-security-checks.md) turns this from an authenticated exchange into an
> unauthenticated one. The most consequential omissions are: not verifying that `p` is a
> safe prime, not verifying the range of `g_a`, and not verifying `new_nonce_hash1`.

---

## 4. Permanent vs temporary keys

There are two flavours, selected by which inner-data constructor you send:

| | Permanent | Temporary (PFS) |
|---|---|---|
| Inner data | `p_q_inner_data_dc#a9f55f95` | `p_q_inner_data_temp_dc#56fddf88` |
| Extra field | — | `expires_in:int` |
| Lifetime | Until explicitly destroyed | `expires_in` seconds |
| Server stores it | Yes, tied to your authorization | Yes, but discards it on expiry |
| Requires binding | No | Yes — `auth.bindTempAuthKey` |

TDLib requests temporary keys with a validity of `Random::fast(23h, 24h)`
(`td/telegram/net/Session.cpp`), then binds them to the permanent key.

**For this guide, use a permanent key** (`p_q_inner_data_dc`). Perfect forward secrecy is
covered in [9.1](../09-advanced/01-perfect-forward-secrecy.md) once everything else works.

---

## 5. Message format during the handshake

All three requests and all three replies are **unencrypted MTProto messages**
(`td/mtproto/Transport.cpp:36-47`, `write_no_crypto`):

```
┌────────────────────┬────────────────┬────────────────────────┬──────────┬─────────┐
│ auth_key_id = 0    │ msg_id         │ message_data_length    │ data     │ padding │
│ 8 bytes (all zero) │ 8 bytes        │ 4 bytes                │ N bytes  │ 0..?    │
└────────────────────┴────────────────┴────────────────────────┴──────────┴─────────┘
```

* `auth_key_id` is 8 zero bytes — that is how the server knows this is plaintext.
* `msg_id` follows the normal rules from [5.1](../05-session/01-msg-id-and-seq-no.md):
  unix time × 2³², divisible by 4, strictly increasing.
* `message_data_length` is the byte length of `data`, **excluding** padding.
* There is **no** `session_id`, **no** `salt`, and **no** `seq_no`.

Padding, from `NoCryptoStorer` (`td/mtproto/NoCryptoStorer.h:20-25`):

```cpp
size_t pad_size = (-static_cast<int32>(data_size)) & 15;
pad_size += 16 * (Random::secure_uint32() % 16);
```

That is: pad to a multiple of 16, then add a random multiple of 16 in `[0, 240]`. TDLib
adds this padding only when `packet_info->is_creator` is false or for specific handshake
variants; padding to a 16-byte multiple is the part that matters, since the Intermediate
framing requires a multiple of 4 and the server expects 16-alignment here.

The reply has the same shape. Parse it by:

1. checking the first 8 bytes are zero,
2. reading `msg_id` (you may sanity-check it, but it is not used further),
3. reading `message_data_length`,
4. taking exactly that many bytes as the TL-serialized reply,
5. ignoring the rest.

---

## 6. Timeouts

TDLib enforces staged deadlines against a total handshake timeout `timeout_in_`
(`Handshake.cpp:87, 164, 257`):

| Stage | Deadline |
|-------|----------|
| `resPQ` received | 60 % of the total timeout |
| `server_DH_params_ok` received | 80 % |
| `dh_gen_ok` received | 100 % |

If the deadline passes, abandon and restart from `req_pq_multi` with a fresh `nonce`.
A total of 10 seconds is a reasonable budget on a healthy network; the `pq` factorization
and the two modular exponentiations are the slow parts on your side (tens of
milliseconds each with a decent bignum library).

---

## 7. What you need before starting

| Item | Where from |
|------|-----------|
| A TCP connection with framing established | [Chapter 2](../02-transport/02-tcp-framings.md) |
| Telegram's RSA public key(s) | [Appendix B](../appendix/B-datacenters-and-keys.md) |
| A CSPRNG | OS: `getrandom`, `/dev/urandom`, `BCryptGenRandom` |
| SHA-1 and SHA-256 | OpenSSL, or any library |
| AES-256 in IGE mode | You will likely implement IGE yourself over AES-ECB — see [4.3](../04-encrypted-messages/03-aes-ige.md) |
| Arbitrary-precision integers | GMP, OpenSSL `BIGNUM`, Rust `num-bigint`/`rug` |

> **💡 Implementation note.** AES-IGE is not in OpenSSL's EVP interface. TDLib implements
> it manually over the raw block cipher in
> `tdutils/td/utils/crypto.cpp:467-592`. It is about 30 lines. Chapter 4.3 gives the exact
> formulas.

---

## 8. Chapter map

| Section | Contents |
|---------|----------|
| [3.2](02-step1-req-pq.md) | `req_pq_multi` → `resPQ`, RSA fingerprints, factorizing `pq` |
| [3.3](03-step2-req-dh-params.md) | Building `p_q_inner_data_dc`, RSA_PAD, `req_DH_params` → `server_DH_params_ok`, decrypting the answer |
| [3.4](04-step3-set-client-dh-params.md) | Choosing `b`, `set_client_DH_params`, `dh_gen_ok`, computing `auth_key`, `auth_key_id`, `server_salt` |
| [3.5](05-security-checks.md) | Every validation you must perform, and what breaks if you skip it |
| [3.6](06-worked-example.md) | A complete annotated byte trace of one handshake |

---

[← Previous](../02-transport/04-transport-errors.md) · [Index](../README.md) · [Next: Step 1, req_pq →](02-step1-req-pq.md)
