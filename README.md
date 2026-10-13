# fastapi-agents-md

**English** | [简体中文](README.zh-CN.md)

> An `AGENTS.md` rule set for FastAPI projects that makes AI coding agents write layered, typed, maintainable code — instead of dumping everything into one file.

When you vibe-code with an AI agent, the default output is the shortest path to the current prompt: one big file, business logic inside route handlers, untyped functions, and an OpenAPI page full of empty `{}` responses. This repository turns architectural taste into **explicit, checkable rules** the agent reads on every task, and backs the important ones with **linters and tests** so they do not depend on the agent's good will.

## What's inside

| File | Purpose |
|------|---------|
| [`templates/AGENTS.md`](templates/AGENTS.md) | The rule set (English). Drop it in your project root. |
| [`templates/AGENTS.zh-CN.md`](templates/AGENTS.zh-CN.md) | Chinese version, same structure and section numbers. |
| [`templates/pyproject.tooling.snippet.toml`](templates/pyproject.tooling.snippet.toml) | Ruff, mypy and import-linter configuration that enforces the rules mechanically. |

## What the rules cover

| # | Section | Highlights |
|---|---------|------------|
| 1 | Working agreement | Read before writing, plan non-trivial changes, smallest diff, ask when requirements are ambiguous |
| 2 | Design principles & patterns | SOLID, DRY + Rule of Three, KISS, YAGNI, fail fast, CQS, idempotency; when to use (and not use) Strategy, Facade, Adapter, Repository…; code smells |
| 3 | Size & complexity limits | File ~300 lines, function ~30 lines, ≤ 5 params, complexity ≤ 10 |
| 4–5 | Structure & layering | Domain-based layout, "where does it go?" table, controller → service → rules, dependency direction, transaction ownership |
| 6–9 | Async, Pydantic, dependencies, REST | Blocking-call pitfalls, v2 serialization, dependencies as validators, paginated lists |
| 10 | Database | Naming, indexes, N+1, safe queries, reviewed and reversible Alembic migrations |
| 11 | External integrations | One module per external service, typed errors, bounded retries, stub-at-the-boundary testing |
| 12–13 | Errors, security, logging | Domain exceptions mapped by handlers, auth by default, secrets handling |
| 14 | API docs & testing | **Route return annotations that generate OpenAPI** (direct schemas, plus an optional generic response-envelope variant), OpenAPI contract test, testing pyramid |
| 15 | Definition of done | Checks the agent must run and report |
| 16 | Python conventions | **Mandatory type annotations** (modern 3.12 syntax), naming |
| 17–18 | Commands, quick reference | One-page cheat sheet |

Rules are marked **MUST** (enforced in review or by tooling) or **SHOULD** (default; break only with a reason). They apply to new and modified code, so adopting them never requires a big-bang refactor.

## Quick start

```shell
# 1. Copy the rules into your project root
cp templates/AGENTS.md /path/to/your-project/AGENTS.md

# 2. Merge the tooling config into your pyproject.toml
#    (see templates/pyproject.tooling.snippet.toml)

# 3. Install the dev tools the rules rely on
pip install ruff mypy import-linter pytest pytest-asyncio httpx

# 4. Verify
ruff check app tests && ruff format --check app tests
mypy app
lint-imports
pytest -q
```

**Which file does your tool read?** Tools that read `AGENTS.md` natively need nothing more. Claude Code reads `CLAUDE.md`, so add one that imports the rules:

```shell
echo "@AGENTS.md" > CLAUDE.md
```

Check your tool's documentation for the exact filename it loads.

## Recommended workflow

1. **Plan first.** Ask the agent for a module structure, abstractions, dependency direction and extension points — no code yet. Review it.
2. **Implement one module at a time**, each with tests.
3. **Let tooling be the referee.** The agent must run lint, type check, import-linter and tests, and report what it ran (Definition of Done, §15).
4. **Review periodically.** Ask the agent to audit a package against `AGENTS.md` and list violations with file and line, before changing anything.

## Customize it

The template assumes **Python 3.12+, FastAPI, Pydantic v2, SQLModel/SQLAlchemy (async), Alembic and pytest**. Adapt before use:

- [ ] **§4 Project structure** — match your real directory layout and update the import-linter contracts accordingly.
- [ ] **Scope & Assumptions** — check the required/optional stack at the top of `AGENTS.md` against your project. The file tells the agent that names like `BaseService` or `ResponseModel` are *examples*; still, replace them with your real names where they exist, so the agent has concrete anchors.
- [ ] **§11 External integrations** — delete it if you have none.
- [ ] **§3 Limits** — tune the numeric thresholds (and the matching Ruff settings) to your team.
- [ ] **§1 git-safety rule** — adapt it to your team's branching model, and add a section for any project-specific agent skills or commands you have.

## Adopting it in an existing codebase

1. Run `ruff check --statistics app` and see how large the gap is.
2. Fix or baseline existing violations with `per-file-ignores`; let mypy `overrides` exempt legacy modules.
3. Hold new and modified code to the full bar.
4. Shrink the exemptions over time.

## Repository layout

```
fastapi-agents-md/
├── README.md
├── README.zh-CN.md
├── LICENSE
└── templates/
    ├── AGENTS.md
    ├── AGENTS.zh-CN.md
    └── pyproject.tooling.snippet.toml
```

The rules live under `templates/` on purpose: an `AGENTS.md` at this repository's root would instruct agents working on *this* repo, not on yours.

## Contributing

Issues and pull requests are welcome. Please:

- keep rules **concrete and checkable** (prefer "functions ≤ 30 lines" over "keep functions short");
- when a rule can be enforced by a tool, add the config to the tooling snippet;
- keep the English and Chinese versions in sync, including section numbers.

## License

[MIT](LICENSE) © 2026 <YOUR NAME OR ORGANIZATION>

## Acknowledgements

Several ideas here — domain-based project layout, dependencies as request validators, async/sync pitfalls — were inspired by community FastAPI guides, notably [zhanymkanov/fastapi-best-practices](https://github.com/zhanymkanov/fastapi-best-practices). This repository is an independent work, is not affiliated with it, and its code samples are written from scratch.
