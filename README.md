# Agent Dev Pipeline

A structured software development methodology and coding pipeline for AI agent-driven development.

## Why this exists

AI agents writing production code need **mechanically enforceable guardrails**, not trust-based guidelines. This repo defines the methodology, framework, and pipeline skill that governs how agents build, test, and deploy code across our projects.

## Contents

- `methodology/research.md` — Comparison of 10 software development methodologies evaluated for AI agent workflows
- `methodology/framework.md` — The concrete 4-layer hybrid framework we use
- `SKILL.md` — The coding pipeline skill (OpenClaw-compatible) that agents follow when building features

## The 4-Layer Hybrid

1. **Branch-PR-CI** — No direct-to-main commits. All changes go through PRs with CI gates.
2. **TDD-Lite** — Write/run tests before claiming success. Build passing ≠ working.
3. **Scope Locking** — Agents declare which files they'll touch upfront. No collateral damage.
4. **Preview Deployments** — Visual verification on preview URLs before merging to production.

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
