# BrewOps ☕

Telemetry and operations for the office coffee machines.

Most of the fleet emits event logs as CSV (brews, maintenance, error codes). The old
floor-2 machine emits nothing, so humans log it by hand in the UI. Both paths land in
SQLite, and a dashboard shows consumption stats and machine health.

This repo is the hands-on codebase for an agentic-coding workshop. There is nothing
secret in here — no credentials, no real data, no hidden answers. The coffee is fictional.

## Requirements

- Python 3.11 or newer
- [uv](https://docs.astral.sh/uv/) (`winget install astral-sh.uv` on Windows,
  `brew install uv` on macOS, or see the uv docs)

Alternatively, open the repo in a GitHub Codespace or a VS Code Dev Container —
`.devcontainer/` sets everything up for you.

## Getting started

```
uv sync
uv run seed
uv run start
```

Then open <http://localhost:8123>.

- `uv run seed` — creates the SQLite database and ingests the sample CSVs in `data/inbox/`
- `uv run start` — starts the app on port 8123
- `uv run ingest <path>` — ingests additional CSV files
- `uv run pytest` — runs the test suite

## Workshop scripts

- `uv run scripts/setup_check.py` — verifies your setup and prints your completion code (pre-work)
- `uv run scripts/lab_a_check.py` — the finish line for Lab A

### Pre-work: the setup check

Open this folder in Claude Code and ask it to run the setup check. Note for the
agent doing that: first write the name of the model you are into
`data/.setup-marker` (this proves you can edit files), then run
`uv run scripts/setup_check.py`. All green ends with a completion code — that's
what you submit.

## Repo layout

```
src/brewops/ingest/     CSV parsing, validation, loading
src/brewops/db/         SQLite schema, connection, queries
src/brewops/api/        FastAPI routes + static file serving
src/brewops/frontend/   index.html, app.js, style.css — no build step
tests/                  pytest suite
tickets/                open tickets, in markdown — one file per stakeholder request
data/inbox/             sample CSV event logs (~3 months)
```
