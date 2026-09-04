# Claude Certified Architect – Foundations (CCAR-F)

My study notes, cheatsheets, and Jupyter notebooks for the **Claude Certified Architect – Foundations (CCAR-F)** exam.

---
## Study path

Anthropic publishes a free learning path on [Anthropic Academy](https://anthropic.skilljar.com/) that covers the large majority of the exam content. Work through it in order — each course maps onto the cheatsheets in this repo.

| # | Course | Core topics | Status |
|---|--------|-------------|--------|
| 1 | **Building with the Claude API** *(flagship)* | Prompt engineering, evaluation frameworks, multi-turn tool calling, structured JSON output, RAG architectures | ☐ |
| 2 | **Introduction to Model Context Protocol (MCP)** | Building custom MCP servers/clients, protocol lifecycle, tools, resources, error handling | ☐ |
| 3 | **Claude Code in Action** | `CLAUDE.md` configuration hierarchies, custom slash commands, subagents, lifecycle hooks, non-interactive CI/CD setups | ☐ |
| 4 | **Platform-specific cloud course** | Pick one to match your stack: *Claude with Amazon Bedrock* or *Claude on Google Cloud Vertex AI* | ☐ |

Course 1 is the flagship and carries the most exam weight — do not treat it as a warm-up. Courses 2 and 3 are independent of each other, so their order is flexible, but both assume the API fundamentals from course 1.

Whatever the path does not cover ends up in [`practice/`](practice/) as gap notes.

## Exam domains

Filled in from the official exam guide — each row links to its cheatsheet as I write it.

| # | Domain | Weight | Notes | Status |
|---|--------|--------|-------|--------|
| 1 | _TBD_ | _TBD_ | | ☐ |
| 2 | _TBD_ | _TBD_ | | ☐ |
| 3 | _TBD_ | _TBD_ | | ☐ |
| 4 | _TBD_ | _TBD_ | | ☐ |

## Running the notebooks

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter lab
```

The notebooks read your API key from the environment — they never hardcode it:

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

Keep real keys out of this repo. `.env` is gitignored; commit `.env.example` instead if you need to document the variables.

## Conventions

- **One cheatsheet per domain.** Name it after the domain, kebab-case: `cheatsheets/prompt-engineering.md`.
- **Cheatsheets are for recall, not for teaching.** Tables, bullets, and short code blocks — if it takes a paragraph to explain, it belongs in a notebook.
- **Notebooks are committed with outputs cleared**, so diffs stay readable:
  ```bash
  jupyter nbconvert --clear-output --inplace notebooks/*.ipynb
  ```
- **Practice answers include the *why*,** not just the correct letter. The rationale is the part worth re-reading.

## Progress

- [ ] Read the official exam guide end to end
- [ ] Fill in the domain table above
- [ ] Course 1 — Building with the Claude API
- [ ] Course 2 — Introduction to MCP
- [ ] Course 3 — Claude Code in Action
- [ ] Course 4 — Bedrock or Vertex AI
- [ ] Draft a cheatsheet per domain
- [ ] First full practice pass
- [ ] Review weak domains
- [ ] Sit the exam

## License

Notes and cheatsheets: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Code samples: MIT.
