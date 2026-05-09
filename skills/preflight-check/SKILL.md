---
name: preflight-check
description: Use when validating an implementation plan before launching a team via /team-building. Runs plan quality, project readiness, and team hygiene checks. Auto-repairs safe issues, asks for risky ones, blocks on critical failures.
---

# preflight-check

Gatekeeper skill that runs **before** the `/team-building` command spins up a
team. It walks five phases (0.1 → 0.5), each delegating its detail to a
reference file. The output is both a human-readable markdown report **and** a
machine-readable JSON summary block that the invoking command parses to decide
whether to proceed, surface warnings, or block.

> **Read references on demand.** Do **not** preload all five — each phase pulls
> in the file it needs at the moment it runs. This keeps your context budget
> spent on the actual plan and project, not on rules you may not need.

## Inputs

| Input | Required | Description |
|---|---|---|
| `plan_path` | yes | Absolute path to the plan markdown |
| `project_root` | yes | Absolute path to the project |
| `proposed_agents` | no | Roster for the Phase 0.3 budget heuristic |
| `skip_checks` | no | If `true`, lower CRITICAL → WARN |

## Five-phase flow

### 0.1 — Plan-quality scan

Verify the plan file exists, parses as Markdown, contains parseable tasks, and
ideally lists file paths, acceptance criteria, and dependencies.

→ Delegate detail to **`references/plan-quality-rules.md`** (severity table +
detection heuristics + failure examples).

### 0.2 — Project-readiness probe

Detect the project's tech stack(s) from sentinel files in `project_root`,
record canonical test/build/typecheck commands, and run baseline git checks
(in-repo, on a branch, clean tree, `CLAUDE.md` present).

→ Delegate detail to **`references/project-probes.md`** (probe table + git
checks + multi-stack disambiguation rules).

### 0.3 — Team-state hygiene

Confirm no leftover active team is hogging resources, flag orphan tasks, and
sanity-check the context budget against the proposed agent count.

→ Delegate detail to **`references/team-hygiene.md`** (`TeamList` /
`TaskList` checks, context-budget heuristic).

### 0.4 — Auto-repair pass

For every finding from 0.1–0.3, look up its repair recipe. SAFE recipes run
inline (logged). RISKY recipes always go through `AskUserQuestion`. NO_RECIPE
findings are reported untouched.

→ Delegate detail to **`references/auto-repair-recipes.md`** (recipe map +
trigger conditions + pseudocode action steps).

### 0.5 — Gate decision

Aggregate everything still unrepaired, classify by severity, and emit the
final status: `PASS` / `WARN` / `BLOCK`.

→ Delegate detail to **`references/severity-rubric.md`** (comprehensive
severity table aggregating every check + gate decision logic).

## Output contract

Always emit two artefacts in this order:

1. A markdown body summarizing what ran, what failed, what was repaired, and
   what the user must decide next.
2. A fenced JSON block — last thing in the response, exact shape below. The
   command parses **only** this block.

```json
{
  "status": "PASS|WARN|BLOCK",
  "criticals": [{"check": "...", "detail": "..."}],
  "warns": [{"check": "...", "detail": "..."}],
  "repaired": [{"check": "...", "action": "..."}]
}
```

Field rules:
- `status` is `BLOCK` if `criticals` is non-empty (and `skip_checks` is not
  set), otherwise `WARN` if `warns` is non-empty, otherwise `PASS`.
- `repaired` lists every auto-repair that ran successfully (do not re-list
  those entries under `criticals` or `warns`).
- Every entry's `check` must match a check name from the severity rubric so
  downstream tooling can map it.

## Examples

### Example — PASS

```json
{"status": "PASS", "criticals": [], "warns": [], "repaired": []}
```

Body: `Pre-flight OK — 12 checks ran. Plan parsed (8 tasks, 14 file paths, 6 acceptance criteria). Project: Java/Maven on branch feat/x, clean tree. No active team, no orphan tasks.`

### Example — WARN

```json
{
  "status": "WARN",
  "criticals": [],
  "warns": [
    {"check": "dirty_git_tree", "detail": "3 modified files in src/main/java"},
    {"check": "missing_claudemd", "detail": "no CLAUDE.md in project root"}
  ],
  "repaired": [
    {"check": "missing_claudemd", "action": "wrote stub CLAUDE.md with project name + Java/Maven stack"}
  ]
}
```

Body lists each warning, the repair that ran, and asks the user one summary
question: `2 warnings (1 repaired) — proceed with team launch?`

### Example — BLOCK

```json
{
  "status": "BLOCK",
  "criticals": [
    {"check": "plan_no_parseable_tasks", "detail": "no headings or numbered list found in plan.md"},
    {"check": "no_tech_stack_detected", "detail": "no pom.xml/package.json/pyproject.toml/go.mod/Cargo.toml in /Users/x/proj"}
  ],
  "warns": [],
  "repaired": []
}
```

Body numbers each blocker, names the offending file, and tells the user
exactly what to fix before re-running. No team is launched.

## Operating rules

- Run phases in order; a Phase-0.2 failure short-circuits 0.3.
- Read each reference file only when its phase runs.
- Auto-repairs must log to `repaired[]`. On failure, demote back into
  `criticals[]` / `warns[]` with the reason in `detail`.
- Never silently downgrade a CRITICAL — only a successful repair, an
  explicit user decision, or `skip_checks: true` may.
