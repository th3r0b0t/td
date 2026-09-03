# 7.5 — Two-Factor Authentication (SRP)

[← Previous](04-sign-in.md) · [Index](../README.md) · [Next: After login →](06-after-login.md)

---

If `auth.signIn` returned `401 SESSION_PASSWORD_NEEDED`, the account has a cloud password.
You must prove you know it **without sending it**, using a variant of SRP-6a.

This is the single most error-prone computation in the whole client. Every step is
specified below, transcribed from `td/telegram/PasswordManager.cpp:37-142`.

---

## 1. Get the parameters

```
account.getPassword#548a30f5 = account.Password;
```
(`td/generate/scheme/telegram_api.tl:2367`)

```
account.password#957b50fb flags:# has_recovery:flags.0?true has_secure_values:flags.1?true
    has_password:flags.2?true current_algo:flags.2?PasswordKdfAlgo srp_B:flags.2?bytes
    srp_id:flags.2?long hint:flags.3?string email_unconfirmed_pattern:flags.4?string
    new_algo:PasswordKdfAlgo new_secure_algo:SecurePasswordKdfAlgo secure_random:bytes
    pending_reset_date:flags.5?int login_email_pattern:flags.6?string = account.Password;
```
(`telegram_api.tl:700`)

The three fields you need — `current_algo`, `srp_B`, `srp_id` — are **all gated on flag bit
2** (`has_password`). If bit 2 is clear there is no password set and you should not be here.

`hint` (bit 3) is the user's password hint; show it.

> **⚠ `srp_B` and `srp_id` are single-use.** They expire. If you take too long, or retry
> after a failure, you must call `account.getPassword` **again** for fresh values. Reusing
> them gives `400 SRP_ID_INVALID` or `400 PASSWORD_HASH_INVALID`.

### The KDF algorithm

```
passwordKdfAlgoUnknown#d45ab096 = PasswordKdfAlgo;
passwordKdfAlgoSHA256SHA256PBKDF2HMACSHA512iter100000SHA256ModPow#3a912d4a
    salt1:bytes salt2:bytes g:int p:bytes = PasswordKdfAlgo;
```
(`telegram_api.tl:1245, 1246`)

Only `0x3a912d4a` is usable. If you get `passwordKdfAlgoUnknown`, your client is too old to
handle this account's password; abort with a clear message.

| Name here | Field | Notes |
|-----------|-------|-------|
| `client_salt` | `salt1` | |
| `server_salt` | `salt2` | Unrelated to the MTProto server salt of [5.4](../05-session/04-salts-and-time.md) |
| `g` | `g` | A small integer, e.g. 2 or 3 |
| `p` | `p` | A 2048-bit prime, 256 bytes |

> **⚠ Validate `g` and `p`.** TDLib runs the *same* DH parameter validation used in the
> handshake (`PasswordManager.cpp:56, 79`):
> ```cpp
> TRY_STATUS(mtproto::DhHandshake::check_config(g, p, DhCache::instance()));
> ```
> That means `p` must be exactly 2048 bits, `p` and `(p−1)/2` must both be prime, and `g`
> must satisfy the quadratic-residue condition for its value — see
> [3.5](../03-auth-key/05-security-checks.md). Skipping this check is a real vulnerability:
> a malicious server could supply a weak `p` and recover your password hash.

---

## 2. Step 1 — the password hash `x`

`PasswordManager::calc_password_hash` (`PasswordManager.cpp:41-51`):

```cpp
static void hash_sha256(Slice data, Slice salt, MutableSlice dest) {
  sha256(PSLICE() << salt << data << salt, dest);
}

BufferSlice PasswordManager::calc_password_hash(Slice password, Slice client_salt, Slice server_salt) {
  BufferSlice buf(32);
  hash_sha256(password, client_salt, buf.as_mutable_slice());          // (1)
  hash_sha256(buf.as_slice(), server_salt, buf.as_mutable_slice());    // (2)
  BufferSlice hash(64);
  pbkdf2_sha512(buf.as_slice(), client_salt, 100000, hash.as_mutable_slice());   // (3)
  hash_sha256(hash.as_slice(), server_salt, buf.as_mutable_slice());   // (4)
  return buf;
}
```

Written out, with `H(data, salt) = SHA256(salt ‖ data ‖ salt)`:

```
PH1  = H(password_utf8, client_salt)                              32 bytes
PH2  = H(PH1,           server_salt)                              32 bytes
PBK  = PBKDF2-HMAC-SHA512(password = PH2,
                          salt     = client_salt,
                          iterations = 100000,
                          dkLen    = 64)                          64 bytes
x    = H(PBK,           server_salt)                              32 bytes
```

Four things people get wrong here:

1. **The salt goes on both sides.** `SHA256(salt ‖ data ‖ salt)`, not `SHA256(salt ‖ data)`.
2. **PBKDF2 uses HMAC-SHA512**, not SHA256, and outputs **64** bytes.
3. **PBKDF2's "password" is `PH2`** (raw 32 bytes), and its **salt is `client_salt`** — the
   *client* salt, even though the step before and after use the server salt.
4. **Exactly 100000 iterations.** The constructor name says so; do not make it configurable.

The password must be encoded as **UTF-8** with no trailing newline. Telegram's official
clients also strip leading and trailing whitespace before hashing — if the user's password
has surrounding spaces, behaviour differs between clients, so trim.

---

## 3. Step 2 — validate `B`

`PasswordManager.cpp:84-90`:

```cpp
auto p_bn = BigNum::from_binary(p);
auto B_bn = BigNum::from_binary(B);
auto zero = BigNum::from_decimal("0").move_as_ok();
if (BigNum::compare(zero, B_bn) != -1 || BigNum::compare(B_bn, p_bn) != -1 ||
    B.size() < 248 || B.size() > 256) {
  LOG(ERROR) << "Receive invalid value of B(" << B.size() << ")…";
  return make_tl_object<telegram_api::inputCheckPasswordEmpty>();
}
```

Four conditions, **all** required:

```
0 < B
B < p
248 ≤ len(B) ≤ 256      (as received, before padding)
```

> **⚠ Do not skip this.** `B = 0` or `B = p` would make the shared secret trivially
> predictable. This is the classic SRP implementation flaw.

---

## 4. Step 3 — the SRP computation

`PasswordManager.cpp:92-141`, transcribed exactly:

```cpp
BigNum g_bn; g_bn.set_value(g);
auto g_padded = g_bn.to_binary(256);                        // g as 256 big-endian bytes

auto x = calc_password_hash(password, client_salt, server_salt);
auto x_bn = BigNum::from_binary(x.as_slice());

BufferSlice a(2048 / 8);
Random::secure_bytes(a.as_mutable_slice());                 // 256 random bytes
auto a_bn = BigNum::from_binary(a.as_slice());

BigNum A_bn;
BigNum::mod_exp(A_bn, g_bn, a_bn, p_bn, ctx);               // A = g^a mod p
string A = A_bn.to_binary(256);                             // 256 bytes

string B_pad(256 - B.size(), '\0');                         // left-pad B to 256
string u = sha256(PSLICE() << A << B_pad << B);
auto u_bn = BigNum::from_binary(u);
string k = sha256(PSLICE() << p << g_padded);
auto k_bn = BigNum::from_binary(k);

BigNum v_bn;  BigNum::mod_exp(v_bn, g_bn, x_bn, p_bn, ctx); // v = g^x mod p
BigNum kv_bn; BigNum::mod_mul(kv_bn, k_bn, v_bn, p_bn, ctx);// kv = k*v mod p
BigNum t_bn;  BigNum::sub(t_bn, B_bn, kv_bn);               // t = B - kv
if (BigNum::compare(t_bn, zero) == -1) {
  BigNum::add(t_bn, t_bn, p_bn);                            // if t < 0, t += p
}
BigNum exp_bn;
BigNum::mul(exp_bn, u_bn, x_bn, ctx);                       // exp = u*x
BigNum::add(exp_bn, exp_bn, a_bn);                          // exp = u*x + a

BigNum S_bn;  BigNum::mod_exp(S_bn, t_bn, exp_bn, p_bn, ctx);// S = t^exp mod p
string S = S_bn.to_binary(256);
auto K = sha256(S);

auto h1 = sha256(p);
auto h2 = sha256(g_padded);
for (size_t i = 0; i < h1.size(); i++) { h1[i] ^= h2[i]; }  // h1 = SHA256(p) XOR SHA256(g_pad)

auto M = sha256(PSLICE() << h1 << sha256(client_salt) << sha256(server_salt)
                         << A << B_pad << B << K);

return make_tl_object<telegram_api::inputCheckPasswordSRP>(id, BufferSlice(A), BufferSlice(M));
```

### As a formula

All big integers are big-endian; `to_binary(256)` means **left-zero-padded to exactly 256
bytes**.

```
g_pad = g as 256-byte big-endian
a     = 256 cryptographically secure random bytes, as an integer
A     = g^a mod p                        , serialized to 256 bytes
B_pad = (256 − len(B)) zero bytes         ; so B_pad ‖ B is B in 256 bytes

u     = SHA256(A ‖ B_pad ‖ B)
k     = SHA256(p ‖ g_pad)
v     = g^x mod p
kv    = (k · v) mod p
t     = B − kv          ; if t < 0 then t = t + p
exp   = u · x + a                         (NOT reduced mod anything)
S     = t^exp mod p                      , serialized to 256 bytes
K     = SHA256(S)

h1    = SHA256(p) XOR SHA256(g_pad)
M1    = SHA256(h1 ‖ SHA256(client_salt) ‖ SHA256(server_salt)
               ‖ A ‖ B_pad ‖ B ‖ K)
```

### Pitfalls

| Pitfall | Consequence |
|---------|-------------|
| `A` or `S` not padded to 256 bytes | Wrong `u`, wrong `M1` |
| `g_pad` not padded to 256 bytes | Wrong `k`, wrong `h1` |
| Using `B` unpadded in `u` or `M1` | Wrong hash — note the code writes `B_pad` **then** `B` |
| Using `p` padded in `h1` | `p` is used **as received**, not re-padded |
| Reducing `exp` mod `p` or mod `p−1` | Wrong `S` |
| Forgetting `t += p` when negative | Wrong `S` |
| `a` shorter than 256 bytes | Weaker, and TDLib uses exactly 256 |
| Little-endian big-number serialization | Everything wrong |

> **💡 Debugging tip.** If you get `PASSWORD_HASH_INVALID`, the fault is almost always in
> `x` (chapter §2) or in a padding step. Test `x` in isolation: two independent
> implementations of §2 fed the same password and salts must agree byte-for-byte.

---

## 5. Step 4 — submit

```
inputCheckPasswordSRP#d27ff082 srp_id:long A:bytes M1:bytes = InputCheckPasswordSRP;
inputCheckPasswordEmpty#9880f658 = InputCheckPasswordSRP;
```
(`telegram_api.tl:1255, 1254`)

```
auth.checkPassword#d18b4d16 password:InputCheckPasswordSRP = auth.Authorization;
```
(`telegram_api.tl:2325`)

Sent unauthenticated, like the rest of the login calls
(`td/telegram/AuthManager.cpp:1364`):

```cpp
G()->net_query_creator().create_unauth(telegram_api::auth_checkPassword(std::move(hash)))
```

* `srp_id` — from `account.password`, unmodified.
* `A` — the 256-byte serialization.
* `M1` — the 32-byte SHA256.

The reply is `auth.authorization`, exactly as in [7.4](04-sign-in.md). You are logged in.

Note that `PasswordManager::get_input_check_password` returns `inputCheckPasswordEmpty`
whenever validation fails (`PasswordManager.cpp:76, 81, 89`) — an empty password rather
than a wrong one. Your implementation should instead **abort with a clear error**; silently
sending an empty proof just produces a confusing rejection.

---

## 6. Errors

| Error | Meaning | Action |
|-------|---------|--------|
| `400 PASSWORD_HASH_INVALID` | Wrong password, or wrong computation | Re-fetch `account.getPassword`, retry |
| `400 SRP_ID_INVALID` | Stale `srp_id` | Re-fetch and recompute |
| `400 SRP_PASSWORD_CHANGED` | Password changed mid-flow | Re-fetch and recompute |
| `420 FLOOD_WAIT_X` | Too many attempts | Wait |

**Every retry must start from a fresh `account.getPassword`.** `srp_B` and `srp_id` are
single-use.

---

## 7. Verifying your implementation

You cannot test SRP against the test datacenters without setting a password on a test
account. A practical approach:

1. Implement §2 (`x`) first and unit-test it against a second implementation — Python's
   `hashlib` plus `hashlib.pbkdf2_hmac('sha512', PH2, client_salt, 100000, 64)` is a good
   independent check.
2. Implement §4 and check the *shapes*: `A` is 256 bytes, `S` is 256 bytes, `M1` is 32
   bytes, `u`/`k`/`K` are 32 bytes.
3. Only then test against the live server.

---

## 8. Checklist

- [ ] `account.getPassword` called fresh for every attempt
- [ ] Flag bit 2 checked before reading `current_algo`/`srp_B`/`srp_id`
- [ ] `passwordKdfAlgoUnknown` rejected with a clear message
- [ ] `g` and `p` validated with the full DH parameter checks
- [ ] `0 < B < p` and `248 ≤ len(B) ≤ 256` enforced
- [ ] `H(data, salt) = SHA256(salt ‖ data ‖ salt)` — salt on both sides
- [ ] PBKDF2 is **HMAC-SHA512**, 100000 iterations, 64-byte output, salted with `client_salt`
- [ ] `a` is 256 secure random bytes
- [ ] `A`, `S`, `g_pad` all serialized to exactly 256 bytes
- [ ] `B` used as `B_pad ‖ B` in both `u` and `M1`
- [ ] `p` used unpadded in `k` and `h1`
- [ ] `t += p` when `B − kv` is negative
- [ ] `exp = u·x + a` not reduced
- [ ] `srp_id` echoed unchanged
- [ ] Validation failure aborts rather than sending `inputCheckPasswordEmpty`

---

[← Previous](04-sign-in.md) · [Index](../README.md) · [Next: After login →](06-after-login.md)
