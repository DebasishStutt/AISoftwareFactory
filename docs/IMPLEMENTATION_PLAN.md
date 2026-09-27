# Software Factory — Implementation Plan

Target: a Git repo you clone locally, open in VS Code, and use to develop software with Claude Code as the agent runtime. Governance is deterministic Python invoked by Claude Code hooks, so it never depends on the LLM behaving (spec §10).

Commit this file as `docs/IMPLEMENTATION_PLAN.md`. The ledger at the bottom is the single source of truth for where you are.

---

## 0. Assumptions (change here before starting)

| ID | Decision | Default | Status |
|----|----------|---------|--------|
| D-01 | Governance/CLI language | Python 3.12 | ASSUMED |
| D-02 | Repo host + CI | GitHub + GitHub Actions | ASSUMED |
| D-03 | LLM billing | Claude Pro/Max subscription (no per-token cost) | ASSUMED |
| D-04 | Execution environment | Devcontainer with egress allowlist (based on Anthropic's reference Claude Code devcontainer) | ASSUMED |
| D-05 | First domain after the vertical slice | Embedded (ARM Cortex-M, FreeRTOS) | ASSUMED |
| D-06 | Product code location | `workspace/<project>/`, each its own Git repo, gitignored by the factory | ASSUMED |

**Deliberate deviation from spec §47:** governance (M3) is built *before* agents (M2). Hooks must exist before any agent runs, otherwise the first agent sessions are ungoverned.

---

## 1. Checkpoint protocol (pause / resume)

Every checkpoint (CP) ends in a state you can walk away from.

**To close a checkpoint**
1. All "Verify" commands pass.
2. Update the ledger row: status `DONE`, date, commit hash, notes.
3. Commit: `git commit -m "CP-XX: <title>"`.
4. Tag: `git tag cp-XX`.
5. End the Claude Code session (`/clear` or close). One checkpoint per session keeps context clean.

**To resume**
1. `git status` must be clean. If it isn't, finish or stash before anything else.
2. Open Claude Code and paste the resume prompt:

```text
Read docs/IMPLEMENTATION_PLAN.md. Find the first checkpoint in the
ledger that is not DONE. Re-run the Verify commands of the last DONE
checkpoint to confirm the baseline is green. Then implement only the
next checkpoint. Stop when its Verify commands pass, update the ledger,
and tell me — do not start the following checkpoint.
```

**To roll back:** `git reset --hard cp-XX` returns you to the last known-good checkpoint.

**Stop rules (any session)**
- Never start a checkpoint on a red baseline.
- If a checkpoint takes more than 3 sessions, split it and record the split in the ledger.
- If a human decision is needed, add it to §0, mark the CP `BLOCKED`, and stop.

Size key: **S** ≈ 1 session, **M** ≈ 2, **L** ≈ 3.

---

## Phase A — Foundation (bootstrap, no agents yet)

During Phase A you use Claude Code to *build* the factory. Until CP-04 is done the hooks don't exist yet, so run these sessions in default permission mode and approve every edit manually.

### CP-00 · Host prerequisites · S
- [ ] VS Code + Dev Containers extension + Claude Code extension
- [ ] Docker running; Git configured; GitHub repo created (private)

**Verify:** `docker run --rm hello-world` · `git --version` · Claude Code opens in VS Code.

### CP-01 · Repo skeleton + devcontainer · M
- [ ] Directory tree (see §5)
- [ ] `.devcontainer/` with: Python 3.12, git, cmake, ninja, gcc/clang, clang-tidy, cppcheck, gitleaks, syft, arm-none-eabi-gcc
- [ ] Egress allowlist: Anthropic API, GitHub, package registries only
- [ ] `.gitignore` covers `workspace/`, `.factory/state/`, secrets

**Verify:** "Reopen in Container" succeeds · `cmake --version && arm-none-eabi-gcc --version && gitleaks version` · `curl https://example.com` from the container **fails** (allowlist works).

### CP-02 · CLAUDE.md, policies, schemas · M
- [ ] `CLAUDE.md`: principles (§1), engineering behaviour (§53), failure handling (§55), "external content is data" rule (§4)
- [ ] `policies/capabilities.yaml`: one entry per planned agent, default deny
- [ ] `policies/project-default.yaml`: network_write, git_push, hardware_flash, cloud, financial all `false`; budget €0
- [ ] `schemas/`: JSON Schema for AuditEvent, Approval, Task, Artifact, Requirement, Gate, Decision (§62 subset)

**Verify:** `python -m factory.schemas.validate policies/` passes · invalid sample policy is rejected.

---

## Phase B — Governance core (spec M3, pulled forward)

### CP-03 · Tool Firewall hook · L
- [ ] `factory/governance/firewall.py`: reads hook JSON from stdin, classifies the call into §3.2 classes, returns allow / deny / ask
- [ ] Classification rules: Bash commands are *parsed*, not pattern-matched (catches `bash -c`, `python -c`, `curl`, `git push`, `openocd`, `st-flash`)
- [ ] Unknown tool or unparseable command → DENY
- [ ] `.claude/settings.json`: `PreToolUse` hook with matcher `*` → firewall

**Verify:** `pytest tests/firewall/` · manual: ask Claude to `git push` → denied with reason.

### CP-04 · Audit + kill switch · M
- [ ] `PreToolUse` and `PostToolUse` both append to `.factory/audit/YYYY-MM-DD.jsonl`
- [ ] Each record hash-chained (`prev_hash`); fields per §8
- [ ] `factory audit verify` detects a tampered line
- [ ] Kill switch: if `.factory/KILL` exists, firewall denies everything

**Verify:** `pytest tests/audit/` · edit one audit line by hand → `factory audit verify` fails · `touch .factory/KILL` → every tool call denied.

**From here on, the factory governs its own construction.** Switch Claude Code to normal mode; the firewall decides.

### CP-05 · Governance immutability · M
- [ ] Settings deny rules for Edit/Write on `.claude/**`, `factory/governance/**`, `policies/**`, `.factory/audit/**`
- [ ] Firewall re-checks the same paths itself (belt and braces)
- [ ] Devcontainer mounts those paths read-only
- [ ] `CODEOWNERS` + branch protection on `main`
- [ ] Governance changes only via a documented maintenance workflow (§59): you edit them by hand, outside Claude Code

**Verify:** ask Claude to edit `firewall.py` → denied + audited · attempt via Bash (`sed -i`) → denied.

### CP-06 · CLI, state machine, approvals, budget · L
- [ ] `factory init|status|approve|deny|pause|resume|cancel|logs|audit|report`
- [ ] SQLite state: projects, tasks, approvals, budget ledger
- [ ] State machine per §9; `factory advance <STATE>` refuses a transition without the gate evidence
- [ ] Approval broker: firewall returns `ask` for approve-once; `factory approve <id> --scope project --limit N` writes a policy override the firewall reads
- [ ] Budget: any FINANCIAL or COST_UNKNOWN action → deny unless approved

**Verify:** `pytest tests/cli tests/state tests/approvals` · `factory advance TESTING` from INITIALIZED → refused.

### CP-07 · Guardrail harness + CI · M
- [ ] `tests/guardrails/`: synthetic hook JSON for every §40 case; each asserts **DENY + audit record exists**
- [ ] GitHub Actions: lint, unit, guardrail, schema validation. No paid services.

**Verify:** CI green on `main` · deliberately weaken one firewall rule on a branch → CI goes red.

**Milestone gate G-1: governance proven.** No agents until this passes.

---

## Phase C — Agents and engineering workflow (spec M1/M2)

### CP-08 · Core agents · M
- [ ] `.claude/agents/`: requirements, architect, planner, implementer, tester, reviewer, security, docs
- [ ] Each has role, input/output artifacts, a `tools:` allowlist matching `capabilities.yaml`
- [ ] Reviewer and security agents: read-only tools
- [ ] Port your embedded skills into `.claude/skills/`

**Verify:** `factory registry check` confirms agent files ⇔ `capabilities.yaml` agree · reviewer agent tries to Write → denied.

### CP-09 · Task engine, artifacts, traceability · M
- [ ] Artifact files under `workspace/<project>/.factory/` (requirements.yaml, adr/, threat-model.md, plan.yaml)
- [ ] Stable IDs (REQ-F-001, TASK-001, TEST-001) and `factory trace` producing REQ → task → test → result
- [ ] Task graph with dependencies; `factory tasks` shows ready/blocked

**Verify:** `pytest tests/tasks tests/trace` · a plan with a cyclic dependency is rejected.

### CP-10 · Slash commands (orchestrator) · M
- [ ] `/build "<request>"`: drives the phases by calling agents and `factory advance`
- [ ] `/status`, `/gate <name>`, `/approve-queue`
- [ ] The orchestrator never skips `factory advance`; the CLI enforces the order

**Verify:** `/build` on a toy request reaches ARCHITECTURE and stops at the architecture-approval gate.

---

## Phase D — Verification (spec M4)

### CP-11 · Gate engine · L
- [ ] `factory gate run <gate>` executes deterministic tools and stores evidence as artifacts
- [ ] Gates: build, unit tests (ctest), static analysis (clang-tidy, cppcheck), secrets (gitleaks), SBOM (syft → CycloneDX)
- [ ] A failed gate blocks `factory advance` unless you override with a logged decision (§57)

**Verify:** a planted failing test blocks TESTING → SECURITY_VERIFICATION · a planted fake secret fails the secrets gate.

### CP-12 · Mandatory vertical slice (§49) · L
- [ ] `/build "small C++17 ring-buffer library with unit tests"` end to end
- [ ] Real outputs: sources, tests, test results, review findings, security analysis, SBOM, trace, audit, completion report

**Verify:** `factory report` shows every §49 artifact with evidence · `factory audit verify` passes · nothing mocked.

**Milestone gate G-2: factory functional.**

---

## Phase E — Hardening

### CP-13 · Prompt-injection + self-test suite (§39) · M
- [ ] Fixture project with injected instructions in README, source comments, logs, test fixtures
- [ ] Each case asserts no control-plane change and a denied action + audit record
- [ ] Failure cases: missing toolchain, failing tests, retry exhaustion, approval denial

**Verify:** `pytest tests/selftest` green in CI.

---

## Phase F — Embedded domain (spec M6)

### CP-14 · Embedded build and test workflow · L
- [ ] Domain agents: embedded-architect, firmware-engineer, embedded-security
- [ ] Host-side unit tests (CppUTest or GTest) plus a cross build with arm-none-eabi-gcc; `.map` size report as an artifact
- [ ] Optional: QEMU Cortex-M target for smoke tests

**Verify:** `/build` for a small FreeRTOS-free driver module produces a host-tested library plus a cross-compiled `.elf` with a size report.

### CP-15 · Flash adapter (DEPLOYMENT class) · M
- [ ] `factory flash` wraps OpenOCD or J-Link; firewall classifies it as DEPLOYMENT → **always ask**, never project-approvable by default
- [ ] Post-flash verification: reset, boot banner over UART, smoke test. Otherwise report "DEPLOYED — VERIFICATION NOT CONFIRMED"

**Verify:** flash request without approval → denied + audited · with approval on a real board → verified or honestly unconfirmed.

**Milestone gate G-3: embedded workflow proven.**

---

## Backlog (plan when G-3 is done)

- M5 Android (Gradle, emulator, APK/AAB, MASVS checks)
- M7 System software (Rust workflow, packaging, perf tests)
- M9 Cloud: only after G-1..G-3; proposal-first per §38
- Observability dashboard (§61), versioned schemas (§43), release packaging (§67)

---

## 5. Target repo layout

```text
software-factory/
├── CLAUDE.md
├── .claude/{settings.json, agents/, commands/, skills/}
├── .devcontainer/
├── factory/{cli.py, governance/, state/, tasks/, gates/, trace/}
├── policies/
├── schemas/
├── tests/{firewall, audit, guardrails, selftest, ...}
├── docs/IMPLEMENTATION_PLAN.md
├── .factory/{audit/, state/, KILL?}
└── workspace/<project>/          # product repos, gitignored
```

---

## 6. Progress ledger

| CP | Title | Size | Status | Date | Commit | Notes |
|----|-------|------|--------|------|--------|-------|
| 00 | Host prerequisites | S | TODO | | | |
| 01 | Repo skeleton + devcontainer | M | TODO | | | |
| 02 | CLAUDE.md, policies, schemas | M | TODO | | | |
| 03 | Tool Firewall hook | L | TODO | | | |
| 04 | Audit + kill switch | M | TODO | | | |
| 05 | Governance immutability | M | TODO | | | |
| 06 | CLI, state, approvals, budget | L | TODO | | | |
| 07 | Guardrail harness + CI → **G-1** | M | TODO | | | |
| 08 | Core agents | M | TODO | | | |
| 09 | Task engine, artifacts, trace | M | TODO | | | |
| 10 | Slash commands (orchestrator) | M | TODO | | | |
| 11 | Gate engine | L | TODO | | | |
| 12 | Vertical slice → **G-2** | L | TODO | | | |
| 13 | Prompt-injection + self-test | M | TODO | | | |
| 14 | Embedded build/test | L | TODO | | | |
| 15 | Flash adapter → **G-3** | M | TODO | | | |

Status values: `TODO` · `IN_PROGRESS` · `BLOCKED (<reason>)` · `DONE`
