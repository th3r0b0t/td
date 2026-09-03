# Appendix A — Constructor Reference

[← Previous](../10-implementation/04-testing-and-debugging.md) · [Index](../README.md) · [Next: Appendix B →](B-datacenters-and-keys.md)

---

Every constructor needed to log in and send a message. All ids verified against
`td/generate/scheme/mtproto_api.tl` and `td/generate/scheme/telegram_api.tl` (layer 229).

> **⚠ Leading zeros.** TL schema files omit leading zero bytes from ids:
> `users.getUsers#d91a548` is `0x0d91a548`, and `auth.sentCodeTypeFirebaseSms#9fd736` is
> `0x009fd736`. Constructor ids are always 32 bits.

---

## A.1 — MTProto (`mtproto_api.tl`)

### Handshake

| Id | Constructor | Line |
|----|-------------|------|
| `0x05162463` | `resPQ nonce:int128 server_nonce:int128 pq:string server_public_key_fingerprints:Vector<long>` | 13 |
| `0xa9f55f95` | `p_q_inner_data_dc pq p q nonce server_nonce new_nonce:int256 dc:int` | 15 |
| `0x56fddf88` | `p_q_inner_data_temp_dc pq p q nonce server_nonce new_nonce dc:int expires_in:int` | 16 |
| `0xd0e8075c` | `server_DH_params_ok nonce server_nonce encrypted_answer:string` | 18 |
| `0xb5890dba` | `server_DH_inner_data nonce server_nonce g:int dh_prime:string g_a:string server_time:int` | 20 |
| `0x6643b654` | `client_DH_inner_data nonce server_nonce retry_id:long g_b:string` | 22 |
| `0x3bcbf734` | `dh_gen_ok nonce server_nonce new_nonce_hash1:int128` | 24 |
| `0x46dc1fb9` | `dh_gen_retry nonce server_nonce new_nonce_hash2:int128` | 25 |
| `0xa69dae02` | `dh_gen_fail nonce server_nonce new_nonce_hash3:int128` | 26 |
| `0x75a3f765` | `bind_auth_key_inner nonce temp_auth_key_id perm_auth_key_id temp_session_id expires_at:int` | 28 |
| — | `rsa_public_key n:string e:string` (**bare**, no id) | 65 |

Functions:

| Id | Function | Line |
|----|----------|------|
| `0xbe7e8ef1` | `req_pq_multi nonce:int128 = ResPQ` | 73 |
| `0xd712e4be` | `req_DH_params nonce server_nonce p:string q:string public_key_fingerprint:long encrypted_data:string = Server_DH_Params` | 75 |
| `0xf5045f1f` | `set_client_DH_params nonce server_nonce encrypted_data:string = Set_client_DH_params_answer` | 77 |

> **Note.** TDLib's schema contains **no** `req_pq#60469778` and **no**
> `server_DH_params_fail#79cb045d`. Use `req_pq_multi`; treat an unparseable
> `Server_DH_Params` as a failure.

### Service messages

| Id | Constructor | Line |
|----|-------------|------|
| `0xf35c6d01` | `rpc_result req_msg_id:long result:Object` | 30 (commented; parsed by hand) |
| `0x2144ca19` | `rpc_error error_code:int error_message:string` | 31 |
| `0x5e2ad36e` | `rpc_answer_unknown` | 33 |
| `0xcd78e586` | `rpc_answer_dropped_running` | 34 |
| `0xa43ad8b7` | `rpc_answer_dropped msg_id seq_no:int bytes:int` | 35 |
| `0x0949d9dc` | `future_salt valid_since:int valid_until:int salt:long` | 37 |
| `0xae500895` | `future_salts req_msg_id:long now:int salts:vector<future_salt>` | 38 |
| `0x347773c5` | `pong msg_id:long ping_id:long` | 40 |
| `0x9ec20908` | `new_session_created first_msg_id:long unique_id:long server_salt:long` | 45 |
| `0x73f1f8dc` | `msg_container messages:vector<message>` | 47 (commented) |
| — | `message msg_id:long seqno:int bytes:int body:string` (**bare**) | 48 (commented) |
| `0xe06046b2` | `msg_copy orig_message:Object` | 49 (commented) |
| `0x3072cfa1` | `gzip_packed packed_data:string` | 51 |
| `0x62d6b459` | `msgs_ack msg_ids:Vector<long>` | 53 |
| `0xa7eff811` | `bad_msg_notification bad_msg_id:long bad_msg_seqno:int error_code:int` | 55 |
| `0xedab447b` | `bad_server_salt bad_msg_id:long bad_msg_seqno:int error_code:int new_server_salt:long` | 56 |
| `0x7d861a08` | `msg_resend_req msg_ids:Vector<long>` | 58 |
| `0xda69fb52` | `msgs_state_req msg_ids:Vector<long>` | 59 |
| `0x04deb57d` | `msgs_state_info req_msg_id:long info:string` | 60 |
| `0x8cc0d131` | `msgs_all_info msg_ids:Vector<long> info:string` | 61 |
| `0x276d3ec6` | `msg_detailed_info msg_id answer_msg_id bytes:int status:int` | 62 |
| `0x809db6df` | `msg_new_detailed_info answer_msg_id bytes:int status:int` | 63 |
| `0xf660e1d4` | `destroy_auth_key_ok` | 67 |
| `0x0a9f2259` | `destroy_auth_key_none` | 68 |
| `0xea109b13` | `destroy_auth_key_fail` | 69 |

Functions:

| Id | Function | Line |
|----|----------|------|
| `0x58e4a740` | `rpc_drop_answer req_msg_id:long` | 79 |
| `0xb921bd04` | `get_future_salts num:int` | 80 |
| `0x7abe77ec` | `ping ping_id:long` | 81 (commented) |
| `0xf3427b8c` | `ping_delay_disconnect ping_id:long disconnect_delay:int` | 82 |
| `0x9299359f` | `http_wait max_delay:int wait_after:int max_wait:int` | 85 |
| `0xd1435160` | `destroy_auth_key` | 87 |

---

## A.2 — Telegram API (`telegram_api.tl`, layer 229)

### Wrappers

| Id | Constructor | Line |
|----|-------------|------|
| `0xda9b0d0d` | `invokeWithLayer layer:int query:!X` | 2307 |
| `0xc1cd5ea9` | `initConnection flags:# api_id:int device_model:string system_version:string app_version:string system_lang_code:string lang_pack:string lang_code:string proxy:flags.0?InputClientProxy params:flags.1?JSONValue query:!X` | 2306 |
| `0xcb9f372d` | `invokeAfterMsg msg_id:long query:!X` | — |
| `0x3dc4b4f0` | `invokeAfterMsgs msg_ids:Vector<long> query:!X` | — |
| `0x75588b3f` | `inputClientProxy address:string port:int` | 1186 |
| `0x1cb5c415` | `Vector t` (boxed vector) | — |

### JSON

| Id | Constructor | Line |
|----|-------------|------|
| `0xc0de1bd9` | `jsonObjectValue key:string value:JSONValue` | 1265 |
| `0x3f6d7b68` | `jsonNull` | 1267 |
| `0xc7345e6a` | `jsonBool value:Bool` | 1268 |
| `0x2be0dfa4` | `jsonNumber value:double` | 1269 |
| `0xb71e767a` | `jsonString value:string` | 1270 |
| `0xf7444763` | `jsonArray value:Vector<JSONValue>` | 1271 |
| `0x99c1d49d` | `jsonObject value:Vector<JSONObjectValue>` | 1272 |

### Config

| Id | Constructor | Line |
|----|-------------|------|
| `0xc4f9186b` | `help.getConfig = Config` | 2791 |
| `0xcc1a241e` | `config …` | 538 |
| `0x18b7a10d` | `dcOption flags:# ipv6:flags.0?true media_only:flags.1?true tcpo_only:flags.2?true cdn:flags.3?true static:flags.4?true this_port_only:flags.5?true id:int ip_address:string port:int secret:flags.10?bytes` | 536 |
| `0x1fb33026` | `help.getNearestDc = NearestDc` | 2792 |
| `0x8e1a1775` | `nearestDc country:string this_dc:int nearest_dc:int` | 540 |

### Login

| Id | Constructor | Line |
|----|-------------|------|
| `0xa677244f` | `auth.sendCode phone_number:string api_id:int api_hash:string settings:CodeSettings = auth.SentCode` | 2316 |
| `0xad253d78` | `codeSettings flags:# …` | 1319 |
| `0x5e002502` | `auth.sentCode flags:# type:auth.SentCodeType phone_code_hash:string next_type:flags.1?auth.CodeType timeout:flags.2?int` | 260 |
| `0x2390fe44` | `auth.sentCodeSuccess authorization:auth.Authorization` | 261 |
| `0xf8827ebf` | `auth.sentCodePaymentRequired …` | 262 |
| `0x8d52a951` | `auth.signIn flags:# phone_number:string phone_code_hash:string phone_code:flags.0?string email_verification:flags.1?EmailVerification = auth.Authorization` | 2318 |
| `0xaac7b717` | `auth.signUp flags:# no_joined_notifications:flags.0?true phone_number:string phone_code_hash:string first_name:string last_name:string` | 2317 |
| `0xcae47523` | `auth.resendCode flags:# phone_number:string phone_code_hash:string reason:flags.0?string` | 2328 |
| `0x1f040578` | `auth.cancelCode phone_number:string phone_code_hash:string = Bool` | 2329 |
| `0x2ea2c0d4` | `auth.authorization flags:# setup_password_required:flags.1?true otherwise_relogin_days:flags.1?int tmp_sessions:flags.0?int future_auth_token:flags.2?bytes user:User` | 264 |
| `0x44747e9a` | `auth.authorizationSignUpRequired flags:# terms_of_service:flags.0?help.TermsOfService` | 265 |
| `0x3e72ba19` | `auth.logOut = auth.LoggedOut` | 2319 |
| `0xc3a2835f` | `auth.loggedOut flags:# future_auth_token:flags.0?bytes` | 1541 |
| `0xe5bfffcd` | `auth.exportAuthorization dc_id:int = auth.ExportedAuthorization` | 2321 |
| `0xb434e2b8` | `auth.exportedAuthorization id:long bytes:bytes` | 267 |
| `0xa57a7dad` | `auth.importAuthorization id:long bytes:bytes = auth.Authorization` | 2322 |
| `0xcdd42a05` | `auth.bindTempAuthKey perm_auth_key_id:long nonce:long expires_at:int encrypted_message:bytes = Bool` | 2323 |

### Code delivery types

| Id | Constructor | Line |
|----|-------------|------|
| `0x3dbb5986` | `auth.sentCodeTypeApp length:int` | 854 |
| `0xc000bba2` | `auth.sentCodeTypeSms length:int` | 855 |
| `0x5353e5a7` | `auth.sentCodeTypeCall length:int` | 856 |
| `0xab03c6d9` | `auth.sentCodeTypeFlashCall pattern:string` | 857 |
| `0x82006484` | `auth.sentCodeTypeMissedCall prefix:string length:int` | 858 |
| `0xf450f59b` | `auth.sentCodeTypeEmailCode …` | 859 |
| `0xa5491dea` | `auth.sentCodeTypeSetUpEmailRequired …` | 860 |
| `0xd9565c39` | `auth.sentCodeTypeFragmentSms url:string length:int` | 861 |
| `0x009fd736` | `auth.sentCodeTypeFirebaseSms …` | 862 |
| `0xa416ac81` | `auth.sentCodeTypeSmsWord flags:# beginning:flags.0?string` | 863 |
| `0xb37794af` | `auth.sentCodeTypeSmsPhrase flags:# beginning:flags.0?string` | 864 |

### Two-factor authentication

| Id | Constructor | Line |
|----|-------------|------|
| `0x548a30f5` | `account.getPassword = account.Password` | 2367 |
| `0x957b50fb` | `account.password flags:# has_recovery:flags.0?true has_secure_values:flags.1?true has_password:flags.2?true current_algo:flags.2?PasswordKdfAlgo srp_B:flags.2?bytes srp_id:flags.2?long hint:flags.3?string …` | 700 |
| `0xd45ab096` | `passwordKdfAlgoUnknown` | 1245 |
| `0x3a912d4a` | `passwordKdfAlgoSHA256SHA256PBKDF2HMACSHA512iter100000SHA256ModPow salt1:bytes salt2:bytes g:int p:bytes` | 1246 |
| `0x9880f658` | `inputCheckPasswordEmpty` | 1254 |
| `0xd27ff082` | `inputCheckPasswordSRP srp_id:long A:bytes M1:bytes` | 1255 |
| `0xd18b4d16` | `auth.checkPassword password:InputCheckPasswordSRP = auth.Authorization` | 2325 |

### Peers

| Id | Constructor | Line |
|----|-------------|------|
| `0x7f3b18ea` | `inputPeerEmpty` | 38 |
| `0x7da07ec9` | `inputPeerSelf` | 39 |
| `0x35a95cb9` | `inputPeerChat chat_id:long` | 40 |
| `0xdde8a54c` | `inputPeerUser user_id:long access_hash:long` | 41 |
| `0x27bcbbfc` | `inputPeerChannel channel_id:long access_hash:long` | 42 |
| `0xa87b0a1c` | `inputPeerUserFromMessage peer:InputPeer msg_id:int user_id:long` | 43 |
| `0xbd2a0840` | `inputPeerChannelFromMessage peer:InputPeer msg_id:int channel_id:long` | 44 |
| `0xb98886cf` | `inputUserEmpty` | 46 |
| `0xf7c1b13f` | `inputUserSelf` | 47 |
| `0xf21158c6` | `inputUser user_id:long access_hash:long` | 48 |
| `0x1da448e2` | `inputUserFromMessage peer:InputPeer msg_id:int user_id:long` | 49 |
| `0x59511722` | `peerUser user_id:long` | 99 |
| `0x36c6019a` | `peerChat chat_id:long` | 100 |
| `0xa2a5371e` | `peerChannel channel_id:long` | 101 |
| `0xb1b8cc83` | `user flags:# … flags2:# … id:long access_hash:flags.0?long first_name:flags.1?string …` | 115 |
| `0xd3bc4b7a` | `userEmpty id:long` | 114 |

### Resolving and messaging

| Id | Constructor | Line |
|----|-------------|------|
| `0x725afbbc` | `contacts.resolveUsername flags:# username:string referer:flags.0?string = contacts.ResolvedPeer` | 2493 |
| `0x7f077ad9` | `contacts.resolvedPeer peer:Peer chats:Vector<Chat> users:Vector<User>` | 778 |
| `0x5dd69e12` | `contacts.getContacts hash:long = contacts.Contacts` | 2485 |
| `0xeae87e42` | `contacts.contacts contacts:Vector<Contact> saved_count:int users:Vector<User>` | 305 |
| `0xa0f4cb4f` | `messages.getDialogs flags:# … offset_date:int offset_id:int offset_peer:InputPeer limit:int hash:long` | 2513 |
| `0x15ba6c40` | `messages.dialogs dialogs:Vector<Dialog> messages:Vector<Message> chats:Vector<Chat> users:Vector<User>` | 312 |
| `0x71e094f3` | `messages.dialogsSlice count:int …` | 313 |
| `0xfc89f7f3` | `dialog flags:# … peer:Peer top_message:int …` | 243 |
| `0x0d91a548` | `users.getUsers id:Vector<InputUser> = Vector<User>` | 2475 |
| `0xfef48f62` | `messages.sendMessage flags:# … peer:InputPeer reply_to:flags.0?InputReplyTo message:string random_id:long … = Updates` | 2521 |

### Updates

| Id | Constructor | Line |
|----|-------------|------|
| `0xe317af7e` | `updatesTooLong` | 520 |
| `0x313bc7f8` | `updateShortMessage …` | 521 |
| `0x4d6deea5` | `updateShortChatMessage …` | 522 |
| `0x78d4dec1` | `updateShort update:Update date:int` | 523 |
| `0x725b04c3` | `updatesCombined updates:Vector<Update> users:Vector<User> chats:Vector<Chat> date:int seq_start:int seq:int` | 524 |
| `0x74ae4240` | `updates updates:Vector<Update> users:Vector<User> chats:Vector<Chat> date:int seq:int` | 525 |
| `0x9015e101` | `updateShortSentMessage flags:# out:flags.1?true id:int pts:int pts_count:int date:int …` | 526 |
| `0x1f2b0afd` | `updateNewMessage message:Message pts:int pts_count:int` | 347 |
| `0x4e90bfd6` | `updateMessageID id:int random_id:long` | 348 |
| `0x62ba04d9` | `updateNewChannelMessage message:Message pts:int pts_count:int` | 373 |
| `0x7600b9d3` | `message flags:# … flags2:# … id:int … peer_id:Peer … date:int message:string …` | 150 |
| `0xedd4882a` | `updates.getState = updates.State` | 2772 |
| `0xa56c2a3e` | `updates.state pts:int qts:int date:int seq:int unread_count:int` | 513 |
| `0x19c2f763` | `updates.getDifference flags:# pts:int … date:int qts:int …` | 2773 |
| `0x5d75a138` | `updates.differenceEmpty date:int seq:int` | 515 |
| `0x00f49ca0` | `updates.difference new_messages:Vector<Message> … state:updates.State` | 516 |
| `0xa8fb1981` | `updates.differenceSlice … intermediate_state:updates.State` | 517 |
| `0x4afe8f6d` | `updates.differenceTooLong pts:int` | 518 |

---

## A.3 — Transport magic values

| Value | Meaning | Source |
|-------|---------|--------|
| `0xeeeeeeee` | Intermediate framing | `td/mtproto/TcpTransport.cpp:75-78` |
| `0xdddddddd` | Padded Intermediate framing | same |
| `0x1cb5c415` | Boxed `Vector` | — |

Obfuscation header exclusions (`TcpTransport.cpp:88-108`), as the first little-endian
`uint32`: `0x44414548`, `0x54534f50`, `0x20544547`, `0x4954504f`, `0xdddddddd`,
`0xeeeeeeee`, `0x02010316`. The first byte must not be `0xef`, and bytes `[4..8)` must not
be all zero.

---

## A.4 — Deriving an id yourself

CRC-32 (IEEE/zlib) of the normalized declaration with `#id` removed
(`td/generate/tl-parser/tl-parser.c:1468-1480`):

* Single spaces between tokens, no leading or trailing space.
* `name:Type` with no spaces around the colon.
* `Vector<long>` → `Vector long`.
* Conditionals stay as `name.N?Type`.
* `{X:Type}` dropped; `!X` → `X`.
* The trailing `= ResultType` **is** included.

Example:

```
CRC32("msgs_ack msg_ids:Vector long = MsgsAck") = 0x62d6b459
```

---

[← Previous](../10-implementation/04-testing-and-debugging.md) · [Index](../README.md) · [Next: Appendix B →](B-datacenters-and-keys.md)
