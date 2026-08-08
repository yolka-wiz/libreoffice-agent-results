# AGENTS.md — LibreOffice Agent Results

> For the agents in the LibreOffice core playground. Read this before uploading.

## What this repo is

The output channel for agent work on LibreOffice core. Everything here was
(or will be) produced by the agno fleet in `/home/agent/workspace/playground`.

## Upload workflow

1. **Write your result** to `results/YYYY-MM-DD-<short-task>.md`
   (or the appropriate folder: `findings/`, `reports/`, `memory/`).
2. **Keep it structured** — H1 title, short summary up front, then sections.
3. **Cite real evidence** — repo paths, line numbers, symbol names. No invented
   facts; if you could not verify something, say so.
4. **No secrets, no binaries, no junk** — check the `.gitignore` spirit:
   `.env`, `*.db`, `*.tags`, `.venv/`, model files never belong here.
5. **Commit small and push** — use `scripts/publish_results.sh` from the
   playground, or `git add` the specific file, commit with a clear message,
   push to `main`.

## Structure conventions

- `results/YYYY-MM-DD-<task>.md` — one file per completed task
- `findings/` — distill reusable knowledge (one finding per file or section)
- `reports/` — larger, multi-step team outputs
- `memory/lo_memory-<date>.json` — full export of the shared sql-vec memory

## Quality bar

- Verifiable claims only (paths + line numbers for code claims).
- If a tool failed, report the failure — never fabricate success.
- Prefer concise, dense output over essays.
