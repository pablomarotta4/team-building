# team-building

A Claude Code plugin that turns an implementation plan into a coordinated team of specialist agents — with pre-flight validation, parallel execution, mandatory code review, and a security audit.

You write the plan. The plugin assembles the right agents for the stack, runs them as a real **Team** (using Claude Code's Team Agents feature so they share task state and can message each other), and gates everything behind a quality check that runs before any code gets written.

---

## What's inside

| Component | Type | Purpose |
|---|---|---|
| `/team-building` | Command | Entry point. Reads a plan, runs preflight, builds the team, executes, reviews. |
| `preflight-check` | Skill | Plan-quality + project-readiness gatekeeper. Auto-repairs safe issues, blocks on critical failures. |
| `plan-analysis` | Skill | Detects tech stack from the plan, matches each task to the best specialist agent available. |
| `generic-code-reviewer` | Agent | Language-agnostic code reviewer. Bundled fallback when no stack-specific reviewer is installed. |
| `generic-security-auditor` | Agent | Language-agnostic security auditor. Bundled fallback. Covers OWASP Top 10. |

The two pinned review agents are language-agnostic on purpose so the plugin works out-of-the-box on any project. If you have stack-specific reviewers (`java-code-reviewer`, `python-code-reviewer`, `node-code-reviewer`, etc.) installed, the plugin prefers those automatically.

---

## Installation

Inside Claude Code, run:

```
/plugin marketplace add pablomarotta4/team-building
/plugin install team-building@team-building-marketplace
```

That's it. The `/team-building` command is now available, the two skills are registered, and the two fallback agents are discoverable.

To verify:

```
/plugin
```

You should see `team-building` listed as an installed plugin.

### Updating

When the repo changes upstream, refresh with:

```
/plugin marketplace update pablomarotta4/team-building
```

Then `/plugin reload` (or restart your session) to pick up the new version.

### Uninstalling

```
/plugin uninstall team-building
/plugin marketplace remove team-building-marketplace
```

---

## Usage

### Basic flow

1. **Write a plan** as a markdown file — typically `docs/plans/<date>-<feature>.md`. The `writing-plans` skill (or `superpowers:writing-plans`) is a good way to produce one.
2. **Run the command** with the plan path:
   ```
   /team-building docs/plans/2026-04-09-payment-webhook.md
   ```
3. The command will:
   - **Phase 0** — Run `preflight-check`. If it returns `BLOCK`, you'll see the failures and the team is not assembled. If `WARN`, it'll surface issues and ask you whether to proceed. If `PASS`, it moves on.
   - **Phase 1** — Run `plan-analysis`. Detects the stack, classifies each task, proposes a team roster (specialists + the two pinned review agents).
   - **Phase 2** — `TeamCreate` and dispatch tasks. Specialists work in parallel where possible, with explicit file-ownership boundaries to avoid conflicts.
   - **Phase 3** — Implementation runs. Lead routes blockers, mediates inter-agent questions.
   - **Phase 4** — Code review + security audit run on the diff. Findings are surfaced to you grouped by severity. **Nothing auto-fixes.** You decide what to apply.
   - **Phase 5** — Team shutdown and cleanup.

### Skipping preflight (emergency)

```
/team-building docs/plans/myplan.md --skip-checks
```

This downgrades `CRITICAL` failures to warnings and proceeds. Use rarely — preflight catches real problems.

### Customizing the pinned agents

Open `commands/team-building.md` (in the installed plugin folder or in this repo if you forked it) and edit the **Pinned Agents** table. Remove rows to drop an agent from the Review phase, or add new ones to force-include them. Common customizations:

- Add a `pr-test-analyzer` row to require test-coverage review on every team run.
- Swap `generic-code-reviewer` out for a stricter house reviewer.
- Add a `comment-analyzer` row to enforce documentation hygiene.

---

## Requirements

- **Claude Code** with plugin and Team Agents features enabled. The plugin uses `TeamCreate` / `TaskCreate` / `Agent(team_name=...)` / `SendMessage` for coordination.
- A plan file you actually want executed. The plugin's quality is bounded by the plan's clarity — preflight catches the worst sins (missing acceptance criteria, ambiguous scope, missing test commands) but it's not a substitute for a thoughtful plan.

---

## Repo layout

```
team-building/
├── .claude-plugin/
│   ├── plugin.json           # plugin manifest
│   └── marketplace.json      # marketplace manifest (source: "./")
├── agents/
│   ├── generic-code-reviewer.md
│   └── generic-security-auditor.md
├── commands/
│   └── team-building.md      # the /team-building command
└── skills/
    ├── plan-analysis/
    │   ├── SKILL.md
    │   └── references/
    └── preflight-check/
        ├── SKILL.md
        └── references/
```

The `.claude-plugin/marketplace.json` points at `"./"` so this single folder is both the plugin **and** its own self-contained marketplace — that's why `/plugin marketplace add pablomarotta4/team-building` works directly against the repo root.

---

## License

MIT.
