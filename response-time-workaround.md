# On-Prem LLM Response Time — Issue & Workaround

## Summary

Long prompts sent from Salesforce Prompt Builder to the on-prem **Qwen 3 (32B)** model were timing out. The cause is a fixed **120-second limit** in the Salesforce Einstein LLM Gateway. Long answers from the 32B model take longer than that to generate.

The workaround has two parts:

1. **An output cap in the on-prem worker**, so every Qwen response finishes well inside the 120 seconds.
2. **Prompt guidance**: ask for a word count that fits inside that cap, so answers are complete rather than cut off.

With both in place, a prompt that used to time out now returns in about **52 seconds**.

---

## The problem

| Observation | Detail |
|---|---|
| Symptom | "Error when invoking openConnector's chat completions endpoint" / timeout in Prompt Builder |
| Salesforce limit | The Einstein LLM Gateway waits at most **120 s** for a response. This can't be configured. |
| Model speed (test environment) | Qwen 3 32B generates about **11 tokens/second** (about 8 words/second) |
| Test prompt result | About 1,900 tokens in about **172 s**, which exceeds the 120 s limit |
| Side effect | Salesforce **automatically retries** a timed-out call. One click became up to 3 identical requests, and they queued behind each other on the single GPU. |

### Why "In 500 words" in the prompt doesn't prevent the timeout

A word count in the prompt is only a *request* to the model, and LLMs follow it loosely. In testing:

| Prompt asked for | Qwen actually wrote |
|---|---|
| 500 words | 807 – 1,112 words |

Only a hard **token limit** reliably controls response length and time.

---

## The workaround

### 1. Output cap in the on-prem worker (implemented)

The on-prem worker now limits Qwen responses to **600 tokens**. That is roughly 55 seconds of generation, leaving about 60 seconds of headroom under Salesforce's 120-second limit.

| Qwen output limit | Measured end-to-end time | Result |
|---|---|---|
| None (Salesforce sends 4,096) | ~172 s | Timeout |
| 1,000 tokens | 87 s | OK, but little headroom |
| **600 tokens (current)** | **52 s** | **OK** |

Other resilience changes made at the same time:

- **One request at a time**: the worker processes requests sequentially to match the single GPU, so requests no longer compete and time out together.
- **Stale-request skip**: if Salesforce has already given up on a request (older than 120 s), the worker discards it instead of spending GPU time on an answer nobody will receive.

### 2. Prompt guidance (action for prompt authors)

The cap stops generation at 600 tokens wherever the model happens to be. **The model doesn't know about the cap**, so if the prompt asks for more than about 450 words, the answer is cut off mid-sentence.

**Keep requested length at or under 250–300 words for Qwen prompts.** Also ask for the conclusion first.

| Prompt asks for | Tokens needed (approx.) | Fits in the 600-token cap? |
|---|---|---|
| 250 words | ~350 | Yes, with room to spare |
| 300 words | ~420 | Yes |
| 500 words | ~700 | No, cut off |

Example instruction to add to prompt templates:

> *Answer in under 250 words. Give the result first, then a brief explanation.*

---

## Options for longer answers

| Option | Trade-off |
|---|---|
| Use **Llama 3 (8B)** for long-form prompts | About 4× faster (about 49 tokens/s), so 2,000+ tokens fit in under a minute. Lower reasoning quality than Qwen 32B. |
| Raise the Qwen cap to about 800 tokens | Allows about 500-word answers at about 75 s per response. Less headroom when several users send requests at once. |
| Faster GPU hardware | Raises tokens/second, so larger caps fit in the same 120 s. |

---

## Key points

- The 120-second limit is set by Salesforce and can't be changed. Response time has to fit inside it.
- Word counts in prompts guide the model; token caps enforce the limit. Use both, and make them agree.
- The current setting is **Qwen: 600 tokens, about 52 s per response**. Prompts should ask for **250 words or fewer**.
