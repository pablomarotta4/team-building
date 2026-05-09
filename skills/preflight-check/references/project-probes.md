# Project-Readiness Probes (Phase 0.2)

Detect the project's tech stack(s) and confirm the working tree is in a sane
state before launching agents. All probes run from `project_root`.

## Tech-stack probes

Each probe is a **sentinel file** (or set of files) that signals a stack, plus
the canonical commands that stack uses for testing/building/typechecking.

| Sentinel | Stack | Test command | Build / compile | Typecheck |
|---|---|---|---|---|
| `pom.xml` | Java / Maven | `mvn test` | `mvn compile` | (compiler) |
| `build.gradle` or `build.gradle.kts` | Java / Gradle | `./gradlew test` | `./gradlew compileJava` | (compiler) |
| `package.json` with `"type":"module"` | Node / ESM | `npm test` | `npm run build` (if defined) | `tsc --noEmit` (if `tsconfig.json` present) |
| `package.json` (no type module) | Node / CJS | `npm test` | `npm run build` (if defined) | `tsc --noEmit` (if `tsconfig.json` present) |
| `pyproject.toml` or `requirements.txt` | Python | `pytest` | (none — interpreted) | `pyright` (if config present) or `mypy` |
| `go.mod` | Go | `go test ./...` | `go build ./...` | (compiler) |
| `Cargo.toml` | Rust | `cargo test` | `cargo build` | (compiler) |

### Detection rules

1. **Iterate every sentinel.** Check existence with `os.path.exists` or a
   shell `test -f`. Don't stop at the first hit — multi-stack projects exist.
2. **`package.json` disambiguation.** Read the file, look for `"type":
   "module"`. ESM and CJS are recorded as different stacks because lint /
   migration concerns differ.
3. **Multiple stacks detected.** List all of them. Use `AskUserQuestion` to
   ask the user which one is **primary** for this plan (the rest are still
   recorded but only the primary's test command is treated as canonical).
4. **No stack detected.** This is **CRITICAL**. The team has no way to verify
   work — fail with check `no_tech_stack_detected` and stop Phase 0.2.

### Probing pseudocode

```text
detected = []
for sentinel, stack_info in PROBES:
    if file_exists(project_root / sentinel):
        if sentinel == "package.json":
            type_module = json.load(...).get("type") == "module"
            stack_info = NODE_ESM if type_module else NODE_CJS
        detected.append(stack_info)

if not detected:
    emit_critical("no_tech_stack_detected", ...)
elif len(detected) > 1:
    primary = ask_user("Which stack is primary for this plan?", detected)
else:
    primary = detected[0]
```

### Recording the canonical commands

Once the primary stack is locked, record the resolved commands so the
implementer/reviewer agents can be told exactly what to run. These should
flow back into the plan-analysis skill's roster output.

## Git checks

Run all four. Severities below.

| Check | Severity | Probe |
|---|---|---|
| Inside a git repo | CRITICAL | `git rev-parse --is-inside-work-tree` returns `true` |
| On a branch (not detached HEAD) | CRITICAL | `git symbolic-ref -q HEAD` succeeds |
| Clean working tree | WARN | `git status --porcelain` returns empty |
| `CLAUDE.md` exists in project root | WARN | `os.path.exists(project_root / "CLAUDE.md")` |

### Per-check details

#### Inside a git repo — CRITICAL

If the user dropped a plan into a directory that isn't a repo, the team
cannot create commits, branches, or PRs. Stop.

Failure example. User runs `/team-building` against `/tmp/scratch` —
`is-inside-work-tree` returns false.

#### On a branch — CRITICAL

Detached HEAD means commits go nowhere. The team would silently lose work.

Failure example. After `git checkout <sha>`, HEAD is detached. Refuse to
launch.

Repair. Tell the user to `git switch -c <branch>` from current HEAD, or
checkout an existing branch.

#### Clean working tree — WARN

Uncommitted changes mean the team's diff will be tangled with whatever the
user was doing. Three options for the user (see `dirty_git_tree` in
auto-repair-recipes):
1. Include uncommitted changes in scope (commit them first).
2. Stash them.
3. Abort.

#### `CLAUDE.md` exists — WARN

Specialist agents rely on CLAUDE.md for project conventions. Missing it →
generic recommendations. Auto-repair writes a minimal stub (project name +
detected stack + TODO marker) — see `missing_claudemd` recipe.

## Outputs from this phase

Stable check names for the JSON summary:

- `no_tech_stack_detected` (CRITICAL)
- `multiple_tech_stacks_unresolved` (WARN — only if user declined to pick)
- `not_in_git_repo` (CRITICAL)
- `detached_head` (CRITICAL)
- `dirty_git_tree` (WARN)
- `missing_claudemd` (WARN)

Successful detections also feed forward — record `primary_stack`,
`test_command`, `build_command`, `typecheck_command` in the markdown body for
the command's downstream use, but they are not severity entries.
