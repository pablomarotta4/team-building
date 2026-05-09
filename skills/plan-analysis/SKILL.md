---
name: plan-analysis
description: Use after preflight-check passes to parse an implementation plan and propose a team roster. Detects tech stack from file paths, matches each domain to the best specialist agent.
---

# plan-analysis

Phase 1 of the `/team-building` flow. Runs **after** `preflight-check`
returns `PASS` or `WARN`. Reads the validated plan, extracts tasks and
detected stacks, and produces a structured roster the command will use to
spawn specialists.

## Inputs

| Input | Required | Description |
|---|---|---|
| `plan_path` | yes | Absolute path to the (already-validated) plan markdown |
| `primary_stack` | yes | Stack chosen in preflight Phase 0.2 (e.g. `java/maven`) |
| `secondary_stacks` | no | Other detected stacks (multi-stack repos) |
| `test_command` / `build_command` / `typecheck_command` | yes | Resolved canonical commands from preflight |

## Procedure

### 1. Extract tasks from the plan

Reuse the regexes from `preflight-check/references/plan-quality-rules.md`:

- `(?m)^#{1,3}\s+(.+)$` — heading-style tasks
- `(?m)^\s*\d+\.\s+(.+)$` — numbered list

For each match:
- `id` = sequential index (1-based)
- `title` = captured text (trimmed)
- `body` = lines from this task to the next match (used for path/keyword
  extraction)

### 2. Detect domain signals per task

For each task `body`, scan for:

- **File extensions.** Regex `[\w./-]+\.([a-z]{1,4})\b` — group by extension.
- **Framework keywords.** Substring search for the signals listed in
  `references/matching-table.md` (Spring, JPA, LangGraph, Express, React,
  etc.).
- **Concept keywords.** `migration`, `schema`, `controller`, `webhook`,
  `frontend`, `async`, etc.

A task may emit **multiple** domain signals — that's fine, it just means
the task touches multiple specialists' areas.

### 3. Look up specialists

Resolution happens on **two axes**: pick the right row of the matching
table, then fill each slot in that row using the fallback chain.

**Axis 1 — row selection.** For each task:

1. **Strong signal** — a file extension or framework keyword that maps to a
   single row.
2. **Concept signal** — a keyword that maps to a row.
3. **Fallback to primary stack** — if the task body has no signals, use the
   primary stack's row (resolved in preflight).
4. **Generic / unknown row** — only if nothing else matches.

**Axis 2 — slot fill.** For each slot (implementer / tester / reviewer /
specialist) in the chosen row, walk the fallback chain in
`references/matching-table.md` § *"Generic fallbacks (bundled)"*:

1. User-installed agent at `~/.claude/agents/<name>.md` if present.
2. Plugin's bundled fallback at `<plugin>/agents/<name>.md` (review →
   `generic-code-reviewer`; security → `generic-security-auditor`).
3. Last resort: `general-purpose` for impl/test slots; `unmatched[]` for
   review/security slots if even the bundled generic file is missing.

For each task, record:
- `implementer`
- `tester` (often the same agent as implementer)
- `reviewer`
- `specialist` (security/database/etc., optional)

### 4. Aggregate into a roster

Deduplicate across tasks. The final roster is:
- The set of unique implementers (with the tasks each owns).
- The set of unique reviewers (one per stack).
- Specialist agents on demand (security auditor if any task has auth/crypto
  signals; database expert if any task has SQL/migration signals).
- A team lead — always added.

### 5. Output

Return a markdown summary plus a fenced JSON block:

```json
{
  "tasks": [
    {"id": 1, "title": "...", "stack": "java/spring", "implementer": "java-spring-expert", "tester": "java-test-engineer", "reviewer": "java-code-reviewer", "specialist": null}
  ],
  "roster": {
    "lead": "team-lead",
    "implementers": ["java-spring-expert"],
    "testers": ["java-test-engineer"],
    "reviewers": ["java-code-reviewer"],
    "specialists": ["java-security-auditor"]
  },
  "unmatched": [],
  "notes": []
}
```

`unmatched[]` lists tasks where no specialist row matched and the user must
intervene (most commonly: frontend React/`.tsx`). `notes[]` is for
human-readable observations the command should surface verbatim (e.g.
"Multi-stack project: Java + Python tasks split across rosters").

## Multi-stack handling

If `secondary_stacks` is non-empty:
- Bucket tasks by their detected primary signal.
- Build a separate sub-roster per stack.
- Add a `notes[]` entry: `"Multi-stack: tasks 1–3 → java/spring, tasks 4–7 → python"`.
- The command may decide to launch one team with mixed specialists or
  split into sequential teams.

## When no agent matches

If a task's strongest signal points at a stack with **no row** in the
matching table (e.g., raw `.tsx` for React frontend):
1. Still record the task in `tasks[]` with `implementer: null`.
2. Append the task ID to `unmatched[]`.
3. Append a `notes[]` line directing the user to the `agent-architect`
   subagent to mint a specialist, or to fall back to `general-purpose`.

## Operating rules

- Read `references/matching-table.md` once at the start, not per-task.
- Don't invent agent names. Every implementer/reviewer/specialist value
  must come from a row in the matching table — or be `null` (with the task
  added to `unmatched[]`).
- Keep the markdown body terse; the JSON is the contract.
