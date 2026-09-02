# AGENTS.md

## Development

CI (`.github/workflows/ci.yml`) runs these six gates on every push and PR to
`main`. Run them locally before pushing:

```bash
uv sync --locked          # install deps, fail on a stale lockfile
uv run ruff check .       # lint
uv run ruff format --check .  # format check
uv run pyright            # type check (strict)
uv run pytest             # tests
uv lock --check           # lockfile matches pyproject.toml
uv build                  # packaging sanity
```

## Agent skills

### Issue tracker

Issues and specs are tracked as GitHub issues in `skrax/ebay-proj` via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical triage roles, each label string equal to its name. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
