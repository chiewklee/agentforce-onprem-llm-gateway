# Agentforce On-Prem LLM Gateway

Use LLMs running in your own data centre (served by [Ollama](https://ollama.com)) from Salesforce Agentforce, in Prompt templates and, experimentally, in the Agent Router, **without opening any inbound access to your network**.

Two MuleSoft applications make this work:

| App | Runs on | What it does |
|---|---|---|
| [`agentforce-gateway`](agentforce-gateway) | CloudHub 2.0 | Receives Agentforce's LLM calls, puts them on Anypoint MQ, waits for the answer and returns it to Salesforce |
| [`agentforce-ollama-worker`](agentforce-ollama-worker) | On-prem Mule runtime | Pulls requests from Anypoint MQ, calls the local LLM, and publishes the answer back. It only makes **outbound** connections |

```
Agentforce ──HTTPS──► agentforce-gateway (CloudHub 2.0)
                          │ publish            ▲ subscribe
                          ▼                    │
                    [chat queue]      [chat-completed queue]     Anypoint MQ
                          │ subscribe          ▲ publish
                          ▼                    │                 (outbound only)
                    agentforce-ollama-worker (on-prem) ──► Ollama
```

**How it works, design decisions, limits and known issues:** see **[IMPLEMENTATION.md](IMPLEMENTATION.md)**.

---

## Quick start

### 1. Prerequisites

- An Anypoint Platform org with **Anypoint MQ** and **CloudHub 2.0**
- An on-prem host with a **Mule EE runtime 4.11.x** (standalone or hybrid) that can reach Anypoint MQ over HTTPS (443)
- **Ollama** on a host the worker can reach, with the models pulled, e.g. `ollama pull qwen3:32b`
- **JDK 17** and **Maven 3.9+** for building
- A Salesforce org with Agentforce and Einstein Studio (Model Builder)

### 2. Create the Anypoint MQ queues

In Anypoint MQ, in the environment you'll deploy to, create two standard queues with **exactly these names**:

| Queue | Direction |
|---|---|
| `agentforce-llm-open-connector-chat` | gateway → worker (requests) |
| `agentforce-llm-open-connector-chat-completed` | worker → gateway (responses) |

Then create an MQ **client app** and note its client ID, client secret and the MQ URL for your region. Both apps use the same client app.

### 3. Configure

Each app reads `src/main/resources/config.yaml`, which is git-ignored. Create it from the template in each project:

```bash
cp agentforce-gateway/src/main/resources/config.example.yaml       agentforce-gateway/src/main/resources/config.yaml
cp agentforce-ollama-worker/src/main/resources/config.example.yaml agentforce-ollama-worker/src/main/resources/config.yaml
```

| App | Keys |
|---|---|
| Gateway | `mq.url`, `mq.clientId`, `mq.clientSecret` |
| Worker | `mq.url`, `mq.clientId`, `mq.clientSecret`, `ollama.host`, `ollama.port`, `ollama.model` (default model), `ollama.defaultMaxTokens`, `ollama.apiKey` (sent as a Bearer token; any value works if your Ollama has no auth in front of it) |

For CloudHub 2.0, keep the file but leave the credentials as placeholders, and set the real values as **secure properties** in Runtime Manager. They override the file.

Also check in the worker's `ollama-worker.xml`:

- `modelAliases`: maps the model display names you register in Salesforce to Ollama model IDs.
- `maxTokensCap`: per-model output caps that keep responses inside Salesforce's 120 s timeout. Tune these for your hardware.

### 4. Build

```bash
export JAVA_HOME=<path-to-jdk-17>
mvn -f agentforce-ollama-worker/pom.xml clean package
mvn -f agentforce-gateway/pom.xml clean package
```

> **Gateway build note:** `agentforce-gateway/pom.xml` has a dependency on the spec asset `salesforce-llm-open-connector` (`<classifier>oas</classifier>`), with the placeholder `groupId` `YOUR_ANYPOINT_ORG_ID`, so the build can't download it as is. Either publish [`llm-open-connector-spec`](llm-open-connector-spec) to **your** Exchange (set the same org ID in its `exchange.json`) and replace the placeholder with your org ID, or remove that `<dependency>` block. The flows don't use it.

### 5. Deploy, in this order

1. **Worker (on-prem):** copy `agentforce-ollama-worker/target/agentforce-ollama-worker-1.0.0-mule-application.jar` into your runtime's `apps/` folder, or deploy it through Runtime Manager for hybrid. Check the log for `ollama-worker` starting without errors.
2. **Gateway (CloudHub 2.0):** in `agentforce-gateway/pom.xml`, set `<environment>` and `<target>` in `<cloudhub2Deployment>` to match your org. Then deploy with the Mule Maven plugin (`mvn deploy -DmuleDeploy`, using your Connected App or credentials) or upload the jar in Runtime Manager. Enable a public endpoint and note the app URL.
3. **Smoke test** the whole chain:

   ```bash
   curl -X POST https://<your-gateway>.cloudhub.io/api/chat/completions \
     -H "Content-Type: application/json" -H "api-key: test" \
     -d '{"model":"","messages":[{"role":"user","content":"Reply with just: OK"}],"max_tokens":20}'
   ```

   You should get an OpenAI-style `chat.completion`. The worker log shows `RAW`, `REQUEST` and `RESPONSE` lines for the same `taskId`.

### 6. Connect Salesforce

In Einstein Studio → Model Builder, use the base URL `https://<your-gateway>.cloudhub.io/api`:

- **Prompt templates:** *Add Foundation Model → LLM Open Connector*. Give the configured model a display name and add that name to `modelAliases` in the worker.
- **Agent Router (experimental, undocumented workaround):** *Connect an OpenAI model*, model version GPT-4 / GPT-4.1.

Step-by-step details and what Salesforce sends on each path: [IMPLEMENTATION.md §4](IMPLEMENTATION.md#4-salesforce-configuration).

---

## Before production

- **The gateway does not authenticate callers yet.** Apply a client-ID or OAuth policy through API Manager or Omni Gateway, and use a Named Credential on the Salesforce side.
- With more than one gateway replica, use a persistent / shared Object Store.
- Read [Performance and limits](IMPLEMENTATION.md#5-performance-and-limits) and [Known issues](IMPLEMENTATION.md#8-known-issues), especially the 120 s timeout and the status of the Agent Router workaround.

## Repository layout

| Path | Contents |
|---|---|
| `agentforce-gateway/` | CloudHub 2.0 gateway app |
| `agentforce-ollama-worker/` | On-prem worker app |
| `llm-open-connector-spec/` | Salesforce LLM Open Connector OAS spec (optional; for Exchange / API Manager) |
| `IMPLEMENTATION.md` | Full implementation guide |
| `component-overview.md`, `diagram*.html`, `diagram.md` | Architecture overview and diagrams |
| `model-builder-setup.md` | Salesforce Model Builder setup notes |
| `response-time-workaround.md` | Response-time guidance for slower models |
