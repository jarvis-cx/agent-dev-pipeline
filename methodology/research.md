# Development Methodology Research for AI Agent-Driven Workflows

> Research date: 2026-03-18 | Context: Cipher/OpenClaw orchestrating Codex/Claude sub-agents across ProtoWeave & TripTrust

---

## Current Pain Points (for reference)

| Problem | Root Cause |
|---------|-----------|
| Frequent regressions | No automated test gates; agents commit without verification |
| Guess-and-patch debugging | No systematic failure analysis; agents retry blindly |
| Breaking unrelated things | `git add -A`, no scoped file guards, no diff review |
| Direct-to-main commits | No branch/PR workflow enforced |
| Unverified visual changes | No screenshot/snapshot comparison |
| Burned tokens on failed fixes | No fail-fast or attempt limits |

---

## Methodology Analysis

### 1. TDD (Test-Driven Development)

**What:** Write a failing test first → write minimal code to pass → refactor. Red-green-refactor cycle.

**Application to agent workflow:** Before an agent writes any implementation code, it must first write (or be given) a test that defines the expected behaviour. The agent's task is "make this test pass" rather than "implement this feature."

**Pros:**
- Constrains agent scope — clear success criteria (test passes)
- Prevents regressions — existing tests catch breakage immediately
- Reduces guess-and-patch — agent knows exactly what "done" looks like
- Tests become living documentation of intended behaviour

**Cons:**
- Agents write mediocre tests — they test implementation details, not behaviour
- Doubles token cost per task (test + implementation)
- UI/visual work doesn't fit well into unit test paradigm
- Requires existing test infrastructure per project

**Feasibility:** ⭐⭐⭐⭐ HIGH — Agents are actually good at TDD when instructed explicitly. The "make this test pass" framing is ideal for LLM agents.

---

### 2. BDD (Behaviour-Driven Development)

**What:** Describe desired behaviour in natural language (Given/When/Then), then implement to satisfy those specs. Often uses Cucumber/Gherkin syntax.

**Application:** Human writes behaviour specs in plain English; agent implements code to satisfy them. Specs live alongside code.

**Pros:**
- Natural language specs are trivial for LLMs to parse
- Bridges human intent → agent implementation cleanly
- Dave can write specs without touching code
- Reduces ambiguity in task descriptions

**Cons:**
- Gherkin tooling (Cucumber etc.) adds infrastructure overhead
- Over-specification leads to brittle tests
- Step definitions still need maintenance
- Overkill for small fixes/bugs

**Feasibility:** ⭐⭐⭐ MEDIUM — The philosophy is excellent for agent work; the tooling is heavy. Best used as a *mindset* (structured specs) without full Cucumber infrastructure.

---

### 3. Trunk-Based Development vs GitFlow

**Trunk-Based:**
- Everyone commits to `main` (or short-lived branches <1 day)
- Relies on CI gates and feature flags
- Fast, simple, requires strong test coverage

**GitFlow:**
- Long-lived `develop`, `release`, `feature/*` branches
- Formal merge process
- Heavier ceremony, clearer separation

**Application:** Currently doing trunk-based *without the safety nets* (no CI gates, no tests). That's the worst of both worlds.

**Pros of short-lived branches (hybrid):**
- Agent works on `fix/issue-123`, PR back to main
- Isolates agent work from production
- PR diff shows exactly what changed — catches `git add -A` disasters
- Can be automated: agent creates branch → works → opens PR

**Cons:**
- Full GitFlow is too heavy for agent speed
- Branch management adds complexity to agent prompts
- Merge conflicts if multiple agents work simultaneously

**Recommendation:** **Short-lived feature branches + PR** (not full GitFlow). Branch per task, merge via PR with automated checks.

**Feasibility:** ⭐⭐⭐⭐ HIGH — Agents handle `git checkout -b`, commit, push, `gh pr create` trivially.

---

### 4. CI/CD (Continuous Integration / Continuous Deployment)

**What:** Automated build, test, and deployment pipeline triggered on every push/PR.

**Application:** GitHub Actions (or similar) runs on every agent PR:
- Lint check
- Type check
- Unit/integration tests
- Build verification
- (Optional) Deploy preview

**Pros:**
- Catches regressions *before* merge — the single biggest fix for current problems
- Agent doesn't need to verify its own work (pipeline does)
- Objective pass/fail — no "agent claims it works"
- Preview deployments let Dave eyeball UI changes

**Cons:**
- Setup cost (one-time, but real)
- CI minutes cost money (GitHub free tier: 2000 min/month — probably enough)
- Slow feedback loop if tests are heavy

**Feasibility:** ⭐⭐⭐⭐⭐ CRITICAL — This is table stakes. Without CI, every other methodology is unenforceable.

---

### 5. Shift-Left Testing

**What:** Move testing/quality activities earlier in the development process (left on the timeline). Test during design, not after deployment.

**Application:** Agent task prompts include acceptance criteria *upfront*. Agent must demonstrate passing criteria before marking done. Linting/type-checking happens in the agent's local environment before commit.

**Pros:**
- Prevents the "commit first, debug later" pattern
- Aligns with how agent prompts already work (task description = spec)
- Cheap to implement — just add `npm run lint && npm run typecheck` to agent workflow

**Cons:**
- Only as good as the acceptance criteria provided
- Agents may game criteria (make test pass without solving the real problem)

**Feasibility:** ⭐⭐⭐⭐ HIGH — Mostly a prompt engineering + workflow design problem. Add verification steps to agent task templates.

---

### 6. Design by Contract

**What:** Functions specify preconditions, postconditions, and invariants. Runtime or compile-time verification.

**Application:** TypeScript interfaces, Zod schemas, runtime assertions. Agent-written code must satisfy type contracts. API boundaries enforce shape.

**Pros:**
- TypeScript already provides this partially
- Prevents agents from returning wrong shapes
- Self-documenting code
- Catches integration errors early

**Cons:**
- Runtime contracts add overhead
- Agents need to understand existing contracts (context window cost)
- Doesn't help with UI/visual issues

**Feasibility:** ⭐⭐⭐ MEDIUM — TypeScript strict mode + Zod schemas are practical. Full DbC (Eiffel-style) is overkill.

---

### 7. Formal Code Review (PR-Based)

**What:** All changes go through pull request review before merge.

**Application:** Two layers possible:
1. **Agent-reviews-agent:** Orchestrator spawns a review agent to check the implementation agent's PR
2. **Human-reviews-agent:** Dave reviews PR diffs (assisted by AI summary)
3. **Automated review:** CI checks + linting + type checking as mandatory PR gates

**Pros:**
- Catches unrelated file changes (the favicon problem)
- `git diff` in PR shows exactly what's changing
- Can be partially automated (AI reviewer flags suspicious changes)
- Mandatory checks prevent broken code from reaching main

**Cons:**
- Human review is a bottleneck (Dave's time is scarce)
- Agent-reviewing-agent may miss the same things
- Adds latency to every change

**Recommendation:** Automated PR gates (CI must pass) + AI-assisted review summary + human review only for significant changes.

**Feasibility:** ⭐⭐⭐⭐ HIGH — GitHub branch protection rules enforce this trivially.

---

### 8. Canary / Staged Deployments

**What:** Deploy to a subset of infrastructure first, monitor, then roll out fully. Or: staging → production pipeline.

**Application:** Deploy agent changes to a preview/staging URL first. Verify (automated or manual) before promoting to production.

**Pros:**
- Catches production-only issues
- Gives Dave a chance to eyeball before users see it
- Rollback is trivial (don't promote)

**Cons:**
- Requires staging infrastructure (cost + maintenance)
- For two small SaaS products, may be over-engineered
- Preview deployments (Vercel/Netlify) may be sufficient substitute

**Feasibility:** ⭐⭐⭐ MEDIUM — Vercel preview deployments on PRs are free and effectively give you this. Full canary is unnecessary at current scale.

---

### 9. Additional: Scope-Locked Agent Tasks (Agent-Specific)

**What:** A methodology specific to AI agent development — each agent task explicitly declares which files/directories it may touch, and the orchestrator verifies compliance post-execution.

**Application:**
- Task prompt includes: "You may ONLY modify files in `src/components/Dashboard/`"
- Post-task: orchestrator runs `git diff --name-only` and rejects if out-of-scope files changed
- Enforced `git add <specific files>` (already in AGENTS.md)

**Pros:**
- Directly addresses the "breaking unrelated things" problem
- Simple to implement — prompt constraint + diff check
- No infrastructure needed

**Cons:**
- May be too restrictive for cross-cutting changes
- Requires good task decomposition upfront

**Feasibility:** ⭐⭐⭐⭐⭐ VERY HIGH — This is prompt engineering + a 3-line shell check.

---

### 10. Additional: Attempt-Budgeted Development (Agent-Specific)

**What:** Each agent task has a maximum retry budget. After N failed attempts, the agent must stop and report rather than continuing to guess-and-patch.

**Application:** Already partially in AGENTS.md (anti-loop rules). Formalize: max 2 attempts → escalate to human with diagnostic info.

**Pros:**
- Directly addresses token waste on failed fixes
- Forces agents to provide useful diagnostic info instead of blind retries
- Human can redirect approach rather than watching agents spin

**Cons:**
- Some legitimate fixes need iteration
- May be too conservative for complex bugs

**Feasibility:** ⭐⭐⭐⭐⭐ VERY HIGH — Prompt-level enforcement.

---

## Comparison Table

| Methodology | Regression Prevention | Scope Control | Token Efficiency | Setup Cost | Agent Feasibility |
|---|---|---|---|---|---|
| TDD | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| BDD | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| Short-lived branches + PR | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| CI/CD | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Shift-Left Testing | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Design by Contract | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| PR-Based Review | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Canary/Staged Deploy | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| Scope-Locked Tasks | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Attempt-Budgeted Dev | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

---

## Recommendation: Agent-Optimized TDD + Branch-PR-CI Hybrid

### The Stack (layered, each builds on the previous)

**Layer 1 — Branch Isolation + CI Gates (foundational)**
- Every agent task → new branch
- CI runs lint + typecheck + tests on push
- PR required to merge; CI must pass
- Catches: regressions, broken builds, unrelated file changes

**Layer 2 — TDD-Lite for Agent Tasks**
- Agent receives failing test (or writes one first) as part of task
- Success = test passes + CI green
- Not full strict TDD — pragmatic "test-first when possible, test-alongside always"
- Catches: vague success claims, guess-and-patch cycles

**Layer 3 — Scope Locking + Attempt Budgets (agent-specific)**
- Task prompts declare allowed file scope
- Post-diff validation rejects out-of-scope changes
- Max 2 retry attempts → escalate with diagnostics
- Catches: favicon deletions, CSS conflicts, token waste

**Layer 4 — Preview Deployments (visual verification)**
- Vercel/Netlify preview URLs on PRs
- UI changes get visual check before merge
- Catches: visual regressions, layout breaks

### Why This Fits Your Situation

| Factor | How the hybrid addresses it |
|--------|---------------------------|
| **AI agents, not humans, write code** | TDD gives agents concrete pass/fail criteria instead of vague specs. Scope locking prevents wandering. |
| **Small team (Dave + agents)** | CI automates what Dave can't review manually. PR gates enforce quality without human bottleneck. |
| **Two production products** | Same pipeline template for both. Shared CI config, shared agent task templates. |
| **Telegram-based coordination** | Agent reports: "PR #47 ready, CI green, preview: [url]" — Dave clicks link, approves or rejects. |
| **Token/cost sensitivity** | Attempt budgets prevent runaway retries. TDD reduces guess-and-patch. Scope locking prevents cascading fixes. |

### What It Looks Like in Practice

**Example: "Fix the dashboard chart not rendering"**

```
1. Orchestrator creates branch: fix/dashboard-chart-rendering
2. Agent receives task with:
   - Failing test: "Dashboard chart renders with mock data"
   - Scope: src/components/Dashboard/, src/lib/charts/
   - Budget: 2 attempts
3. Agent writes fix, runs test locally, commits specific files
4. CI runs: lint ✅ typecheck ✅ tests ✅ build ✅
5. Preview URL generated
6. Orchestrator posts to Telegram: "PR #47 ready — preview: https://preview-47.vercel.app"
7. Dave clicks preview, sees chart working, approves
8. Merge to main → auto-deploy
```

**Example: "Agent fails after 2 attempts"**

```
1. Agent attempt 1: changes X → test still fails
2. Agent attempt 2: changes Y → test still fails
3. Agent STOPS, reports to orchestrator:
   - What was tried
   - Error output
   - Suggested next investigation
4. Orchestrator escalates to Dave via Telegram
5. Dave provides guidance or reassigns with different approach
```

---

## What NOT to Adopt

| Methodology | Why Skip |
|---|---|
| Full GitFlow | Too heavy. Feature branches + PR is enough. |
| Full BDD/Cucumber | Infrastructure overhead not justified. Use the mindset (structured specs) without the tooling. |
| Full Design by Contract | TypeScript strict mode covers 80%. Zod for API boundaries. No need for formal DbC. |
| Full canary deployments | Preview deployments cover visual verification. Canary infra is overkill at current scale. |

---

## Summary

The core insight: **AI agents need guardrails, not guidelines.** Humans can be told "be careful" — agents need automated enforcement. The recommended hybrid prioritizes *mechanically enforceable* practices (CI gates, branch protection, scope validation, attempt limits) over *trust-based* ones (code review, careful commits, thorough testing).

Every layer addresses a specific current pain point:
- Regressions → CI + tests
- Unrelated breakage → scope locking + PR diffs
- Token waste → attempt budgets
- Unverified UI → preview deployments
- Direct-to-main → branch protection

The methodology is deliberately pragmatic — no heavy tooling, no new infrastructure beyond CI pipelines and branch rules, and fully compatible with the existing Telegram-based orchestration model.
