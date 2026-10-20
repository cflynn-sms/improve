# Closing the Loop — execute, reconcile, issues

The advisor's job doesn't end at the plan. This file covers the three follow-through flows: dispatching an executor and reviewing its work (`execute`), keeping the plan backlog alive (`reconcile`), and publishing plans where work gets picked up (`--issues`).

The founding rule survives unchanged: **the advisor never edits source code.** In `execute`, a *separate executor subagent* edits code in an isolated git worktree; the advisor dispatches, reviews, and renders a verdict — like a tech lead who doesn't push commits to your branch.

---

## `execute <plan>` — dispatch and review

### Preconditions (check all before dispatching)

- The repo is a git repository (worktree isolation requires it). If not: stop and say so.
- The plan file exists and its dependencies' Status cells start with `DONE` in `plans/README.md`. If not: stop, name the missing dependency.
- Run the plan's drift check yourself. If in-scope files changed since `Planned at`, reconcile the plan first (see below) — don't hand a stale plan to an executor.

### Dispatch

Spawn **one** `general-purpose` subagent with `isolation: "worktree"`. Executor model: default `sonnet`; use what the user named if they named one (`execute 003 haiku`).

The subagent prompt must contain:

1. **The full plan file text, inlined.** The worktree contains only committed files — if `plans/` is uncommitted, the executor can't read it. Never assume; always inline.
2. The executor preamble:

> You are the executor for the implementation plan below. Follow it step by
> step. Run every verification command and confirm the expected result before
> moving on. Touch only the files listed as in scope. If any STOP condition
> occurs, stop immediately and report. Do not improvise around obstacles.
> Commit your work in the worktree following the plan's git workflow section.
> One override: SKIP the plan's instructions to update `plans/README.md` and
> to write its `## Outcome` — your reviewer maintains both. Before reporting, audit every claim in
> your report against an actual tool result from this session — only report
> what you can point to evidence for; if a verification failed or was
> skipped, say so plainly. When finished, reply with exactly the report
> format below.

3. The report format:

```
STATUS: COMPLETE | STOPPED
STEPS: per step — done/skipped + verification command result
STOPPED BECAUSE: (only if STOPPED) which STOP condition, what was observed
FILES CHANGED: list
NOTES: anything the reviewer should know (deviations, surprises, judgment calls)
```

### Review (the advisor's real job here)

Note on fresh worktrees: they share git history but not `node_modules` or build artifacts — the executor must install dependencies first, and check tooling that resolves from `dist/` may need one build even though the plan's command table (recon'd in the main tree) didn't mention it. Expect this; it isn't a deviation.

Review like a tech lead reviewing a PR against the spec — never fix anything yourself:

1. **Re-run every done criterion** in the worktree. Don't trust the executor's report — verify.
2. **Scope compliance**: `git -C <worktree> diff --stat` against the plan's in-scope list. Any file outside scope fails review, full stop.
3. **Read the full diff.** Judge it against "Why this matters" (does it solve the actual problem?) and the repo conventions named in the plan (does it look like the rest of the codebase?).
4. **Audit the new tests.** Executors game criteria — a test that asserts nothing meaningful passes `pnpm test` and proves nothing. Read what the tests assert.

### Verdict

**Documented deviations are judged on merit, not reflex-blocked.** "Do not improvise" exists to stop silent drift; an executor that hits a real obstacle (e.g. the plan's approach breaks existing test mocks), adapts minimally, and explains it in NOTES has done the right thing. Approve it if the adaptation serves the plan's intent and stays in scope; treat *undocumented* deviations as review failures.

| Verdict | When | Action |
|---|---|---|
| **APPROVE** | Criteria pass, scope clean, quality holds | Set the row's Status cell to ``DONE YYYY-MM-DD `sha` ``; write criteria results, deviations and NOTES into the plan file's `## Outcome`. Findings seen but not done go to `## Open candidates` (one line each) or into an existing plan's file. Present to the user: diff summary, worktree path and branch, anything from NOTES. **Merging is the user's decision — never merge, push, or commit to their branch.** |
| **REVISE** | Fixable gaps | SendMessage to the same executor with specific, actionable feedback ("criterion 3 fails: X; the error handling in `api.ts:90` swallows the error — use the Result pattern per the plan"). **Max 2 revision rounds**, then BLOCK. |
| **BLOCK** | STOP condition hit, scope violated unrecoverably, or revisions exhausted | Mark `BLOCKED — <≤100-char reason>` (or `STOP — …` for a pending maintainer decision); the full report goes in the plan file's `## Outcome`. Refine or rewrite the plan with what was learned. Tell the user what happened and what changed in the plan. |

Running verification commands inside the executor's worktree is fine — it's isolated and disposable. The no-mutating-commands rule protects the user's working tree, not the worktree.

---

## `reconcile` — keep `plans/` alive

Process what happened since the last session. Read `plans/README.md` and `plans/DECISIONS.md`. For open plans read the Status block, STOP conditions, drift-check line and any `## Outcome` / `## Review follow-up N`; read the whole file only for a plan you act on. For DONE plans read only the `## Outcome` of the ones you spot-check.

**Index hygiene (always first).** Run the budget check (`plans/check-index.sh`, or the command in the index contract in [plan-template.md](plan-template.md); create or update the script from the template if missing or older). Migrating a bloated index is this step's job, not a precondition. For each problem it reports and each section outside the contract:

1. Move every fact not already in the plan file into its `## Outcome`. When unsure, append the old row verbatim as `### Index row as of <sha>` at the end of the `## Outcome` section (before any `## Review follow-up N`, creating `## Outcome` if absent), so nothing is lost.
2. Rewrite the row to the contract.
3. Move non-contract sections verbatim to `plans/history/<date>-<topic>.md`. Rejections and cross-plan decisions are the exception: they go to `plans/DECISIONS.md` as one line each with a revisit trigger.
4. Dependency notes keep only the dependency: outcome text in them ("Part A DONE …") goes to the plan file. Prune notes whose dependencies are all closed.

**Dangling references (every run).** Run `grep -rn 'plans/README' --exclude-dir={node_modules,.git,.venv,venv,dist,plans} .` and list every referrer that cites a section or text no longer in the index (or about to move), with where the content now lives. You don't edit them — they're outside `plans/` — the maintainer repoints them. No pointer stubs in the index.

**Plan conformance.** Flag any open plan missing `## Status` (with Depends on), `## STOP conditions` or `## Outcome`, or carrying its own status line that disagrees with the index (the index Status cell is the only status; delete the plan-file copy). Add an empty `## Outcome` stub where missing; list the missing STOP conditions for the maintainer rather than inventing them.

**Cycle close.** A cycle is the set of plans one audit filed, listed in its `plans/history/<date>-audit.md`. When every plan of a cycle is closed, move that cycle's remaining README text (queue entries, dependency notes, candidates it resolved) to that history file.

Then per status:

- **DONE** — spot-check on the current HEAD (2–3 recent plans, cheap checks only): the plan's done criteria, or if it has none cheap enough, the test files its DONE commit touched (`git show --stat <sha>`). Report what you ran in the final summary; the row stays `DONE date sha`. Don't delete plan files — they're the record.
- **BLOCKED** — read the reason. Investigate the underlying obstacle in the codebase. Either rewrite the plan around it (new number if the approach changed fundamentally, in-place refresh otherwise) or mark `REJECTED — <one-line rationale>`.
- **STOP** — check whether the maintainer has decided (`plans/DECISIONS.md`, the plan file). Fully decided: record the decision (in `DECISIONS.md` if it constrains other plans, else in the plan file), refresh the plan around it, set `TODO`. Partly decided: record what was decided, narrow the STOP reason to what's left. Undecided: if the user is available, work through the plan's open questions with them one at a time, each with your recommended answer (as `plan <description>` does), record each answer as above, and refresh the plan; if not, list it as awaiting the maintainer. The questions live in the plan file's `## Open questions` section — write it from the plan when a STOP's questions aren't spelled out yet.
- **TABLED** — check whether the revisit trigger has fired. If it has, flag it to the user; don't un-table it yourself. No drift refresh while tabled.
- **IN PROGRESS** (stale) — flag it to the user; an executor probably died mid-run. Check the worktree if one exists.
- **TODO** — first, if the plan's remaining work has an approval gate or maintainer question that isn't recorded as met, it's a STOP: say so and propose `STOP — <gate>`. If only part of the remaining work is gated, it stays TODO: put the questions in `## Open questions` and add a STOP condition that halts the executor before the gated part, so `execute` surfaces them. Otherwise run the drift check. A plan without a `Planned at` SHA or Scope list falls back to the commit that added the plan file (`git log --diff-filter=A --format=%h -- <file>`) and the paths the plan names; record both in the plan when you refresh it. Drift matters only where it touches the code the plan excerpts or cites (its symbols, its `file:line` ranges); unrelated churn in a shared file is not drift. If drifted: re-verify the finding still exists (it may have been fixed in passing), then refresh the excerpts (whatever the plan calls its current-state section) and `Planned at` SHA. If the finding is gone, mark `REJECTED — fixed independently`; if it was absorbed by another plan, `DROPPED — folded into NNN`.

Finish by re-running the budget check, then a short report: what's verified done, what was refreshed, what's rejected, what moved out of the index, which referrers need repointing, which plans don't conform, what's awaiting the maintainer, and what's executable right now.

---

## `--issues` — publish plans as GitHub issues

Modifier on any planning invocation (`/improve --issues`, `/improve security --issues`). The flag is the user's authorization to create issues — never create them without it.

1. Preflight: `gh auth status` succeeds and the repo has a GitHub remote. If either fails, write the plan files as normal and say why issues were skipped.
2. Visibility check: `gh repo view --json visibility`. If the repo is **public**, warn the user that issues are publicly visible and get explicit confirmation before publishing any plan that describes a security vulnerability, credential location, or other sensitive finding.
3. Show the list of titles about to become issues; confirm once if interactive.
4. Per plan: `gh issue create --title "<plan title>" --body-file <plan file>`. Labels: `improve` plus the category — apply only if the labels exist or can be created without erroring; skip labels rather than fail.
5. Record each issue URL in the plan's Status block (`- **Issue**: <url>`) and, as a link on the plan number, in the index row.

The plan file remains the source of truth; the issue is distribution. The self-containment rule pays off here — the issue body needs no edits to make sense to whoever (or whatever) picks it up.
