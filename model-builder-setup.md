# Registering the Agentforce Gateway in Salesforce Model Builder (BYO LLM)

How to register the **Agentforce Gateway** as a custom foundation model in Salesforce, so it
becomes selectable in Prompt Builder and the Agentforce model pickers.

> **Note:** Salesforce menu names shift between releases. Treat this as the shape of the
> process and confirm the exact labels against Salesforce's current
> **"Bring Your Own LLM" / "LLM Open Connector"** documentation before walking a customer through it.

---

## Prerequisite — Open Connector–compliant endpoint

The Agentforce Gateway must implement the **Salesforce LLM Open Connector** contract:
an OpenAI-compatible `POST /chat/completions` plus the connector's metadata endpoints.
This is already provided by `agentforce-gateway/src/main/resources/llm-open-connector.yaml`.
This is the contract Salesforce calls.

---

## Steps

### 1. Create a Named Credential for the endpoint
- Setup → **Named Credentials** (with an **External Credential** for the auth).
- Point it at the Agentforce Gateway base URL.
- Store the auth header Salesforce will send (the `api-key` / bearer token).
- This is where the credential the gateway is *meant* to validate lives — see **Auth** below.

### 2. Register the model in Model Builder
- Go to **Einstein Studio → Model Builder**.
- Choose **"Connect to a foundation model hosted outside Salesforce"** (Bring Your Own LLM).
- Select the **LLM Open Connector** connection type (may be shown as "OpenAI-compatible / custom endpoint").
- Provide:
  - the **endpoint** (via the Named Credential from step 1),
  - the **model name(s)** to expose (e.g. `qwen3:32b`),
  - any request defaults.

### 3. Save, activate, and test
- Use Model Builder's **test / validate** step — it calls the endpoint to confirm the contract.
- Confirm the call reaches the gateway (visible in the worker log flow).
- **Activate** the model once it passes.

### 4. Use the model
- Once active, it appears as a selectable model in **Prompt Builder** and in the
  **Agentforce model-override** dropdown ("Search models" list).
- Attach it to a **subagent / generation step** — **not** the Agent Router (the router needs a
  fast, certified model like HyperClassifier or GPT-4.1; routing through the on-prem model would
  add latency to every turn and may not be permitted for the router role).

---

## Auth
Salesforce **requires** credentials in the Named / External Credential even though the gateway
does not currently enforce them. Configuring real validation on the gateway to match this
credential is the production-hardening step (see `component-overview.md` → Future hardening).

## Limits that still apply
The **120s hard limit** and **4,096-token** cap live at the Einstein LLM Gateway and apply
regardless of how the model is registered. Model Builder configuration does not change them.

---

## To confirm with Salesforce (SE / support / live docs)
1. Exact current Model Builder path and labels for BYO LLM via Open Connector.
2. Whether a custom BYO LLM model can be selected for the **Agent Router / reasoning** role,
   or only for content generation.
3. Whether the 120s timeout is customer-configurable (assume **fixed / hard** unless told otherwise).
