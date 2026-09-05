# Notebooks

Runnable companions to the [cheatsheets](../cheatsheets/). The notebooks hold **code only** — every
explanation, caveat, and exam trap lives here, so the `.ipynb` files stay short and diff cleanly.

| Notebook | Guide |
|---|---|
| `01-multi-turn-conversation.ipynb` | [Accessing Claude with the API](#01--accessing-claude-with-the-api) |
| `02` *(planned)* | Tool use and the agentic loop |
| `03` *(planned)* | Structured outputs |
| `04` *(planned)* | Prompt caching |
| `05` *(planned)* | MCP servers and clients |

---

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter lab
```

### Credentials

The SDK resolves credentials in this order, first match wins:

1. `ANTHROPIC_API_KEY`
2. `ANTHROPIC_AUTH_TOKEN`
3. An OAuth profile from `ant auth login`, stored under `~/.config/anthropic/`

So a bare `anthropic.Anthropic()` works with no environment variable set, provided you have logged
in with the CLI. Either of these works:

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

```bash
ant auth login
ant auth status   # shows which credential source is active
```

**Never hardcode a key in a notebook.** It ends up in git history, and history is hard to scrub.

### Before committing

Clear outputs so diffs stay readable:

```bash
jupyter nbconvert --clear-output --inplace notebooks/*.ipynb
```

---

## 01 · Accessing Claude with the API

Everything goes through a single endpoint: `POST /v1/messages`. Tools, structured outputs, thinking,
and caching are all *parameters* on that one endpoint, not separate APIs. That is the single most
useful thing to have straight going into the exam.

### 1. Install and authenticate

```bash
pip install anthropic
```

Prefer the zero-arg `anthropic.Anthropic()` constructor over passing `api_key=` — it picks up
whichever credential source is configured (see [Credentials](#credentials) above).

### 2. Your first request

Three required parameters: `model`, `max_tokens`, and `messages`.

`max_tokens` is a hard ceiling on the response — hit it and the output is cut off mid-sentence, so
don't lowball it. Sensible defaults: **~16000** for non-streaming (stays under the SDK's HTTP
timeout) and **~64000** when streaming. Drop it only for a reason, e.g. `256` for classification.

### 3. Reading the response

`response.content` is a **list of content blocks**, not a string. A block can be `text`, `thinking`,
`tool_use`, and more. Reaching for `response.content[0].text` works in simple cases but breaks the
moment thinking or tool use is in play — a thinking block has no `.text`. Always branch on
`block.type`.

### 4. System prompts

The system prompt is a **top-level parameter**, not a message with `role: "system"`. This trips
people up coming from other APIs.

### 5. Multi-turn conversations

**The API is stateless.** There is no session or conversation ID — you resend the entire history on
every request, and roles must alternate `user` / `assistant`.

Append `response.content` (the block list) back onto the history, not just the extracted text.
Stripping it to a string silently loses thinking blocks and tool calls, which breaks features that
depend on them later in the conversation.

### 6. Streaming

Use streaming for anything with long input, long output, or a high `max_tokens` — it avoids request
timeouts. Above roughly 64K `max_tokens` the SDK effectively requires it.

`client.messages.stream()` is the helper you want: it accumulates state and gives you `text_stream`
plus `get_final_message()`.

### 7. Extended thinking and effort

Two separate knobs, often confused:

- **`thinking`** — *whether* the model reasons before answering. On current models use
  `{"type": "adaptive"}`; Claude decides how much to think. On Opus 5 thinking is **on by default**,
  so omitting the parameter is the same as adaptive.
- **`output_config.effort`** — *how much* total token spend to allow: `low` / `medium` / `high` /
  `xhigh` / `max`. Default is `high`.

> **Exam traps.** Three recent changes, and the stale forms are still all over the internet:
>
> 1. `thinking={"type": "enabled", "budget_tokens": N}` is **removed** on current models and returns
>    a **400**. It survives only on older ones such as Haiku 4.5.
> 2. `effort` nests **inside `output_config`**, not top-level.
> 3. `display` defaults to `"omitted"`, so reasoning comes back as empty strings unless you ask for
>    `"summarized"`. Thinking is billed either way — `display` controls visibility only.

### 8. Stop reasons

Always check `stop_reason` before trusting `content`.

| Value | Meaning |
|-------|---------|
| `end_turn` | Finished naturally |
| `max_tokens` | Hit your ceiling — output is truncated |
| `stop_sequence` | Hit a custom stop sequence |
| `tool_use` | Wants a tool executed; run it and continue the loop |
| `pause_turn` | Paused, resumable (agentic flows) |
| `refusal` | Declined on safety grounds — inspect `stop_details` |

`stop_details` is populated **only** when `stop_reason == "refusal"`; it is `None` otherwise, so
guard before reading it.

### 9. Error handling

Catch a **chain**, most specific first. A single broad `except APIStatusError` throws away the
distinction that matters operationally: retryable (429, 5xx, network) versus not (400, 404).

The SDK already auto-retries connection errors, 408, 409, 429 and 5xx with exponential backoff —
twice by default. Don't hand-roll a retry loop on top without turning that off.

### 10. Usage and cost signals

`response.usage` is where cost analysis starts. The two cache fields are the ones to watch: if
`cache_read_input_tokens` stays at zero across requests that should share a prefix, something is
silently invalidating the cache — a timestamp in the system prompt, an unsorted `json.dumps()`, a
tool list whose order shifts.

Log `_request_id` too. It is what Anthropic support asks for, and despite the underscore it is a
public attribute.

`count_tokens` takes the same arguments as `create` and costs nothing — use it for budgeting. Do
**not** reach for `tiktoken`; that is OpenAI's tokenizer and gives wrong numbers for Claude.

### Recap

- One endpoint, `POST /v1/messages`. Everything else is a parameter on it.
- The API is **stateless** — you resend full history each turn.
- `system` is a top-level parameter, not a message role.
- `content` is a **list of blocks**; branch on `block.type`.
- `thinking` (adaptive) and `output_config.effort` are separate knobs. `budget_tokens` is dead on
  current models.
- Check `stop_reason` before reading `content`.
- Stream anything long.
