---
name: coding-pipeline
description: "Orchestrated coding pipeline for feature builds. Spawns a 7-stage sub-agent pipeline (Plan → Setup → Implement → Verify → Test → PR → Review) with dedicated agent personas and retry loops. Use when: building features, fixing bugs, or refactoring code across multiple files. NOT for: simple one-liner fixes, reading code, or non-coding tasks."
---

# Coding Pipeline

Multi-stage sub-agent pipeline for building features on any repo. The orchestrator spawns one child agent per stage, yields, and receives results via auto-announce.

## Methodology

This pipeline enforces a **4-layer hybrid framework** (see `methodology/framework.md`):

1. **Branch-PR-CI** — No direct-to-main. All changes via feature branches + PRs with CI gates.
2. **TDD-Lite** — Reproduce before fixing. Verify before claiming success. Visual checks for UI.
3. **Scope Locking** — Declare files upfront. `git add <specific files>` only. Never `git add -A`.
4. **Preview Deployments** — Verify Vercel preview URL visually for UI changes before merging.

**Attempt budget:** Max 2 retries per fix. If 2 retries fail → stop and report. Do not keep guessing.

## Prerequisites

1. **Spec required.** Write it to `specs/<feature-name>.md` and get approval before spawning.
2. **Main branch clean.** `git checkout main && git pull --rebase origin main`
3. **GitHub CLI authenticated.** `gh auth status` must pass (for PR creation).

## Pipeline Overview

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

## Stages

### 1. PLAN (Reasoning model)
**Agent:** `feature-dev-planner`
- Explore codebase, read spec
- Decompose into ordered, dependency-aware user stories (max 20)
- Order: schema/DB → backend → frontend → integration
- Every story: mechanically verifiable acceptance criteria + test criteria
- Write plan to progress file

### 2. SETUP (Coding model)
**Agent:** `feature-dev-setup`
- Create feature branch from main
- Discover build/test commands
- Ensure .gitignore covers sensitive files
- Run build + tests to establish baseline
- Write baseline state to progress file

### 3. IMPLEMENT (Coding model)
**Agent:** `feature-dev-developer`
- Iterate over stories from plan (fresh session per story)
- Implement completely — no TODOs, no placeholders
- Write tests for each feature
- Commit atomically per logical unit
- **Scope lock:** Only modify files declared in the plan. If unexpected files appear in `git diff --stat`, unstage them.
- **Git safety:** Always `git add <specific files>`. NEVER `git add -A` or `git add .`
- **Protected files:** Do not modify favicon, icons, config files, or unrelated components without explicit approval.
- Update progress file

### 4. VERIFY (Reasoning model)
**Agent:** `feature-dev-verifier`
- Inspect actual diff: `git diff main..HEAD`
- **Scope check:** Verify only declared files were modified. Flag any unexpected changes.
- Security: no leaked secrets, .gitignore correct
- Run full test suite — must pass
- Check acceptance criteria against actual code
- TypeScript/build check must pass
- **For UI changes:** Verify Vercel preview URL visually (screenshot or browser check). Build passing is NOT sufficient.
- **For API changes:** Test endpoint with real data (curl or script).
- **On FAIL:** kick back to IMPLEMENT (max 2 retries, then STOP and report)

### 5. TEST (Coding model)
**Agent:** `feature-dev-tester`
- Full test suite for integration validation
- Cross-cutting concerns: error handling, edge cases
- Classify failures as pre-existing vs branch-introduced
- **On FAIL (branch-introduced):** kick back to IMPLEMENT (max 1 retry)

### 6. PR (Coding model)
**Agent:** `feature-dev-developer`
- `git push -u origin <branch>`
- `gh pr create` with structured title + body
- Update progress file with PR URL

### 7. REVIEW (Reasoning model)
**Agent:** `feature-dev-reviewer`
- Security, performance, backward compatibility
- Test coverage, spec alignment
- **On CHANGES_REQUESTED:** kick back to IMPLEMENT (max 1 retry)

## Model Assignments

| Stage | Model Type | Agent ID |
|-------|-----------|----------|
| Orchestrator | Reasoning | `coding-orchestrator` |
| Plan | Reasoning | `feature-dev-planner` |
| Setup | Coding | `feature-dev-setup` |
| Implement | Coding | `feature-dev-developer` |
| Verify | Reasoning | `feature-dev-verifier` |
| Test | Coding | `feature-dev-tester` |
| PR | Coding | `feature-dev-developer` |
| Review | Reasoning | `feature-dev-reviewer` |

Models are configured per-agent in `openclaw.json` — no need to pass `model` in `sessions_spawn`.

## Execution Model

The orchestrator runs as a **persistent session** (`mode: "session"`). It spawns ONE stage at a time, yields, and wakes when the child auto-announces back.

**Rules:**
- NEVER sleep between stages
- NEVER poll subagents in a loop
- NEVER spawn multiple stages in one turn
- NEVER implement code yourself — only orchestrate
- ONE stage per turn → yield → receive result → next stage
- Worker agents do not coordinate with each other directly
- The orchestrator is the sole coordinator across stages and retries
- The progress file is the only handoff contract between worker stages

## Progress File

Durable state contract between stages at `<repo>/progress-<branch-name>.md`.

Every child sub-agent prompt MUST include:
> "Read progress file at `<repo>/progress-<branch-name>.md` for prior stage context."

## Recovery

If the orchestrator stalls, send: **"Resume from progress file."**

It reads the progress file, finds the next pending stage, and continues.

## Spawn Pattern

See `templates/orchestrator-prompt.md` for the full template with placeholders.

```javascript
sessions_spawn({
  agentId: "coding-orchestrator",
  label: "<feature-name>",
  mode: "session",
  task: `<filled orchestrator template>`
})
```

## Config Requirements

In `openclaw.json`:
```jsonc
{
  "agents": {
    "defaults": {
      "subagents": {
        "maxSpawnDepth": 2,
        "maxChildrenPerAgent": 5
      }
    }
  }
}
```

## PR Discipline

- Always feature branch — never commit to main directly
- Always `gh pr create` — never merge locally
- Branch naming: `<type>/<short-description>` (e.g., `feature/user-dashboard`, `fix/auth-crash`)

## Attempt Budget

- Maximum **2 retries** per stage failure
- If 2 retries fail at any stage → **STOP the pipeline and report**
- Do not keep burning tokens on hope-driven patches
- After 2 failures: escalate to orchestrator with findings, write a proper investigation spec

## Verification Requirements

| Change Type | Build? | Test? | Visual? | Endpoint? |
|---|---|---|---|---|
| New feature (UI) | ✅ | ✅ | ✅ Preview URL | ❌ |
| New feature (API) | ✅ | ✅ | ❌ | ✅ curl test |
| Bug fix (UI) | ✅ | ✅ | ✅ Preview URL | ❌ |
| Bug fix (API) | ✅ | ✅ | ❌ | ✅ curl test |
| Refactor | ✅ | ✅ | ❌ | ❌ |

"Build passes" is necessary but NEVER sufficient for UI changes.
