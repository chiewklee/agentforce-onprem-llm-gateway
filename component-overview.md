# Agentforce On-Prem LLM Gateway — Component Overview

The solution connects Salesforce Agentforce to my on-premise LLMs (Ollama) through an
asynchronous, message-based architecture. This keeps the on-premise LLM network fully
closed to inbound traffic — the on-premise worker only ever makes *outbound* connections. Three
components make this work:

## 1. Gateway *(CloudHub 2.0, MuleSoft — cloud)*

The public entry point. It exposes a standard Salesforce LLM Open Connector–compatible
REST API (`POST /api/chat/completions`) that Agentforce calls directly.

- Accepts the inbound chat request and assigns a unique correlation id (`taskId`).
- Publishes the request onto the message queue and then waits (polls) for the matching response.
- Returns the completed LLM response to Agentforce in OpenAI-compatible format, or a clean
  timeout/error message if the model takes too long.
- Shields callers from the on-premise infrastructure — Agentforce never talks to the LLM
  network directly.

## 2. Message Broker — Anypoint MQ *(managed message broker — cloud)*

The asynchronous messaging layer that decouples cloud from on-premise. Two queues:

- **chat queue** — inbound requests published by the Gateway, waiting to be picked up.
- **chat-completed queue** — finished responses published back by the Worker.
- Every message carries the `taskId` property so requests and responses stay correlated.
  This queue-based design is what allows the on-premise side to pull work outbound, with
  **no inbound firewall openings** into the LLM network.
- **The broker is pluggable.** Anypoint MQ is the default, but the same asynchronous pattern
  works with **other message brokers such as Apache Kafka** (or similar). The Gateway and
  Worker interact with the broker through a well-defined publish/subscribe contract, so the
  broker can be swapped to match a customer's existing messaging platform, governance, or
  infrastructure preferences — without changing the overall architecture.

## 3. Worker *(MuleSoft, on-premise)*

The on-premise MuleSoft application that runs in the same local environment as the LLMs and does the actual request handling.

- Subscribes (outbound) to the chat queue and pulls requests as they arrive.
- Translates the request into Ollama's native `/api/chat` format — including model selection,
  token limits, temperature, and the optional reasoning/"thinking" toggle — and calls the
  local LLM (e.g. qwen3:32b, llama3:8b).
- Reshapes the model's native response back into the standard OpenAI-compatible format.
- Publishes the result to the chat-completed queue, keyed by the same `taskId`.

> **Note on the current setup:** In this deployment, **Ollama is running locally on my Mac**,
> serving the on-premise LLMs (qwen3:32b, llama3:8b) that the Worker calls. This demonstrates
> the "no inbound access" model end-to-end — my Mac never needs to accept any inbound
> connection; the Worker reaches out to the broker and to Ollama entirely from inside the
> local environment.

## In short

Agentforce → **Gateway** (REST API) → **Message Broker** (Anypoint MQ, or Kafka / another
broker) → **Worker** (on-premise) → **Ollama (running locally on my Mac)**, and back again —
correlated end-to-end by `taskId`, with zero inbound access to my Mac.

## Future hardening (recommended before production)

- **Gateway authentication** — enforce API-key or OAuth validation at the Gateway. Today the
  `api-key` header is accepted but not validated; the current setup is intended for demonstration.
- Other production considerations as needed: rate limiting, request/response logging and
  monitoring, and dead-letter handling on the message broker.
