# Severity Rubric (Phase 0.5)

Two levels. There is no "INFO" — if it doesn't matter enough to surface as a
warning, it doesn't go in the JSON output.

## Levels

| Level | Effect on launch | What the user sees |
|---|---|---|
| **CRITICAL** | Blocks team launch. Cannot proceed without successful repair, explicit user decision, or `skip_checks: true` override. | Numbered list of blockers + the file/path/value at fault + suggested next step. No team is launched. |
| **WARN** | Proceeds, but flagged. | One-line `"N warnings — review?"` summary at the top of the report; full list below. User can launch or abort. |

## Comprehensive check table

Aggregates every check defined in tasks #4–#6.

### From `plan-quality-rules.md` (Phase 0.1)

| Check name | Severity | Source |
|---|---|---|
| `plan_file_missing` | CRITICAL | 0.1 #1 |
| `plan_not_markdown` | CRITICAL | 0.1 #2 |
| `plan_no_parseable_tasks` | CRITICAL | 0.1 #3 |
| `plan_no_file_paths` | WARN | 0.1 #4 |
| `plan_no_acceptance_criteria` | WARN | 0.1 #5 |
| `plan_no_dependencies` | WARN | 0.1 #6 |

### From `project-probes.md` (Phase 0.2)

| Check name | Severity | Source |
|---|---|---|
| `no_tech_stack_detected` | CRITICAL | 0.2 stack detection |
| `multiple_tech_stacks_unresolved` | WARN | 0.2 (only if user declined to pick a primary) |
| `not_in_git_repo` | CRITICAL | 0.2 git check |
| `detached_head` | CRITICAL | 0.2 git check |
| `dirty_git_tree` | WARN | 0.2 git check |
| `missing_claudemd` | WARN | 0.2 git check |
| `ambiguous_test_command` | WARN | 0.2 (when multiple runners co-exist) |

### From `team-hygiene.md` (Phase 0.3)

| Check name | Severity | Source |
|---|---|---|
| `leftover_active_team` | WARN | 0.3 #1 |
| `orphan_tasks` | WARN | 0.3 #2 |
| `context_budget_tight` | WARN | 0.3 #3 (skipped if no roster passed in) |

### Cross-phase / propagated from plan-analysis

| Check name | Severity | Notes |
|---|---|---|
| `missing_pinned_reviewer_agent` | WARN | Demote to CRITICAL only if user policy requires a stack-specific reviewer |

## Gate decision logic

This is what the skill returns as the `status` field of its JSON output.

```text
unrepaired_criticals = [c for c in criticals if c not in repaired]
unrepaired_warns     = [w for w in warns      if w not in repaired]

if skip_checks:
    # User opted to lower CRITICALs to WARNs. Still record them, just
    # don't block. Move them into warns[] for the JSON shape.
    unrepaired_warns.extend(unrepaired_criticals)
    unrepaired_criticals = []

if unrepaired_criticals:
    status = "BLOCK"
elif unrepaired_warns:
    status = "WARN"
else:
    status = "PASS"
```

## Body formatting per status

### `PASS`

One-liner. Example: `Pre-flight OK — 12 checks ran.` Optionally append a
brief stack/branch summary, but keep it under three lines.

### `WARN`

```
Pre-flight WARN — N warning(s), M repaired.

Warnings:
  1. <check>: <detail>
  2. ...

Repaired:
  - <check>: <action>
```

End with one summary question for the user via `AskUserQuestion`: *"Proceed
with team launch?"* (Yes / Abort / Show full report). The command consumes the
JSON to decide whether to ask.

### `BLOCK`

```
Pre-flight BLOCKED — N critical issue(s).

  1. <check>: <detail>
     Suggested fix: <one line>
  2. ...

Re-run /team-building once these are resolved, or pass skip_checks: true to
override (not recommended).
```

No team is launched. The command must abort cleanly.

## Notes on stable check names

The `check` field in every JSON entry **must** match a name from this rubric.
Downstream tooling (the `team-building` command, future analytics) keys on
these names. If a new check is added in a future revision:
1. Add it to the appropriate Phase reference file.
2. Add a row in the table above.
3. Add a recipe (or explicitly document it as NO_RECIPE) in
   `auto-repair-recipes.md`.

Don't fork the naming — every check appears exactly once across the four
reference files.
