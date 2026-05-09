---
name: team-building
description: Coordinate a team of specialist agents to execute an implementation plan, with pre-flight validation.
argument-hint: [plan-path] [--skip-checks]
---

# Team Building

You are the **team lead** for this session. Read a plan, discover which specialist agents fit the work, assemble a real Team (using Claude Code's Team Agents feature), and coordinate them to deliver the implementation.

**Critical rule**: use the Team Agents feature (`TeamCreate`, `TaskCreate`, `Agent` with `team_name`, `SendMessage`) for coordination. Do NOT fall back to plain parallel `Agent` calls without a team — that loses inter-agent communication, shared task tracking, and the ability to route fixes back to the same agent that built the code.

## Input

Plan input: `$ARGUMENTS`

- If `$ARGUMENTS` contains `--skip-checks`, note this flag and **skip Phase 0 entirely** (emergency escape hatch). Remove the flag from the path before reading.
- If `$ARGUMENTS` contains a file path, read it with the `Read` tool and proceed to Phase 0 (or Phase 1 if `--skip-checks` was given).
- If `$ARGUMENTS` is empty, ask the user for the plan — they can provide a file path (e.g., `docs/plans/2026-04-09-feature.md`) or paste the plan content directly. Do not proceed until you have the plan.

---

## Gotchas

Multi-agent failure modes this command must prevent:

1. **Falling back to plain `Agent` calls without a Team.** Symptom: agents work in parallel but can't share TaskList state, can't be messaged, and reviewers can't message implementers for fixes. Cause: skipping `TeamCreate`. Fix: always start Phase 2 with `TeamCreate(...)` — no exceptions.
2. **Two agents owning the same file.** Symptom: merge conflicts, lost work, one agent's edits silently overwritten. Cause: weak boundaries in the teammate prompt. Fix: include explicit file ownership in section 3 of every teammate prompt ("you own X; agent Y owns Z; do not touch Z"). One agent per directory or feature slice.
3. **Teammate stuck in a re-prompt loop with the lead.** Symptom: same teammate keeps messaging "what should I do next?" Cause: ambiguous task description or missing context. Fix: re-issue the task with concrete acceptance criteria, OR escalate the question to the user (don't guess).
4. **Auto-fixing review findings.** Symptom: reviewer flags BLOCKING issues, you start a fix subtask without showing the user. Cause: skipping Phase 4.3 (present findings, wait). Fix: NEVER apply review fixes without the user's go-ahead — even for "obvious" ones.
5. **Forgetting to shutdown the team at the end.** Symptom: stale teammates remain across sessions, future TaskList shows leftover tasks. Cause: skipping Phase 5.3. Fix: send `{"type": "shutdown_request"}` to every teammate, then `TeamDelete()`.

---

## Pinned Agents

These agents **always join the team** regardless of what the plan says. They run in the Review phase after implementation is complete.

| Role           | Preferred subagent_type                                              | Fallback (bundled)        | Phase  |
|----------------|----------------------------------------------------------------------|---------------------------|--------|
| Code Reviewer  | java-code-reviewer / python-code-reviewer / node-code-reviewer       | generic-code-reviewer     | Review |
| Security Audit | java-security-auditor (and stack equivalents)                        | generic-security-auditor  | Review |

> The user can customize this table at any time. Remove rows to stop including an agent, add rows to force-include new ones.

---

## Phase 0: Pre-flight Validation

> **Skip this phase entirely if `--skip-checks` was present in `$ARGUMENTS`.**

Before assembling any team, invoke the pre-flight check skill to validate the plan:

```
Skill(skill="team-building:preflight-check", args=<plan-path>)
```

The skill returns a markdown report plus a JSON block:

```json
{"status": "PASS|WARN|BLOCK", "criticals": [...], "warns": [...], "repaired": [...]}
```

> **Parsing note:** The skill response may contain multiple `json` code blocks (the SKILL.md includes pedagogical examples). Parse only the **LAST** fenced ` ```json ` block in the response.

**After parsing the JSON**, extract from the report body and hold for Phase 1.1: `primary_stack` (e.g., `"java"`, `"python"`, `"node"`), `secondary_stacks` (optional), `test_command`, `build_command`, `typecheck_command`.

**Handle each outcome:**

- **`BLOCK`** — Print the full report, then **abort**. Do NOT proceed to Phase 1. Tell the user: "Pre-flight check found critical issues that must be resolved before the team can be assembled. See the report above."
- **`WARN`** — Print the summary of warnings, then ask the user: "Pre-flight found some warnings (non-blocking). Would you like to review them before proceeding, or continue?" Proceed only after the user responds.
- **`PASS`** — Print a single line: `✅ Pre-flight passed — proceeding to team assembly.` Then continue to Phase 1.

---

## Phase 1: Plan Analysis & Agent Discovery

### 1.1 — Analyze the plan

Invoke the plan-analysis skill to parse the plan and get a proposed agent roster, passing the stack values extracted from Phase 0:

```
Skill(skill="team-building:plan-analysis", args="plan_path=<path> primary_stack=<stack> test_command=<cmd> build_command=<cmd> typecheck_command=<cmd> secondary_stacks=<list-or-empty>")
```

> **Parsing note:** The skill response may contain multiple `json` code blocks (the SKILL.md includes pedagogical examples). Parse only the **LAST** fenced ` ```json ` block in the response.

The skill returns:
- **Parsed tasks** with descriptions, file paths, and dependencies
- **Detected domains** (backend, frontend, database, infra, testing, etc.)
- **Proposed agent roster** matched to available agents in `~/.claude/agents/`

Use this output as the basis for Phase 1.2–1.4 below.

### 1.2 — Discover available agents

Glob both sources and merge into one catalog (**user-level wins** on name conflicts):

```
1. Primary   — Glob: pattern="*.md" path="~/.claude/agents/"
2. Fallback  — Glob: pattern="*.md" path="~/.claude/team-building-plugin/agents/"
              (if path not found, treat as empty — never abort)
```

For each `.md` file, read the **first 10 lines** for frontmatter: `name` (= `subagent_type` when spawning) and `description` (used for matching).

### 1.3 — Confirm agent matching

Review the roster proposed by `team-building:plan-analysis`. Apply these rules when adjusting:

- **One agent per domain** — no duplicate specialist roles
- **Always include pinned agents** from the table above
- Prefer **specific agents** over generic ones (e.g., `java-database-expert` over `java-spring-expert` for DB work)

**Fallback ladder** — for any slot still unmatched after checking the merged catalog:
- `code-reviewer` → use bundled `generic-code-reviewer`
- `security-auditor` → use bundled `generic-security-auditor`
- Implementer / tester without a specialist → fall back to `general-purpose` (built-in)
- Any other unmatched slot → escalate to user before proceeding

> **Transparency rule:** Tell the user whenever a generic fallback is used instead of a specialist — they may want to install a more specific agent.

### 1.4 — Present the team roster

Show the user the proposed team and **wait for confirmation**:

```markdown
## Proposed Team: "[Plan Name]"

| Teammate Name  | Agent Type             | Assigned Tasks       | Phase          |
|----------------|------------------------|----------------------|----------------|
| backend-impl   | java-spring-expert     | Tasks 1, 2, 3        | Implementation |
| db-expert      | java-database-expert   | Task 4               | Implementation |
| test-engineer  | java-test-engineer     | Tasks 5, 6           | Implementation |
| code-reviewer  | java-code-reviewer     | All changed files    | Review         |
| security-audit | java-security-auditor  | All changed files    | Review         |

### Execution Order
- **Group 1** (parallel): backend-impl, db-expert, test-engineer
- **Group 2** (after Group 1): code-reviewer, security-audit

Launch this team?
```

**Do NOT proceed until the user confirms.** If they want changes (swap agents, reassign tasks, add/remove teammates), adjust and re-present.

---

## Phase 2: Team Setup

### 2.1 — Create the team

```
TeamCreate(team_name="<plan-name-slugified>", description="Implementation of <plan name>")
```

Use a short, descriptive slug derived from the plan name (e.g., `payment-webhook-impl`, `user-auth-refactor`).

### 2.2 — Create tasks

For each task from the plan, create a tracked task:

```
TaskCreate(title="<task title>", description="<task details from plan>")
```

Include enough detail so any agent reading the task list understands what needs to happen.

### 2.3 — Spawn implementation teammates

For each **implementation** agent in the confirmed roster, spawn them using the Agent tool. **Spawn all implementation agents in a single message** so they launch in parallel:

```
Agent(
  team_name="<team-name>",
  name="<teammate-name>",         # e.g., "backend-impl"
  subagent_type="<agent-type>",   # e.g., "java-spring-expert"
  prompt="<see prompt structure below>"
)
```

**Do NOT spawn review agents yet** — they come in Phase 4.

### Teammate prompt structure

Each teammate's prompt must be **self-contained**. Paste everything they need — never tell them to read a file for instructions. Include:

1. **Role**: "You are the [role] on a team implementing [plan name]."
2. **Their tasks**: Paste the full task text from the plan. Include exact file paths, expected behavior, and acceptance criteria.
3. **Boundaries**: Which files they own. Which files are owned by OTHER agents (stay out). What is NOT their responsibility.
4. **Integration context**: What other teammates are building and how pieces connect (interfaces, contracts, shared types).
5. **Project conventions**: Tech stack, naming patterns, coding standards relevant to their work.
6. **Team coordination**: "Check TaskList for your assigned tasks. Mark each task completed via TaskUpdate when done. If you're blocked or have questions, send a message to the team lead."
7. **When finished**: "After completing all your tasks, send a summary of what you built, files changed, and any concerns."

---

## Phase 3: Parallel Execution

While teammates work, you coordinate:

1. **Monitor progress** — use `TaskList` to check status periodically
2. **Respond to messages** — teammates will message you with questions or status updates. Messages arrive automatically — you don't need to poll.
3. **Unblock teammates** — if someone is stuck:
   - Provide missing context via `SendMessage`
   - Reassign the task to another teammate if appropriate
   - Escalate to the user if it requires a decision you can't make
4. **Track completion** — when ALL implementation tasks are marked complete, move to Phase 4

Teammates going **idle** between turns is normal. It means they're waiting for input. Send them a message to wake them up when needed.

---

## Phase 4: Review Pipeline

### 4.1 — Run tests

Execute the project's test command. Detect from the plan or project structure:
- `mvn test` (Maven/Java)
- `./gradlew test` (Gradle/Java)
- `npm test` (Node.js)
- `pytest` (Python)
- Or as specified in the plan

### 4.2 — Spawn review agents

Now spawn the **pinned review agents** as teammates. Launch all reviewers in a single message (parallel):

```
Agent(
  team_name="<team-name>",
  name="<reviewer-name>",
  subagent_type="<reviewer-agent-type>",
  prompt="Review all files changed during implementation: <list of changed files with paths>. Report your findings organized by severity: BLOCKING (must fix), IMPORTANT (should fix), MINOR (nice to have)."
)
```

### 4.3 — Present findings to the user

When reviewers finish, **present ALL findings to the user**. Do NOT auto-fix anything.

```markdown
## Review Findings

### Code Review (code-reviewer)
- **BLOCKING**: [description] — `path/to/file.java:42`
- **IMPORTANT**: [description] — `path/to/other.java:15`
- **MINOR**: [description] — `path/to/file.java:88`

### Security Audit (security-audit)
- **BLOCKING**: [description] — `path/to/file.java:78`

---

**What would you like to do?**
1. Fix all blocking issues
2. Fix specific issues (tell me which)
3. Proceed as-is
```

**Wait for the user's decision.**

### 4.4 — Route fixes (if requested)

If the user wants fixes:
- Send fix instructions to the **original implementer** via `SendMessage` — they already have full context from building the code
- After fixes are applied, re-run tests
- If the user wants, re-run reviewers on the fixed code

---

## Phase 5: Wrap Up

Once the user is satisfied:

### 5.1 — Final test run
Run the test suite one last time to confirm everything passes.

### 5.2 — Present summary

```markdown
## Implementation Complete

### What was built
- [bullet points summarizing delivered functionality]

### Files created/modified
- [grouped by teammate/domain]

### Test results
- [pass/fail count, any skipped]

### Review findings addressed
- [what was fixed based on review feedback]

### Remaining suggestions (non-blocking)
- [optional improvements the user can tackle later]
```

### 5.3 — Shutdown the team

Send a shutdown message to each teammate:
```
SendMessage(to="<teammate-name>", message={"type": "shutdown_request"})
```

Do this for every teammate (implementation agents AND review agents).

### 5.4 — Delete the team

```
TeamDelete()
```

This cleans up the team and task directories.

---

## Principles

- **The plan is the spec** — execute what's written, don't re-plan or add scope.
- **Parallel by default** — if tasks don't share state, they run in parallel.
- **One agent, one domain** — clear ownership prevents file conflicts.
- **Always use Team Agents** — TeamCreate + team_name on every Agent call. No exceptions.
- **Context is king** — each teammate gets ALL context upfront. They can't see the plan or the conversation.
- **User decides on review findings** — never auto-fix. Present findings, wait for instructions.
- **Escalate, don't guess** — architectural trade-offs and unclear requirements go to the user.
