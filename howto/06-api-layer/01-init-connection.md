# 6.1 — `initConnection` and `invokeWithLayer`

[← Previous](../05-session/05-reliability.md) · [Index](../README.md) · [Next: rpc_result and errors →](02-rpc-results-and-errors.md)

---

Before the server will answer any API call, you must tell it two things: which **schema
layer** you speak, and who you are. Both are done by wrapping your first query.

---

## 1. The two wrappers

```
invokeWithLayer#da9b0d0d {X:Type} layer:int query:!X = X;
```
(`td/generate/scheme/telegram_api.tl:2307`)

```
initConnection#c1cd5ea9 {X:Type} flags:# api_id:int device_model:string
    system_version:string app_version:string system_lang_code:string
    lang_pack:string lang_code:string proxy:flags.0?InputClientProxy
    params:flags.1?JSONValue query:!X = X;
```
(`telegram_api.tl:2306`)

They nest, outermost first:

```
invokeWithLayer(
    layer = 229,
    query = initConnection(
        api_id, device_model, …,
        query = <your actual first call>
    )
)
```

The result type `X` propagates through both wrappers, so the reply you get is simply the
result of your inner query — there is no extra envelope.

---

## 2. The layer number

`td/telegram/Version.h:13`:

```cpp
constexpr int32 MTPROTO_LAYER = 229;
```

The layer tells the server which version of `telegram_api.tl` you were compiled against.
The server keeps old layers working, so a slightly stale number is safe; a wildly stale one
means you lose access to newer fields.

**Use the layer that matches the schema file you generated your code from.** Mixing a
layer-229 declaration with a layer-200 number will get you fields the server does not send,
or vice versa.

---

## 3. Building the header

`MtprotoHeader::store` (`td/telegram/net/MtprotoHeader.cpp:28-101`) is the authoritative
construction:

```cpp
store(static_cast<int32>(0xda9b0d0d), storer);   // invokeWithLayer
store(MTPROTO_LAYER, storer);                    // 229
store(static_cast<int32>(0xc1cd5ea9), storer);   // initConnection
int32 flags = 0;
bool have_proxy = !is_anonymous && options.proxy.type() == Proxy::Type::Mtproto;
if (have_proxy)          flags |= 1 << 0;
if (!is_anonymous)       flags |= 1 << 1;
if (options.is_emulator) flags |= 1 << 10;
store(flags, storer);
store(options.api_id, storer);
…
```

### Flags

| Bit | Field | When set |
|-----|-------|----------|
| 0 | `proxy:InputClientProxy` | Connecting through an MTProxy |
| 1 | `params:JSONValue` | Always, unless anonymous |
| 10 | *(no field)* | The client is running in an emulator |

Bit 10 is a **flag-only** bit — it has no associated field, so it consumes no bytes. It is
not even declared in the schema line; TDLib sets it as an out-of-band signal.

### Field values

| Field | Type | TDLib's value | Yours |
|-------|------|---------------|-------|
| `api_id` | int | From `td_api::setTdlibParameters` | Your own — see [7.2](../07-login/02-api-credentials.md) |
| `device_model` | string | e.g. `"Desktop"` | Anything non-empty |
| `system_version` | string | e.g. `"Linux 6.1"` | Anything non-empty |
| `app_version` | string | Your app's version | Anything non-empty |
| `system_lang_code` | string | e.g. `"en"` | `"en"` |
| `lang_pack` | string | `"tdlib"`, or empty | **Empty string** |
| `lang_code` | string | e.g. `"en"`, or empty | **Empty string** |
| `proxy` | InputClientProxy | Only with an MTProxy | Omit |
| `params` | JSONValue | A JSON object | `jsonObject` with no fields |

> **⚠ Common bug.** `lang_pack` and `lang_code` must be **empty strings**, not omitted —
> they are unconditional fields, not flag-gated. TDLib stores empty slices for them when
> there is no language pack (`MtprotoHeader.cpp:58-61`). Sending a `lang_pack` you have not
> registered gets you `400 LANG_PACK_INVALID`.

### The anonymous variant

`is_anonymous` (`MtprotoHeader.cpp:49-55, 76`) substitutes `"n/a"` for
`device_model` and `system_version` and drops both `proxy` and `params`. TDLib uses it for
requests that should not be linkable to a device.

---

## 4. The `params` JSON value

`MtprotoHeader.cpp:76-100`:

```cpp
if (!is_anonymous) {
  telegram_api::object_ptr<telegram_api::JSONValue> json_value;
  if (options.parameters.empty()) {
    json_value = make_tl_object<telegram_api::jsonObject>(vector<…>());
  } else {
    …get_input_json_value(parameters_copy)…
  }
  if (json_value->get_id() == telegram_api::jsonObject::ID) {
    // inject or overwrite "tz_offset"
    …
    values.push_back(make_tl_object<telegram_api::jsonObjectValue>(
        "tz_offset", make_tl_object<telegram_api::jsonNumber>(options.tz_offset)));
  }
  TlStoreBoxedUnknown<TlStoreObject>::store(json_value, storer);
}
```

TDLib always injects a `tz_offset` number (seconds east of UTC). The relevant schema:

```
jsonObjectValue#c0de1bd9 key:string value:JSONValue = JSONObjectValue;
jsonNull#3f6d7b68 = JSONValue;
jsonBool#c7345e6a value:Bool = JSONValue;
jsonNumber#2be0dfa4 value:double = JSONValue;
jsonString#b71e767a value:string = JSONValue;
jsonArray#f7444763 value:Vector<JSONValue> = JSONValue;
jsonObject#99c1d49d value:Vector<JSONObjectValue> = JSONValue;
```
(`telegram_api.tl:1265-1272`)

The simplest legal value is an empty object — 12 bytes:

```
49 d4 c1 99      jsonObject#99c1d49d
15 c4 b5 1c      Vector
00 00 00 00      count = 0
```

You may omit `params` altogether by clearing flag bit 1. TDLib always sets it; either works.

---

## 5. When to send it

**Once per connection**, wrapping your first query. Subsequent queries on the same
connection are sent bare.

TDLib tracks this per connection (`td/telegram/net/Session.cpp`) and re-sends the header if
the server tells it to — see §7.

A typical first packet is therefore:

```
invokeWithLayer(229, initConnection(…, help.getConfig()))
```

`help.getConfig` is a good choice: it is cheap, needs no authorization, and its reply gives
you the datacenter list ([2.1](../02-transport/01-datacenters.md)).

---

## 6. Byte layout

For `api_id = 94575`, `device_model = "PC"`, `system_version = "Linux"`,
`app_version = "1.0"`, `system_lang_code = "en"`, empty `lang_pack`/`lang_code`, an empty
`params` object, wrapping `help.getConfig#c4f9186b`:

```
0d 0d 9b da            invokeWithLayer#da9b0d0d
e5 00 00 00            layer = 229
a9 5e cd c1            initConnection#c1cd5ea9
02 00 00 00            flags = 0x02  (bit 1: params present)
ef 71 01 00            api_id = 94575
02 50 43 00            "PC"     (len 2, 2 data, 1 pad → 4 bytes)
05 4c 69 6e 75 78 00 00  "Linux" (len 5, 5 data, 2 pad → 8 bytes)
03 31 2e 30 00          "1.0"   (len 3, 3 data, 0 pad → 4 bytes)
02 65 6e 00            "en"
00 00 00 00            "" lang_pack  (1 len byte + 3 pad)
00 00 00 00            "" lang_code
49 d4 c1 99            jsonObject
15 c4 b5 1c            Vector
00 00 00 00            count = 0
6b 18 f9 c4            help.getConfig#c4f9186b
```

Total: 68 bytes. Note how the empty strings still occupy 4 bytes each — a single length
byte `0x00` plus three padding bytes ([1.2 §3](../01-serialization/02-wire-encoding.md)).

---

## 7. What happens if you get it wrong

| Error | Meaning | Fix |
|-------|---------|-----|
| `400 CONNECTION_NOT_INITED` | You sent a query before `initConnection` on this connection | Resend the query wrapped in the header |
| `400 CONNECTION_LAYER_INVALID` | `invokeWithLayer` missing, or the layer is unusable | Same |
| `400 API_ID_INVALID` | Bad `api_id` | Check your credentials |
| `400 API_ID_PUBLISHED_FLOOD` | You used a publicly known `api_id` | Get your own |
| `400 LANG_PACK_INVALID` | Non-empty `lang_pack` you have not registered | Send an empty string |

TDLib's recovery (`td/telegram/net/Session.cpp:970-974`) is to mark the connection as
un-initialized and rewrite the error as a retryable `500`, so the query is resent — this
time with the header:

```cpp
if (error_code == 400 && (message == "CONNECTION_NOT_INITED" || message == "CONNECTION_LAYER_INVALID")) {
  LOG(WARNING) << "Receive " << message;
  auth_data_.on_connection_not_inited();
  error_code = 500;
}
```

Implement this — it also fires legitimately when the server restarts and forgets your
connection state.

---

## 8. Checklist

- [ ] `invokeWithLayer#da9b0d0d` outermost, then the layer as `int32`
- [ ] `initConnection#c1cd5ea9` next
- [ ] `flags` bit 1 set if `params` is present, bit 0 if `proxy` is present
- [ ] `api_id` is your own registered id
- [ ] `device_model`, `system_version`, `app_version`, `system_lang_code` non-empty
- [ ] `lang_pack` and `lang_code` present as **empty strings**
- [ ] `params` is a valid `JSONValue` (an empty `jsonObject` is fine) or the flag is clear
- [ ] Sent **once per connection**, wrapping the first query
- [ ] `CONNECTION_NOT_INITED` / `CONNECTION_LAYER_INVALID` trigger a resend with the header

---

[← Previous](../05-session/05-reliability.md) · [Index](../README.md) · [Next: rpc_result and errors →](02-rpc-results-and-errors.md)
