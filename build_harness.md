# Build a Harness

> A step-by-step recipe for turning any repository into a repository that an AI coding agent can start, verify, resume, and improve without asking a human.

This guide is distilled from the reference implementation in [`projects/project-06/solution`](projects/project-06/solution). That project is a complete Electron + TypeScript + React application wrapped in a strong operating harness (with a few deliberate gaps, listed in Section 1.6): instruction files, a machine-readable feature list, a one-command initializer, session handoff notes, a clean-state checklist, structured logging, benchmarks, a cleanup scanner, an evaluator rubric, and a quality document.

The goal is not to copy those files blindly. The goal is to reproduce the **roles** they play, using the commands and conventions of whatever repository you are working in.

**How to use this guide.**

- Work through the steps in order. Each step ends with an exit gate; do not start the next step until the gate passes.
- Commit after each step, so a failed experiment can be reverted without losing the previous improvements.
- Treat the templates as starting points, not scripture. Replace every placeholder with a command or path from your repository.
- If you only have one day, do Steps 0-6. If you have one week, do Steps 0-11. The full sequence is for repositories that will be maintained by agents over many sessions.

**The phases.**

1. Measure the starting point (Step 0).
2. Make the repository self-describing (Steps 1-3).
3. Make state durable and scope explicit (Steps 4-6).
4. Make "done" objective (Steps 7-9).
5. Make runtime observable (Steps 10-11).
6. Refactor, enforce, and operate (Steps 12-14).

---

## 0. What You Get

After following this recipe, a brand-new agent session — with no memory of previous chats — should be able to answer five questions using only the repository:

1. **What is this system?**
2. **How is it organized?**
3. **How do I run it?**
4. **How do I verify it?**
5. **Where are we now, and what is the next best step?**

If any answer requires asking a human, the repository has a blank spot. The harness exists to remove those blank spots.

A harness is not a framework, a library, or a prompt. In plain English:

> **The harness is everything around the model that determines whether the model's work can be started, checked, resumed, and trusted.**

That includes the instructions the agent reads, the commands it can run, the state it can inspect, the feedback it receives, and the environment it runs in.

---

## 1. The Reference: What a Standard Harness Looks Like

### 1.1 Inventory of `projects/project-06/solution`

| Artifact | Role | What "good" contains | Reference size |
|---|---|---|---|
| `AGENTS.md` | Entry router and operating rules | Project context, ordered startup, hard constraints, commands, docs links, definition of done, end-of-session routine | 104 lines |
| `CLAUDE.md` | Tool-specific quick reference | Commands, key files, architecture rules, how to add a feature | 86 lines |
| `docs/ARCHITECTURE.md` | System map | Layer diagram, dependency direction, data flows, IPC channels, storage layout, logging contract | 208 lines |
| `docs/PRODUCT.md` | Product contract | User-visible features, flows, UI layout, behavior | 105 lines |
| `docs/RELIABILITY.md` | Operational contract | Logging levels, clean-state reset, benchmarks, cleanup scanner | 178 lines |
| `feature_list.json` | Scope and state | 15 features, each with status, evidence, and a `testedAt` timestamp | 126 lines |
| `claude-progress.md` | Session log | Current verified state plus one record per session: goal, completed, decisions, evidence, next action | 40 lines |
| `session-handoff.md` | Compact handoff | What was accomplished, what remains, decisions, files modified, blockers, next steps | 81 lines |
| `init.sh` | One-command startup | Install, type-check, build, verify harness files, verify sample data, print start command | 70 lines |
| `clean-state-checklist.md` | Exit gate | 41 checkboxes across build, architecture, runtime, logging, data integrity, performance, repository, scripts | 68 lines |
| `evaluator-rubric.md` | Review rubric and completed assessment | Scored criteria, evidence, defects, overall verdict | 70 lines |
| `quality-document.md` | Quality snapshot and tracker template | Per-area or per-dimension grades, evidence, known debt, overall grade | 61 lines |
| `scripts/check-architecture.sh` | Boundary enforcement | Executable checks for forbidden imports and layer violations | 85 lines |
| `scripts/benchmark.sh` | Performance signal | Import, indexing, query, and integrity tasks with pass/fail thresholds | 180 lines |
| `scripts/cleanup-scanner.sh` | Hygiene signal | Orphaned, dangling, missing, inconsistent, and stale artifacts | 253 lines |
| `src/services/logger.ts` | Runtime observability | JSON log entries with timestamp, level, service, message, and structured data | 136 lines |

This is not a checklist of files you must create. It is a checklist of **jobs that must be done**. A Python CLI project may not need an Electron preload layer, but it still needs an architecture map, a feature list, a startup command, an exit gate, and runtime signals.

### 1.2 The Five Subsystems

The course model describes a harness as five subsystems. The reference project maps to them cleanly:

| Subsystem | Question it answers | In `project-06/solution` |
|---|---|---|
| **Instructions** | What are the rules and where is the map? | `AGENTS.md`, `CLAUDE.md`, `docs/` |
| **Environment** | Can I run it reproducibly? | `package.json`, `init.sh`, TypeScript configs; no lockfile is checked in (see Section 1.6) |
| **Tools** | What can I execute to inspect and change the system? | Shell, build scripts, test runner, benchmark, scanner |
| **State** | Where are we, and what is next? | `feature_list.json`, `claude-progress.md`, `session-handoff.md`, git history |
| **Feedback** | How do I know it actually works? | Tests, `init.sh`, architecture checks, benchmark, clean-state checklist, evaluator rubric, quality document |

A missing subsystem shows up as a specific failure:

- Missing instructions -> the agent invents conventions.
- Missing environment -> the agent spends the session fixing setup.
- Missing tools -> the agent cannot verify or diagnose.
- Missing state -> the next session redoes completed work.
- Missing feedback -> the agent declares victory too early.

### 1.3 The Fresh-Session Test

This is the cheapest harness audit. Open a new agent session with access only to the repository. Ask the five questions from Section 0. Do not explain anything from chat.

Score each answer:

- **2** — answered directly from repository files.
- **1** — answered, but only after searching and guessing.
- **0** — not answerable without a human.

A repository is harness-ready when all five answers score 2. This test is intentionally blunt. It exposes missing maps faster than reading a long checklist.

### 1.4 Why the Starter Matters

`projects/project-06/starter` contains essentially the same product code as the solution. The difference is the operating surface:

- The starter has only a short `AGENTS.md` (about 1.2 KB).
- It has no `CLAUDE.md`, no `feature_list.json`, no `init.sh`, no `session-handoff.md`, no benchmark script, and no cleanup scanner.
- Its own `AGENTS.md` says: "This starter intentionally has a weak harness surface."

The lesson is important: **the product code is not the differentiator; the ability to operate and verify the product is.** A capable model in a weak harness will still wander, forget, and declare false victories. The same model in a strong harness has a map, a queue, a definition of done, and a flight recorder.

### 1.5 Maturity Levels

Use these levels to locate your repository before you start:

| Level | Name | What exists | Typical failure |
|---|---|---|---|
| **L0** | Ad hoc | README only | Agent guesses commands and conventions |
| **L1** | Startable | Entry file + one-command setup | Agent can run it, but cannot verify or resume |
| **L2** | Stateful | Feature list + progress/handoff | Agent knows what is next, but "done" is still subjective |
| **L3** | Verifiable | Tests, e2e, architecture checks, definition of done | Agent can prove behavior, not diagnose runtime |
| **L4** | Observable | Structured logs, benchmarks, cleanup scanner | Agent can diagnose, measure, and detect drift |
| **L5** | Self-correcting | CI enforcement, independent evaluation, periodic audit, module refactoring | Harness decays unless actively maintained |

Many repositories start at L0 or L1. The recipe below moves you to L5 in small, reversible steps.

### 1.6 What the Reference Does Not Prove

The reference is a teaching artifact, not a certification. It demonstrates the shape of a strong harness, but it has several deliberate or accidental gaps. Do not copy these:

- **Automated tests are configured but missing.** `package.json` defines `npm test` with Vitest, but the checked-in solution contains no `.test` or `.spec` files. Its evidence is the build, runtime checklist, and benchmark. Treat this as a gap to close in your repository; Step 7 requires real tests and e2e coverage.
- **The evaluator rubric is a completed assessment, not an enforced independent role.** The file contains scores and evidence, but nothing prevents the same session from generating and grading the work. Step 8 adds role separation; the reference does not demonstrate it.
- **The quality document is a snapshot, not a trend.** It records one set of grades and an overall A, without history across updates. Use it as a template, then update yours over time.
- **No lockfile or runtime version pin.** The checked-in solution has `package.json` but no `package-lock.json`, `pnpm-lock.yaml`, or `yarn.lock`, and no `.nvmrc`. `npm install` can resolve different versions over time. Commit a lockfile and pin the runtime.
- **`init.sh` installs with `npm install`, not a frozen-lockfile install.** Prefer `npm ci` or the equivalent for reproducibility.
- **The feature list uses `status: "pass"` and has no per-feature `verification` field.** It records evidence and `testedAt`, but the verification steps are not in the file. The recipe's schema adds `verification`; use one schema consistently and validate it.
- **`AGENTS.md` and `CLAUDE.md` overlap.** The reference serves two tools, but duplicated rules drift. For a new repository, keep one canonical file and make the other a thin pointer.
- **The benchmark's timing fallback can be coarse.** It prefers `python3` for sub-second timing and falls back to `date +%s`, which has one-second resolution. Use a monotonic millisecond clock in your implementation.
- **No CI configuration is checked in.** `init.sh` and the scripts can be run manually, but nothing enforces them on every change. Add the pipeline from Step 13.
- **The helper scripts are bash/Python.** They assume a Unix-like shell and Python 3. On Windows, run them under WSL/Git Bash or port them, and test the port against seeded failures.
- **The layers are Electron-specific.** The main/preload/renderer/services/shared split illustrates boundaries, not a universal architecture. Adapt the dependency direction to your stack.

None of these gaps invalidates the reference. They are the difference between a teaching example and a production harness. Close them in your repository, and record the decisions.

---

## 2. The Recipe at a Glance

### 2.1 The One-Sentence Formula

> **A harness-ready repository = a short entry file + a tested startup path + machine-readable state + an executable definition of done + runtime signals + a clean exit + independent review + automated enforcement.**

The rest of this guide expands that sentence into 15 concrete steps (0-14), including the baseline audit.

### 2.2 The 15 Steps

| # | Step | Output | Exit gate |
|---|---|---|---|
| 0 | Baseline audit | `docs/HARNESS-AUDIT.md` or a section in `claude-progress.md` | Commands measured, five-question score recorded, top failures listed |
| 1 | `AGENTS.md` | Short entry router | Fresh session answers all five questions; file is <= 200 lines |
| 2 | Docs map | `docs/ARCHITECTURE.md`, `docs/PRODUCT.md`, module docs | A new agent can trace one feature end-to-end from docs |
| 3 | `init.sh` | One-command setup and baseline verification | Clean clone passes in under 5 minutes |
| 4 | `feature_list.json` | Machine-readable scope contract | Validator passes; at most one `in_progress`; every `passing` has evidence |
| 5 | Progress and handoff | `claude-progress.md`, `session-handoff.md` | A new session reconstructs state in under 5 minutes |
| 6 | Clean state | `clean-state-checklist.md` plus reset mechanism | Checklist passes; reset is repeatable and scoped |
| 7 | Verification ladder | Full check, e2e smoke, architecture check | Clean clone passes; a failing test blocks `passing`; seeded boundary violation fails |
| 8 | Evaluator rubric | `evaluator-rubric.md` plus role separation | Score is evidence-backed; evaluator can work without the generator |
| 9 | Quality document | `quality-document.md` | Every module has a grade and the lowest one has an action |
| 10 | Structured logging | Logger module and event contract | 100% parseable JSON; critical flows emit start/success/failure |
| 11 | Benchmark and scanner | `scripts/benchmark.*`, `scripts/cleanup-scanner.*` | Baseline recorded; scanner finds a seeded issue; clean repo has no false positives |
| 12 | Module refactor | Boundaries, module docs, dependency rules | Architecture check passes; tests stay green; a new agent can find the right module |
| 13 | CI and operating loop | CI pipeline, session protocol, maintenance cadence | Red CI blocks merge; two consecutive sessions follow the protocol |
| 14 | Acceptance gate | Scorecard and rollout plan | Score >= 20/24 with no critical zero |

### 2.3 Why This Order

The order is deliberate:

1. **Make the repository readable first.** An agent that cannot find the commands will not benefit from a feature list or a logging system.
2. **Make state durable before long work begins.** Feature scope and session notes prevent duplicate work and drift.
3. **Make "done" objective before adding more code.** Tests, e2e, architecture checks, and independent review are the highest-return investment.
4. **Add observability before refactoring.** Logs, benchmarks, and scanners give you a safety net and a diagnosis path. Refactoring without them is unverifiable churn.
5. **Refactor modules after the safety net exists.** Module boundaries are easier to enforce when tests, logs, and performance baselines can prove that behavior did not change.
6. **Automate enforcement last.** CI turns the harness from a document into a gate.

### 2.4 Minimum Viable Harness

If you can only do part of the work, do Steps 0-7. That gives you:

- a short entry file,
- a documentation map,
- a one-command startup,
- a machine-readable feature list,
- cross-session state,
- a clean exit,
- and an executable definition of done.

This alone eliminates most "the agent forgot / wandered / claimed done too early" failures. Steps 8-14 make the harness stronger, measurable, and self-maintaining.

---

## 3. Before You Start: Project Profile

Fill this in before creating files. Use commands you have actually run, not commands you think should work. If a row cannot be filled, that is a finding to investigate, not a blocker to skip.

| Field | Your repository |
|---|---|
| Repository / product name | |
| One-sentence purpose | |
| Primary language and runtime version | |
| Package manager and lockfile | |
| Install command | `<INSTALL>` |
| Dev / start command | `<DEV>` |
| Build command | `<BUILD>` |
| Type-check command | `<TYPECHECK>` |
| Lint / format command | `<LINT>` |
| Unit / integration test command | `<TEST>` |
| End-to-end or smoke command | `<E2E>` |
| Full verification command | `<FULL_CHECK>` |
| Runtime data directories | |
| Reset / clean-data command | `<RESET>` |
| Log destination and level variable | `<LOG_LEVEL>` |
| Three to seven critical user flows | |
| External services or credentials needed | |
| Owner and maintenance cadence | |

Keep this profile in `AGENTS.md`, `docs/ARCHITECTURE.md`, or `claude-progress.md`. It is the source for every template in this guide.

---

## 4. Phase 0 — Measure Before You Change

### Step 0 — Baseline Audit

**Goal.** Establish a before-picture so you can prove the harness improved something. Without a baseline, every change feels like progress.

**Do this.**

1. Run the existing install, build, type-check, lint, and test commands. Record exact outcomes and versions.
2. Run the fresh-session test from Section 1.3. Record the score for each of the five questions.
3. Score the five subsystems: `0` absent, `1` ad hoc, `2` documented, `3` enforced and automated.
4. List the top five failure modes from recent agent sessions. Examples: forgot the startup command, modified unrelated files, declared done without running tests, reimplemented a completed feature, could not diagnose a runtime failure.
5. Map each failure to a missing subsystem. This becomes your work plan.
6. Write the result to `docs/HARNESS-AUDIT.md` (or a clearly marked section in `claude-progress.md`).

**Minimum artifact.**

```markdown
# Harness Audit — <date> — commit <sha>

## Baseline commands
- Install: `<INSTALL>` -> PASS/FAIL
- Build: `<BUILD>` -> PASS/FAIL
- Type-check: `<TYPECHECK>` -> PASS/FAIL
- Tests: `<TEST>` -> PASS/FAIL (x/y)

## Fresh-session test
| Question | Score | Missing information |
|---|---:|---|
| What is this? | 1 | README is outdated |
| How organized? | 0 | No architecture doc |
| How to run? | 1 | Commands scattered in chat |
| How to verify? | 0 | No test command documented |
| Where are we? | 0 | No progress file |

## Subsystem scores
Instructions: 1 · Environment: 1 · Tools: 2 · State: 0 · Feedback: 0

## Top failure modes -> missing subsystem
1. Agent guessed the package manager -> Environment
2. Agent marked features done without running tests -> Feedback
3. New session repeated completed work -> State
```

Keep this audit as a dated snapshot. After Step 14, archive or delete it so it does not become another stale doc.

**Exit gate.**

- [ ] Baseline commands and results recorded.
- [ ] Five-question score recorded.
- [ ] Top five failure modes mapped to subsystems.
- [ ] No production code changed yet.

**Pitfall.** Do not audit by reading README only. Run the commands. A README that says "npm test" while the actual command is "pnpm test" is worse than no README, because it sends the agent in the wrong direction.

**Reference.** The starter in `projects/project-06/starter` is the control group; the solution is the treatment. Compare the harness surface, not the product code.

---

## 5. Phase 1 — Make the Repository Self-Describing

### Step 1 — Create `AGENTS.md` as a Router, Not an Encyclopedia

**Goal.** Give every new session one short file that answers: what is this, how do I start, what are the non-negotiable rules, how do I know I am done, and where do I read more?

Think of `AGENTS.md` as the **front desk of a building**, not the entire floor plan. It tells you where to go; the detailed docs live in the rooms.

**Do this.**

1. Keep it to roughly 50-200 lines. The reference is 104 lines.
2. Put these sections in this order:
   - Project context (one or two sentences).
   - Startup rules (ordered, with exact commands).
   - Commands (install, dev, build, check, test, full verification).
   - Hard constraints (no more than 15; each must be enforceable).
   - Docs map (one line per doc, with "read when"). Only link docs that already exist; add links as you create them.
   - Work rules (WIP=1, no drive-by refactors, no changing verification rules).
   - Definition of Done (behavior + verification + evidence + clean state).
   - End-of-session routine.
3. Put critical constraints at the top or bottom, never buried in the middle. Long files suffer from the "lost in the middle" effect.
4. For every rule, know why it exists, when it applies, and when it can be removed. A rule with no expiry condition becomes permanent debt.
5. If you use `CLAUDE.md` too, make one file canonical and the other a thin pointer. Duplicating the same rules in two files guarantees drift.
6. Apply the same Definition of Done to refactors, infrastructure changes, and documentation changes. They are work too: verify them and leave evidence.
7. Measure the entry file's cost. `wc -w` is a rough proxy; a token counter is better. If `AGENTS.md` plus the docs loaded every session exceeds roughly 10% of the model's context budget, move detail into on-demand docs.

**Minimum artifact.**

```markdown
# AGENTS.md — <Project>

## What This Is
<One or two sentences: product, users, stack, runtime.>

## Startup (in order)
1. Read `claude-progress.md` for the current verified state and next action.
2. Read `feature_list.json` and pick the highest-priority unfinished item.
3. Read the relevant doc for the area you will touch.
4. Run `bash init.sh`. If the baseline fails, fix it before new work.
5. Run the focused smoke check for the selected feature.

## Commands
- Install: `<INSTALL>`
- Dev: `<DEV>`
- Build: `<BUILD>`
- Type-check / lint: `<TYPECHECK>` / `<LINT>`
- Tests: `<TEST>`
- End-to-end: `<E2E>`
- Full verification: `<FULL_CHECK>`

## Hard Constraints
- <e.g. Renderer code must not import Node.js core modules.>
- <e.g. All IPC channel names live in `src/shared/types.ts`.>
- <e.g. No secrets in logs, fixtures, or committed files.>
- <e.g. All data access goes through the service layer.>
- <Maximum 15. Each rule must be checkable.>

## Work Rules
- One feature in progress at a time (WIP=1).
- Do not mark a feature `passing` without running its verification and recording evidence.
- Stay inside the selected feature's scope; no drive-by refactors.
- Do not change verification rules during implementation.
- Prefer repository artifacts over chat summaries.

## Docs Map
- `docs/ARCHITECTURE.md` — read before changing module boundaries or data flow.
- `docs/PRODUCT.md` — read before changing user-visible behavior.
- `docs/RELIABILITY.md` — read before changing logging, reset, or benchmarks.
- `<module>/README.md` — read before changing that module.

## Definition Of Done
A feature is done only when:
1. The target behavior is implemented.
2. Required verification ran and passed: static checks, tests, and end-to-end when cross-component.
3. Evidence is recorded in `feature_list.json`.
4. Architecture and product docs are updated if boundaries or behavior changed.
5. `clean-state-checklist.md` passes.
6. The work is committed with a descriptive message.

## End Of Session
1. Run full verification.
2. Update `feature_list.json`, `claude-progress.md`, and `session-handoff.md`.
3. Remove debug artifacts and temporary files.
4. Commit; leave `bash init.sh` green.
```

**Exit gate.**

- [ ] A new agent session can answer all five questions from `AGENTS.md` plus linked docs.
- [ ] File is under 200 lines.
- [ ] Every command in the file has been run successfully.
- [ ] Hard constraints are specific and checkable.
- [ ] Definition of Done includes evidence and clean state.
- [ ] There is one canonical source of rules; no contradictory duplicates.

**Pitfall.** Adding a new rule after every mistake. That produces a 600-line file where the important rule is buried at line 300. Put detailed rules in topic docs and link them.

**Reference.** `projects/project-06/solution/AGENTS.md` has 104 lines, seven startup steps, explicit layer boundaries, and a seven-item Definition of Done.

### Step 2 — Build a Progressive-Disclosure Docs Map

**Goal.** Keep the entry file short while making detailed knowledge available on demand. Put knowledge next to the code it describes.

Think of it as **packing cubes for a suitcase**: essentials in the front pocket, detailed material in labeled compartments. You do not empty the whole suitcase to find a charger.

**Do this.**

1. Create `docs/ARCHITECTURE.md`:
   - System overview and purpose.
   - Layer or module diagram.
   - Dependency direction and forbidden edges.
   - Main data flows (three to seven critical paths).
   - Storage layout and external boundaries.
   - "Where to add a new feature" section.
2. Create `docs/PRODUCT.md`:
   - User-visible features.
   - Main flows and UI layout.
   - Acceptance behavior and non-goals.
3. Create `docs/RELIABILITY.md` later, when you add logging, reset, and benchmarks. It should define the operational contract.
4. Add short module docs for non-obvious areas: responsibility, public API, invariants, examples, tests. Twenty to fifty lines is enough. A `README.md` or `AGENTS.md` inside the module directory works.
5. Link every doc from `AGENTS.md` with a one-line "read when" condition.
6. Keep each doc to roughly 50-200 lines. If it grows, split by topic.
7. Update docs in the same commit as the code change. Stale documentation is worse than no documentation because it actively misleads.
8. If `AGENTS.md` already links `docs/RELIABILITY.md`, create the file now or remove the link until Step 10. Broken links are worse than no links.

**Exit gate.**

- [ ] A new agent can trace one feature end-to-end using only the docs and code.
- [ ] Critical module boundaries and data flows are documented.
- [ ] Every doc is linked from the entry file.
- [ ] No critical knowledge exists only in chat or in someone's head.
- [ ] Docs match the current code; uncertain areas are marked as assumptions.

**Pitfall.** Writing a 500-page design document that nobody maintains. A 50-line doc next to the module is more useful than a perfect document in another system.

**Reference.** `docs/ARCHITECTURE.md` (208 lines), `docs/PRODUCT.md` (105 lines), and `docs/RELIABILITY.md` (178 lines) in the solution.

### Step 3 — Create `init.sh` as the Repository Power-On Self-Test

**Goal.** Make one command prove that the repository is installable, buildable, and verifiable. Remove the "it works on my machine" and "which command do I run?" ambiguity.

Think of `init.sh` as a **pre-flight checklist for a plane**: it does not fly the plane, but it catches a missing wheel before takeoff.

**Do this.**

1. Start with `set -euo pipefail`.
2. Resolve the repository root and `cd` into it.
3. Run, in order:
   - Install dependencies from a lockfile.
   - Type-check and lint.
   - Build.
   - Run tests.
   - Verify required harness files exist.
   - Verify sample data or fixtures exist.
   - Print the start command.
4. Make every failure actionable: say what failed, why it matters, and how to fix it.
5. Make the script idempotent: running it twice must be safe.
6. Do not prompt for input, do not open a browser, and do not start a long-running foreground server by default. Offer an opt-in flag such as `RUN_START_COMMAND=1`.
7. Put a timeout around network operations. Pin tool versions where possible.
8. Use the exact same command in CI.

**Minimum artifact.**

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$ROOT_DIR"

# The commands below are examples. Replace them with the values from your Project Profile.
INSTALL_CMD=(npm ci)
STATIC_CMD=(npm run check)
BUILD_CMD=(npm run build)
TEST_CMD=(npm test)
START_CMD=(npm run dev)

echo "==> Working directory: $PWD"
echo "==> Installing dependencies"
"${INSTALL_CMD[@]}"

echo "==> Static checks"
"${STATIC_CMD[@]}"

echo "==> Build"
"${BUILD_CMD[@]}"

echo "==> Tests"
"${TEST_CMD[@]}"

echo "==> Verifying harness files"
for file in AGENTS.md feature_list.json claude-progress.md clean-state-checklist.md; do
  if [ ! -f "$file" ]; then
    echo "MISSING: $file"
    echo "FIX: restore the file or update this check if the harness has moved."
    exit 1
  fi
done

echo "==> Validating feature list"
if [ -f scripts/validate-features.mjs ]; then
  node scripts/validate-features.mjs
fi

echo "==> Architecture checks"
if [ -f scripts/check-architecture.sh ]; then
  bash scripts/check-architecture.sh
fi

echo "==> Startup command"
printf '    %q' "${START_CMD[@]}"
printf '\n'

if [ "${RUN_START_COMMAND:-0}" = "1" ]; then
  exec "${START_CMD[@]}"
fi
```

**Exit gate.**

- [ ] A clean clone passes `bash init.sh`.
- [ ] An intentionally broken check produces a clear, actionable error and a non-zero exit code.
- [ ] The script completes in under five minutes, or the slow parts are documented.
- [ ] CI runs the same command.
- [ ] Running it twice is safe.

**Pitfall.** Using `npm install` with unpinned versions, swallowing failures with `|| true`, or requiring manual database setup. A green `init.sh` that lies is worse than a red one that tells the truth.

**Reference.** `projects/project-06/solution/init.sh` has five stages: install, type-check, build, harness-file verification, sample-data verification. It exits non-zero when anything is missing. It uses `npm install`; prefer a frozen-lockfile install such as `npm ci` in your repository.

### Environment and Tool Access (Companion to Step 3)

`init.sh` proves the repository works on your machine. It does not prove that another machine can reproduce it, or that the agent is allowed to run the commands it needs. Two more checks close that gap.

**Pin the environment.**

- Commit the lockfile and use a frozen install in CI: `npm ci`, `pnpm install --frozen-lockfile`, `uv sync --frozen`, `go mod download` with a committed `go.sum`, or `cargo fetch` with `Cargo.lock`.
- Pin the runtime: `.nvmrc`, `.python-version`, `rust-toolchain.toml`, the `go` directive in `go.mod`, or `.tool-versions`.
- Optional but useful: a devcontainer or Dockerfile that reproduces the runtime and system dependencies.
- Document external services (database, cache, API) and how `init.sh` handles them: local mock, container, or a clear skip with instructions.

**Constrain tool access.**

- In `AGENTS.md`, list allowed and forbidden commands. Example: allowed to run tests, build, and lint; forbidden to deploy, rotate credentials, or delete data outside the application's data directory.
- Use least privilege for tokens and secrets. Put secrets in environment variables or a secret manager, never in the repository.
- If a destructive command is necessary, require an explicit flag and a confirmation step.
- Make network access explicit. If a service is unreachable, fail with a useful message instead of hanging.

**Exit gate.**

- [ ] A fresh clone on a clean machine passes `init.sh`.
- [ ] The runtime version and lockfile are committed.
- [ ] Tool permissions and forbidden actions are documented.
- [ ] External services are mocked, containerized, or clearly documented.
- [ ] Secrets are not in the repository or logs.

---

## 6. Phase 2 — Make State Durable and Scope Explicit

### Step 4 — Create `feature_list.json` as the Scope Contract

**Goal.** Turn "what needs to be done" and "what counts as done" into a machine-readable file that every session reads and updates. Enforce one active feature at a time.

The feature list is not a memo. It is the **queue and the contract**: the scheduler reads it to pick work, the verifier reads it to judge completion, and the handoff reporter reads it to summarize state.

**Do this.**

1. Choose one schema and keep it consistent. A good schema includes:
   - `id`, `priority`, `area`, `title`.
   - `user_visible_behavior`: what a user can observe when it works.
   - `status`: `not_started`, `in_progress`, `blocked`, or `passing`.
   - `verification`: commands or step-by-step checks.
   - `evidence`: proof that verification ran (command output, test report, log excerpt, screenshot path).
   - `notes`: constraints, dependencies, links.
2. Add top-level rules and a status legend.
3. Enforce the state machine:
   - At most one `in_progress` at a time. Zero is fine between sessions.
   - `blocked` requires a documented reason.
   - `passing` requires evidence.
   - The agent may propose a state change, but the verification result decides.
4. Calibrate granularity to "completable in one session." "Implement the shopping cart" is too broad. "User can add an item to the cart and see the count update" is right-sized.
5. Add a validator script (JSON Schema, `jq`, Node, Python, or Go) that checks unique IDs, valid statuses, at most one `in_progress`, and evidence on every `passing` item. Run it in `init.sh` and CI. A minimal Node validator is in Appendix B.

**Minimum artifact.**

```json
{
  "project": "<project-name>",
  "last_updated": "YYYY-MM-DD",
  "rules": {
    "single_active_feature": true,
    "passing_requires_evidence": true,
    "do_not_skip_verification": true
  },
  "status_legend": {
    "not_started": "Work has not begun.",
    "in_progress": "The current active task.",
    "blocked": "Cannot continue until a documented blocker is resolved.",
    "passing": "Required verification passed and evidence is recorded."
  },
  "features": [
    {
      "id": "auth-001",
      "priority": 1,
      "area": "auth",
      "title": "User can log in with email and password",
      "user_visible_behavior": "A valid user reaches the dashboard after submitting credentials.",
      "status": "not_started",
      "verification": [
        "Run `<TEST>`.",
        "Start the app with `<DEV>`.",
        "Log in with a seeded valid account.",
        "Confirm the dashboard is visible and the session persists after refresh."
      ],
      "evidence": [],
      "notes": "Invalid-password path is covered by auth-002."
    }
  ]
}
```

**What good evidence looks like.** Evidence should let a skeptical reviewer reproduce the result without asking the generator. Examples:

- `npm test -- auth.test.ts` -> `12 passed`.
- A log excerpt: `{"event":"answer_generated","confidence":0.85,"citationCount":2}`.
- A path to a screenshot or test report, plus the exact command that produced it.
- A commit SHA and the feature ID it implements.

"I ran it and it worked" is not evidence. Neither is a screenshot with no command or context.

Store large evidence outside the JSON: keep a short path or identifier in `evidence`, and put the actual report or screenshot under `evidence/<feature-id>/` or in CI artifacts. Add a retention rule so evidence does not grow forever.

**Exit gate.**

- [ ] Every feature has behavior, verification, and state.
- [ ] At most one feature is `in_progress` (zero is allowed between sessions).
- [ ] Every `passing` feature has evidence.
- [ ] The validator passes in `init.sh`.
- [ ] A fresh session can pick the next feature without asking.

**Pitfall.** Treating the feature list as a wish list. A feature without a verification method is not a contract; it is a hope. Also avoid multiple `in_progress` items — that is how agents overreach and under-finish.

**Reference.** `projects/project-06/solution/feature_list.json` tracks 15 features with status, evidence, and `testedAt`. It uses `status: "pass"` (not `passing`) and does not include a per-feature `verification` field; the verification steps live in the evidence text. That is a weaker schema than the template above. If you copy it, add `verification`, standardize on `passing` or `pass`, and validate the result.

### Step 5 — Create `claude-progress.md` and `session-handoff.md` for Continuity

**Goal.** Make the next session's rebuild cost small. A fresh agent should reconstruct the state in under five minutes, including decisions that are not obvious from the code.

Think of the progress file as the **shift log** at a hospital: what happened, what is still open, what the next shift must watch. The handoff is the **note to the next shift** that highlights the critical items.

**Do this.**

1. In `claude-progress.md`, keep:
   - Current verified state: repository root, standard startup path, standard verification path, highest-priority unfinished feature, current blocker.
   - A session record with goal, completed work, verification run, evidence, decisions, files modified, commits, known risks, and next best action.
2. In `session-handoff.md`, keep:
   - What is verified now.
   - What changed this session, including decisions and files modified.
   - What is broken or unverified.
   - The next best step, why it is next, what counts as passing, and what must not change.
   - The startup, verification, and focused debug commands.
3. Record decisions with date, reason, alternatives considered, and constraints. The code shows "what"; the decision log preserves "why."
4. Use git commits as checkpoints. Commit after each verified atomic unit, with a message that explains what and why.
5. Put the clock-in and clock-out routine in `AGENTS.md`.
6. Update state after verification, never before. Do not claim a check that did not run.
7. For short sessions, the handoff can be a short note; the progress file is still required. The handoff is most valuable when the session is long or the state is risky.

**Exit gate.**

- [ ] A new session can read only the entry file, feature list, progress, and handoff and explain the current state.
- [ ] Every unfinished item has a concrete next action.
- [ ] Important decisions are recorded.
- [ ] The last commit is clean and the standard startup path works.
- [ ] The handoff does not contain unsupported "done" claims.

**Pitfall.** Turning the progress file into a diary that repeats the feature list. Keep it short and link to commits and logs. Also avoid stale handoffs — an outdated handoff is worse than none because it confidently points the next session in the wrong direction.

**Reference.** `claude-progress.md` (40 lines) and `session-handoff.md` (81 lines) in the solution. The handoff records decisions such as using a singleton logger with `forService()`, storing feedback in a separate file, and using a destructive reset for simplicity. Templates are in Appendix D and Appendix E.

### Step 6 — Create a Clean-State Checklist and Reset Mechanism

**Goal.** End every session in a state where the next session can start immediately. Control entropy before it compounds.

Think of clean state as **leaving the kitchen usable for the next cook**: dishes washed, surfaces clear, ingredients labeled. "The food was good" is not enough if the kitchen is a disaster.

**Do this.**

1. Create `clean-state-checklist.md` with objective items in these categories:
   - Build and static checks.
   - Architecture boundaries.
   - Runtime or smoke behavior.
   - Logging and observability.
   - Data integrity.
   - Performance.
   - Repository hygiene (git status, no secrets, no build output).
   - Scripts and state files.
   - Docs, feature list, progress, and handoff.
2. Make as many items automatable as possible. Manual items must require evidence, not a feeling.
3. Provide a reset command or in-app action that:
   - Removes only application data, never broad filesystem paths.
   - Is idempotent.
   - Recreates the directory structure.
   - Requires confirmation for destructive action.
   - Verifies the empty state afterward.
4. Add "clean state passes" to the Definition of Done. A session with failing checks is not done, even if the feature works.
5. Schedule a deeper cleanup weekly: temporary files, debug code, stale docs, flaky tests, TODOs.

**Exit gate.**

- [ ] The checklist can be completed in under ten minutes.
- [ ] Every item is objective or has a clear evidence requirement.
- [ ] Reset can run twice with the same result.
- [ ] After reset, the standard build, test, and startup paths still work.
- [ ] No debug artifacts or secrets remain.

**Pitfall.** A 200-item manual checklist that nobody runs. Also dangerous: a reset script that deletes a broad directory or follows user input directly. Scope deletion to known application data paths, and fail closed if the path is unexpected.

**Reference.** `clean-state-checklist.md` has 41 checkboxes across eight categories. The app exposes a reset IPC channel; `PersistenceService.resetAll()` removes and recreates the data directory, and the renderer clears its state after the reset. A smaller template is in Appendix F.

---

## 7. Phase 3 — Make "Done" Objective

### Step 7 — Build a Verification Ladder and Enforce Architecture Boundaries

**Goal.** Replace "the code looks done" with executable evidence. Catch defects that unit tests systematically miss: interface mismatches, state propagation errors, resource leaks, environment differences.

Think of the verification ladder as a **three-stage rocket**: static checks, behavior checks, system checks. Skipping a stage means the mission is not complete.

**Do this.**

1. Define three levels:
   - **L1 — Static:** type-check, lint, build. Always required. Cheapest and least informative, but non-negotiable.
   - **L2 — Behavior:** unit and integration tests, plus a startup smoke check. Always required.
   - **L3 — System:** end-to-end checks for critical user flows, especially cross-component changes. Required before declaring a feature `passing`.
2. Create one full-check command (`make check`, `scripts/check.sh`, or a package script) that runs L1 and L2 at minimum, and L3 when relevant.
3. Ensure at least one example test passes before feature work begins. A test framework that has never run is not configured.

**Legacy repository shortcut.** If the repository has no test framework, do not let that block the ladder. Add one smoke test that boots the app or CLI and asserts a known output or exit code. Then add characterization tests for the module you are about to change. That is enough to start; expand coverage as you touch code.

4. Choose three to seven critical flows and automate them. Run them from a clean state. Include failure paths. Save artifacts: command output, screenshots, logs, test reports.
5. Create `scripts/check-architecture.sh` for boundary rules:
   - Forbidden imports between layers.
   - Business logic in the UI layer.
   - Direct filesystem access where it is not allowed.
   - Shared types imported from the wrong place.
6. Make error messages agent-oriented: **what** failed, **why** it matters, **how** to fix it. For example:
   ```
   ERROR: src/renderer/App.tsx imports 'fs'.
   WHY: The renderer process has no Node.js access; this breaks the security boundary.
   FIX: Move file operations to src/services/file-service.ts and expose them through the preload bridge.
   ```
7. Run architecture checks in `init.sh` and CI. If a rule cannot be checked, it is a suggestion, not a constraint.

**Exit gate.**

- [ ] A clean clone passes the full check.
- [ ] A failing test blocks a feature from moving to `passing`.
- [ ] A seeded architecture violation fails the check with an actionable message.
- [ ] Critical flows have end-to-end coverage or a documented manual equivalent with evidence.
- [ ] Tests do not rely on a dirty data state.

**Pitfall.** Unit tests only. They are designed to isolate components, which is exactly why they miss boundary defects. Also avoid "fixing" a failing test by weakening the assertion, and avoid changing verification rules during implementation.

**Reference.** `scripts/check-architecture.sh` checks three rules: no Node core imports in the renderer, no Electron IPC in services, and no React imports in services or main. The solution also has a runtime checklist and benchmark-based verification. It has a `test` script but no checked-in test files, and it does not automate unit or full UI end-to-end tests; treat that as a gap to close, not a pattern to copy. If your product has critical UI flows, add them. A boundary-check template is in Appendix J.

### Step 8 — Add an Evaluator Rubric and Separate the Roles

**Goal.** Prevent the agent from grading its own work generously. Make acceptance evidence-based and reproducible.

Think of it as **code review by a different person**: the person who wrote the code is the worst judge of whether it is finished.

**Do this.**

1. Define three roles:
   - **Planner:** defines scope, acceptance criteria, and exclusions.
   - **Generator:** implements and runs focused checks.
   - **Evaluator:** runs the full verification, scores against the rubric, and reports defects. The evaluator does not edit code.
2. The same model can play all three roles, but use separate sessions or contexts. The evaluator should not see the generator's excuses; it should see the repository, the feature list, the rubric, and the evidence.
3. Define rubric dimensions and 1-5 anchors. Example dimensions:
   - Build and static checks.
   - Behavior correctness.
   - Architecture compliance.
   - Test coverage and stability.
   - Observability.
   - Documentation and state accuracy.
   - Performance and resource use.
   - Security and data safety.
4. Set a threshold, for example: overall >= 4, no dimension below 3, and mandatory dimensions (build, behavior, security) must be 4 or 5.
5. For larger work, write a short sprint contract before implementation:
   - Scope.
   - Verification standards.
   - Exclusions.
   - What counts as done.
6. Require evidence for every score: command output, test name, log excerpt, file and line, screenshot path. "Looks good" is not evidence.
7. The generator fixes the defects; the evaluator re-runs and re-scores. Only then can the feature move to `passing`.

**Exit gate.**

- [ ] The evaluator can score without asking the generator questions.
- [ ] Every score has evidence.
- [ ] Failing dimensions produce a concrete fix list.
- [ ] No feature passes below the threshold.
- [ ] The evaluator's output is recorded with the feature or session.

**Pitfall.** Same-context self-review. The model is systematically overconfident about its own work. Also avoid a rubric with vague words like "clean" or "good" that different evaluators interpret differently.

**Reference.** `projects/project-06/solution/evaluator-rubric.md` is a completed assessment with 15 criteria, notes, and evidence. It is not an enforced independent evaluation: nothing prevents the same author or session from producing and scoring the work. The checked-in Project 05 variants report single-role 1.6/5, generator + evaluator 3.3/5, and planner + generator + evaluator 4.9/5. Treat those as reported comparisons, not controlled measurements. The pattern matters more than the numbers: independent evaluation catches more defects. A rubric template is in Appendix G; a compact sprint contract template is in Appendix K.

### Step 9 — Maintain a Quality Document

**Goal.** Track module health over time, direct work to the weakest area, and detect decay before it becomes a crisis.

Think of it as a **health dashboard for the codebase**, not a one-time report card.

**Do this.**

1. For each module or major area, record:
   - Grade: A, B, C, or D.
   - Verification status: passing, partial, failing.
   - Agent readability: easy, difficult, or unclear.
   - Test stability: stable, flaky, or untested.
   - Architecture compliance: compliant or violations.
   - Conventions: followed or deviations.
   - Known debt and the next action.
   - Last updated and evidence links.
2. Update it at milestones or weekly, not after every commit.
3. Rule: fix the lowest-scoring module first. Do not add features to a D-grade module without an explicit exception and a plan.
4. Use it during planning: the next feature should either improve the weakest module or be the highest business priority, and the trade-off should be recorded.

**Exit gate.**

- [ ] Every module or major area is represented.
- [ ] Grades are backed by evidence.
- [ ] The lowest grade has a concrete next action.
- [ ] The trend is visible across updates.
- [ ] The quality document agrees with the feature list and checklist.

**Pitfall.** A one-time document with inflated grades. If every module is an A while tests are flaky and docs are stale, the document is lying. Also avoid excessive granularity: module-level is usually enough.

**Reference.** `projects/project-06/solution/quality-document.md` grades 15 dimensions and records evidence from the build, runtime, observability, performance, and verification checklist. It is a snapshot rather than a trend: there is one overall grade and no history across updates. Use it as the template, then keep your own dated history. A compact template is in Appendix H.

---

## 8. Phase 4 — Make Runtime Observable

### Step 10 — Add Structured Logging

**Goal.** Make runtime behavior visible and machine-parseable. When something fails, the agent should diagnose from evidence instead of guessing.

Think of structured logs as the **flight recorder** of the application. A plain text log is a diary; a structured log is a data source you can query.

**Do this.**

1. Create one logger module. Ban raw `console.log` in production code (allow it only in throwaway scripts, and mark those scripts clearly).
2. Use a single-line JSON schema:
   - `timestamp`: ISO 8601.
   - `level`: DEBUG, INFO, WARN, ERROR.
   - `service`: module or service name.
   - `event` or `message`: a stable, searchable name.
   - `data`: structured payload with IDs, counts, durations, and safe context.
   - `correlationId` or `traceId`: links events in one flow.
   - `durationMs`: for operations where latency matters.
   - `error`: name, message, and stack for failures.
3. Use levels consistently:
   - **DEBUG:** routine data access and detailed state.
   - **INFO:** significant business events (import, index, answer, reset).
   - **WARN:** missing but non-critical data, recoverable fallback.
   - **ERROR:** failures that need attention.
4. Provide child loggers per service, for example `logger.forService('document-service')`, so the service tag is automatic.
5. Log at boundaries:
   - Application startup and shutdown.
   - API, IPC, or CLI entry points.
   - Service operations: start, success, failure.
   - External calls and I/O.
   - State changes and destructive actions.
6. Include useful context: document ID, user ID (if allowed), filename, size, chunk count, confidence, citation count, duration. Never log secrets, tokens, passwords, or personal data.
7. Make the level configurable with an environment variable. Default to INFO in production and DEBUG in development. Send logs to stdout for containers; use rotation if writing files.
8. Add a test that parses every emitted line as JSON and asserts required fields. Add a check that production code contains no raw `console.log`.
9. Document the event catalog in `docs/RELIABILITY.md`: event name, level, fields, and when it fires.

**Example entry.**

```json
{
  "timestamp": "2026-03-30T12:00:00.000Z",
  "level": "INFO",
  "service": "document-service",
  "event": "document_imported",
  "data": {
    "documentId": "abc-123",
    "filename": "design-notes.md",
    "sizeBytes": 2048,
    "durationMs": 42
  },
  "correlationId": "req-7f3a"
}
```

**Exit gate.**

- [ ] All application log lines are valid JSON.
- [ ] Every critical flow emits at least a start and a success or failure event.
- [ ] Logs include the identifiers needed to debug that flow.
- [ ] No secrets or personal data are logged.
- [ ] The level is configurable.
- [ ] The event catalog is documented.

**Pitfall.** Logging only errors. That gives you a record of failures with no context about what succeeded before them. Also avoid logging everything at DEBUG in production; noise hides signal and costs money.

**Reference.** `projects/project-06/solution/src/services/logger.ts` implements levels, a child logger, and JSON output. All five services emit structured logs; the document, indexing, and QA services use `logger.forService()` child loggers, while persistence and the IPC handlers use the base logger with an explicit service tag. `docs/RELIABILITY.md` documents the levels and logging points. A language-agnostic contract is in Appendix I.

### Step 11 — Add a Benchmark and a Cleanup Scanner

**Goal.** Turn performance and hygiene into repeatable signals. Detect regressions and stale artifacts before they surprise you.

Think of the benchmark as a **scale** and the cleanup scanner as a **Roomba that only reports dust** — it tells you where the mess is before you decide whether to clean it.

**Do this.**

1. Define three to five critical benchmark tasks. For the reference app: import, indexing, query, and data integrity.
2. Use fixed sample data and a clean state. Measure wall time, throughput, and pass/fail against a threshold.
3. Output structured results (JSON or Markdown) that can be stored and compared over time.
4. Store the baseline in `claude-progress.md`, `quality-document.md`, or a `benchmarks/` directory. Compare against it on each run.
5. Exit non-zero when a threshold is breached, so CI can catch regressions.
6. Build a cleanup scanner that checks:
   - Orphaned files with no metadata.
   - Dangling metadata with no files.
   - Missing required files.
   - Inconsistent status flags.
   - Stale references to deleted entities.
   - Temporary files, debug code, TODO/FIXME markers, and stale docs.

   Exclude `node_modules`, `dist`, generated code, and vendor directories from the scan.

7. Make the scanner read-only by default. If it supports `--fix`, require an explicit flag, scope it to known paths, make it idempotent, and print exactly what it will delete.
8. Run the benchmark and scanner in `init.sh` (fast checks) or CI/weekly (slow checks).

**Exit gate.**

- [ ] One command runs the benchmark.
- [ ] A baseline is recorded.
- [ ] The scanner finds a deliberately seeded issue.
- [ ] A clean repository produces no false positives.
- [ ] Both scripts are safe to run repeatedly.

**Pitfall.** Benchmarking a dirty state, or comparing numbers collected under different conditions. Do not use a one-second clock (`date +%s`) for sub-second thresholds; use a monotonic millisecond timer. The reference's `python3`/`date` fallback is a teaching shortcut, not a production timer. Also avoid a scanner that deletes data by default; a hygiene tool that can destroy work will not be trusted or run. Finally, test the scanner's exit code and issue count, not just its printed text: shell pipelines run in subshells, and a counter incremented inside `while read` may silently reset.

**Reference.** `scripts/benchmark.sh` runs four tasks and reports pass/fail. `scripts/cleanup-scanner.sh` checks orphaned content, dangling chunks, missing content, inconsistent metadata, and stale Q&A references. Both are shell scripts with no application dependency. The scanner uses Python 3; port it if your environment does not provide Python.

---

## 9. Phase 5 — Refactor, Enforce, and Operate

### Step 12 — Refactor Modules Behind a Safety Net

**Goal.** Reduce the amount of context an agent must load to make a change. Put related code and docs together, make dependencies explicit, and prevent boundary violations.

Think of a warehouse: if nails, screws, and anchors are mixed in one bin, every task starts with sorting. Module refactoring labels the bins and puts them where the picker expects them.

**Why this step comes late.** A large refactor without tests, logs, and benchmarks is unverifiable. You cannot prove that behavior did not change. By now you have the safety net: the full check, the architecture script, logs, and performance baselines. Refactor behind that net.

**Do this.**

1. **Inventory the current state.**
   - List modules and their one-sentence responsibilities.
   - List public entry points and internal imports.
   - Use a dependency analyzer appropriate to the stack: `madge` or `dependency-cruiser` for JS/TS, `import-linter` for Python, `cargo modules` or `cargo-deny` for Rust, `go list -deps` for Go. Do not guess the graph from grep alone.
   - Find cycles, hidden globals, duplicated logic, and oversized files.
   - Identify where the UI reaches into infrastructure, or where business logic lives in the wrong layer.
   - Exclude generated and vendor directories from the analysis; document how they are generated.
2. **Define the target.**
   - Group by responsibility, not by file type.
   - Choose a dependency direction (for example: UI -> application -> domain -> infrastructure; or renderer -> preload -> main -> services -> shared).
   - Define a public API per module (an `index.ts` or equivalent).
   - Put shared types in one place.
   - Forbid cycles and upward dependencies.
3. **Document before moving.**
   - Update `docs/ARCHITECTURE.md` with the target module map and dependency rules.
   - Add a short `README.md` or `AGENTS.md` in each non-obvious module: responsibility, public API, invariants, examples, tests.
   - Add a "where to add a feature" section.
4. **Refactor in small, behavior-preserving steps.**
   - One seam at a time: extract a module, move a file, rename a concept, introduce an interface.
   - Do not mix a refactor with a feature change.
   - Run the full check after each step.
   - Use characterization tests if the module has no tests.
   - Update imports, docs, and architecture checks in the same commit.
   - For legacy code, use the strangler pattern: wrap the old module, migrate callers incrementally, delete the old path when no callers remain.
5. **Enforce the boundaries.**
   - Extend `scripts/check-architecture.sh` or add a linter rule.
   - Error messages must say what, why, and how to fix.
   - Run the check in `init.sh` and CI.
6. **Reduce context load.**
   - Split giant files by responsibility.
   - Colocate tests and docs with the module.
   - Remove dead code and unused exports.
   - Use consistent naming and explicit exports.
7. **Measure the result.**
   - File size, import cycles, fan-in/fan-out, change coupling.
   - Time for a fresh agent to locate the right module.
   - Test coverage per module.
   - Number of boundary violations.

**Concrete example from the reference.**

- Layers: `src/main/` (window and IPC), `src/preload/` (typed bridge), `src/renderer/` (React UI), `src/services/` (business logic), `src/shared/` (types and channel constants).
- Rules: the renderer never imports Node.js modules; services never import Electron or React; all IPC channel names live in `src/shared/types.ts`; services receive `PersistenceService` through the constructor.
- Enforcement: `scripts/check-architecture.sh` checks three forbidden patterns and exits non-zero on violations.

**Exit gate.**

- [ ] The dependency graph is acyclic and follows the documented direction.
- [ ] Each module has a one-sentence responsibility and a public API.
- [ ] Module docs exist where the code is non-obvious.
- [ ] The architecture check passes.
- [ ] The full test suite, e2e checks, and benchmark remain green.
- [ ] A fresh agent can find where to add a small feature without scanning the whole repository.

**Pitfall.** Big-bang refactoring, or "while I am here" refactoring during a feature. Both destroy the ability to attribute failures. Also avoid ceremonial architecture that adds layers without reducing coupling, and never refactor generated or vendor code by hand.

**Reference.** The reference app's boundary checks and `docs/ARCHITECTURE.md` are a good template for the rules; the exact layers are Electron-specific and should be adapted.

### Step 13 — Automate Enforcement and Run the Operating Loop

**Goal.** Make the harness self-enforcing. A document that nobody runs is a suggestion; CI and a session protocol are constraints.

Think of CI as the **automatic gatekeeper** at the door: it checks the ticket every time, without getting tired or optimistic.

**Do this.**

1. Build a CI pipeline that runs:
   - Install from the lockfile.
   - Type-check and lint.
   - Build.
   - Unit and integration tests.
   - End-to-end checks for critical flows.
   - Architecture checks.
   - Log-format checks.
   - Feature-list and harness-file validation.
   - A fast benchmark or a nightly benchmark.
2. Cache dependencies, set timeouts, upload artifacts (test reports, logs, screenshots), and fail fast.
3. Protect the main branch: no merge on red. A skipped test is a failure, not a pass.
4. Encode the session protocol in `AGENTS.md`:
   - **Clock in:** read progress, feature list, and recent commits; run `init.sh`; run the baseline smoke check.
   - **Work:** WIP=1; read the relevant docs; write or update a test first where practical; implement; run focused checks; record evidence.
   - **Clock out:** run full verification; update feature list, progress, and handoff; remove debug artifacts; commit; verify clean state.
5. Schedule maintenance:
   - **Weekly:** cleanup scanner, flaky-test review, stale-doc review, TODO triage, dependency check.
   - **Monthly:** harness audit against the five subsystems, fresh-session test, quality-document update, benchmark trend review.
   - **Quarterly:** prune rules and files that no longer pay for themselves. Harness components can become obsolete as models improve.
6. Track a small set of metrics:
   - Time from clone to first green check.
   - Time for a fresh session to reconstruct state.
   - Verified completion rate: verified features / activated features.
   - Escaped defects: bugs found after a feature was marked `passing`.
   - Rework rate: features reopened after `passing`.
   - Log parse rate: valid JSON lines / total lines.
   - Benchmark trend.
   - Count of stale docs and open TODOs.
7. Run a controlled ablation occasionally: remove one harness component, measure the effect, restore it. This tells you which components still earn their keep.
8. Treat flaky tests as failures, not noise. Quarantine the test with an owner and a fix deadline; do not add retries just to make CI green. A retry that hides a race condition is a false pass.
9. When multiple agents work in parallel, isolate them with git worktrees or branches. Give each feature one owner and avoid concurrent writes to the same feature list, progress, or handoff files. Merge through CI, not by copying files between worktrees.
10. If a session cannot be recovered, revert to the last green commit, record the failure and the decision in the handoff, and start the next session from a known-good state. Do not build on a broken baseline.

A generic CI sketch is in Appendix L; adapt the stages to your stack.

**Exit gate.**

- [ ] CI blocks a merge when any required check fails.
- [ ] Two consecutive sessions follow the clock-in/clock-out protocol.
- [ ] The maintenance cadence has an owner and a calendar.
- [ ] Metrics are recorded, not just discussed.
- [ ] At least one obsolete rule or file has been removed in the last quarter.

**Pitfall.** CI that only builds. Also common: a session protocol that exists in a document but not in practice, and a maintenance loop with no owner. A harness rots like code; if nobody prunes it, it becomes noise.

**Reference.** `init.sh` is the single verification entry point. The evaluator rubric, quality document, benchmark, and cleanup scanner provide the evidence the operating loop needs.

### Step 14 — Apply the Acceptance Gate

**Goal.** Decide objectively when the repository is harness-ready, and avoid endless polishing.

Use this scorecard. Score each category 0-2:

- **0** — missing or unreliable.
- **1** — present but inconsistent or manual.
- **2** — present, evidence-backed, and enforced.

| Category | 0 | 1 | 2 |
|---|---|---|---|
| Fresh clone and startup | Fails or undocumented | Works with manual steps | `init.sh` passes in under 5 minutes |
| Fresh-session test | 0-1 answers | 2-3 answers | 4-5 answers from repo alone |
| Instructions | Missing or huge | Present but stale/duplicated | Short, linked, tested, one source of truth |
| State | No state files | Notes exist but ambiguous | Feature list + progress + handoff, validated |
| Verification | No tests | Some tests, not enforced | Full check + e2e + architecture checks enforced |
| Definition of done | Subjective | Written but ignored | Evidence required; `passing` gated |
| Observability | `console.log` | Some structured logs | JSON logs, levels, critical flows, no secrets |
| Quality review | Self-review | Rubric exists but informal | Independent evaluator, evidence-backed scores |
| Hygiene | No scanner | Manual cleanup | Benchmark baseline + read-only scanner |
| Module structure | Tangled | Some boundaries | Acyclic, documented, enforced |
| CI | None | Build only | Full gate, branch protection, artifacts |
| Operating loop | Ad hoc | Occasional | Session protocol + weekly/monthly cadence |

**Scoring.**

- Maximum: 24.
- Harness-ready target: **>= 20/24**, with no `0` in fresh clone, verification, or state.
- If a critical category is `0`, fix it before adding more features.

**Rollout plan for a typical repository.**

| Time | Work | Steps |
|---|---|---|
| Day 1 | Entry file, docs map, init script | 0-3 |
| Day 2 | Feature list, progress/handoff, clean state | 4-6 |
| Week 1 | Tests, e2e, architecture checks, rubric | 7-8 |
| Week 2 | Quality doc, structured logging, benchmark, scanner | 9-11 |
| Week 3 | Module refactor and CI | 12-13 |
| Ongoing | Acceptance gate, audits, pruning | 14 |

Adjust to the size of the repository. The sequence matters more than the calendar.

**Before/after evidence.** Record at least these metrics before Step 0 and after Step 13:

- Time to first passing test on a clean clone.
- Time for a fresh session to answer the five questions.
- Verified completion rate.
- Escaped defects and rework rate.
- Number of boundary violations.
- Log parse rate.
- Benchmark trend.

If the numbers did not improve, the harness changes were theater. Use the ablation test to find out which components actually matter.

---

## 10. Adapting the Recipe to Any Stack

The roles are universal; the commands are not. Replace the placeholders with real commands.

| Stack | Install | Static checks | Tests | E2E | Build | Structured logs |
|---|---|---|---|---|---|---|
| Node / TypeScript | `npm ci`, `pnpm install --frozen-lockfile` | `tsc --noEmit`, `eslint` | `vitest`, `jest` | `playwright`, `cypress` | `vite build`, `tsc` | `pino`, `winston`, custom JSON logger |
| Python | `uv sync --frozen`, `pip install -r requirements.txt` | `mypy`, `pyright`, `ruff` | `pytest` | `playwright`, `behave` | `python -m build` | `structlog`, `logging` with JSON formatter |
| Go | `go mod download` | `go vet`, `golangci-lint` | `go test ./...` | `playwright-go`, CLI integration | `go build ./...` | `log/slog` JSON handler |
| Rust | `cargo fetch` | `cargo check`, `clippy` | `cargo test` | CLI/integration tests | `cargo build --release` | `tracing` + `tracing-subscriber` JSON |
| Generic | Make target | Make target | Make target | Script | Make target | JSON lines to stdout |

Rules that apply to every stack:

- One command for full verification.
- Lock dependencies.
- Fail loudly, with a fix instruction.
- Keep logs JSON-serializable.
- Scope destructive cleanup to application data.
- Do not let the harness depend on one editor or one agent vendor.

### Monorepos, Legacy Code, and Generated Code

- **Monorepo:** keep a root `init.sh` that delegates to the affected packages. Use one root feature list with an `area` field, or per-package lists with a clear owner. Enforce package boundaries with the same architecture check.
- **Legacy code:** start with a smoke test and characterization tests around the module you will touch. Add the verification ladder incrementally; do not wait for a perfect suite before making the first safe change.
- **Generated code:** exclude generated and vendor directories from architecture and scanner checks. Record the generation command and verify that regenerating produces no diff.

---

## 11. Anti-Patterns

| Anti-pattern | Why it hurts | Fix |
|---|---|---|
| Giant `AGENTS.md` | Eats context, buries critical rules, accumulates contradictions | Keep 50-200 lines; move detail to linked docs |
| Feature list as a wish list | No shared definition of done; false passing | Require behavior, verification, and evidence per item |
| Marking `passing` without running checks | The most expensive lie in the harness | Gate state changes on verification output |
| Unit tests only | Misses interface mismatches and cross-layer defects | Add e2e for critical flows |
| `console.log` everywhere | Logs are unparseable and inconsistent | One logger, JSON schema, levels |
| Logging secrets or personal data | Security and compliance incident | Redaction, allowlists, review |
| Cleanup scanner that deletes by default | Destroys trust and data | Read-only default; explicit `--fix`; scoped paths |
| Big-bang refactor | Unverifiable; failures cannot be attributed | Small behavior-preserving steps behind tests |
| Refactor mixed with feature work | Two variables change at once; review and rollback are hard | Separate commits and scopes |
| Stale docs | Actively misleads the agent | Update in the same commit; delete what is not true |
| CI that skips tests | Green pipeline that proves nothing | Treat skipped as failed; protect the branch |
| Harness files written once | Decay and drift | Weekly cleanup; monthly audit; quarterly prune |
| Copying the reference files verbatim | Commands and paths will be wrong | Adapt roles, not text |
| Too much harness ceremony | Agents spend more time reading process than doing work | Remove components that do not change outcomes |

---

## 12. Mapping to the Course

| Recipe step | Related lecture | Related project |
|---|---|---|
| 0 Baseline audit | Lecture 02: What a harness actually is | Project 01: baseline vs minimal harness |
| 1 `AGENTS.md` | Lecture 04: split instructions across files | Project 02: agent-readable workspace |
| 2 Docs map | Lecture 03: repository as system of record; Lecture 04 | Project 02 |
| 3 `init.sh` | Lecture 06: initialization as its own phase | Project 03: multi-session continuity |
| 4 Feature list | Lecture 07: scope boundaries; Lecture 08: feature lists as primitives | Project 04: incremental indexing |
| 5 Progress and handoff | Lecture 05: cross-session continuity | Project 03 |
| 6 Clean state | Lecture 12: clean handoff every session | Project 06 |
| 7 Verification ladder | Lecture 09: premature victory; Lecture 10: end-to-end testing | Project 05: grounded Q&A verification |
| 8 Evaluator rubric | Lecture 09; Lecture 11: process observability | Project 05: role separation |
| 9 Quality document | Lecture 12 | Project 06 |
| 10 Structured logging | Lecture 11: runtime observability | Project 06 |
| 11 Benchmark and scanner | Lecture 11; Lecture 12 | Project 06 |
| 12 Module refactor | Lecture 03: knowledge next to code; Lecture 10: architecture boundaries | Project 06 |
| 13 CI and operating loop | Lecture 12; Lecture 13: loop engineering | Project 06 |
| 14 Acceptance gate | All of the above | Project 06 capstone |

---

## 13. Appendix: Copy-Paste Templates

These are **minimum viable** versions. Replace placeholders with real commands and paths. The full reference versions are in `projects/project-06/solution`.

### A. `AGENTS.md`

```markdown
# AGENTS.md — <Project>

## What This Is
<One or two sentences: product, users, stack, runtime.>

## Startup (in order)
1. Read `claude-progress.md` for current state and next action.
2. Read `feature_list.json` and pick the highest-priority unfinished item.
3. Read the relevant doc for the area you will touch.
4. Run `bash init.sh`. If the baseline fails, fix it before new work.
5. Run the focused smoke check for the selected feature.

## Commands
- Install: `<INSTALL>`
- Dev: `<DEV>`
- Build: `<BUILD>`
- Type-check / lint: `<TYPECHECK>` / `<LINT>`
- Tests: `<TEST>`
- End-to-end: `<E2E>`
- Full verification: `<FULL_CHECK>`

## Hard Constraints
- <Specific, checkable rule.>
- <Specific, checkable rule.>
- <Maximum 15.>

## Work Rules
- One feature in progress at a time.
- Do not mark `passing` without running verification and recording evidence.
- Stay in scope; no drive-by refactors.
- Do not change verification rules during implementation.
- Prefer repository artifacts over chat summaries.

## Docs Map
- `docs/ARCHITECTURE.md` — read before changing modules or data flow.
- `docs/PRODUCT.md` — read before changing user-visible behavior.
- `docs/RELIABILITY.md` — read before changing logging, reset, or benchmarks.

## Definition Of Done
1. Behavior implemented.
2. Required verification ran and passed.
3. Evidence recorded in `feature_list.json`.
4. Docs updated if boundaries or behavior changed.
5. `clean-state-checklist.md` passes.
6. Work committed with a descriptive message.

## End Of Session
1. Run full verification.
2. Update `feature_list.json`, `claude-progress.md`, `session-handoff.md`.
3. Remove debug artifacts.
4. Commit; leave `bash init.sh` green.
```

### B. `feature_list.json`

```json
{
  "project": "<project-name>",
  "last_updated": "YYYY-MM-DD",
  "rules": {
    "single_active_feature": true,
    "passing_requires_evidence": true,
    "do_not_skip_verification": true
  },
  "status_legend": {
    "not_started": "Work has not begun.",
    "in_progress": "The current active task.",
    "blocked": "Cannot continue until a documented blocker is resolved.",
    "passing": "Required verification passed and evidence is recorded."
  },
  "features": [
    {
      "id": "<area>-001",
      "priority": 1,
      "area": "<area>",
      "title": "<short title>",
      "user_visible_behavior": "<what a user can observe when it works>",
      "status": "not_started",
      "verification": [
        "Run `<TEST>`.",
        "Start the app with `<DEV>`.",
        "<observable check>."
      ],
      "evidence": [],
      "notes": ""
    }
  ]
}
```

Minimal validator (`scripts/validate-features.mjs`):

```js
import fs from "node:fs";

const data = JSON.parse(fs.readFileSync("feature_list.json", "utf8"));
const valid = new Set(["not_started", "in_progress", "blocked", "passing"]);
const errors = [];
const ids = new Set();
let active = 0;

for (const f of data.features ?? []) {
  if (!f.id || ids.has(f.id)) errors.push(`Invalid or duplicate id: ${f.id}`);
  ids.add(f.id);
  if (!valid.has(f.status)) errors.push(`${f.id}: invalid status "${f.status}"`);
  if (f.status === "in_progress") active += 1;
  if (f.status === "passing" && (!Array.isArray(f.evidence) || f.evidence.length === 0)) {
    errors.push(`${f.id}: passing without evidence`);
  }
  if (!Array.isArray(f.verification) || f.verification.length === 0) {
    errors.push(`${f.id}: missing verification steps`);
  }
}

if (active > 1) errors.push(`More than one in_progress feature: ${active}`);
if (errors.length) {
  console.error(errors.join("\n"));
  process.exit(1);
}
console.log("feature_list.json OK");
```

Run `node scripts/validate-features.mjs` from `init.sh` and CI. If you keep the reference's `status: "pass"`, add `"pass"` to the valid set or migrate the file.

### C. `init.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$ROOT_DIR"

# The commands below are examples. Replace them with the values from your Project Profile.
INSTALL_CMD=(npm ci)
STATIC_CMD=(npm run check)
BUILD_CMD=(npm run build)
TEST_CMD=(npm test)
START_CMD=(npm run dev)

echo "==> Working directory: $PWD"
echo "==> Installing dependencies"
"${INSTALL_CMD[@]}"

echo "==> Static checks"
"${STATIC_CMD[@]}"

echo "==> Build"
"${BUILD_CMD[@]}"

echo "==> Tests"
"${TEST_CMD[@]}"

echo "==> Verifying harness files"
for file in AGENTS.md feature_list.json claude-progress.md clean-state-checklist.md; do
  if [ ! -f "$file" ]; then
    echo "MISSING: $file"
    echo "FIX: restore the file or update this check if the harness has moved."
    exit 1
  fi
done

echo "==> Validating feature list"
if [ -f scripts/validate-features.mjs ]; then
  node scripts/validate-features.mjs
fi

echo "==> Architecture checks"
if [ -f scripts/check-architecture.sh ]; then
  bash scripts/check-architecture.sh
fi

echo "==> Startup command"
printf '    %q' "${START_CMD[@]}"
printf '\n'

if [ "${RUN_START_COMMAND:-0}" = "1" ]; then
  exec "${START_CMD[@]}"
fi
```

### D. `claude-progress.md`

```markdown
# Progress

## Current Verified State
- Repository root: `<path>`
- Standard startup: `<DEV>`
- Standard verification: `<FULL_CHECK>`
- Highest-priority unfinished feature: `<id / title>`
- Current blocker: `<none or description>`
- Last verified commit: `<sha>`

## Session Record — <date>
- Goal:
- Completed:
- Verification run:
- Evidence:
- Decisions:
- Files modified:
- Commits:
- Known risks:
- Next best action:
```

### E. `session-handoff.md`

```markdown
# Session Handoff — <date>

## Verified Now
- What works:
- What verification ran:

## Changed This Session
- Code or behavior:
- Infrastructure or harness:
- Decisions:
- Files modified:

## Broken Or Unverified
- Known defect:
- Unverified path:
- Risk for the next session:

## Next Best Step
- Feature:
- Why this is next:
- What counts as passing:
- What must not change:

## Commands
- Startup: `<DEV>`
- Verification: `<FULL_CHECK>`
- Focused debug: `<COMMAND>`
```

### F. `clean-state-checklist.md`

```markdown
# Clean State Checklist

## Build
- [ ] Type-check passes.
- [ ] Lint passes.
- [ ] Build passes.

## Architecture
- [ ] No forbidden imports.
- [ ] All shared types live in the agreed location.
- [ ] No business logic in the UI layer.

## Runtime
- [ ] Standard startup works.
- [ ] Critical smoke flow works.
- [ ] Reset works and is idempotent.

## Observability
- [ ] Logs are valid JSON.
- [ ] Critical flows emit start/success/failure.
- [ ] No secrets in logs.

## Data Integrity
- [ ] No orphaned or dangling data.
- [ ] State persists across restart.
- [ ] Clean reset removes all app data.

## Performance
- [ ] Benchmark runs.
- [ ] Results are within the documented threshold.

## Repository
- [ ] `git status` contains only intended changes.
- [ ] No secrets, build output, or temp files staged.
- [ ] Feature list and progress are current.
- [ ] Handoff is current if the session ends.
- [ ] Docs are updated or explicitly marked unchanged.

## Scripts
- [ ] `init.sh` passes.
- [ ] Cleanup scanner reports no issues.
```

### G. `evaluator-rubric.md`

```markdown
# Evaluator Rubric — <feature / milestone>

## Verdict
- Overall score: `<1-5>`
- Threshold: `>= 4`, no dimension below `3`, mandatory dimensions `>= 4`.
- Decision: `PASS / FAIL`

## Scores
| Dimension | Score | Evidence |
|---|---:|---|
| Build and static checks (mandatory) | | |
| Behavior correctness (mandatory) | | |
| Architecture compliance | | |
| Test coverage and stability | | |
| Observability | | |
| Documentation and state | | |
| Performance and resources | | |
| Security and data safety (mandatory) | | |

## Defects
| Severity | Location | Problem | Required fix |
|---|---|---|---|
| | | | |

## Notes
- What was actually run:
- What was not run and why:
```

### H. `quality-document.md`

```markdown
# Quality Document

## Overall
- Overall grade: `<A-D>`
- Last updated: `<date>`

## Summary
| Area | Grade | Verification | Agent readability | Test stability | Architecture |
|---|---|---|---|---|---|
| `<module>` | A/B/C/D | passing/partial/failing | easy/difficult | stable/flaky/untested | compliant/violations |

## Details

### `<module>` — Grade `<A-D>`
- Evidence:
- Known debt:
- Next action:
- Last updated:
```

### I. Logger Contract

```ts
export type LogLevel = "DEBUG" | "INFO" | "WARN" | "ERROR";

export interface LogEntry {
  timestamp: string;
  level: LogLevel;
  service: string;
  event: string;
  data?: Record<string, unknown>;
  correlationId?: string;
  durationMs?: number;
  error?: {
    name: string;
    message: string;
    stack?: string;
  };
}

export interface ServiceLogger {
  debug(event: string, data?: Record<string, unknown>): void;
  info(event: string, data?: Record<string, unknown>): void;
  warn(event: string, data?: Record<string, unknown>): void;
  error(event: string, data?: Record<string, unknown>): void;
}
```

Implementation rules:

- One `logger.forService(name)` factory.
- `JSON.stringify` one line per entry.
- Route WARN and ERROR to stderr if your platform expects it; otherwise stdout is fine.
- Read `LOG_LEVEL` from the environment.
- Never interpolate untrusted strings into a non-JSON format.
- Add a test that every line parses and has `timestamp`, `level`, `service`, and `event`.

### J. `check-architecture.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

VIOLATIONS=0

echo "=== Architecture Boundary Checks ==="

# Requires ripgrep (`rg`). Replace with `grep -rn` if ripgrep is unavailable.
if ! command -v rg >/dev/null 2>&1; then
  echo "ERROR: ripgrep (rg) is required for this check."
  echo "FIX: install ripgrep, or replace the rg calls below with grep -rn."
  exit 1
fi

# Example: renderer must not import Node.js core modules.
if rg -n "from ['\"](fs|path|os|child_process)['\"]" src/renderer 2>/dev/null; then
  echo "ERROR: Renderer imports a Node.js core module."
  echo "WHY: The renderer has no Node.js access; this breaks the security boundary."
  echo "FIX: Move the operation to the service layer and expose it through the preload bridge."
  VIOLATIONS=$((VIOLATIONS + 1))
fi

# Example: services must not import Electron or React.
if rg -n "from ['\"](electron|react)['\"]" src/services 2>/dev/null; then
  echo "ERROR: A service imports Electron or React."
  echo "WHY: Services must stay pure and testable."
  echo "FIX: Inject the dependency at the main-process boundary."
  VIOLATIONS=$((VIOLATIONS + 1))
fi

if [ "$VIOLATIONS" -gt 0 ]; then
  echo "FAIL: $VIOLATIONS violation(s)."
  exit 1
fi

echo "PASS: All architecture checks passed."
```

Adapt the paths and rules to your repository. If you use `ripgrep` in CI, pin the version or use a portable grep wrapper.

### K. `sprint-contract.md`

```markdown
# Sprint Contract — <feature>

## Scope
- Files or modules in scope:
- Behavior to add or change:
- Data or API changes:

## Verification
- Commands to run:
- Critical flow to exercise:
- Evidence to capture:

## Exclusions
- Not doing:
- Not touching:

## Done Criteria
- [ ] Behavior is observable and verified.
- [ ] Required checks passed.
- [ ] Evidence is recorded.
- [ ] Docs and state are updated.
```

### L. CI Pipeline Sketch

```yaml
name: harness
on:
  push:
  pull_request:

jobs:
  verify:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm
      - run: npm ci
      - run: npm run check
      - run: npm test
      - run: npm run build
      - run: bash scripts/check-architecture.sh
      - run: node scripts/validate-features.mjs
      - run: bash scripts/cleanup-scanner.sh
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: verification-output
          path: |
            coverage/
            test-results/
```

This is a Node example; adapt the setup action and commands to your stack, and pin action versions in a real repository. Enable branch protection so this job must pass before merge. Do not use `|| true` or `continue-on-error` to make it green.

---

## 14. Final Checklist

Use this before calling a repository "harness-ready":

- [ ] `AGENTS.md` is short, tested, and links to detailed docs.
- [ ] A fresh session answers all five questions from the repository alone.
- [ ] `init.sh` passes on a clean clone in under five minutes.
- [ ] `feature_list.json` is valid, has at most one active item, and every `passing` item has evidence.
- [ ] `claude-progress.md` and `session-handoff.md` reflect the real state.
- [ ] The clean-state checklist passes, and reset is idempotent.
- [ ] The full check covers static checks, tests, and e2e for critical flows.
- [ ] Architecture boundaries are enforced by a script or linter.
- [ ] An independent evaluator scores against a rubric with evidence; the evaluator is not the generator.
- [ ] Environment and tool access are pinned and documented (lockfile, runtime version, secrets, permissions).
- [ ] The quality document is current and the weakest module has an action.
- [ ] Logs are structured JSON, parseable, and free of secrets.
- [ ] Benchmarks have a baseline and thresholds.
- [ ] The cleanup scanner is read-only by default and finds seeded issues.
- [ ] Module dependencies are documented, acyclic, and enforced.
- [ ] CI blocks merge on failure, and the session protocol is actually followed.
- [ ] Metrics show improvement over the Step 0 baseline.

If the scorecard is at least 20/24 and most boxes here are checked, you have a repository that an agent can operate. If not, pick the lowest-scoring subsystem and fix that first. **Do less, finish more, leave evidence.** That is the whole harness discipline.

---

## 15. Known Limitations

This guide is intentionally stack-agnostic and role-focused. It does not include:

- **Full benchmark and scanner implementations.** Step 11 defines the checks; the reference has shell scripts. Port them to your language and test them against seeded failures.
- **Full architecture and product document templates.** Step 2 lists the required sections; use the reference docs as examples and adapt them to your domain.
- **Detailed 1-5 rubric anchors.** Appendix G lists dimensions; define concrete anchors for your product so two evaluators reach similar scores.
- **A full logger implementation.** Appendix I is a contract; the reference `logger.ts` is a TypeScript implementation. Adapt it or use your stack's structured logger.
- **A security threat model.** The guide covers secrets, boundaries, and least privilege, but not authentication, authorization, dependency supply chain, or sandboxing. Add those for production.
- **A complete multi-agent protocol.** Step 13 gives isolation basics; larger teams need branch ownership, merge queues, and conflict-resolution policies.
- **Deep monorepo guidance.** Section 10 gives the shape; per-package versioning, affected-only CI, and ownership need separate design.
- **A migration and rollback playbook.** Step 13 says to revert to the last green commit; schema and data migrations need their own forward and backward plan.
- **A performance profiling methodology.** The benchmark measures outcomes; it does not explain how to profile CPU, memory, or I/O.
- **Product and UX quality criteria.** The rubric covers behavior, architecture, and security; accessibility, visual quality, and product fit need domain-specific criteria.
- **Harness cost measurement.** The context-budget note is a heuristic; exact token accounting per file and per session is tool-specific.

Add the missing pieces that matter for your risk profile. Do not add all of them by default: harness ceremony has a real cost, and every component must earn its keep.
