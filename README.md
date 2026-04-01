# Agent Dev Pipeline

A structured software development methodology and coding pipeline for AI agent-driven development.

## Why this exists

AI agents writing production code need **mechanically enforceable guardrails**, not trust-based guidelines. This repo defines the methodology, framework, and pipeline skill that governs how agents build, test, and deploy code across our projects.

## Contents

- `methodology/research.md` — Comparison of 10 software development methodologies evaluated for AI agent workflows
- `methodology/framework.md` — The concrete 4-layer hybrid framework we use
- `SKILL.md` — The coding pipeline skill (OpenClaw-compatible) that agents follow when building features

## The Original 7-Stage Pipeline

This pipeline is built on top of the [openclaw-coding-pipeline](https://github.com/Clintos11/openclaw-coding-pipeline) by [@Clintos11](https://github.com/Clintos11). The original pipeline defines 7 orchestrated stages, each run by a dedicated sub-agent with its own persona and model assignment:

| # | Stage | Agent | Model Type | Purpose |
|---|-------|-------|------------|---------|
| 1 | **Plan** | `feature-dev-planner` | Reasoning | Decompose spec into ordered, dependency-aware user stories |
| 2 | **Setup** | `feature-dev-setup` | Coding | Create feature branch, establish build/test baseline |
| 3 | **Implement** | `feature-dev-developer` | Coding | Build each story, write tests, commit atomically |
| 4 | **Verify** | `feature-dev-verifier` | Reasoning | Quality gate: diff review, security checks, test suite |
| 5 | **Test** | `feature-dev-tester` | Coding | Integration/E2E testing, classify failures |
| 6 | **PR** | `feature-dev-developer` | Coding | Push branch, create GitHub PR |
| 7 | **Review** | `feature-dev-reviewer` | Reasoning | Final code review, merge readiness check |

### Pipeline flow with retry loops

```
PLAN → SETUP → IMPLEMENT ←──┐
                              │
         VERIFY ──(fail)──────┘ (max 2 retries)
           │
         (pass)
           │
         TEST ──(fail)──→ IMPLEMENT fix (max 1) → re-TEST
           │
         (pass)
           │
         PR → REVIEW ──(changes)──→ IMPLEMENT fix (max 1) → re-REVIEW
                │
              (approved)
                │
              DONE
```

### Key design principles from the original pipeline

- **One stage per turn, then yield.** Never sleep, poll, or spawn multiple stages at once.
- **Progress file is the source of truth**, not session memory. Located at `<repo>/progress-<branch-name>.md`.
- **`mode: "session"` is non-negotiable** for orchestrators. `mode: "run"` dies mid-pipeline.
- **Never implement code as the orchestrator.** Only orchestrate.
- **Fresh context per child** is a feature — the progress file is the handoff contract.
- **Always push to feature branches** and create PRs — never merge to main locally.
- **Repo-agnostic** — agents discover build commands, test commands, and conventions from the repo itself.

Typical runtime: 15–25 minutes for a medium feature.

## Our Extensions: The 4-Layer Hybrid

1. **Branch-PR-CI** — No direct-to-main commits. All changes go through PRs with CI gates.
2. **TDD-Lite** — Write/run tests before claiming success. Build passing ≠ working.
3. **Scope Locking** — Agents declare which files they'll touch upfront. No collateral damage.
4. **Preview Deployments** — Visual verification on preview URLs before merging to production.

### Architecture Enhancements

In our deployment, `coding-orchestrator` owns the existing main workspace while each `feature-dev-*` worker runs in its own dedicated workspace with its own bootstrap files (`AGENTS.md`, `SOUL.md`, and related workspace context).

Workers do not coordinate directly with each other. The orchestrator is the sole coordinator, and the progress file is the cross-stage handoff contract between stages. This isolation model reduces drift, keeps worker identities role-specific, and makes retries/debugging more deterministic.

## Usage

This skill is installed at `~/.openclaw/skills/coding-pipeline/SKILL.md` and referenced by sub-agents when building features or fixing bugs.

## Projects governed by this pipeline

- [ProtoWeave](https://protoweave.ai) — AI-powered business validation
- [TripTrust](https://triptrust.co) — AI travel safety reports

## Credits

This pipeline was originally based on [openclaw-coding-pipeline](https://github.com/Clintos11/openclaw-coding-pipeline) by [@Clintos11](https://github.com/Clintos11). We extended it with a 4-layer methodology framework (Branch-PR-CI, TDD-Lite, Scope Locking, Preview Deployments) tailored for AI agent-driven development.

## Related

- [openclaw-coding-pipeline](https://github.com/Clintos11/openclaw-coding-pipeline) — Original pipeline this was forked from
- [openclaw-cipher](https://github.com/jarvis-cx/openclaw-cipher) — Cipher agent config/memory
- [OpenClaw](https://openclaw.ai) — The agent platform
