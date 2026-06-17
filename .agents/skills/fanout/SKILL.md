---
name: fanout
description: Ship several INDEPENDENT editor backlog items in parallel — the parallel twin of /next. Selects unblocked items, proves they share no files, picks the optimum model for each, spawns one worktree-isolated agent per item concurrently, merges clean branches one at a time, runs ONE end-gate over the merged tree, then commits locally (never pushes). Caps at 5 parallel agents. Use when the user says "/fanout", "fan out these items", "ship these in parallel", "/fanout 3", "/fanout <id> <id>".
---

# /fanout — ship independent backlog items in parallel (editor)

`/next` ships items one after another. `/fanout` ships **independent** ones at the same time — one git-worktree-isolated agent each, each on the model that best fits its item. The hard parts aren't spawning agents; they are (a) proving the items won't collide, (b) choosing the right model per item, and (c) getting each agent's work back onto the branch cleanly. Those three are the core of this skill.

## Invocation
```
/fanout <id> <id> <id>   fan out exactly these backlog items
/fanout N                pick N independent unblocked items and fan them out
/fanout                  ≡ /fanout 3
/fanout --plan           print the independence analysis + model plan; write nothing
```
Numeric arg = how many to fan out. **Hard cap: 5** — beyond that, merge friction outweighs the wall-clock win. If fewer than 2 independent items exist, say so and fall back to `/next`.

## Model selection — pick the optimum model per agent

`/fanout` chooses a model for **each** dispatched agent independently; a fan-out batch is routinely mixed (e.g. one Opus + two Sonnet). Read your own session model first (`claude-opus-4-*` → Opus available; `claude-sonnet-4-*` → Sonnet; `claude-haiku-4-*` → Haiku) and classify each item:

- **Opus** — design/architecture decisions, safety- or correctness-sensitive logic, parity/invariant-preserving math, multi-file integration, schema changes that propagate, ambiguous specs, anything high-risk.
- **Sonnet** — the pattern already exists in the codebase, a bounded feature in one or two modules, mechanical edits, doc/config/test additions, low risk.
- **Haiku** — pure transcription where the backlog item already carries the complete code, one-line fixes, trivial renames.

Rules:
- **Default to Opus when unsure** — a wrong-cheap agent costs a failed gate and a re-run; a wrong-expensive agent costs only tokens.
- **Always pass `model:` explicitly on every `Agent` dispatch.** An omitted model silently inherits the session's model (often the most expensive), which defeats this section.
- You can dispatch an agent on a model **above** your session model (a Sonnet session may spawn an Opus agent) — the orchestrator picks per item, it is not capped at the session tier.
- The orchestrator (this controller) keeps running as the session model — it only does selection, merge, and gating, so it does not need to be the most capable model.
- Print the chosen model beside each item in the plan and honor it at dispatch.

## The independence check (safety gate)
Two items are fan-out-safe only if ALL hold. *Default when unsure: treat as colliding* — a false-independent costs a merge conflict; a false-collide costs only a little wall-clock.
1. **Unblocked** — neither has an unshipped dependency, and neither depends on the other.
2. **No shared source file** — not both editing the same code file (or the same function/region of a shared seam file).
3. **No shared test file** — not both editing the same test file.
4. **No shared config/doc region** — not both appending to the same config key, manifest, or doc section; each touches only its own backlog row.

Items that fail the check go to a single sequential `/next` instead.

## Backlog + gates — detect them, don't assume

This body is repo-agnostic. Resolve both at runtime, in this order:

- **Backlog source:** if this repo has a `/next` skill, use exactly its backlog file, id pattern, unblocked rule, and layer rule. Otherwise look for `BACKLOG.md` (root), `docs/BACKLOG.md`, or a `backlog/` directory (one file per item). If none exists, treat the explicit ids/task list the user gave as the work set.
- **Gates:** if `/next` or `CLAUDE.md` documents the project's gates, use them verbatim. Otherwise detect: `pyproject.toml`/`setup.cfg` → `pytest -q` then `ruff check .` (prefer the repo's `.venv` interpreter if present); `package.json` → `npm test` then `npm run lint` if that script exists; `*.sln`/`*.csproj` → `dotnet build` then `dotnet test`. If no gate can be detected, review the merged diff by hand before committing and say so in the report.

## Pipeline

### Phase 0 — Pre-flight (abort before any work)
For each id, confirm it is unblocked in the backlog: unblocked → OK; dependency- or externally-blocked → ABORT, list the exact blocker; already done / not found → ABORT. Also: HEAD must be on a branch (refuse detached HEAD); the working tree must be clean (refuse if dirty — fan-out merges back into it).

### Phase 1 — Select + verify independence + assign models
Resolve the set (explicit ids, or top-N unblocked by the backlog's sort). Run the independence check pairwise; keep a mutually-independent set, report the rest as a sequential remainder. Assign each kept item a model per § Model selection. Print the plan: each item, its files, and its model. With `--plan`, stop here and write nothing.

### Phase 2 — Worktrees
For each parallel item, off the current HEAD:
```bash
git worktree remove --force .claude/worktrees/fanout-<id> 2>/dev/null
git branch -D fanout/<id> 2>/dev/null
git worktree add .claude/worktrees/fanout-<id> -b fanout/<id>
```
If any worktree fails to create, ABORT — don't spawn a partial fan-out.

### Phase 3 — Spawn one agent per worktree (single message, parallel)
Issue all `Agent` calls in ONE message, each with its chosen `model:`. Each agent prompt (filled per item):
```
You are in an isolated git worktree at <abs path> on branch fanout/<id>.
Ship exactly ONE item — <id> — end to end. Do NOT call /next or /fanout
yourself. Use the worktree as cwd on every command (absolute paths or cd).
Touch only this item's files and its own backlog row, nothing else.

1. Read the backlog row for <id> + any linked spec/doc.
2. Plan (5-10 lines): which layer(s)/files, test plan, project invariants
   (parity, lookahead-safety, no-secrets, encoding/path rules — whatever
   this repo's CLAUDE.md / next skill documents).
3. Implement, following existing patterns; touch only your files.
4. Test: add/extend tests that verify real behavior; run the focused test.
5. Gate (run the repo's gates once; halt on first failure, commit nothing
   on failure). Skip any whole-repo republish/deploy — the parent does
   end-gating once after merge.
6. Wrap: flip <id> to Done in the backlog with an honest one-line note.
7. Commit in the worktree (the Stop hook does NOT auto-commit here):
     git add <only files you touched> && git commit -m "<id> — <summary>

     Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
   Do NOT push.
8. Return ONE final message with this block:
   ===FANOUT_RESULT===
   {"id":"<id>","shipped":true|false,"branch":"<git branch --show-current>","commit":"<git rev-parse HEAD>","files_touched":<int>,"gate_passed":true|false,"failure":"<verbatim or null>","summary":"<one line>"}
   ===END===
```

### Phase 4 — Collect
Wait for all agents. Parse each `===FANOUT_RESULT===`. A failed agent does NOT abort the others — collect them all, then build a result table.

### Phase 5 — Integrate (local, one branch at a time, id ascending)
For each `shipped:true`, from the current branch:
```bash
git merge --no-ff -m "fanout: <id>" fanout/<id>
```
Clean → continue. **Conflict → `git merge --abort`, leave that branch unmerged, stop integrating** (halt-and-keep — earlier clean merges stay).

### Phase 6 — End-gate (once, over the merged tree)
The merge is a new combination, so re-run the project gates ONCE on the merged result (§ Backlog + gates). Fail here → STOP, leave the merged commits in place for inspection, do not commit the merge as "shipped" / do not push.

### Phase 7 — Wrap + cleanup
Gate green → the integrated merges are the deliverable. Per the global rule, **commit locally, do not push** (a repo whose own CLAUDE.md mandates push overrides this — follow that repo's rule). Then for each merged item:
```bash
git worktree remove .claude/worktrees/fanout-<id>
git branch -d fanout/<id>
```
Leave failed/conflicted worktrees + branches in place; print rescue commands.

### Phase 8 — Report
Summary: shipped+merged ids (with the model each ran on), failed (verbatim reason), conflicts, end-gate result, the sequential remainder (`/next <id> …`), and one `▶ Next:` line.

## Does NOT
- Parallelise items that share a source/test/config/doc file — those go to sequential `/next`.
- Exceed 5 parallel agents; roll back already-merged items on a later conflict (halt-and-keep); skip the end-gate; bypass gate failures with `--no-verify`.
- Dispatch an agent without an explicit `model:` (silent most-expensive inheritance).
- Push (unless the repo's own CLAUDE.md requires it); have agents recurse into `/next` / `/fanout` (each agent's pipeline is inlined in its prompt).
