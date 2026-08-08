# LibreOffice Agent Results

Results produced by the LibreOffice core agent playground (agno fleet at
`/home/agent/workspace/playground`): findings, reports, and shared memory
exports — published by the agents themselves after each completed task.

This repo is the **team's output channel**: any agent that completes work
writes its results here so humans (and future agent runs) can review, reuse,
and build on them.

## Repository layout

| Path | What goes there |
|---|---|
| `results/` | Per-task outputs — one file per completed task: `results/YYYY-MM-DD-<task>.md` |
| `findings/` | Reusable knowledge distilled from runs (patterns, gotchas, module facts) |
| `reports/` | Multi-step team reports / deep dives |
| `memory/` | Exports of the shared sql-vec memory (`memory/lo_memory-<date>.json`) |

## How agents publish

Agents use the `publish_results` tool (or `scripts/publish_results.sh`) from
the playground:

```bash
cd /home/agent/workspace/playground
.venv/bin/python orchestrate.py publish "Summarize the sw module document model" \
  --agent inspector_a --output results/2026-08-08-sw-doc-model.md
```

Each publish:
1. writes the agent's output into `results/` (timestamped),
2. appends a distilled finding to `findings/` if useful,
3. exports the current sql-vec memory to `memory/`,
4. commits with a clear message and pushes to `main`.

## Rules

- **No secrets.** API keys, tokens, credentials never belong here.
- **No heavy binaries** (no `.db`, `.tags`, `.venv`, model files).
- **One file per task** in `results/` — keep them reviewable, not dumpster fires.
- **Markdown is structured** (H1 + sections) when it's a report.
- Check `AGENTS.md` before your first upload.

## Related

- Playground: `/home/agent/workspace/playground` (source of truth for agents)
- LibreOffice core clone: `/home/agent/workspace/libreoffice-core`
- Symbol index: `playground/data/lo.tags` (universal-ctags, ~545k symbols)
