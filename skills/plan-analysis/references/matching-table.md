# Matching Table — Stack/Domain → Specialist Agents

Lookup table consumed by `plan-analysis`. Each row maps a stack or domain
signal to the four specialist roles. A blank cell means "no specific agent —
fall through to the implementer for that role, or escalate."

## Primary table

| Stack / Domain signal | Implementer | Tester | Reviewer | Security / Specialist |
|---|---|---|---|---|
| `.java`, `.kt`, Spring Boot, controller, `@RestController`, `@Service` | java-spring-expert | java-test-engineer | java-code-reviewer | java-security-auditor |
| `.sql`, migration, JPA, schema, `@Entity`, Flyway, Liquibase | java-database-expert | java-test-engineer | java-code-reviewer | — |
| `.py`, async, LangGraph, `StateGraph`, redis.asyncio, pytest-asyncio | python-langgraph-impl | python-langgraph-impl | python-code-reviewer | — |
| `.ts`, `.js` ESM, Express, Node, `app.use`, `router.get` | node-refactor-specialist | node-refactor-specialist | node-code-reviewer | — |
| `.tsx`, React, frontend, JSX, hooks, components | *(no specific agent — flag in `unmatched[]`)* | — | — | — |
| Generic / unknown | general-purpose | general-purpose | general-purpose | — |

## Generic fallbacks (bundled)

The primary table names **user-installed** specialist agents (e.g.,
`java-code-reviewer` from `~/.claude/agents/`). When the user's
`~/.claude/agents/` does **not** contain the matched specialist,
plan-analysis must walk a fallback chain before declaring the slot
unmatched.

### Fallback chain by role

| Slot | User-specific (preferred) | Generic fallback (bundled) | Last resort |
|---|---|---|---|
| Code review | `java-code-reviewer` / `python-code-reviewer` / `node-code-reviewer` | `generic-code-reviewer` (bundled at `<plugin>/agents/`) | escalate to user |
| Security audit | `java-security-auditor` (and future `python-security-auditor` / `node-security-auditor`) | `generic-security-auditor` (bundled at `<plugin>/agents/`) | escalate to user |
| Implementation | `java-spring-expert` / `python-langgraph-impl` / `node-refactor-specialist` | *(none — no generic implementer is bundled)* | `general-purpose` (built-in) |
| Testing | `java-test-engineer` (or specialist where applicable) | *(none — no generic tester is bundled)* | `general-purpose` (built-in) |

### Resolution rule (this is the exact phrasing for downstream code)

For each slot in the matched row:

1. **User-level match.** Check whether the user has the named agent in
   `~/.claude/agents/<name>.md`. If yes, use it.
2. **Bundled fallback.** If not, check the plugin's bundled directory
   `<plugin_root>/agents/<name>.md` for the role's generic fallback
   (`generic-code-reviewer` for review, `generic-security-auditor` for
   security). If present, use it.
3. **Last resort.**
   - For **implementation** and **testing** slots: fall through to
     `general-purpose` (always available — it is a built-in subagent type).
     Add a `notes[]` line so the user knows quality may be lower than a
     specialist.
   - For **review** and **security** slots: if even the generic fallback is
     missing (the bundled file isn't on disk), the slot is **unfillable**.
     Add the task ID to `unmatched[]` and tell the user to either restore
     the bundled agent or install a stack-specific one.

### What goes in `unmatched[]`

After fallback resolution, `unmatched[]` carries only the cases where:
- A review or security slot has no user-specific agent **and** no bundled
  generic agent on disk.
- A task's strongest stack signal points at a row whose implementer cell is
  intentionally empty (e.g., the React `.tsx` row).

Implementation/testing slots that fell through to `general-purpose` are
**not** added to `unmatched[]` — that's a quality-of-output concern, not a
"can't proceed" concern. They are flagged via `notes[]` instead.

### Implementer/tester slots — why no generic fallback?

Implementation and testing demand stack-specific idiom (Spring annotations,
pytest fixtures, Express middleware order, etc.). A generic implementer
would write code that compiles but doesn't follow the project's framework
conventions — worse than `general-purpose`, which at least admits ignorance
and asks. So we deliberately bundle only the two roles where stack-agnostic
review can still add value: code review and security.

## Signal-detection notes

### Java / Spring
Signals: file extensions `.java`, `.kt`; keywords `Spring`, `@RestController`,
`@Service`, `@Repository`, `@Component`, `controller`, `endpoint`, `bean`.
Also catches `pom.xml` / `build.gradle` mentions.

### Java / Database
Signals: file extension `.sql`; keywords `migration`, `schema`, `JPA`,
`@Entity`, `@Table`, `Flyway`, `Liquibase`, `repository` (when paired with
JPA), `index`, `foreign key`. Use this row when the task is data-layer
focused — even if the file extension is `.java`, the database expert wins
over the generic Spring expert.

### Python (LangGraph / async)
Signals: file extension `.py`; keywords `async def`, `await`, `LangGraph`,
`StateGraph`, `Command(resume=`, `redis.asyncio`, `aioboto3`,
`pytest-asyncio`, `fakeredis`. Note that the same agent handles both
implementation and testing.

### Node / Express ESM
Signals: file extensions `.ts`, `.js`; keywords `Express`, `app.use`,
`router.get`, `import` (ES modules), `package.json` with `"type":"module"`.
Note that the same agent handles both implementation and testing.

### React / frontend
Signals: file extension `.tsx`; keywords `React`, `useState`, `useEffect`,
JSX `<Component`, `props`, `hooks`. **No specialist agent exists.** The
plan-analysis skill must:
1. Add the task to `unmatched[]`.
2. Append a `notes[]` line: `"React/.tsx tasks have no specialist — use agent-architect to mint one, or assign general-purpose for now."`

### Generic / unknown
Default fallback when no signals fire and no primary stack is set. Uses
`general-purpose` for all roles. Should be rare — preflight Phase 0.2
guarantees a stack, so the implementer fallback should usually flow back to
the primary stack's row, not this one.

## Multi-stack projects

When the plan's tasks fan out across multiple stacks (e.g., a Java backend
plus a TypeScript SDK):

1. **Split tasks per stack.** Bucket tasks by their dominant signal — not
   by the project's primary stack.
2. **Build sub-rosters.** Each bucket gets its own implementer / tester /
   reviewer triplet from the table. The team lead remains a single agent.
3. **Note it.** Append a `notes[]` line: `"Multi-stack: <count> tasks per
   stack: <java/spring=N>, <node/esm=M>"`.
4. **Order matters.** If the plan declares dependencies, respect them. The
   command may decide to run sub-rosters sequentially.

Example: a plan with 5 tasks where tasks 1–3 are `.java` Spring controllers
and tasks 4–5 are `.ts` Express routers produces:
- Implementers: `java-spring-expert`, `node-refactor-specialist`
- Reviewers: `java-code-reviewer`, `node-code-reviewer`
- Note: `"Multi-stack: tasks 1–3 → java/spring, tasks 4–5 → node/esm"`

## What to do when no agent matches

If a task's strongest signal points at a row whose implementer is missing
(only the React row, today):

1. Set `implementer: null` on the task.
2. Add the task ID to the top-level `unmatched[]` array.
3. Add a `notes[]` line telling the user how to recover:
   - **Preferred.** Spawn `agent-architect` to design a specialist (e.g.
     `react-frontend-expert`) and re-run plan-analysis.
   - **Fallback.** Assign `general-purpose` and accept lower-quality output.
4. **Do not** silently substitute `general-purpose` — the user owns the
   trade-off.

## Adding a new row

When extending this table (e.g., adding a Rust or Go specialist):

1. Pick the strongest signal — typically a file extension plus 2–3
   distinctive keywords.
2. Add the row in alphabetical / stack-grouped order.
3. Write a "Signal-detection notes" subsection covering ambiguity (which
   keywords win against neighbouring stacks).
4. If the new agent is shared between implementer and tester (like
   `python-langgraph-impl`), say so explicitly in the row and the notes.
5. Update the preflight `severity-rubric.md` only if you also add a new
   check — most additions don't need that.

Treat this file as the **single source of truth** for agent selection. If a
hard-coded agent name appears anywhere else in the plugin, that's a bug.
