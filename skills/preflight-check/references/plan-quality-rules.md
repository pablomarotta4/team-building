# Plan-Quality Rules (Phase 0.1)

Detection rules for the plan markdown file. Run them in the order below — each
check assumes the previous ones passed (no point counting tasks if the file
doesn't exist).

## Severity table

| # | Check | Severity | Detection heuristic |
|---|---|---|---|
| 1 | File exists, readable | CRITICAL | `os.path.exists(plan_path)` AND file is non-empty |
| 2 | Parses as Markdown | CRITICAL | Successfully read as UTF-8 text; no binary bytes |
| 3 | ≥1 task/step parseable | CRITICAL | Regex `(?m)^#{1,3}\s+\S` OR `(?m)^\s*\d+\.\s+\S` matches at least once |
| 4 | File paths mentioned | WARN | Regex `[\w./-]+\.[a-z]{1,4}\b` matches ≥1 occurrence (excluding URLs) |
| 5 | Acceptance criteria present | WARN | Case-insensitive keyword search: `must`, `should`, `given`, `when`, `then`, `tests pass`, `verify`, `acceptance` — at least 2 distinct keywords |
| 6 | Dependencies declared | WARN | Case-insensitive search: `after Task`, `blocks`, `depends on`, `Group \d`, `prerequisite` — at least 1 hit |

## Per-check details

### 1. File exists, readable — CRITICAL

**Heuristic.** `Read(plan_path)` succeeds and returns non-zero bytes.

**Failure example.** `plan_path=/Users/x/docs/plans/missing.md` → file not
found.

**Suggested fix.** Ask the user for the correct plan path, or list candidates:
`ls docs/plans/*.md`. Do NOT auto-create — that mints a phantom plan.

### 2. Parses as Markdown — CRITICAL

**Heuristic.** The file decodes as UTF-8 and contains at least one ASCII-range
character. (We don't run a full markdown AST parse — too heavy. Just confirm
it's text.)

**Failure example.** Binary `.docx` accidentally renamed to `.md`, or
UTF-16-encoded export with BOM the parser chokes on.

**Suggested fix.** Convert to plain UTF-8 markdown. Tell the user the encoding
detected.

### 3. ≥1 task/step parseable — CRITICAL

**Heuristic.** At least one of:
- `(?m)^#{1,3}\s+\S` — markdown heading levels 1–3
- `(?m)^\s*\d+\.\s+\S` — numbered list

**Failure example.** A plan written as a single prose paragraph with no
headings, no list. Nothing for the team to claim as a task.

**Suggested fix.** Trigger the `plan_no_acceptance_criteria` repair recipe
(spawn the `Plan` agent to flesh out the document) — see auto-repair-recipes.

### 4. File paths mentioned — WARN

**Heuristic.** Regex `[\w./-]+\.[a-z]{1,4}\b` (excluding `http://`/`https://`
matches). Need ≥1 hit.

**Failure example.** Plan describes "add an endpoint and write tests" without
naming any concrete file. Implementers will guess paths and drift.

**Suggested fix.** Warn the user; recommend they annotate each task with the
file(s) it touches before launching the team.

### 5. Acceptance criteria present — WARN

**Heuristic.** Case-insensitive keyword presence. Looking for at least **two
distinct** keywords from: `must`, `should`, `given`, `when`, `then`, `tests
pass`, `verify`, `acceptance`. Two-distinct rule avoids false positives from a
single throwaway "should" in a paragraph.

**Failure example.** Tasks listed as bare imperatives ("Add login form", "Wire
the route") with no testable success criteria. The reviewer can't tell when
implementation is done.

**Suggested fix.** Trigger the `plan_no_acceptance_criteria` repair recipe.

### 6. Dependencies declared — WARN

**Heuristic.** Case-insensitive search for `after Task`, `blocks`, `depends
on`, `Group \d` (regex), `prerequisite`. ≥1 hit.

**Failure example.** Eight tasks listed flat with no ordering. The team may
parallelise tasks that actually need each other's output.

**Suggested fix.** Warn the user; recommend they annotate dependencies or
group tasks. The orchestrator may still run, but parallel-execution gains will
be limited.

## Outputs from this phase

Each rule that fails contributes one entry to either `criticals[]` or
`warns[]` in the JSON summary. Use a stable `check` name matching the rubric
table:

- `plan_file_missing`
- `plan_not_markdown`
- `plan_no_parseable_tasks`
- `plan_no_file_paths`
- `plan_no_acceptance_criteria`
- `plan_no_dependencies`

`detail` should include the offending value (path, encoding, regex miss
count) — anything that helps the user act without re-running the check.
