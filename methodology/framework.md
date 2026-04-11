# Agent Development Framework

The concrete 4-layer hybrid methodology for AI agent-driven development.

---

## Layer 1: Branch-PR-CI

### Rules
- **No direct commits to `main`** for non-trivial changes
- Every feature/fix gets a branch: `feature/<name>` or `fix/<name>`
- PR required before merging to main
- CI must pass on the PR (build + lint + type check)
- Vercel/preview deployment auto-generated on PR push

### When to skip (small changes only)
- Single-line config changes
- Documentation-only updates
- Memory/TODO file updates
- These can go direct to main

### Agent enforcement
- Sub-agent task prompts must include: "Create a feature branch. Do not push to main."
- Orchestrator merges to main only after verification

---

## Layer 2: TDD-Lite

### Rules
- Before implementing a fix: **reproduce the bug first**
- Before claiming success: **verify the fix works** (not just "build passes")
- For UI changes: **visual verification required** (browser screenshot or manual check)
- For API changes: **curl/test the endpoint** with real data

### What "TDD-Lite" means in practice
- Not full test suites for everything
- But: if you're fixing a bug, prove it's fixed
- If you're adding a feature, prove it works
- If you're changing UI, look at it

### Agent enforcement
- Sub-agent tasks must include verification steps
- "Build passes" is necessary but NOT sufficient
- Report must include evidence of verification (endpoint response, visual check, etc.)

---

## Layer 3: Scope Locking

### Rules
- Before implementing, agent declares which files it will modify
- After implementing, `git diff --stat` must match the declared scope
- Unexpected file changes → unstage and investigate
- **Never use `git add -A` or `git add .`**
- Always `git add <specific files>`

### Protected files
These files must never be modified without explicit approval:
- `favicon.ico`, `icon.svg`, `icon.png` (any favicon/icon files)
- `package.json` (unless installing a declared dependency)
- `.env*` files
- `tailwind.config.*`
- `next.config.*`
- `tsconfig.json`

### Agent enforcement
- Pre-commit hook blocks:
  - File deletions (requires `--no-verify` override)
  - Staging 20+ files (likely accidental `git add -A`)
- Sub-agent prompts must include: "Only modify files directly related to this task."

---

## Layer 4: Preview Deployments

### Rules
- For UI/frontend changes: push to feature branch → get Vercel preview URL
- Verify on preview URL before merging
- For backend-only changes: verify via API endpoint test
- Never claim a UI fix works without visual evidence

### Workflow
1. Push to feature branch
2. Vercel generates preview URL automatically
3. Check the preview URL (browser screenshot or manual visit)
4. If good → merge PR to main
5. If bad → fix on branch, re-verify

### Agent enforcement
- Sub-agent tasks for UI work must include: "After pushing, verify the Vercel preview URL visually."
- Orchestrator checks preview before approving merge

---

## Attempt Budget

### Rules
- Maximum **2 retries** per fix attempt
- If 2 retries fail → **stop and report**
- Do not keep burning tokens on hope-driven patches
- After 2 failures: write a proper spec, investigate root cause, then try again with fresh context

### Agent enforcement
- Sub-agent prompts must include: "If this fails twice, stop and report the failure. Do not retry further."

---

## Decision Matrix: When to use what

| Change Type | Branch? | PR? | Preview? | Tests? |
|---|---|---|---|---|
| New feature | ✅ | ✅ | ✅ (if UI) | ✅ |
| Bug fix (backend) | ✅ | ✅ | ❌ | ✅ (verify fix) |
| Bug fix (frontend/UI) | ✅ | ✅ | ✅ | ✅ (visual) |
| Config change | ❌ | ❌ | ❌ | ❌ |
| Documentation | ❌ | ❌ | ❌ | ❌ |
| Memory/TODO update | ❌ | ❌ | ❌ | ❌ |
| Database migration | ✅ | ✅ | ❌ | ✅ (verify schema) |
| Hotfix (production down) | Direct to main | ❌ | ❌ | ✅ (verify fix) |

---

## Layer 5: Deterministic Production Control

### Hard rule
- **Never use non-deterministic production design/flows again**
- LLMs may generate content, analysis, classifications, and structured outputs
- LLMs must NOT be the sole controller of queue polling, claiming, retries, recovery, finalization, publish steps, or exactly-once workflow behavior

### Rules
- Production workflow state transitions must be deterministic and code-controlled
- Queue polling, locks, stale recovery, retries, and completion writes belong in deterministic code/scripts/services
- LLMs belong inside a deterministic envelope, not as the workflow engine itself
- Every production workflow must fail closed and expose bounded recovery behavior
- One live execution path only, no conflicting crons/prompts/scripts

### Agent enforcement
- If a proposed production design puts workflow control inside prompts or agent discretion, reject it and redesign the orchestration layer deterministically
- Specs must identify which parts are deterministic control logic vs LLM generation logic
- Any stateful production workflow must document lock/claim semantics, retry bounds, and terminal failure behavior before implementation

## Summary

The framework is designed to be **lightweight enough that agents can follow it** but **strict enough to prevent the regressions** we've been experiencing. It's not bureaucracy — it's guardrails.

Core principle: **Mechanically enforceable > trust-based**.
