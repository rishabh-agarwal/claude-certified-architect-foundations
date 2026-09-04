# Claude Certified Architect – Foundations (CCAR-F)

My study notes, cheatsheets, and Jupyter notebooks for the **Claude Certified Architect – Foundations (CCAR-F)** exam.

> ⚠️ These are personal study materials, written by me while preparing. They are **not** official Anthropic content, are not endorsed by or affiliated with Anthropic, and may contain mistakes or go stale as the exam evolves. Always check the official exam guide as the source of truth.

---

## Repo layout

```
.
├── cheatsheets/     # Markdown quick-reference, one file per exam domain
├── notebooks/       # Jupyter notebooks — runnable API examples and experiments
├── practice/        # Practice questions and answer keys with rationale
└── requirements.txt # Python deps for the notebooks
```

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
- [ ] Draft a cheatsheet per domain
- [ ] First full practice pass
- [ ] Review weak domains
- [ ] Sit the exam

## License

Notes and cheatsheets: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Code samples: MIT.
