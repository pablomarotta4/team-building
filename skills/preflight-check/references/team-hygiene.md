# Team-State Hygiene (Phase 0.3)

Make sure the runtime is in a clean state before launching a new team. A
leftover team or orphan tasks from a previous session will collide with the
new team's coordination loop.

## Checks

### 1. No leftover active team

**Probe.** Call `TeamList()`. Inspect the returned teams.

**Pass condition.** No team is currently active in this session/scope.

**Failure (severity: WARN).**
- An active team exists. Two follow-up actions, presented to the user via
  `AskUserQuestion`:
  - **Resume.** Don't launch a new team — hand control back to the existing
    one.
  - **Cleanup.** Call `TeamDelete()` on the leftover team, then proceed with
    the new launch.

**Stable check name.** `leftover_active_team`.

**Why this is WARN, not CRITICAL.** Plenty of legitimate flows resume an
existing team. The user owns the call.

### 2. No orphan tasks

**Probe.** Call `TaskList()`. An orphan task is one with `status` ≠
`completed` AND no team context (i.e. it doesn't belong to any active team
the user is about to resume).

**Pass condition.** Either no incomplete tasks, or all incomplete tasks
belong to a team the user just opted to resume.

**Failure (severity: WARN).**
- List the orphan IDs and subjects in the markdown body.
- Offer two `AskUserQuestion` actions:
  - **Cleanup.** `TaskUpdate(taskId, status="deleted")` for each orphan.
  - **Keep.** Leave them — user knows what they are.

**Stable check name.** `orphan_tasks`.

### 3. Context-budget heuristic

**Probe (heuristic — leave a comment in code that this is approximate).**

```text
# Heuristic: assume each spawned agent receives ~5K tokens of prompt
# (system prompt + plan excerpt + tool list). This is a rough estimate;
# real values vary with plan size and agent definitions.
ESTIMATED_PROMPT_PER_AGENT = 5_000
proposed_agents = len(roster)  # passed in from the command
estimated_total = proposed_agents * ESTIMATED_PROMPT_PER_AGENT

remaining_budget = current_context_window_remaining  # best-effort
threshold = remaining_budget * 0.5

if estimated_total > threshold:
    emit_warn("context_budget_tight", ...)
```

**Pass condition.** `estimated_total <= remaining_budget * 0.5`.

**Failure (severity: WARN).**
- Tell the user how many agents were proposed, the estimated cost, and the
  remaining budget.
- Offer one `AskUserQuestion` action: reduce the roster (drop the lowest-
  priority specialists) or proceed anyway.

**Stable check name.** `context_budget_tight`.

**Caveat.** The 5K-per-agent figure is intentionally conservative. Treat the
output as a yellow flag, not a hard cap. If `proposed_agents` isn't supplied,
**skip this probe** rather than guess.

## Outputs from this phase

Stable check names:

- `leftover_active_team` (WARN)
- `orphan_tasks` (WARN)
- `context_budget_tight` (WARN — only if `proposed_agents` was provided)

Each entry's `detail` should carry the actionable specifics: team name + ID
for leftover team, comma-separated task IDs for orphans, and the
agent-count + estimated tokens + remaining budget for the budget warning.
