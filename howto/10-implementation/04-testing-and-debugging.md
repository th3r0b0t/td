# 10.4 — Testing and Debugging

[← Previous](03-rust-guide.md) · [Index](../README.md) · [Next: Appendix A →](../appendix/A-constructor-ids.md)

---

## 1. Test on the test datacenters

Test DC addresses are in [Appendix B](../appendix/B-datacenters-and-keys.md), and they use
a **different RSA public key**. Getting `req_DH_params` rejected on a test DC almost always
means you used the production key's fingerprint.

Test accounts:

* Phone numbers of the form `99966XYYYY`, where `X` is the DC number. TDLib's own tests use
  `9996636437` and `9996636438` (`test/tdclient.cpp:676-677`).
* The login code is the DC number repeated to the required length — `22222` on DC 2.
* No real SMS is sent and no rate limits worth worrying about.

Do all early development here.

---

## 2. Test bottom-up

| Layer | Test without a network |
|-------|------------------------|
| TL serialization | Round-trip; compare against [1.3](../01-serialization/03-worked-examples.md) |
| SHA-1/SHA-256 | NIST known-answer vectors |
| AES-IGE | Encrypt then decrypt; the IV split is the usual bug |
| KDF2 | Fixed `auth_key` + `msg_key` → fixed `aes_key`/`aes_iv` |
| Envelope | `decrypt(encrypt(m)) == m`, with X=0 and X=8 |
| Padding | Every size 1…2000 lands on a legal bucket, `12 ≤ pad ≤ 1024` |
| Framing | `size % 4 == 0`, length round-trips |

Loopback-testing the envelope catches most crypto errors before you ever connect.

---

## 3. Failure symptoms

| Symptom | Most likely cause |
|---------|-------------------|
| Connection closed immediately, no data | Framing magic missing or wrong |
| 4-byte reply `9c ff ff ff` (`-100`) | Malformed first packet |
| 4-byte reply `7c fe ff ff` (`-388`) | Bad auth key / transport confusion |
| 4-byte reply `74 fe ff ff` (`-396`) or `-404` | Unknown `auth_key_id` — key revoked or byte order wrong |
| Handshake hangs after `req_pq_multi` | `msg_id` wrong, or `auth_key_id` not zero in plaintext mode |
| `server_DH_params_fail` | Wrong RSA key, wrong fingerprint, or bad `RSA_PAD` |
| SHA-1 mismatch on `server_DH_inner_data` | `tmp_aes_key`/`tmp_aes_iv` derivation wrong |
| `dh_gen_retry`/`dh_gen_fail` | `g_b` or the encrypted `client_DH_inner_data` is malformed |
| Handshake fine, then silence | `msg_id` from local time — see [5.4](../05-session/04-salts-and-time.md) |
| `bad_msg_notification` 18 | `msg_id` not divisible by 4 |
| `bad_msg_notification` 32/34 | `seq_no` not `counter*2+1` for content messages |
| `msg_key` mismatch on decrypt | Wrong X (0 vs 8), or padding not stripped before hashing |
| `400 CONNECTION_NOT_INITED` | `initConnection` missing on this connection |
| `400 PEER_ID_INVALID` | Stale or missing `access_hash` |
| `400 PASSWORD_HASH_INVALID` | SRP — check `x` first, then padding |

---

## 4. Hex dumps

Log every packet in both directions, at four levels:

```
[send] raw framed   : ee ee ee ee | 28 00 00 00 | ...
[send] envelope     : auth_key_id=0000000000000000 msg_id=... len=40
[send] plaintext    : 78 97 46 60 ...
[send] decoded      : req_pq_multi{nonce=...}
```

Being able to see the same bytes at four levels of interpretation is what makes protocol
debugging tractable. TDLib's `format::as_hex_dump<4>` exists for exactly this.

> **⚠ Redact `auth_key`, `new_nonce`, the password, and SRP intermediates from all dumps.**
> A debug log containing an `auth_key` is a full account compromise.

---

## 5. Comparing against TDLib

TDLib itself is the best oracle. Build it and run with verbose MTProto logging:

```
td_set_log_verbosity_level(5)
```

or via `td_api::setLogVerbosityLevel`. The `VLOG(mtproto)` statements in
`td/mtproto/SessionConnection.cpp` print `msg_id`, `seq_no`, and message types for every
packet — line them up against your own trace.

For the handshake specifically, `td/mtproto/Handshake.cpp` logs each step's nonces, so you
can confirm your `new_nonce` handling matches.

---

## 6. Milestones

Work toward these in order. Each proves a specific set of layers.

1. **TCP connects and stays open** → addresses, ports, framing magic.
2. **`resPQ` parses** → framing, plaintext envelope, TL parsing.
3. **`server_DH_params_ok`** → RSA, `RSA_PAD`, fingerprints.
4. **`dh_gen_ok`** → DH, `tmp_KDF`, AES-IGE, SHA-1.
5. **`help.getConfig` returns a `config`** → encrypted envelope, KDF2, `msg_key`, session.
6. **Code arrives on your phone** → API layer, `initConnection`.
7. **`auth.authorization`** → login.
8. **Message in Saved Messages** → done.

Milestone 5 is the big one: reaching it means the entire cryptographic stack is correct.

---

## 7. Checklist

- [ ] Development done against test DCs with test numbers
- [ ] Test-DC RSA key used for test DCs
- [ ] Crypto primitives covered by known-answer tests
- [ ] Envelope loopback-tested for both X=0 and X=8
- [ ] Padding rules property-tested across a size range
- [ ] Four-level hex logging available
- [ ] Secrets redacted from logs
- [ ] Progress tracked against the eight milestones

---

[← Previous](03-rust-guide.md) · [Index](../README.md) · [Next: Appendix A →](../appendix/A-constructor-ids.md)
