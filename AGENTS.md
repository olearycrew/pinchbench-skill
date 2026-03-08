# AGENTS.md

## Purpose

PinchBench Skill is the benchmark runner. It runs 23 real-world tasks against an LLM (via OpenClaw + OpenRouter), grades results, and uploads them to the API at https://api.pinchbench.com. Results then appear on the public leaderboard at https://pinchbench.com.

## System Architecture

This repo fits into a three-part system:

- **pinchbench-skill** (this repo): runs benchmarks, uploads results
- **pinchbench-api** (api.pinchbench.com): receives results via `POST /api/results`, stores in Cloudflare D1, serves leaderboard data. Has an admin section at `/admin` used by the system owner.
- **pinchbench-leaderboard** (pinchbench.com): Next.js frontend that reads from the API and displays the leaderboard

## Tech Stack

- Python 3.10+ managed by `uv`
- OpenClaw agent framework (Node.js, installed globally via npm)
- OpenRouter for all LLM access (model IDs: `openrouter/<provider>/<model>`)
- GitHub Actions CI (scheduled daily, Blacksmith runners, Docker)
- Judge model: `anthropic/claude-opus-4.5`

## Key Files

- `scripts/benchmark.py` — main entry point
- `scripts/lib_agent.py` — OpenClaw agent lifecycle
- `scripts/lib_grading.py` — automated + LLM judge grading engine
- `scripts/lib_tasks.py` — Markdown+YAML task file parser
- `scripts/lib_upload.py` — HTTP upload to api.pinchbench.com
- `tasks/task_NN_name.md` — the 23 benchmark task definitions
- `.github/benchmark-models.yml` — matrix of models run in CI
- `.github/workflows/scheduled-benchmarks.yml` — CI workflow
- `Dockerfile.benchmark` — Docker image for CI runs

## Running a Benchmark

```bash
./scripts/run.sh --model anthropic/claude-sonnet-4
# or directly:
uv run scripts/benchmark.py --model anthropic/claude-sonnet-4
```

Required env vars: `OPENROUTER_API_KEY`, `PINCHBENCH_TOKEN` (or `scripts/.pinchbench/config.json`)

## Upload Flow

- Token registered once via `--register` flag, saved to `scripts/.pinchbench/config.json`
- Results POSTed to `https://api.pinchbench.com/api/results` with `X-PinchBench-Token` header
- `PINCHBENCH_SERVER_URL` env var can override the server (for testing)

## Task File Format

Markdown with YAML frontmatter. Required sections: `Prompt`, `Expected Behavior`, `Grading Criteria`, `Automated Checks` (if `automated`/`hybrid`), `LLM Judge Rubric` (if `llm_judge`/`hybrid`). Task IDs follow `task_NN_descriptive_name` convention.

## Grading

- `automated`: Python `grade()` function, receives transcript + `workspace_path`, returns `dict[str, float]` 0.0–1.0
- `llm_judge`: Claude Opus agent, strict JSON output `{scores, total, notes}`
- `hybrid`: weighted average of both (default 50/50)

## Key Gotchas

- OpenClaw ignores `--session-id`; `lib_agent.py` has a 3-strategy retry to find the real JSONL transcript
- The CI workflow writes `/tmp/pinchbench-workspace/AGENTS.md` to prevent `NO_REPLY` behavior from OpenClaw defaults
- `uv run` is required (not `python` directly)
- Workspace files in frontmatter reference the `assets/` directory at repo root
