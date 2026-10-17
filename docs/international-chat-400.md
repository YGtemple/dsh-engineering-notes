# Diagnosing an empty-body HTTP 400 on a third-party gateway

## Symptom

A DSH plugin (`dsh-multi-router`) proxies chat completions for two regions of the same vendor:

| Provider ID | Login domain | Region |
| --- | --- | --- |
| `workbuddy` | `codebuddy.cn` | CN |
| `workbuddy-ai` | `workbuddy.ai` | International |

The CN path worked. The international path failed on **every** chat turn:

- Catalog endpoint `GET /v3/config` → **200**, models enumerated correctly
- Chat endpoint `POST /v2/chat/completions` → **400**, response body **empty**

An empty body is significant: the vendor's own error envelope always carries `{code, msg, ...}`. An empty body means the request was rejected by the gateway before reaching the application layer. No model was involved.

The plugin's telemetry recorded the failure as `status=400, outcome=failed` with no payload.

## Hypotheses tested and rejected

Each of these was plausible and each was wrong.

### 1. User-Agent shape

The vendored source contains a comment (from an earlier research round) stating that the international gateway splits on User-Agent and that the space form `WorkBuddy AI/<v>` is rejected with `400 / code 12403`, while the terse form `WorkBuddyAI/<v>` succeeds.

The catalog path sends the terse form; the chat path builds:

```ts
const product = region === 'global' ? 'WorkBuddy AI' : 'WorkBuddy'
const parts = [`WorkBuddy/${identity.clientVersion}`, `${product}/${identity.clientVersion}`]
```

That is the space form. This looked like a strong candidate.

**Test:** request the catalog endpoint with each UA variant.

```
WorkBuddyAI/5.5.2               → 200 (26161 bytes)
WorkBuddy/5.5.2 WorkBuddy AI/5.5.2 → 200 (26161 bytes)
```

Both succeeded. The UA hypothesis does not explain a 400 on chat.

### 2. Stale app version

`resolveAppVersion()` returns a fallback when the app cannot be located:

```ts
async function installedAppVersion() {
  if (process.platform !== 'darwin') return void 0   // macOS only
  ...
}
```

On Windows this never resolves an installed version, so the request ships `5.5.2` while the installed client is `5.8.4`.

**Rejected by the source comment itself**, which states the gateway ignores the version component when splitting the UA — `CLI/1.0.0`, `CLI/99.0.0`, and the real version all return the same document. The constant is a shape requirement, not a currency claim.

Confirmed empirically: the catalog returned identical bytes for both UA forms.

### 3. Wrong endpoint path

**Test:** probe three candidate paths.

```
/v2/chat/completions       → 400
/v1/chat/completions       → 404
/api/v2/chat/completions   → 404
```

The 404s confirm `/v2/chat/completions` is the correct route. Rejected.

### 4. Invalid model identifier

Model IDs were read from the live catalog response rather than assumed, then used verbatim. Rejected.

### 5. Expired or malformed credential

Decoded the JWT from the plugin's credential copy:

```json
{ "iss": "https://www.workbuddy.ai/auth/realms/copilot",
  "exp": 1823602783796, "sub": "1e600eb7-..." }
```

`exp` corresponds to 2027-07-27 — not expired. The catalog call using the same token succeeded. Rejected.

## Isolating the cause

Two independent dimensions remained: **the request body** and **the header set**. Reduced to a 2×2 matrix, with `model=hy3` and the token held constant:

| Leading `system` message | `X-Conversation-*` headers | Result |
| --- | --- | --- |
| yes | no | **400** (empty) |
| yes | yes | **200** (7.6 KB SSE) |
| no | no | 400 |
| no | yes | 400 |

**Both conditions are required.** Neither alone is sufficient.

The body-side requirement was already satisfied in production — `prepareInternationalChatBody()` prepends a system prompt when the first message is not one. The header-side requirement was missing entirely.

## Root cause

The vendored transport builds only the CN header set:

```ts
function chatHeaders(credential, userAgent, clientVersion) {
  const headers = {
    ...commonHeaders(credential),
    'X-IDE-Type': 'WorkBuddy',
    'X-Product': 'SaaS',
  }
  return headers
}
```

A sibling plugin (`dsh-workbuddy-connect`, from which this transport was originally derived) already handles it:

```ts
...international ? conversationHeaders() : {}

function conversationHeaders() {
  return {
    'X-Conversation-ID': randomUUID(),
    'X-Conversation-Request-ID': randomUUID(),
    'X-Conversation-Message-ID': randomUUID(),
  }
}
```

The branch was dropped during vendoring. The global gateway requires all three headers per turn.

## Fix

Four changes to `upstream/workbuddy/src/upstream.ts`, +27 −4 lines:

1. `import { randomUUID } from 'node:crypto'`
2. New `conversationHeaders()` producing a per-turn UUID triple
3. `chatHeaders()` gains `international = false`; the triple is injected only when the region is global
4. Both call sites pass the region flag

Note that `probeEffort` also calls `chatHeaders()`. Patching only `chatStream` leaves probing broken; the call there uses a differently-named local (`international` rather than `region`), which is easy to miss.

## Verification

Post-fix, per model, live account, Windows:

```
hy3                  200
gpt-6-luna           200
gemini-3.8-flash     200
deepseek-v4.1-flash  200
kimi-k3              200
glm-5.3              200
grok-4.7             200
step-5-preview       429   (rate limit, not 400 — the request now reaches the model layer)
```

The 429 is itself evidence the fix works: the request cleared the gateway.

Production telemetry confirmed the same transition — six consecutive `status=400` before, `status=200 completed` after.

## The same request body, two different outcomes

During verification, `gpt-6-luna` produced `400` on some `max_tokens` values and `200` on others, inconsistently across repetitions:

```
7:16 → 400    7:16 → 200   (same value, different runs)
7:18 → 200    7:32 → 400
```

This was initially mistaken for a model-specific constraint. It was not. The vendor's own desktop client was mid-update (`WorkBuddy-Setup-5.8.3.exe` running), so the backend was restarting and requests were landing on healthy and unhealthy instances non-deterministically.

**Lesson:** a non-deterministic result across identical requests is a signal about the environment, not about the request. Treating it as a parameter constraint produces a wrong conclusion — one that would have shipped as "this model only accepts `max_tokens=16`".

## Upstream contribution

Submitted as [AndroidSix/dsh-multi-router#2](https://github.com/AndroidSix/dsh-multi-router/pull/2): one commit, +27 −4, no dependency changes, CN path untouched.

A standalone mirror with the fix applied is maintained at [YGtemple/dsh-multi-router-intl](https://github.com/YGtemple/dsh-multi-router-intl) for use until the patch is released upstream.
