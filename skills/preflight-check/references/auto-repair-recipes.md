# Auto-Repair Recipes (Phase 0.4)

Every finding from Phase 0.1–0.3 is keyed by its stable `check` name. This
file maps each name to one of three categories:

- **SAFE** — execute inline, log to `repaired[]`, no user prompt needed.
- **RISKY** — always ask the user via `AskUserQuestion` before acting.
- **NO_RECIPE** — surface in the report, take no automated action.

Findings without an entry here default to **NO_RECIPE**.

---

## SAFE recipes

### `leftover_active_team`

**Trigger.** Phase 0.3 found an active team that the user did **not** elect to
resume.

**Action steps (pseudocode).**
```text
team = the leftover team object
confirm_message = f"About to delete leftover team '{team.name}' (id {team.id}). Anything still in flight?"
# Confirmation here is a one-line "ok to delete?" — SAFE because we already
# asked resume-vs-cleanup in Phase 0.3, and the user picked cleanup.
TeamDelete(team_id=team.id)
log_repair("leftover_active_team", f"deleted team {team.id}")
```

**Success criteria.** `TeamList()` no longer includes that team.

**Log line.** `repaired[].action = "deleted leftover team <id> (<name>)"`.

### `orphan_tasks`

**Trigger.** Phase 0.3 found orphan tasks the user opted to clean up.

**Action steps.**
```text
for tid in orphan_task_ids:
    TaskUpdate(taskId=tid, status="deleted")
log_repair("orphan_tasks", f"deleted N orphan tasks: <id1>, <id2>, ...")
```

**Success criteria.** `TaskList()` returns no incomplete orphan tasks.

**Log line.** `repaired[].action = "deleted N orphan tasks: <id list>"`.

### `missing_claudemd`

**Trigger.** Phase 0.2 reported `CLAUDE.md` missing in `project_root`.

**Action steps.**
```text
project_name = basename(project_root)
stack_summary = primary_stack + (", " + ", ".join(secondary) if secondary else "")
content = f"""# {project_name}

Tech stack: {stack_summary}

TODO: project description — replace this stub with conventions, workflows,
and architectural notes the team should follow.
"""
Write(project_root / "CLAUDE.md", content)
log_repair("missing_claudemd", "wrote stub CLAUDE.md (project name + stack + TODO)")
```

**Success criteria.** File exists and contains the project name plus stack
summary.

**Log line.** `repaired[].action = "wrote stub CLAUDE.md (project name + stack + TODO)"`.

---

## RISKY recipes (always ask via AskUserQuestion)

### `plan_no_acceptance_criteria`

**Trigger.** Phase 0.1 didn't find ≥2 acceptance-criteria keywords. Also
applicable to `plan_no_parseable_tasks` if the user wants to recover instead
of abort.

**Action steps.**
1. `AskUserQuestion`: *"Plan lacks acceptance criteria. Spawn the `Plan` agent
   to flesh it out before launching the team?"* Options:
   - **Yes — flesh it out.** Spawn `Agent(subagent_type="Plan", prompt="Read <plan_path>, identify gaps in acceptance criteria, and produce a revised plan with explicit Given/When/Then or 'tests pass' style criteria for every task.")`.
   - **No — proceed as-is.** Demote the finding to a sticky WARN and continue.
   - **Abort.** Return BLOCK.
2. If the user chose **flesh it out**, present the agent's draft for approval
   via a second `AskUserQuestion`:
   - **Save and use.** Write to `<plan_path>` (back up original to
     `<plan_path>.bak`).
   - **Discard.** Demote to sticky WARN.

**Success criteria.** Either the rewritten plan now passes the acceptance-
criteria heuristic on a re-scan, or the user explicitly chose to proceed
without one.

**Log line.** `repaired[].action = "Plan agent rewrote <path> with acceptance criteria; backup at <path>.bak"`.

### `ambiguous_test_command`

**Trigger.** Phase 0.2 detected multiple test runners (e.g. `mvn test` AND
`./gradlew test` both viable, or `pytest` plus `tox`), and no single canonical
command can be inferred.

**Action steps.**
1. `AskUserQuestion`: *"Multiple test commands detected. Which is canonical
   for this plan?"* Options listed dynamically from probe results, plus
   "Specify another (free text)".
2. Record the user's choice as the project's `test_command` for the
   downstream roster.

**Success criteria.** A single test command is locked in. Any subsequent
agent told "run tests" will use this exact string.

**Log line.** `repaired[].action = "user selected test command: <command>"`.

### `dirty_git_tree`

**Trigger.** Phase 0.2 found uncommitted changes.

**Action steps.**
1. `AskUserQuestion`: *"Working tree has uncommitted changes. How should we
   handle them?"* Options:
   - **Include in scope.** Commit them first (`git add -A && git commit -m
     "pre-team WIP"`), then proceed. We do NOT amend or force-push.
   - **Stash.** `git stash push -u -m "preflight: pre-team-launch <date>"`,
     proceed.
   - **Abort.** Return BLOCK.
2. Execute the user's choice.

**Success criteria.** `git status --porcelain` is empty (or the user
explicitly opted to include changes via a fresh commit).

**Log line.** `repaired[].action = "user chose <option>; tree clean"` or
`"user chose stash; stash ref: <ref>"`.

---

## NO_RECIPE (report-only)

### `missing_pinned_reviewer_agent`

**Trigger.** Plan-analysis (the next skill) couldn't find a reviewer agent
matching the detected stack — for example, a Rust project with no
`rust-code-reviewer` configured.

**Action.** Surface in the markdown body with this exact instruction:
*"No reviewer agent matches the detected stack. Install one or create a new
specialist via the `agent-architect` subagent before launching."*

**No state change.** This finding stays in `criticals[]` (or `warns[]` if the
project allows fall-through to `general-purpose` review).

### Catch-all

Anything not listed above stays in its severity bucket untouched. Do not
invent recipes — silent unsafe repairs are worse than visible failures.

---

## Repair-pass loop

```text
for finding in collected_findings:
    recipe = RECIPE_MAP.get(finding.check)
    if recipe is None:
        keep finding in original bucket  # NO_RECIPE
        continue
    if recipe.kind == "SAFE":
        try:
            recipe.action(finding)
            move finding into repaired[]
            log details
        except Exception as e:
            keep finding; append "auto-repair failed: " + str(e) to detail
    elif recipe.kind == "RISKY":
        decision = recipe.ask_user(finding)
        execute decision; move into repaired[] iff success
```

After the loop, recompute the JSON `status` from the remaining `criticals[]`
and `warns[]`.
