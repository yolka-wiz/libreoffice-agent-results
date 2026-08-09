# sw/ (Writer) module — git history analysis

_Generated 2026-08-09T16:22:36_

## Task
Find what changed recently in the `sw/` (Writer) module via git history and summarize top commits with purpose, citing hashes.

## Finding: no usable incremental history exists in this checkout

The workspace at `/home/agent/workspace/libreoffice-core` is a **depth-1 shallow clone** containing exactly **one commit**. Evidence:

- `git log` (no path filter, count=100) and `git log -- sw` both return a single commit:
  - **`88771d0a64e06297b7f82d6cc00cdf0d60c199cc`** — Patrick Luby, 2026-08-08 — *"tdf#172927 don't rely on pointer comparison for equality"*
- `git show 88771d0a6` shows a diff of the **entire repository** (~10.1M chars: `.buckconfig`, `COPYING`, `Repository.mk`, every module…). It is an import snapshot, not a per-change commit.
- `.git/shallow` contains exactly `88771d0a64e06297b7f82d6cc00cdf0d60c199cc` → depth-1 shallow clone confirmed.
- Pickaxe (`git log -S tdf#172927`) returns only the same commit.
- `git blame sw/inc/swtypes.hxx` attributes every line to `88771d0a6`.

**Consequence:** it is not possible to enumerate "top recent commits" for `sw/` — there is only one hash in the entire object database. Any per-file history question (blame, -S, bisect) will trivially resolve to this single snapshot commit.

## What the one available commit actually is
The message refers to a real upstream fix, but it lives in **vcl/ (macOS)**, not sw/:
- `vcl/osx/salframeview.mm` — `mpLastEvent` handling at lines 1125, 1706, 1772 (`- [NSEvent isEqual:]` deeper equality check to avoid infinite recursion in `-[NSApp sendSuperEvent:]` when `mpLastEvent` is a copy).
- `vcl/inc/osx/salframeview.h:96` — `NSEvent* mpLastEvent;` member.

No sw/ file is touched by the real change.

## Current sw/ snapshot (fallback overview, not history)
Per `REPO_MAP.md` (line 120): **sw = 2563 source files** (1249 cxx, 1006 hxx, 298 py, 8 h, 1 java, 1 sh), **77103 symbols**. This reflects the tree state at the snapshot point, not a delta.

## Recommendation
To actually answer "what changed recently in sw/", the clone must be unshallowed / history fetched from `git.libreoffice.org` (e.g. `git fetch --unshallow origin`), then `git log --oneline -30 -- sw/` will yield real per-change commits with distinct hashes and Change-Ids.

Recorded in shared memory (knowledge#259, fact `repo.git_is_shallow`).
