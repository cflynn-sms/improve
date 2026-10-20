# Handoff Plan Template

Every plan is written for an executor model that has **zero context**: it has not seen the advisor session, the audit, the other plans, or any prior conversation. It may be a smaller/cheaper model. Assume it is competent at following explicit instructions and weak at filling gaps, recovering from ambiguity, or knowing when to stop.

Three properties make a plan executable by a weaker model:

1. **Self-contained context** — everything needed is in the file: paths, code excerpts, conventions, commands.
2. **Verification gates** — every step ends with a command and its expected result. The executor never has to *judge* whether it succeeded.
3. **Hard boundaries and escape hatches** — explicit out-of-scope list, and "STOP and report" conditions instead of letting the model improvise when reality doesn't match the plan.

File naming: `plans/NNN-short-slug.md`, numbered in recommended execution order.

Ad-hoc plans written outside `/improve` (by an orchestrator, a reviewer, or by hand) must still include `## Status` (with Depends on), `## STOP conditions` and `## Outcome` — the executor instructions live in those sections, and a plan without them never passes them on.

---

## Template

```markdown
# Plan NNN: <Imperative title — what will be true after this plan>

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, write a dated `## Outcome` section at
> the end of this plan file: commits, verification results, deviations, and
> anything seen but not done. Then, unless a reviewer or orchestrator told you
> they maintain the index, change only this plan's Status cell in
> `plans/README.md` to ``DONE YYYY-MM-DD `sha` `` (or `BLOCKED — <reason>` /
> `STOP — <reason>`, ≤100 chars). Never add outcome text, counts or notes to
> the index.
>
> **Drift check (run first)**: `git diff --stat <planned-at SHA>..HEAD -- <in-scope paths>`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P1 | P2 | P3
- **Effort**: S | M | L
- **Risk**: LOW | MED | HIGH
- **Depends on**: plans/NNN-*.md (or "none")
- **Category**: bug | security | perf | tests | tech-debt | migration | dx | docs | direction
- **Planned at**: commit `<short SHA>`, <YYYY-MM-DD>
- **Issue**: <GitHub issue URL — only when published via `--issues`; omit otherwise>

## Why this matters

2–5 sentences. The problem, its concrete cost, and what improves when this
lands. Written so the executor (and a human reviewer) understands the intent —
intent is what lets a correct judgment call happen when a detail is off.

## Current state

The facts the executor needs, inlined — never "as discussed" or "see audit":

- The relevant files, each with one line on its role:
  - `src/orders/api.ts` — order-list endpoint; contains the N+1 (lines 130–160)
- Excerpts of the code as it exists today (short, with `file:line` markers),
  enough that the executor can confirm it's looking at the right thing.
- The repo conventions that apply here, with a pointer to one exemplar file:
  "Error handling follows the Result pattern — see `src/lib/result.ts` and its
  use in `src/users/api.ts:40-60`. Match it."
- Any documented vocabulary or design constraints the plan must honor, inlined
  from the intent/design docs found in recon: the relevant `CONTEXT.md` terms
  the executor should use in names and comments, the `DESIGN.md` tokens/components
  to reuse, or the ADR whose decision this work must stay consistent with. Quote
  the specific lines — the executor has not read those docs.

## Commands you will need

| Purpose   | Command                  | Expected on success |
|-----------|--------------------------|---------------------|
| Install   | `pnpm install`           | exit 0              |
| Typecheck | `pnpm typecheck`         | exit 0, no errors   |
| Tests     | `pnpm test -- <filter>`  | all pass            |
| Lint      | `pnpm lint`              | exit 0              |

(Exact commands from this repo — verified during recon, not guessed.)

## Suggested executor toolkit

(Optional — include only when relevant skills/tools plausibly exist in the
executor's environment. Skip the section otherwise.)

- Skills the executor should invoke if available, and for what:
  "use `vercel-react-best-practices` when writing the memoization in step 3".
- Reference docs worth reading before starting, by path or URL.

## Scope

**In scope** (the only files you should modify):
- `src/orders/api.ts`
- `src/orders/api.test.ts` (create)

**Out of scope** (do NOT touch, even though they look related):
- `src/orders/legacy-api.ts` — deprecated path, scheduled for deletion;
  changing it wastes effort and risks the v1 clients still pinned to it.
- Any change to the public response shape — clients depend on it.

## Git workflow

(Filled from recon — match the repo's observed conventions.)

- Branch: `advisor/NNN-<slug>` (or the repo's branch-naming convention if one is evident)
- Commit per step or per logical unit; message style: <match repo, e.g. conventional commits — include an example from `git log`>
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: <imperative title>

What to do, precisely. Reference exact files/symbols. Include the target code
shape when it's load-bearing (the pattern to produce, not necessarily every
line).

**Verify**: `<command>` → <expected output>

### Step 2: ...

(Each step small enough to verify independently. Order steps so the codebase
is never broken between steps when possible — e.g. add new path, switch
callers, then remove old path.)

## Test plan

- New tests to write, in which file, covering which cases (list them:
  happy path, the specific bug/regression this plan fixes, named edge cases).
- Which existing test to use as the structural pattern:
  "model after `src/users/api.test.ts`".
- Verification: `<test command>` → all pass, including N new tests.

## Done criteria

Machine-checkable. ALL must hold:

- [ ] `pnpm typecheck` exits 0
- [ ] `pnpm test` exits 0; new tests for <X> exist and pass
- [ ] `grep -rn "<old pattern>" src/` returns no matches
- [ ] No files outside the in-scope list are modified (`git status`)
- [ ] `## Outcome` written in this plan file; README Status cell changed (and nothing else in the README)

## STOP conditions

Stop and report back (do not improvise) if:

- The code at the locations in "Current state" doesn't match the excerpts
  (the codebase has drifted since this plan was written).
- A step's verification fails twice after a reasonable fix attempt.
- The fix appears to require touching an out-of-scope file.
- You discover the assumption "<key assumption>" is false.

## Maintenance notes

For the human/agent who owns this code after the change lands:

- What future changes will interact with this (e.g. "if pagination is added
  to this endpoint, the batching in step 2 must be revisited").
- What a reviewer should scrutinize in the PR.
- Any follow-up explicitly deferred out of this plan (and why).

## Outcome

Filled when the plan closes; one dated `## Review follow-up N` per later round.
This, not `plans/README.md`, is the record.
```

---

## Index contract: `plans/README.md`

The index is a **lookup table, not a log**. Agents load it whole to find what exists, what's open and what's next, so it must stay readable in one pass. Every record of *what happened* lives elsewhere. (Paths say `plans/`; substitute `advisor-plans/` when that is the directory in use.)

**Sections, fixed, in this order:** a header of 5 lines or fewer (the contract, links to `DECISIONS.md` and `history/`, the status values) · `## Queue` · `## Open plans` (table) · `## Open candidates` (bullets) · `## Dependency notes` (open plans only) · `## Closed plans` (table). No other sections.

**Row format:** `| NNN | Title | P | Effort | Depends on | Status |`. One row per plan number. A separately tracked part gets its own row (`102 D1`) only if it has its own status; otherwise the plan's row covers it. A number that was assigned but never got its own file gets a row with `DROPPED — folded into NNN` or `DROPPED — never filed`, so the gap is explained.

- **Title**: the plan file's H1 minus its `Plan NNN…:` prefix, shortened to 100 chars or fewer, with provenance and scope notes ("from the 069 review", "(plans 110 / 122)", "(tracked initiative)") stripped — it need not match the H1 verbatim. Scope learned later goes in the plan file. New plans: keep the H1 itself within these rules.
- **Status** starts with the exact keyword, no bold or other markup, so `execute` can read it: `TODO` · `IN PROGRESS` · `BLOCKED — why` · `STOP — why` (can't proceed without a maintainer decision — whether an executor hit it, the plan has an unmet approval gate, or it was filed as a proposal; the questions go in the plan's `## Open questions`) · ``DONE YYYY-MM-DD `sha` `` · `REJECTED — why` · `DROPPED — why` · `TABLED — revisit trigger`. Reasons are one line, ≤100 chars. DONE rows carry date + commit only.
- **No raw `|` inside cells** — it splits the row. Write "or", or escape it as `\|`.
- A plan moves from Open plans to Closed plans when its Status becomes DONE, REJECTED or DROPPED. TABLED stays under Open plans.

**Budgets:** every table row ≤300 chars; the whole file ≤40 KB. Check with `plans/check-index.sh` (it also checks the rules above), else the minimal check:

```sh
awk '/^\| *[0-9]/ && length($0)>300 {print NR": "length($0)}' plans/README.md; wc -c plans/README.md
```

**Never in the index**, and where each kind goes instead:

| Kind | Goes to |
|---|---|
| Outcomes, counts, verification results, repair logs, deviations | the plan file's `## Outcome` |
| Review-round summaries and their fixes | the plan file's `## Review follow-up N` |
| Seen-but-not-done items | `## Open candidates` (one line each), or the owning plan's file |
| Rejections; maintainer decisions that constrain more than one plan | `plans/DECISIONS.md` |
| Decisions scoped to one plan (an approved DDL, a picked option) | that plan's file |
| Sequencing rationale, audit write-ups, findings tables, closed-cycle narrative | `plans/history/YYYY-MM-DD-<topic>.md` |

### README skeleton

```markdown
# Implementation Plans

Index of the `/improve` plans: one line per plan (≤300 chars a row, file ≤40 KB, check with `plans/check-index.sh`). Outcomes and follow-ups live in each `NNN-*.md`.
Maintainer decisions and rejection memory (don't re-propose): [`DECISIONS.md`](DECISIONS.md). Past cycles' narrative: [`history/`](history/).
Status values: `TODO` · `IN PROGRESS` · `BLOCKED — why` · `STOP — why` · `DONE date sha` · `REJECTED — why` · `DROPPED — why` · `TABLED — revisit trigger`.

## Queue

1. 001 → 002 → 004.
2. 003: STOP, pending the maintainer's pick of approach.

## Open plans

| Plan | Title | P | Effort | Depends on | Status |
|------|-------|---|--------|------------|--------|
| 002 | ... | P1 | M | 001 | TODO |

## Open candidates

Seen in a plan or review, neither planned nor rejected.

- <one line>. (seen in: plan 002)

## Dependency notes

- 002 requires 001 because <reason>.

## Closed plans

| Plan | Title | P | Effort | Depends on | Status |
|------|-------|---|--------|------------|--------|
| 001 | ... | P1 | S | — | DONE 2026-01-15 `abc1234` |
```

### `plans/DECISIONS.md` skeleton

```markdown
# Maintainer decisions and rejection memory

Decisions that still constrain future plans, and findings considered and **not**
planned, so nobody re-proposes them without new evidence. One line each. Every
audit and `reconcile` reads this file; treat each "Not planned" line as
rejection memory.

## Decisions

- YYYY-MM-DD **topic**: decision and why. Revisit if: trigger. (src: plan NNN / audit date)

## Not planned

- YYYY-MM-DD **finding**: not worth doing because <reason>. Revisit if: trigger. (src: …)

## Removed as obsolete

- <entry>: why it no longer applies (date).
```

### `plans/check-index.sh`

Create it when first writing the index, if missing (`chmod +x`); replace an older copy that only checks row length. Reconcile and executors run it. It checks row and title length, cell count (a raw `|`), Status keywords and closed/open placement, then prints the byte count.

```sh
#!/bin/sh
# Index contract check for README.md next to this script. Prints each problem
# as "line: reason", then the file's byte count (limit 40 KB). Lengths count
# characters, not bytes.
cd "$(dirname "$0")" || exit 1
LC_ALL=C awk '
  function chars(s) { gsub(/[\200-\277]/, "", s); return length(s) }
  function trim(s) { sub(/^ +/, "", s); sub(/ +$/, "", s); return s }
  /^## / { section = $0 }
  /^\| *[0-9]/ {
    row = $0
    gsub(/\\\|/, "", row)
    if (chars($0) > 300) print NR ": row is " chars($0) " chars (max 300)"
    n = split(row, c, "|")
    if (n != 8) { print NR ": " n - 2 " cells (want 6; raw | in a cell?)"; next }
    title = trim(c[3]); status = trim(c[7])
    if (chars(title) > 100) print NR ": title is " chars(title) " chars (max 100)"
    if (status !~ /^(TODO|IN PROGRESS)$/ &&
        status !~ /^(BLOCKED|STOP|REJECTED|DROPPED|TABLED) — ./ &&
        status !~ /^DONE [0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9] `[0-9a-f]+`$/)
      print NR ": Status not in the contract: " status
    closed = status ~ /^(DONE|REJECTED|DROPPED)/
    if (closed && section ~ /Open plans/) print NR ": closed plan under Open plans"
    if (!closed && section ~ /Closed plans/) print NR ": open plan under Closed plans"
  }
' README.md
wc -c README.md
```

## Quality bar — check before finishing each plan

- Could a model that has never seen this repo execute this with only the plan file and the repo? If any step requires knowledge from the advisor session, inline that knowledge.
- Is every verification a command with an expected result, not a judgment ("make sure it works")?
- Does every step name exact files and symbols, not "the relevant module"?
- Are the STOP conditions specific to this plan's actual risks, not boilerplate?
- Would a reviewer reading only "Why this matters" + "Done criteria" understand what they're approving?
- No secret values anywhere in the file — locations and credential types only.
- "Planned at" SHA is filled in and the in-scope paths in the drift check match the Scope section.
- The plan has `## Status` (with Depends on), `## STOP conditions` and an empty `## Outcome`.
