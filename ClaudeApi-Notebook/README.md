# Notebooks

Runnable companions for the **Claude Certified Architect – Foundations (CCA-F)** study path. The
notebooks hold **code only** — every explanation, caveat, and exam trap lives here, so the `.ipynb`
files stay short and diff cleanly.

| Notebook | Guide | Model used |
|---|---|---|
| `01-multi-turn-conversation.ipynb` | [Multi-turn conversation](#01--multi-turn-conversation) | `claude-sonnet-5` |
| `02-system-prompt.ipynb` | [System prompts](#02--system-prompts) | `claude-sonnet-5` |
| `03-temperature.ipynb` | [Temperature](#03--temperature) | `claude-haiku-4-5` |
| `04-Streaming.ipynb` | [Streaming](#04--streaming) | `claude-haiku-4-5` |
| `05-StructuredData.ipynb` | [Structured outputs](#05--structured-outputs) *(scaffold only)* | `claude-haiku-4-5` |
| `06` *(planned)* | Tool use and the agentic loop | — |
| `07` *(planned)* | Prompt caching | — |
| `08` *(planned)* | MCP servers and clients | — |

---

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install anthropic python-dotenv
jupyter lab
```

Each notebook also opens with `%pip install anthropic python-dotenv`, so it is self-contained if you
open one cold.

> **Pick the right kernel.** The venv lives at the repo root (`.venv/`). If VS Code's kernel picker
> points somewhere else, you will `pip install` into one interpreter and run in another — the
> classic "I just installed it and it says `ModuleNotFoundError`" loop.

### Credentials

The SDK resolves credentials in this order, first match wins:

1. `ANTHROPIC_API_KEY`
2. `ANTHROPIC_AUTH_TOKEN`
3. An OAuth profile from `ant auth login`, stored under `~/.config/anthropic/`

So a bare `anthropic.Anthropic()` works with no argument passed, which is what every notebook here
does. The key reaches the environment via `python-dotenv` reading `.env` at the repo root.

**Never hardcode a key in a notebook, and never `print()` one.** Notebook *output* is saved into the
`.ipynb` and committed. GitHub push protection will block the push — and by then the key is already
in a local commit object. Print a masked check instead:

```python
k = os.environ["ANTHROPIC_API_KEY"]
print(f"key {k[:13]}...{k[-4:]} (len={len(k)})")
```

### The setup cell

All four notebooks open with the same cell. Three details in it are deliberate:

```python
load_dotenv(find_dotenv(usecwd=True), override=True)

client = Anthropic()          # reads ANTHROPIC_API_KEY from the environment
model = "claude-sonnet-5"
```

- **`find_dotenv(usecwd=True)`** searches upward from the working directory, so `.env` is found
  whether it sits in `/` or the repo root. A bare relative `".env"` breaks the moment the
  file moves or Jupyter's CWD differs. Plain `find_dotenv()` inspects the call stack for the
  caller's `__file__`, which notebooks do not have — always pass `usecwd=True` here.
- **`override=True`** is what makes a **rotated key** take effect. `load_dotenv()` will *not*
  overwrite a variable already present in `os.environ`, so without this the old key survives in the
  kernel and every request 401s.
- **Loading and constructing the client live in one cell.** `Anthropic()` captures the key once, at
  construction. Split them across cells and re-running only the loader leaves a stale `client`.

> **The single biggest time-waster in these notebooks is stale kernel state.** A `.ipynb` edit is
> inert until the cell is re-run. If behaviour disagrees with the code you are looking at, check the
> execution counts in the margin — `None` means never run, and out-of-order numbers mean cells ran
> piecemeal across restarts. **Restart Kernel and Run All Cells** is the only state that matches the
> file.

### Before committing

Clear outputs so diffs stay readable and no key leaks:

```bash
jupyter nbconvert --clear-output --inplace ClaudeApi-Notebook/*.ipynb
```

---

## Shared helpers

Notebooks 01–04 define the same two helpers. Messages are plain dicts; there is no message class to
learn.

```python
def add_user_message(messages, text):
    messages.append({"role": "user", "content": text})

def add_assistant_message(messages, text):
    messages.append({"role": "assistant", "content": text})
```

Watch the quoting: `{"content": text}` uses the parameter, `{"content": "text"}` sends the literal
word. The second is valid Python, raises nothing, and quietly sends the wrong prompt.

---

## 01 · Multi-turn conversation

Everything goes through a single endpoint: `POST /v1/messages`. Tools, structured outputs, thinking,
and caching are all *parameters* on that one endpoint, not separate APIs. That is the single most
useful thing to have straight going into the exam.

### Three required parameters

`model`, `max_tokens`, and `messages`.

`max_tokens` is a hard ceiling on the response — hit it and output is cut off mid-sentence. Sensible
defaults: **~16000** non-streaming (stays under the SDK's HTTP timeout), **~64000** streaming. Drop
it only for a reason, e.g. `256` for classification. The notebooks use `1000` because the answers
are one-liners.

### Reading the response

`response.content` is a **list of content blocks**, not a string. A block can be `text`, `thinking`,
`tool_use`, and more.

```python
def chat(messages):
    response = client.messages.create(
        model=model,
        max_tokens=1000,
        messages=messages,
    )
    return "".join(b.text for b in response.content if b.type == "text")
```

> **Exam trap.** `response.content[0].text` works right up until it doesn't. On a model with
> adaptive thinking on, block `0` is a `ThinkingBlock` and you get
> `AttributeError: 'ThinkingBlock' object has no attribute 'text'`. Because `display` defaults to
> `"omitted"`, that block arrives with *empty* text — invisible, but still occupying index 0.
> Always branch on `block.type`.

### The API is stateless

There is no session or conversation ID. You resend the entire history on every request, and roles
must alternate `user` / `assistant`. That is what the accumulating `messages` list is doing — and
why cost grows with each turn, since the full history is re-sent and re-billed as input every time.

The notebook closes with an interactive REPL:

```python
while True:
    user_input = input("> ")
    if user_input.strip().lower() in {"quit", "exit"}:
        break
    add_user_message(messages, user_input)
    answer = chat(messages)
    add_assistant_message(messages, answer)
```

Two things that bite here: everything must be **indented into the loop** (dedent it and you get an
infinite input-echo that never calls Claude), and `while True` needs a **break condition** or the
only way out is interrupting the kernel.

---

## 02 · System prompts

The system prompt is a **top-level parameter**, not a message with `role: "system"`. This trips up
everyone arriving from other APIs.

```python
def chat(messages, system=None):
    params = {
        "model": model,
        "max_tokens": 1000,
        "messages": messages,
    }

    if system:
        params["system"] = system

    message = client.messages.create(**params)
    return "".join(b.text for b in message.content if b.type == "text")
```

Building a `params` dict and adding `system` **conditionally** is the pattern worth keeping —
passing `system=None` explicitly is not the same as omitting it, and this scales cleanly as you add
`temperature`, `tools`, and `output_config` in later notebooks.

The notebook's example pins a persona that constrains behaviour rather than topic:

```python
system = """
You are a patient math tutor
Do not directly answer a student's questions.
Guide them to a solution step by step
"""
```

That is the useful demonstration — the same user question produces a fundamentally different
*shape* of answer, not just a different tone.

> Later, when you reach prompt caching: `system` is rendered **after** `tools` and **before**
> `messages`. Keeping it byte-stable is what makes a cached prefix hold.

---

## 03 · Temperature

`temperature` controls sampling randomness — `0.0` for near-deterministic output, higher for
variety. Run the same movie-idea prompt several times at each setting to see the spread.

```python
def chat(messages, system=None, temperature=1.0):
    params = {
        "model": model,
        "max_tokens": 1000,
        "messages": messages,
        "temperature": temperature,
    }
    ...
```

> **Exam trap — and the reason this notebook pins a different model.** `temperature`, `top_p`, and
> `top_k` are **removed on current-generation models**. Sending `temperature` to Fable 5/5.1,
> Opus 5/4.8/4.7, or Sonnet 5 returns:
>
> ```
> 400 invalid_request_error: `temperature` is deprecated for this model.
> ```
>
> Verified against the live API:
>
> | Model | `temperature` |
> |---|---|
> | `claude-sonnet-5` | ❌ 400 |
> | `claude-sonnet-4-6` | ✅ |
> | `claude-opus-4-6` | ✅ |
> | `claude-haiku-4-5` | ✅ |
>
> This notebook uses **`claude-haiku-4-5`** so the parameter is actually exercisable. On current
> models the replacement knobs are `thinking` and `output_config.effort` — a different concept, not
> a rename.

---

## 04 · Streaming

Use streaming for anything with long input, long output, or a high `max_tokens` — it avoids request
timeouts. Above roughly 64K `max_tokens` the SDK effectively requires it.

The notebook shows **both** forms, which is the point of the comparison.

### Raw events — `stream=True`

```python
stream = client.messages.create(
    model=model,
    max_tokens=1000,
    messages=messages,
    stream=True,
)

for event in stream:
    print(event)
```

You get the raw SSE event objects: `message_start`, `content_block_start`,
`content_block_delta`, `content_block_stop`, `message_delta`, `message_stop`. Printing them is the
fastest way to *see* that a response is assembled from deltas rather than arriving whole. You would
reach for this only when you need per-event control.

### The helper — `client.messages.stream()`

```python
with client.messages.stream(
    model=model,
    max_tokens=1000,
    messages=messages,
) as stream:
    for text in stream.text_stream:
        print(text, end="")

stream.get_final_message()
```

This is the one you want. It accumulates state for you and gives you two things the raw form does
not: `text_stream`, which yields just the text deltas, and `get_final_message()`, which returns the
fully assembled `Message` — same shape as a non-streaming response, so `usage` and `stop_reason` are
there.

Note it is a **context manager**. Call `get_final_message()` after the `with` block has consumed the
stream, or you will be asking for a message that has not finished arriving.

---

## 05 · Structured outputs

> **Scaffold only.** As of now this notebook is a byte-for-byte copy of `04-Streaming` — the setup
> cell, the helpers, and both streaming cells, with no structured-output code written yet. Treat the
> section below as the outline to fill in, not a description of what runs.

Structured outputs constrain the shape of what comes back, and there are two distinct mechanisms
that get conflated:

- **`output_config.format`** — constrains the **response** to a JSON schema. This is what you want
  when you need parseable data rather than prose.
- **`strict: true`** on a tool definition — constrains **tool arguments** so `tool_use.input`
  validates exactly against your schema. Requires `additionalProperties: false` and `required` in
  the schema.

`client.messages.parse()` is the ergonomic path: it validates the response against your schema for
you rather than leaving you to `json.loads()` and hope.

> **Exam traps.**
>
> 1. The parameter is `output_config: {format: {...}}`. The older top-level `output_format` is
>    **deprecated** — and it is what most search results still show.
> 2. `strict` is a field on the **tool definition**, alongside `name` / `description` /
>    `input_schema`. It is *not* a field on `tool_choice`.
> 3. Structured outputs are **incompatible with citations** — setting both returns a 400.

Because the model still returns a block list, keep branching on `block.type`; a schema-constrained
response does not flatten `content` into a string.

---

## General reference

### Extended thinking and effort

Two separate knobs, often confused:

- **`thinking`** — *whether* the model reasons before answering. On current models use
  `{"type": "adaptive"}`; Claude decides how much to think. On Opus 5 and Sonnet 5 thinking is
  **on by default**, so omitting the parameter is the same as adaptive.
- **`output_config.effort`** — *how much* total token spend to allow: `low` / `medium` / `high` /
  `xhigh` / `max`. Default is `high`.

> **Exam traps.** Three recent changes, and the stale forms are still all over the internet:
>
> 1. `thinking={"type": "enabled", "budget_tokens": N}` is **removed** on current models and returns
>    a **400**. It survives only on older ones such as Haiku 4.5.
> 2. `effort` nests **inside `output_config`**, not top-level.
> 3. `display` defaults to `"omitted"`, so reasoning comes back as empty strings unless you ask for
>    `"summarized"`. Thinking is billed either way — `display` controls visibility only.

**Thinking tokens bill as output.** On a small prepaid balance this is the surprise that empties it:
the tokens are invisible under the default `display`, but charged at the full output rate. Drop
`output_config={"effort": "low"}` on routine exercises.

### Stop reasons

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

### Error handling

Catch a **chain**, most specific first. A single broad `except APIStatusError` throws away the
distinction that matters operationally: retryable (429, 5xx, network) versus not (400, 404).

The SDK already auto-retries connection errors, 408, 409, 429 and 5xx with exponential backoff —
twice by default. Don't hand-roll a retry loop on top without turning that off.

Three you will actually meet while working through these notebooks:

| Error | Cause |
|---|---|
| `401 authentication_error: invalid x-api-key` | Malformed key — check `.env` for a doubled `ANTHROPIC_API_KEY=` prefix |
| `401 authentication_error: API key is invalid.` | Key is well-formed but deleted or rotated |
| `400 invalid_request_error: Your credit balance is too low` | Auth succeeded; the account has no prepaid credits |

That middle one is the useful tell: the shape parsed, the key just does not exist any more.

### Usage and cost signals

`response.usage` is where cost analysis starts. The two cache fields are the ones to watch: if
`cache_read_input_tokens` stays at zero across requests that should share a prefix, something is
silently invalidating the cache — a timestamp in the system prompt, an unsorted `json.dumps()`, a
tool list whose order shifts.

Log `_request_id` too. It is what Anthropic support asks for, and despite the underscore it is a
public attribute.

`count_tokens` takes the same arguments as `create` and costs nothing — use it for budgeting. Do
**not** reach for `tiktoken`; that is OpenAI's tokenizer and gives wrong numbers for Claude.

---

## Recap

- One endpoint, `POST /v1/messages`. Everything else is a parameter on it.
- The API is **stateless** — you resend full history each turn, and pay for it each turn.
- `system` is a top-level parameter, not a message role.
- `content` is a **list of blocks**; branch on `block.type`, never index `[0]`.
- `temperature` is **gone on current models** — use an older model to study it.
- `thinking` (adaptive) and `output_config.effort` are separate knobs. `budget_tokens` is dead on
  current models.
- Check `stop_reason` before reading `content`.
- Stream anything long; prefer `client.messages.stream()` over `stream=True`.
- `load_dotenv(..., override=True)` + client construction in **one cell**, and when in doubt,
  **Restart Kernel and Run All**.
