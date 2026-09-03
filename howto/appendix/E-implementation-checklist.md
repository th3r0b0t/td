# Appendix E — Implementation Checklist

[← Previous](D-tdlib-source-map.md) · [Index](../README.md)

---

Every checklist item from the guide, in build order. Work top to bottom; each stage
depends on the one above it.

Items marked **⚠** are security-critical. Skipping them does not merely produce a broken
client — it produces a client that an active attacker can compromise.

---

## Stage 0 — Foundations

Before any network code.

- [ ] Modules layered: crypto → TL → transport → MTProto → API
      ([10.1](../10-implementation/01-project-skeleton.md))
- [ ] Built and tested bottom-up
- [ ] Crypto library chosen; nothing hand-rolled except AES-IGE
- [ ] AES-256 ECB/CBC single-block, AES-256-CTR, SHA-1, SHA-256, HMAC-SHA-512, PBKDF2,
      and a big-integer library all available
- [ ] **⚠** CSPRNG wired up (`RAND_bytes`, `getrandom`); RNG failure is **fatal**, never
      silently ignored
- [ ] Client state small and explicit
- [ ] Transport, protocol, and API errors modelled as three distinct types
      ([C](C-error-reference.md))

---

## Stage 1 — Serialization

- [ ] `int32`, `int64`, `double` stored little-endian
      ([1.2](../01-serialization/02-wire-encoding.md))
- [ ] `int128`/`int256` are raw byte blobs, no endianness applied
- [ ] Strings: `<254` → 1 length byte; `<2²⁴` → `0xFE` + 3-byte LE length; else `0xFF` +
      4-byte LE length + 3 zero bytes
- [ ] **String padding computed from the field start**, not from the data start
- [ ] Boxed vectors prefixed with `0x1cb5c415`; bare vectors are not
- [ ] `flags:#` written as `int32`; `flags.N?true` consumes **zero bytes**
- [ ] Unknown flag bits rejected on parse
- [ ] **⚠** Every parser read bounds-checked; length arithmetic cannot overflow
- [ ] Constructor ids computed by CRC-32 of the normalized declaration, or transcribed from
      [Appendix A](A-constructor-ids.md)
- [ ] Round-trip tests: serialize → parse → compare
- [ ] `msg_id` handled as **unsigned** 64-bit throughout
      ([10.2](../10-implementation/02-c-guide.md), [10.3](../10-implementation/03-rust-guide.md))
- [ ] No unaligned pointer casts; all multi-byte access via explicit accessors

---

## Stage 2 — Transport

- [ ] Intermediate framing: `0xeeeeeeee` magic, 4-byte LE length prefix
      ([2.2](../02-transport/02-tcp-framings.md))
- [ ] Padded intermediate: `0xdddddddd`, 0-15 random pad bytes **included** in the length
- [ ] `size % 4 == 0` enforced for plain Intermediate
- [ ] `MAX_PACKET_SIZE` = `(1 << 22) + 1024` enforced **before** allocating
- [ ] Length bit 31 recognized as a quick-ack token, not a length
- [ ] `TCP_NODELAY` set; short reads and writes looped; `SIGPIPE` ignored

### Obfuscation ([2.3](../02-transport/03-obfuscation.md))

- [ ] **⚠** 64 header bytes from a CSPRNG
- [ ] First byte is not `0xEF`
- [ ] First LE `uint32` is none of the seven forbidden values
- [ ] Second LE `uint32` is non-zero
- [ ] Bytes `[56..60)` set to the framing magic
- [ ] Bytes `[60..62)` set to `dc_id` as LE `int16` (proxies only)
- [ ] `send_key = header[8..40)`, `send_iv = header[40..56)`
- [ ] `recv_key = reverse(header)[8..40)`, `recv_iv = reverse(header)[40..56)`
- [ ] Both keys passed through `SHA256(key ‖ secret)` when a proxy secret is present
- [ ] **All 64 bytes** encrypted (to advance the counter); only `[56..64)` of the
      ciphertext transmitted
- [ ] Framing applied *before* encryption on send, *after* decryption on receive
- [ ] CTR counters never reset for the life of the connection

### Connectivity ([2.1](../02-transport/01-datacenters.md), [B](B-datacenters-and-keys.md))

- [ ] Bootstrap addresses from [Appendix B](B-datacenters-and-keys.md)
- [ ] Addresses shuffled; ports tried 443, 80, 5222
- [ ] Connect, handshake, and query timeouts all set
- [ ] Frames under 16 bytes parsed as transport error codes, never decrypted
      ([2.4](../02-transport/04-transport-errors.md))
- [ ] `0` (nop) and `-1` (quick ack) not treated as errors

---

## Stage 3 — Auth key

The single hardest stage. Everything here is security-critical.

### Step 1 — `req_pq_multi` ([3.2](../03-auth-key/02-step1-req-pq.md))

- [ ] **⚠** 16-byte `nonce` from a CSPRNG
- [ ] Sent `0xbe7e8ef1 ‖ nonce` unencrypted with `auth_key_id = 0`
- [ ] Reply constructor id is `0x05162463`
- [ ] **⚠** `resPQ.nonce == nonce` — abort if not
- [ ] `server_nonce` saved
- [ ] **⚠** A fingerprint matches one of your hardcoded RSA keys — abort if not
- [ ] Fingerprints computed from the PEMs at startup, not hardcoded
      ([B.3](B-datacenters-and-keys.md))
- [ ] **Test DCs use the test RSA key**
- [ ] `pq` factorized into `p < q`
- [ ] `p` and `q` encoded as **minimal** big-endian (no leading zeros)
- [ ] **⚠** 32-byte `new_nonce` from a CSPRNG

### Step 2 — `req_DH_params` ([3.3](../03-auth-key/03-step2-req-dh-params.md))

- [ ] `p_q_inner_data_dc` built with `dc` set to the DC you are connected to
- [ ] `data` padded to exactly 192 bytes with random
- [ ] **⚠** Fresh 32-byte `aes_key` per attempt
- [ ] `data_with_hash = data ‖ SHA256(aes_key ‖ data)`, first 192 bytes **reversed**
- [ ] AES-256-IGE with a **32-byte zero IV**, output at `out[32..256)`
- [ ] `out[0..32) = aes_key ⊕ SHA256(out[32..256))`
- [ ] Raw RSA `x^e mod n`
- [ ] **⚠** Retry the **whole** loop if `x ≥ n`
- [ ] `encrypted_data` is exactly 256 bytes
- [ ] **⚠** Both nonces verified in the reply
- [ ] `encrypted_answer.size() % 16 == 0`
- [ ] `tmp_KDF` argument order correct (`new_nonce` first for the key, `server_nonce` first
      for the IV)
- [ ] A copy of `tmp_aes_iv` saved before decrypting
- [ ] Inner constructor id is `0xb5890dba`
- [ ] Leftover padding < 16 bytes
- [ ] **⚠** SHA-1 over the inner data (excluding padding) matches
- [ ] **⚠** Both nonces inside `server_DH_inner_data` verified
- [ ] `server_time_diff` recorded

### DH validation ([3.5](../03-auth-key/05-security-checks.md))

- [ ] **⚠** `p` is exactly 2048 bits
- [ ] **⚠** `g ∈ {2,3,4,5,6,7}` and the corresponding `p mod 4g` condition holds
- [ ] **⚠** `p` is prime
- [ ] **⚠** `(p−1)/2` is prime
- [ ] **⚠** `2¹⁹⁸⁴ ≤ g_a ≤ p − 2¹⁹⁸⁴`
- [ ] **⚠** `2¹⁹⁸⁴ ≤ g_b ≤ p − 2¹⁹⁸⁴`

### Step 3 — `set_client_DH_params` ([3.4](../03-auth-key/04-step3-set-client-dh-params.md))

- [ ] **⚠** `b` is 2048 bits from a CSPRNG
- [ ] `g_b = g^b mod p`, minimal big-endian
- [ ] `client_DH_inner_data` with `retry_id = 0`
- [ ] Encrypted as `SHA1(data) ‖ data ‖ random pad to a multiple of 16`
- [ ] **Fresh** `tmp_aes_key`/`tmp_aes_iv`, not the mutated ones
- [ ] `auth_key = g_a^b mod p` as **exactly 256 bytes, left-zero-padded**
- [ ] `auth_key_id = SHA1(auth_key)[12..20)`
- [ ] `server_salt = new_nonce[0..8) ⊕ server_nonce[0..8)`
- [ ] Reply is `dh_gen_ok` (`0x3bcbf734`), not retry (`0x46dc1fb9`) or fail (`0xa69dae02`)
- [ ] **⚠** Both nonces verified
- [ ] **⚠** `new_nonce_hash1 == SHA1(new_nonce ‖ 0x01 ‖ SHA1(auth_key)[0..8))[4..20)`
- [ ] `auth_key` persisted along with its DC id
- [ ] **⚠** `b`, `new_nonce`, and the temporary keys wiped
- [ ] Handshake bounded by a timeout

---

## Stage 4 — Encrypted messages

### `msg_key` and KDF ([4.2](../04-encrypted-messages/02-msg-key-and-kdf.md))

- [ ] `msg_key_large = SHA256(auth_key[88+X .. 120+X) ‖ plaintext)`
- [ ] `msg_key = msg_key_large[8..24)` — the **middle** 16 bytes
- [ ] The hashed plaintext **includes the padding**
- [ ] `A = SHA256(msg_key ‖ auth_key[X .. X+36))` — `msg_key` **first**
- [ ] `B = SHA256(auth_key[40+X .. 76+X) ‖ msg_key)` — `msg_key` **last**
- [ ] `aes_key = A[0..8) ‖ B[8..24) ‖ A[24..32)`
- [ ] `aes_iv = B[0..8) ‖ A[8..24) ‖ B[24..32)`
- [ ] **`X = 0` for sending, `X = 8` for receiving**
- [ ] `aes_iv` is 32 bytes, not 16
- [ ] **⚠** `msg_key` verified on receive, in constant time

### AES-IGE ([4.3](../04-encrypted-messages/03-aes-ige.md))

- [ ] Key is 32 bytes, IV is **32 bytes**
- [ ] `c₀ = iv[0..16)`, `p₀ = iv[16..32)`
- [ ] Encrypt: `cᵢ = AES_enc(pᵢ ⊕ cᵢ₋₁) ⊕ pᵢ₋₁`
- [ ] Decrypt: `pᵢ = AES_dec(cᵢ ⊕ pᵢ₋₁) ⊕ cᵢ₋₁`
- [ ] Decryption uses the AES **decryption** primitive
- [ ] Input length asserted to be a multiple of 16
- [ ] IV taken by value, or explicitly copied before use
- [ ] Plaintext block captured **before** the buffer is overwritten (in-place aliasing)
- [ ] Round-trip test passes
- [ ] Avalanche test passes (all later blocks garbled)

### Envelope ([4.1](../04-encrypted-messages/01-envelope.md))

- [ ] Layout: `auth_key_id(8) ‖ msg_key(16) ‖ AES-IGE(salt(8) ‖ session_id(8) ‖ msg_id(8)
      ‖ seq_no(4) ‖ length(4) ‖ body ‖ padding)`
- [ ] Padding is 12-1024 bytes, chosen so the encrypted part is a multiple of 16
- [ ] **⚠** On receive: `message_data_length % 4 == 0`
- [ ] **⚠** On receive: the declared length fits inside the decrypted buffer
- [ ] **⚠** On receive: `12 ≤ pad_size ≤ 1024`
- [ ] **⚠** On receive: `auth_key_id` matches your key

---

## Stage 5 — Session

### `msg_id` and `seq_no` ([5.1](../05-session/01-msg-id-and-seq-no.md))

- [ ] `session_id` random, non-zero, stable across reconnects
- [ ] `msg_id` derived from **server** time, not local time
- [ ] `msg_id` low two bits cleared (`& ~3`)
- [ ] `msg_id` strictly increasing; bumped by a multiple of 8 on collision
- [ ] `seq_no` starts at 0; `| 1` then `+= 2` only for content-related messages
- [ ] **⚠** Incoming `session_id` verified
- [ ] **⚠** Incoming `msg_id` verified odd
- [ ] **⚠** Incoming `msg_id` inside the −300 s / +30 s window
- [ ] **⚠** Duplicate incoming `msg_id`s dropped (window of ~1000)
- [ ] Messages with odd incoming `seq_no` acknowledged

### Containers and acks ([5.2](../05-session/02-containers-and-acks.md))

- [ ] Container id `0x73f1f8dc`, then `int32` count, then bare inner messages
- [ ] Inner message = `msg_id(8) ‖ seqno(4) ‖ bytes(4) ‖ body`
- [ ] **No** `0x1cb5c415` vector tag inside the container
- [ ] The container's own `seq_no` is even
- [ ] The container's own `msg_id` is greater than every inner `msg_id`
- [ ] Containers built only when there is more than one message
- [ ] Received containers unpacked and each inner message dispatched
- [ ] **⚠** Inner lengths validated against the remaining buffer
- [ ] **⚠** Nested containers rejected
- [ ] `msgs_ack` sent with an even `seq_no`
- [ ] Acks flushed within 30 s or after 100 pending

### Service messages ([5.3](../05-session/03-service-messages.md))

- [ ] Containers unpacked before dispatch
- [ ] `gzip_packed` handled as a message body **and** inside `rpc_result`
- [ ] **⚠** Decompressed size bounded
- [ ] `rpc_result` matched to its query by `req_msg_id`
- [ ] `rpc_error` inside `rpc_result` detected by peeking at `0x2144ca19`
- [ ] `bad_server_salt` updates the salt **and** resends
- [ ] `bad_msg_notification` 16/20 resend; 17 resynchronizes time
- [ ] `new_session_created` resends everything below `first_msg_id`
- [ ] `msgs_ack` clears the resend queue
- [ ] Unknown constructors treated as updates, not errors

### Salts and time ([5.4](../05-session/04-salts-and-time.md))

- [ ] Initial `server_time_diff` from `server_DH_inner_data.server_time`
- [ ] Updated from the `msg_id` of every incoming message
- [ ] Updates **monotonically increasing** after the first
- [ ] Reset on `bad_msg_notification` code 17
- [ ] Reset when an `rpc_result` predates its query by more than 15 s
- [ ] `msg_id` generated from `local_time + server_time_diff`
- [ ] `server_time_diff` and `server_salt` both persisted
- [ ] Initial salt = `new_nonce[0..8) ⊕ server_nonce[0..8)`

### Reliability ([5.5](../05-session/05-reliability.md))

- [ ] Sent queries retained until `rpc_result` or `msgs_ack`
- [ ] Resends allocate a **new** `msg_id` and `seq_no`
- [ ] Resends reuse the **same** `random_id`
- [ ] Resend on: disconnect, `bad_server_salt`, `bad_msg_notification` 16/20,
      `new_session_created`
- [ ] `session_id` and `seq_no` preserved across reconnects
- [ ] Obfuscation header regenerated on each new connection
- [ ] Unexpected quick acks handled without crashing
- [ ] Reconnect uses exponential backoff with jitter, capped around 60 s
- [ ] Backoff counter reset after a stable connection
- [ ] Optional: periodic `ping_delay_disconnect` as a liveness check

---

## Stage 6 — API layer

### `initConnection` ([6.1](../06-api-layer/01-init-connection.md))

- [ ] `invokeWithLayer#da9b0d0d` outermost, then the layer as `int32`
- [ ] `initConnection#c1cd5ea9` next
- [ ] `flags` bit 1 set if `params` is present, bit 0 if `proxy` is present
- [ ] `api_id` is your own registered id
- [ ] `device_model`, `system_version`, `app_version`, `system_lang_code` non-empty
- [ ] `lang_pack` and `lang_code` present as **empty strings**
- [ ] `params` a valid `JSONValue` (an empty `jsonObject` is fine) or the flag is clear
- [ ] Sent **once per connection**, wrapping the first query
- [ ] `CONNECTION_NOT_INITED` / `CONNECTION_LAYER_INVALID` trigger a resend with the header

### RPC results ([6.2](../06-api-layer/02-rpc-results-and-errors.md))

- [ ] `rpc_result.req_msg_id` matched against a pending-query map
- [ ] `req_msg_id == 0` treated as an error
- [ ] The next `int32` peeked, with a rewind in the default branch
- [ ] `gzip_packed` recognized inside `rpc_result` **and** as a top-level body
- [ ] `FLOOD_WAIT_X` parsed, clamped to 1 s … 14 days, and honoured
- [ ] All five `420` prefixes recognized ([C.4](C-error-reference.md))
- [ ] Non-numeric `420` suffixes rewritten as `400`, not parsed
- [ ] `303` errors routed to the migration handler
- [ ] Unknown constructor ids logged and ignored, never fatal
- [ ] A per-`msg_id` record of what was asked, so the answer can be typed

### Migration ([6.3](../06-api-layer/03-migration-and-multi-dc.md))

- [ ] `303` parsed for all four `*_MIGRATE_N` prefixes
- [ ] **⚠** `N` validated as a plausible DC id before use
- [ ] `PHONE_`/`NETWORK_`/`USER_MIGRATE_` change the **home** DC
- [ ] `FILE_MIGRATE_` redirects **only that query**
- [ ] The handshake can run more than once, with results kept per DC
- [ ] `initConnection` sent on every new connection, including post-migration
- [ ] `help.getConfig` called before authorization to refresh `dc_options`
- [ ] `dcOption` filtering excludes `cdn` and `media_only` for general API calls
- [ ] Per-DC auth key, salt, and session state kept separate
- [ ] `server_time_difference` shared globally

---

## Stage 7 — Login

### Credentials ([7.2](../07-login/02-api-credentials.md))

- [ ] **⚠** Own `api_id`/`api_hash` from <https://my.telegram.org>
- [ ] **⚠** Loaded from the environment or a config file, not hard-coded
- [ ] **⚠** Not committed to version control
- [ ] **⚠** Not written to logs
- [ ] `API_ID_INVALID` and `API_ID_PUBLISHED_FLOOD` produce a clear message

### Flow ([7.1](../07-login/01-flow-overview.md))

- [ ] Login modelled as an explicit state machine
- [ ] `phone_number` and `phone_code_hash` retained between `sendCode` and `signIn`
- [ ] `303 PHONE_MIGRATE_N` handled before anything else
- [ ] All `auth.*` calls sent **without** the authorization requirement
- [ ] Developed against the test DCs first

### `auth.sendCode` ([7.3](../07-login/03-send-code.md))

- [ ] Phone number in international format, digits only
- [ ] `codeSettings` with `flags = 0` unless you can honour a flag
- [ ] `phone_code_hash` **persisted**
- [ ] All three `auth.SentCode` result types handled
- [ ] `timeout` respected before calling `auth.resendCode`
- [ ] `type.length` used to size the prompt; non-numeric types not assumed to be digits

### `auth.signIn` ([7.4](../07-login/04-sign-in.md))

- [ ] `flags = 1` with `phone_code` present, for a normal code login
- [ ] `phone_number` identical to the `sendCode` value
- [ ] `phone_code_hash` from the `sentCode` reply
- [ ] **⚠** `401 SESSION_PASSWORD_NEEDED` matched on **message**, routed to SRP
- [ ] `PHONE_CODE_INVALID` and `PHONE_CODE_EXPIRED` handled differently
- [ ] `auth.authorizationSignUpRequired` handled or reported clearly
- [ ] The parser copes with `user#b1b8cc83`'s two flag words

### SRP ([7.5](../07-login/05-two-factor-srp.md))

- [ ] **⚠** `account.getPassword` called fresh for every attempt
- [ ] Flag bit 2 checked before reading `current_algo`/`srp_B`/`srp_id`
- [ ] **⚠** `passwordKdfAlgoUnknown` rejected with a clear message
- [ ] **⚠** `g` and `p` validated with the full DH parameter checks
- [ ] **⚠** `0 < B < p` and `248 ≤ len(B) ≤ 256` enforced
- [ ] `H(data, salt) = SHA256(salt ‖ data ‖ salt)` — salt on **both** sides
- [ ] PBKDF2 is **HMAC-SHA512**, 100000 iterations, 64-byte output, salted with `client_salt`
- [ ] **⚠** `a` is 256 secure random bytes
- [ ] `A`, `S`, `g_pad` all serialized to exactly 256 bytes
- [ ] `B` used as `B_pad ‖ B` in both `u` and `M1`
- [ ] `p` used **unpadded** in `k` and `h1`
- [ ] `t += p` when `B − kv` is negative
- [ ] `exp = u·x + a`, **not** reduced
- [ ] `srp_id` echoed unchanged
- [ ] **⚠** Validation failure aborts rather than sending `inputCheckPasswordEmpty`

### After login ([7.6](../07-login/06-after-login.md))

- [ ] `auth_key` and `dc_id` persisted immediately on `auth.authorization`
- [ ] **⚠** Storage is `0600`, outside the repository, never logged
- [ ] `server_salt`, `server_time_difference`, and own `user_id` persisted too
- [ ] **⚠** Password, code, `phone_code_hash`, and SRP values **not** persisted
- [ ] Startup loads the key and skips the handshake
- [ ] `AUTH_KEY_UNREGISTERED` / `-404` clears the stored key and restarts login
- [ ] `auth.logOut` used for real logout; local deletion only for "forget device"
- [ ] The client does not re-authenticate on every start

---

## Stage 8 — Sending a message

### Peers ([8.1](../08-sending-a-message/01-peers-and-access-hashes.md))

- [ ] `Peer` (output) and `InputPeer` (input) not confused
- [ ] `InputPeer` and `InputUser` not confused — different ids for identical shapes
- [ ] `inputPeerSelf#7da07ec9` used for the first end-to-end test
- [ ] An access-hash cache populated from every reply carrying `User`/`Chat` objects
- [ ] `min` users never overwrite full cached hashes
- [ ] `inputPeerUserFromMessage` used when only a `min` object is available
- [ ] Ids kept as plain positive `long`s on the wire
- [ ] `PEER_ID_INVALID` treated as "bad access hash" first

### Resolving ([8.2](../08-sending-a-message/02-resolving-a-recipient.md))

- [ ] `contacts.resolveUsername` sent with `flags = 0` and no leading `@`
- [ ] Access hash taken from the `users`/`chats` vectors, **not** from `peer`
- [ ] All three `Peer` kinds handled
- [ ] `messages.dialogs` **and** `messages.dialogsSlice` both handled
- [ ] `inputPeerEmpty` used as `offset_peer` for the first page
- [ ] Username resolutions cached; `FLOOD_WAIT_X` honoured

### Sending ([8.3](../08-sending-a-message/03-send-message.md))

- [ ] `flags = 0` for a plain text message
- [ ] Fields in declaration order: `flags`, `peer`, `message`, `random_id`
- [ ] **⚠** `random_id` from a secure RNG, non-zero
- [ ] **⚠** `random_id` generated **once** and reused across every retry
- [ ] `random_id` stored with the pending query, not regenerated in the send path
- [ ] `message` is UTF-8, non-empty, within `config.message_length_max` **UTF-16 units**
- [ ] No `entities` unless UTF-16 offsets are computed correctly
- [ ] All six `Updates` constructors at least recognized
- [ ] `users`/`chats` from the reply absorbed into the cache
- [ ] `FLOOD_WAIT_X` and `SLOWMODE_WAIT_X` honoured

### Updates ([8.4](../08-sending-a-message/04-reading-updates.md))

- [ ] Unknown constructor ids logged and ignored, never fatal
- [ ] All seven `Updates` container types recognized
- [ ] Short forms understood to omit peer objects
- [ ] Duplicate and out-of-order updates tolerated
- [ ] If tracking state: `pts`/`qts`/`date`/`seq` stored; gaps trigger `getDifference`
- [ ] If tracking state: `differenceSlice` loops until `difference` or `differenceEmpty`
- [ ] `updatesTooLong` triggers a catch-up
- [ ] Connection kept open with periodic pings for a long-running client

---

## Stage 9 — Optional: perfect forward secrecy

Skip unless you need it. ([9.1](../09-advanced/01-perfect-forward-secrecy.md))

- [ ] **⚠** The temporary key is **never** written to disk
- [ ] Its buffer zeroed on expiry
- [ ] `expires_in` randomized in the 23-24 hour range
- [ ] `bind_auth_key_inner` uses `seq_no = 0`
- [ ] The blob encrypted with **MTProto v1** (SHA-1 KDF)
- [ ] The blob uses **random** `salt` and `session_id`, not the real ones
- [ ] `nonce` and `expires_at` identical inside the blob and in the outer call
- [ ] The outer `auth.bindTempAuthKey` encrypted with the **temporary** key
- [ ] Re-keying starts before expiry, not after
- [ ] Failure falls back to the permanent key rather than breaking the client

---

## Stage 10 — Hardening

### Language-specific

**C** ([10.2](../10-implementation/02-c-guide.md)):

- [ ] Built with `-Wall -Wextra -Wconversion`; ASan + UBSan in development
- [ ] `BN_bn2binpad` for fixed-width big-integer output
- [ ] **⚠** Secrets zeroed with `OPENSSL_cleanse` or equivalent
- [ ] AES-IGE asserts a 16-byte multiple; in-place aliasing handled

**Rust** ([10.3](../10-implementation/03-rust-guide.md)):

- [ ] Newtypes for `AuthKeyId`, `SessionId`, `RandomId`
- [ ] **⚠** All parser reads via `get(..)`/`checked_add`, never `[a..b]` on untrusted input
- [ ] AES-IGE uses `chunks_exact_mut(16)` and captures the plaintext before overwriting
- [ ] `to_binary(n, 256)` helper used for every fixed-width big integer
- [ ] **⚠** `zeroize` on all secret types
- [ ] Distinct error variants for transport, protocol, and RPC failures
- [ ] No `unsafe`

### Testing ([10.4](../10-implementation/04-testing-and-debugging.md))

- [ ] Development against test DCs with test numbers
- [ ] **Test-DC RSA key** used for test DCs
- [ ] Crypto primitives covered by known-answer tests
- [ ] Envelope loopback-tested for both `X=0` and `X=8`
- [ ] Padding rules property-tested across a size range
- [ ] Four-level hex logging available
- [ ] **⚠** Secrets redacted from logs

### Operations

- [ ] A single connection per DC unless there is a specific reason for more
      ([9.2](../09-advanced/02-connection-management.md))
- [ ] Periodic pings with a response timeout
- [ ] **⚠** Shared state protected if multi-threaded
- [ ] Persistence file `0600` with a version field

---

## The eight milestones

If you want a coarser progress measure:

| # | Milestone | Proves |
|---|-----------|--------|
| 1 | TL round-trip test passes | Serialization |
| 2 | TCP connection established, obfuscation applied | Transport |
| 3 | `resPQ` received and nonce-verified | Plaintext messages |
| 4 | `dh_gen_ok` received, `auth_key` derived | Handshake |
| 5 | `help.getConfig` returns a `config` | Encrypted envelope + session |
| 6 | `auth.sendCode` returns a `phone_code_hash` | API layer + login |
| 7 | `auth.authorization` received | Full login |
| 8 | `messages.sendMessage` to yourself returns `Updates` | **Done** |

Milestone 5 is the big one. If `help.getConfig` works, every hard part is behind you.

---

## Non-negotiables

If you implement nothing else on this page, implement these. Each one, omitted, is an
exploitable vulnerability rather than a bug:

1. **Validate the DH parameters** — `p` prime, `(p−1)/2` prime, `g_a` in range.
2. **Verify `msg_key` on every decrypt**, in constant time.
3. **Verify the RSA fingerprint** before encrypting `new_nonce`.
4. **Verify `new_nonce_hash1`** at the end of the handshake.
5. **Verify `session_id`, `msg_id` parity, and the time window** on every inbound message.
6. **Reject duplicate `msg_id`s.**
7. **Use a CSPRNG for every secret**, and treat RNG failure as fatal.
8. **Bounds-check every parser read.**
9. **Never log or persist the auth key, password, or SRP intermediates.**

---

[← Previous](D-tdlib-source-map.md) · [Index](../README.md)
