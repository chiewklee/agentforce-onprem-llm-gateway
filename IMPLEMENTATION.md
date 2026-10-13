# Agentforce → On-Prem LLM Gateway: Implementation Guide

How to connect Salesforce Agentforce to LLMs running on-premises (Ollama) **without opening any inbound traffic** into the on-prem network. This guide covers both MuleSoft applications, the **Gateway** and the **Worker**. It also covers the two ways Salesforce can call them:

| Salesforce path | Used for | Status |
|---|---|---|
| **A. LLM Open Connector (BYOLLM)** | Prompt templates / Prompt Builder generation | Working |
| **B. "Connect an OpenAI model" (BYOLLM)** | Agent **router** (subagent selection) | **Experimental, undocumented.** Transport works end-to-end; routing reliability is still open (see [Known issues](#8-known-issues)). |

---

## 1. Architecture

```
Salesforce (Agentforce / Prompt Builder)
   │  Einstein Trust Layer → Einstein LLM Gateway   (120 s hard timeout)
   ▼
┌──────────────────────────────┐        CloudHub 2.0
│  agentforce-gateway          │  POST /api/chat/completions      (path A)
│  (HTTP → MQ → wait → HTTP)   │  POST /api/v1/chat/completions   (path B)
└──────┬───────────────▲───────┘
       │ publish       │ subscribe
       ▼               │
  [chat queue]   [chat-completed queue]        Anypoint MQ (cloud)
       │               ▲
       │ subscribe     │ publish                (outbound connections only)
┌──────▼───────────────┴───────┐
│  agentforce-ollama-worker    │        On-prem Mule runtime
│  (MQ → Ollama → MQ)          │
└──────────────┬───────────────┘
               ▼
        Ollama  /api/chat                       On-prem GPU host
```

- **Why async MQ:** the on-prem LLM network allows **no inbound connections**. The Worker makes only outbound calls: it pulls from MQ and pushes back to MQ.
- **Correlation:** the Gateway creates a `taskId` (UUID) for each request. The ID travels as an MQ message **property** in both directions.
- **The Gateway is stateless apart from a short-lived Object Store entry** that holds the response until the waiting HTTP request picks it up.

### Components and versions

| | Gateway | Worker |
|---|---|---|
| Runs on | CloudHub 2.0 | On-prem Mule EE standalone |
| Mule runtime | 4.11.x (`minMuleVersion` 4.11.0) | 4.11.x |
| Java | 17 | 17 |
| Connectors | HTTP 1.11.1, Anypoint MQ 4.0.23, Object Store 1.3.1 | HTTP 1.11.1, Anypoint MQ 4.0.23 |
| Files | `chat-gateway.xml`, `llm-open-connector.xml`, `llm-open-connector.yaml` (spec) | `ollama-worker.xml`, `log4j2.xml` |
| Config keys (`config.yaml`) | `mq.url`, `mq.clientId`, `mq.clientSecret` | `mq.*`, `ollama.host`, `ollama.port`, `ollama.model`, `ollama.defaultMaxTokens`, `ollama.apiKey` |

> Keep `mq.clientSecret` and `ollama.apiKey` out of source control. On CloudHub 2.0, use Secure Properties.

---

## 2. Gateway (`agentforce-gateway`)

### 2.1 HTTP endpoints (`llm-open-connector.xml`)

There are two listener flows on port 8081. Both call the **same** processing flow:

| Flow | Path | Caller |
|---|---|---|
| `llm-open-connector-main` | `POST /api/chat/completions` | Open Connector (path A) |
| `llm-openai-compat-main` | `POST /api/v1/chat/completions` | "Connect an OpenAI model" (path B). Salesforce appends **`/v1/chat/completions`** to the base URL. |

- **Base URL to configure in Salesforce (both paths):** `https://<your-app>.cloudhub.io/api`
- **Success response Content-Type** is `#[vars.responseContentType default "application/json"]`, so a streaming reply can switch it to `text/event-stream`.
- **Error handler:** `MULE:RETRY_EXHAUSTED` (no answer within 120 s) returns **504** `{"message":"LLM response timed out"}`.

### 2.2 Processing flow (`chat-gateway.xml`)

```
Mint taskId ─► Publish to chat queue ─► get-llm-response (poll) ─► Stream requested?
                                                                     ├─ yes → SSE
                                                                     └─ no  → JSON
```

1. **Mint taskId.** Sets `vars.taskId = uuid()`. It also records `vars.wantStream = payload.stream default false` and `vars.includeUsage = payload.stream_options.include_usage default false`.
2. **Publish** to `agentforce-llm-open-connector-chat` with these properties:
   ```dataweave
   {"taskId": vars.taskId, "publishedAt": now() as Number {unit: "milliseconds"} as String}
   ```
   `publishedAt` lets the Worker drop requests the Gateway has already given up on.
3. **Wait** (`get-llm-response` sub-flow): `until-successful maxRetries="120" millisBetweenRetries="1000"` around `os:retrieve key=taskId`. After retrieving, it runs **`os:remove`**, so answered entries don't build up.
4. **Return** the answer:
   - `stream != true`: returns the stored OpenAI `chat.completion` JSON as is.
   - `stream == true` **and** `payload.object == "chat.completion"`: replays the answer as an **OpenAI SSE stream**. It is sent as one burst, because the full answer has already arrived through MQ.
     ```
     data: {"object":"chat.completion.chunk", "choices":[{"index":0,"delta":{"role":"assistant","content":null,"tool_calls":[{"index":0,"id":"call_…","type":"function","function":{"name":"go_to_order","arguments":"{…}"}}]},"finish_reason":null}], …}

     data: {"object":"chat.completion.chunk", "choices":[{"index":0,"delta":{},"finish_reason":"tool_calls"}], …}

     data: {"object":"chat.completion.chunk", "choices":[], "usage":{…}}     ← only if include_usage

     data: [DONE]
     ```
     Worker errors (`object: "error"`) are always returned as plain JSON.

**Gotcha:** inside `<ee:set-variable>`, the body is a DataWeave expression. To set a literal string, wrap it in quotes: `<![CDATA["text/event-stream"]]>`. Without the quotes, DataWeave reads `text/event-stream` as a division expression and fails at runtime, not at build time.

### 2.3 Response subscriber flow (`post-chat-llm-response`)

- Subscribes to `agentforce-llm-open-connector-chat-completed`.
- Stores the payload in the Object Store under `attributes.properties.taskId`.

With more than one Gateway replica, use a **persistent / shared Object Store** so that whichever replica receives the response can serve the waiting request.

---

## 3. Worker (`agentforce-ollama-worker`)

### 3.1 Subscriber and concurrency (the most important settings)

```xml
<flow name="ollama-worker" maxConcurrency="1">
  <anypoint-mq:subscriber destination="agentforce-llm-open-connector-chat"
      acknowledgementTimeout="3" acknowledgementTimeoutUnit="MINUTES">
    <anypoint-mq:subscriber-type>
      <anypoint-mq:prefetch maxLocalMessages="1" />
    </anypoint-mq:subscriber-type>
  </anypoint-mq:subscriber>
```

| Setting | Why |
|---|---|
| `maxConcurrency="1"` | A single-GPU Ollama host runs **one inference at a time**. Higher concurrency makes calls queue up inside Ollama until they hit `HTTP:TIMEOUT` and get redelivered, which snowballs. We saw 54 timeouts before this fix. |
| `maxLocalMessages="1"` | The connector default prefetch **locks up to 30 messages** while the flow processes one. Locks then expired and messages were delivered twice. |
| `acknowledgementTimeout=3 min` | Longer than Ollama's 120 s response timeout, so a message is only released back to the queue if the Worker actually dies. |

`numberOfConsumers` is **not** a valid attribute on this connector version (the build fails with `cvc-complex-type`). Use `maxConcurrency` on the flow instead.

### 3.2 Drop stale requests

```dataweave
requestAgeSeconds = if (isEmpty(attributes.properties.publishedAt)) -1
                    else (now_ms - publishedAt) / 1000
```

If the request is more than **120 s** old, the Gateway has already returned 504 and nobody is waiting. The Worker logs `SKIP` and acks the message without calling Ollama. This matters because **Salesforce retries timed-out calls**, so one user click can become 3 queued requests.

### 3.3 Request mapping: OpenAI / Open Connector → Ollama `/api/chat`

| Concern | Implementation |
|---|---|
| **Model name** | Salesforce never sends an Ollama model ID. It sends `""` or the Model Builder **display name** (path A), or an OpenAI model ID such as `gpt-4.1-2025-04-14` (path B). `resolveModel()`: empty → `${ollama.model}`; `modelAliases` lookup; a valid Ollama ID passes through; anything unknown → default. |
| **Thinking** | `think: payload.enable_thinking default false`. Salesforce never sends it, so thinking is off by default to keep latency down. |
| **Output cap** | `num_predict = min(max_tokens or ollama.defaultMaxTokens, maxTokensCap[model])`, with `maxTokensCap = {"qwen3:32b": 600}`. |
| **Content parts** | Path B sends `content: [{"type":"text","text":"…"}]`. Ollama only accepts a string (400 otherwise), so the text parts are joined. |
| **Tools** | `tools` is passed through unchanged (Ollama accepts the OpenAI shape). `tool_choice` and `parallel_tool_calls` have no Ollama equivalent and are dropped. |
| **Tool-call history** | OpenAI sends `function.arguments` as a **JSON string**. Ollama requires an **object**, and otherwise returns 400 `Value looks like object, but can't find closing '}' symbol`. It is converted with `read(args, "application/json")`, with empty args → `{}`. |
| **Streaming** | The Worker always calls Ollama with `stream: false`. Streaming is handled by the Gateway (§2.2). |

Core of the request transform:

```dataweave
var modelAliases = {
    "On-Prem Qwen": "qwen3:32b", "On-Prem Llama": "llama3:8b", "On-Prem Gemma": "gemma4:26b",
    "gpt-4.1-2025-04-14": "qwen3:32b", "gpt-4-0613": "qwen3:32b"
}
fun textOf(c) = if (c is Array) ((c filter ($."type" == "text")) map $.text) joinBy "\n" else c
fun argsObj(a) = if (a is String) (if (isEmpty(a)) {} else read(a, "application/json")) else (a default {})
fun toOllamaMessage(m) = do {
    var base = (m - "content") ++ {content: textOf(m.content) default ""}
    ---
    if (isEmpty(m.tool_calls)) base
    else (base - "tool_calls") ++ {tool_calls: m.tool_calls map ((tc) -> (tc - "function") ++
            {function: {name: tc.function.name, arguments: argsObj(tc.function.arguments)}})}
}
---
{
    model: resolvedModel,
    messages: payload.messages map toOllamaMessage($),
    (tools: payload.tools) if (!isEmpty(payload.tools)),
    think: payload.enable_thinking default false,
    stream: false,
    options: {
        (temperature: payload.temperature) if payload.temperature?,
        num_predict: /* capped as above */
    }
}
```

**Gotcha:** DataWeave `default` only replaces **null or missing** values, not `""`. Use `isEmpty()` to detect Salesforce's empty model name.

### 3.4 Response mapping: Ollama → OpenAI `chat.completion`

- `content` ← `message.content`. `reasoning` ← `message.thinking` (only when present).
- `tool_calls` ← Ollama's `message.tool_calls`, converted to the OpenAI shape: an `id` is added if missing (`call_…`), `type: "function"`, and `arguments` is written back as a **JSON string**.
- `finish_reason` is `"tool_calls"` when tool calls are present, otherwise Ollama's `done_reason` (`stop` / `length`).
- `usage` ← `prompt_eval_count` / `eval_count`.
- The result is published to `agentforce-llm-open-connector-chat-completed` with the same `taskId` property.

### 3.5 Error handling

`on-error-continue type="ANY"` does three things:
- logs `ERROR taskId=… errorType=…`;
- publishes an OpenAI-shaped error (`{"object":"error","error":{…}}`) to chat-completed, so the caller gets an immediate error instead of waiting for the 504;
- **acks** the message, so it isn't redelivered in a loop.

### 3.6 Logging

| Line | Purpose |
|---|---|
| `RAW taskId=… payload=…` | The request **exactly as Salesforce sent it**. Essential for discovering new fields (tools, stream, content parts). |
| `REQUEST taskId=…` | The request after it has been reshaped for Ollama. |
| `RESPONSE` / `ERROR` / `SKIP` | Outcome of each message. |

The payloads may contain customer data, so treat these logs as sensitive. For wire-level debugging, `org.mule.service.http.impl.service.HttpMessageLogger` at `DEBUG` shows raw HTTP traffic. **It also logs the `Authorization` header**, so enable it only temporarily.

---

## 4. Salesforce configuration

### Path A: Prompt templates (Open Connector)

1. Einstein Studio → Model Builder → **Add Foundation Model → LLM Open Connector**. Set the URL to `https://<your-app>.cloudhub.io/api` and add an API key header.
2. Create a configured model with a clear display name (e.g. "On-Prem Qwen"), activate it, and add that display name to the Worker's `modelAliases`.
3. The model appears in Prompt Builder under **Custom → Models**. Salesforce populates that list from its own registrations; there is **no `GET /models`** call.

A BYOLLM model on this path is **not eligible** for the agent's reasoning or router role, only for generation.

### Path B: Agent router ("Connect an OpenAI model", undocumented workaround)

1. Model Builder → **Connect an OpenAI model**. Set the base URL to `https://<your-app>.cloudhub.io/api` and choose model version **GPT-4 / GPT-4.1**. Salesforce calls `/api/v1/chat/completions`.
2. Select that model for the agent.
3. Observed request from the router: `tools` (one `go_to_<subagent>` per subagent, plus actions), `tool_choice: "auto"`, `parallel_tool_calls: true`, `stream: true`, `stream_options.include_usage: true`, **no `max_tokens`**, and the full conversation history, including earlier tool calls and tool results. Prompt size was about 2,300–2,500 tokens.

This is unsupported by Salesforce and may stop working in any release. Validate it before relying on it.

### What we changed to make Path B work

Salesforce thinks it's calling GPT-4.1 and sends a standard OpenAI Chat Completions request. The Gateway and Worker translate that request for Ollama/qwen, then translate the answer back.

**Gateway (CloudHub 2.0)**

1. **Added a second endpoint, `POST /api/v1/chat/completions`** (§2.1). Salesforce appends `/v1/chat/completions` to the base URL, so without this endpoint the call returned 404. It reuses the existing flow: publish to MQ, then wait for the reply.
2. **Added streaming (SSE) responses** (§2.2). The router sends `stream:true` and only accepts a stream. A plain JSON reply made Salesforce retry and then fail with `system_error`. When streaming is requested, the Gateway replays the finished answer in OpenAI's chunk format and ends with `[DONE]`. It includes usage if Salesforce asked for it, and sets `Content-Type: text/event-stream`.

**Worker (on-prem)**

3. **Mapped the model names** (§3.3). `gpt-4.1-2025-04-14` and `gpt-4-0613` both go to `qwen3:32b`.
4. **Flattened content parts.** OpenAI sends `content` as `[{"type":"text","text":"…"}]`, but Ollama only accepts a string (400 otherwise), so the Worker joins the text parts.
5. **Converted tool-call arguments in the conversation history.** OpenAI sends them as a JSON string, but Ollama requires an object (another 400), so the Worker parses them.
6. **Passed the tools through.** Ollama accepts the same format for `tools` (the `go_to_<subagent>` entries and actions). `tool_choice` has no Ollama equivalent, so the Worker drops it.
7. **Converted Ollama's reply back to OpenAI format** (§3.4). Tool calls get an `id`, their arguments are written back as a JSON string, and `finish_reason` is set to `"tool_calls"`. That tells the router which subagent to route to.
8. **Kept responses fast.** Thinking is off, output is capped at 600 tokens, and requests the Gateway has already given up on (older than 120 s) are skipped (§3.2).
9. **Added a `RAW` log of each request exactly as Salesforce sent it** (§3.6). This is how we discovered each of the formats above.

**Status:** when we replayed a real router request captured from Agentforce, qwen correctly picked `go_to_order`. We still need to confirm the whole flow end-to-end in Agentforce, because qwen sometimes answers in text instead of calling a tool (see [Known issues](#8-known-issues)).

---

## 5. Performance and limits

| Model | Throughput (single GPU, test Mac) | Notes |
|---|---|---|
| qwen3:32b | ~11 tokens/s | Best reasoning. Needs the output cap. |
| llama3:8b | ~49 tokens/s | 4–5× faster. Unreliable on logic and reasoning. |

- **Salesforce Einstein LLM Gateway timeout: 120 s, not configurable.** Every call must finish within it, including model load, prompt evaluation and MQ round trips.
- **Salesforce retries after a timeout**, which is why the Worker has the stale-request skip.
- **The token cap is the only hard limit.** "Answer in N words" in a prompt is only guidance: qwen wrote 807–1,112 words when asked for 500. With a 600-token cap, ask for **≤ 250 words** so answers finish before the cap instead of being cut off (`finish_reason: length`).
- Measured end-to-end times through the Gateway:

| Request | Time |
|---|---|
| qwen, 600-token capped generation | ~52 s |
| qwen, router tool selection (~2,460-token prompt, ~26-token output) | 2–15 s warm, ~33 s when the model has to be loaded first |
| llama, short answer | ~6 s |

- Ollama unloads idle models after about 5 minutes (the `keep_alive` setting). The first call afterwards pays the load time.

---

## 6. Testing

```bash
# Path A: Open Connector
curl -X POST https://<your-app>.cloudhub.io/api/chat/completions \
  -H "Content-Type: application/json" -H "api-key: <key>" \
  -d '{"model":"On-Prem Qwen","messages":[{"role":"user","content":"Reply with just: OK"}],"max_tokens":50}'

# Path B: OpenAI-style, streaming (as the agent router calls it)
curl -N -X POST https://<your-app>.cloudhub.io/api/v1/chat/completions \
  -H "Content-Type: application/json" -H "Authorization: Bearer <key>" \
  -d '{"model":"gpt-4.1-2025-04-14","stream":true,"stream_options":{"include_usage":true},
       "messages":[{"role":"user","content":[{"type":"text","text":"Reply with just: OK"}]}]}'
```

To reproduce a real router call, copy a `RAW` payload from the Worker log and POST it to `/api/v1/chat/completions`.

A good known-answer test prompt: *"Swap two letters in 'UNITED' to form a valid English word"*. The only answer is **UNTIED**. Avoid unsolvable puzzles: no two-letter swap of "SALESFORCE" produces a dictionary word.

---

## 7. Extending the design

### 7.1 Policies in front of the Gateway

The current setup has no gateway or policies in front of the Agentforce Gateway app. There are two ways to add them:

| Option | Notes |
|---|---|
| **API Manager policies on the Mule app** (autodiscovery) | Simplest: no extra component, because the app already runs on CloudHub 2.0. Use it for client-ID enforcement, OAuth, rate limiting and IP allowlisting. |
| **Omni Gateway in front of the app** | Adds a dedicated policy tier, including LLM/AI-specific policies, and can front other AI endpoints too. |

Either option closes the "Gateway does not authenticate callers" gap listed in [Known issues](#8-known-issues). Neither changes the on-prem side: both sit at the cloud ingress.

### 7.2 On-prem and cloud LLMs from the same Worker

The Worker can call cloud-hosted LLMs as well as on-prem Ollama. Because it only makes **outbound** calls (to Anypoint MQ and to the model), the on-prem network still never needs to accept inbound traffic. That is the core benefit of this design.

What changes if you add a cloud LLM:

- **Egress:** the Worker needs outbound HTTPS to the provider, as well as to Anypoint MQ (443).
- **Routing:** add a branch per target. Ollama requests need the `/api/chat` mapping described in §3.3. OpenAI-compatible providers can mostly take the request as Salesforce sent it.
- **Data residency:** requests sent to a cloud model leave the on-prem network.
- **Why go through the Worker at all?** Salesforce can connect cloud LLMs directly. Routing them through the Worker gives you one control point:
  - choose on-prem or cloud per request (e.g. sensitive data stays on-prem);
  - fall back to another model when one is slow or unavailable;
  - keep logging and token caps consistent across models.

### 7.3 Trade-offs of the async design

- The MQ round trip adds some latency on top of model time.
- Responses are not streamed token by token; the Gateway replays each complete answer as a single SSE burst.
- Every call, including the MQ hops, must finish within Salesforce's 120 s LLM Gateway timeout.

---

## 8. Known issues

| Issue | Status / mitigation |
|---|---|
| **Router: qwen sometimes answers in text instead of calling a tool** (e.g. "what can you do?" → prose instead of `go_to_GeneralFAQ`). The router then errors with *"Error behavior threshold exceeded: system_error"* after one retry. | **Open.** Ollama has no `tool_choice: "required"`. Options: add a system instruction to always call exactly one tool when `tools` are present; retry once with a stronger instruction if the reply has no tool call; or constrain the output format. |
| **Router: streamed tool-call responses not yet confirmed accepted by Salesforce.** | The Gateway emits valid OpenAI SSE (verified with curl). Needs an end-to-end Agentforce run in which a tool call is returned. |
| Router latency per turn | Every agent turn involves at least one router call. qwen takes about 2–15 s warm. The model's reasoning-quality fit for routing hasn't been assessed yet. |
| Gateway does not authenticate callers | Add an API Manager client-ID or OAuth policy and set up the Salesforce side with a Named Credential before production. |
| Error responses return HTTP 200 with an `error` body | Acceptable for Salesforce. Consider mapping to 502 for other clients. |
| Trust Layer | Path A goes through the Einstein Trust Layer. Custom agent actions / External Services that call the Gateway directly bypass it. |
