# Calling the Trae API from outside the IDE

## Summary

Integrating a third-party gateway usually means replicating its wire protocol exactly. Trae's chat endpoint is not OpenAI-compatible: it needs a different auth scheme, nine request headers derived from a device identity, and a non-standard message body. Getting any one of these wrong produces a 401 or 400 with no hint about which requirement was missed.

This note records the three failure modes encountered, in the order they surfaced.

## Failure 1 — wrong auth scheme (401)

Initial attempt used the conventional form:

```http
Authorization: Bearer <accessToken>
```

The endpoint answered `401` on every request, including the read-only model catalog.

**Correct scheme:**

```http
Authorization: Cloud-IDE-JWT <accessToken>
X-Ide-Token: <accessToken>
X-Cloudide-Token: <accessToken>
```

The credential itself was valid throughout — decoding the JWT gave a correct `exp` and `userId`. The prefix was the only problem.

## Failure 2 — missing client identity headers (401)

After fixing the auth scheme, the catalog endpoint still returned 401. Inspecting the header builder in the plugin source showed nine headers that must be present together:

```http
X-Ide-Token: <accessToken>
X-Cloudide-Token: <accessToken>
x-plugin-channel: icube-ai
x-app-id: <appId>
x-app-version-code: <numeric>
x-ide-version-code: <numeric>
x-custom-trace-id: <uuid hex>
x-flow-traceparent: 04-<traceId>-<traceId[:16]>-01
x-uid: <userId>
request-traffic-type: prod
User-Agent: Trae/<buildVersion>
```

Two of these deserve attention:

- **`x-app-version-code` must be numeric.** A dotted build string such as `2.3.68989` is rejected; the digits are stripped to `2368989`.
- **`x-flow-traceparent`** follows W3C Trace Context shape, but reuses the same trace id in the parent-id slot rather than generating a child span id.

Values come from the IDE's own `identity.json` (`machineId`, `deviceId`, `buildVersion`), not from the token.

With the full set, the catalog endpoint returned `200` and enumerated 11 models.

## Failure 3 — message content must be an array (400)

The chat endpoint then returned a 400 whose body finally named the problem:

```json
{"code":4001,"message":"bad request: json: cannot unmarshal string into Go struct field
 LLMRawMessage.messages.content of type []*idecopilot.LLMRawMessageContent"}
```

This is a Go type error surfaced from the server. `content` is `[]*LLMRawMessageContent`, not a string.

**Wrong:**

```json
{ "role": "user", "content": "hi" }
```

**Right:**

```json
{ "role": "user", "content": [{ "type": "text", "text": "hi" }] }
```

A further `function` field is required at the top level:

```json
{ "model": "glm-5.1", "function": "chat", "messages": [...], "stream": true }
```

Omitting it returns HTTP 200 whose stream carries a business error:

```
event:error
data:{"code":2001,"message":"[LLMUtilsChat.resolveByUsage] function is empty, cannot resolve model by usage"}
```

Note the distinction: **HTTP 200 does not mean success here.** The error arrives inside the SSE stream as an `event:error` frame. A client that only checks `response.ok` will report success and then hand the caller an empty answer.

## Verified request shape

```js
const headers = {
  Authorization: `Cloud-IDE-JWT ${accessToken}`,
  'X-Ide-Token': accessToken,
  'X-Cloudide-Token': accessToken,
  'x-plugin-channel': 'icube-ai',
  'User-Agent': `Trae/${identity.buildVersion}`,
  'x-app-id': appId,
  'x-app-version-code': identity.buildVersion.replace(/\D/g, ''),
  'x-ide-version-code': identity.buildVersion.replace(/\D/g, ''),
  'x-custom-trace-id': traceId,
  'x-flow-traceparent': `04-${traceId}-${traceId.slice(0, 16)}-01`,
  'request-traffic-type': 'prod',
  'x-uid': userId,
  'x-request-id': randomUUID(),
  'x-trae-request-id': randomUUID(),
  'Content-Type': 'application/json',
  Accept: 'text/event-stream',
}

const body = JSON.stringify({
  model: 'glm-5.1',
  function: 'chat',
  messages: [{ role: 'user', content: [{ type: 'text', text: 'say hi' }] }],
  stream: true,
})
```

Response, first frames:

```
data: {"model":"","session_id":"...","prompt_completion_id":0,...}
data: {"name":"llm_raw_chat_v2","preprocess_timing":31,"platform_first_token_timing":44,...}
data: {"response":"\"","reasoning_content":null,"tool_calls":null,...}
```

---

## Routing note: a configured account that never runs

The account was registered in the router and appeared in several routes:

```
deepseek-v4.1-flash:  workbuddy → trae → qoder
glm-5.3:              workbuddy → trae → qoder
```

Its telemetry showed **zero events** across 540 recorded requests, while the first target accumulated 434.

This is correct `failover` behaviour, not a fault: the strategy walks targets in order and only advances on failure. When the first target keeps succeeding, later targets are never exercised. The consequence is worth stating plainly — **an account can be configured, healthy, and completely unused**, and its credentials degrade silently (refresh tokens expire) without ever being noticed.

To exercise a secondary account deliberately, either reorder the targets or bind it as the sole account of a dedicated route.

## Interaction with crediting priority

A route may enable `preferExpiringCredits`, which reorders candidates by the soonest credit-package expiry within a horizon (default 7 days):

```js
const expiry = (c) => preferActive && !completed.includes(c.account.id)
  ? this.expiringAt(c.account, route)
  : Infinity
if (preferActive) candidates.sort((a, b) => expiry(a) - expiry(b))
```

**This overrides target order.** A secondary account whose credits lapse within the horizon jumps ahead of the primary target. That is the intended behaviour — spend credits before they expire — but it means route order is not a stable predictor of which upstream serves a request.

`expiringAt()` returns `Infinity` unless all of these hold:

- the balance snapshot is fresher than 120 seconds
- `unit === 'credits'`
- at least one package with `remaining > 0`, not unlimited, `expiresAt > now` and `expiresAt <= now + horizon`

Stale or missing balance data therefore disables the preference entirely rather than reordering on old numbers — a conservative choice worth preserving.

## Auto-update

Both Trae editions are VS Code forks, so updates are controlled by the Electron update service and the standard settings. Writing to `%APPDATA%\<Product>\User\settings.json`:

```json
{
  "update.mode": "none",
  "update.enableWindowsBackgroundUpdates": false,
  "extensions.autoUpdate": false
}
```

`update.mode: "none"` also disables the manual "check for updates" menu entry; future updates then require downloading an installer.

Two cautions when editing this file programmatically:

- Preserve unrelated keys. These settings files carry per-tool permission allowlists and trusted paths; a round-trip through an object serializer must not drop them.
- Write UTF-8 without BOM. Values may contain JSON-escaped non-ASCII paths (`D:\\浏览…`), and re-encoding through a different code page corrupts them.
