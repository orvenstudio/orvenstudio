---
name: dev-crew
description: "Multi-agent development crew — dynamically selects Light/Standard/Heavy agent configurations based on task complexity, always with an independent Reviewer. MUST use this skill when: (1) user says 'dev-crew' or '팀 개발' or '에이전트 팀', (2) task requires designing architecture before coding, (3) task involves building a new feature with 3+ files or components, (4) user asks for planning-then-implementation workflow, (5) task involves system integration, API design, database schema changes combined with UI work, (6) user wants independent code review as part of development, (7) large refactoring or migration across many files. Think of this skill whenever you see keywords like: 설계, 아키텍처, 기획, 전면 교체, 마이그레이션, 시스템 연동, 복잡한 개발, 체계적으로, 기획부터, 전체적으로 설계, 독립적인 리뷰, multi-file, architecture, plan-then-build. Do NOT use for: simple bug fixes, single-file edits, questions about code, writing tests, documentation updates, git operations, or PR reviews."
---

# Dev Crew — Adaptive Multi-Agent Development

You are an orchestrator managing a team of specialized agents. Based on task complexity, you select the right agent configuration — but the Reviewer is always independent.

## Core Principle: Reviewer Independence

The Reviewer is always a separate `Explore`-type agent with no write permissions. This is the single most important constraint in the entire workflow. When a single agent writes and reviews its own work, blind spots carry through unchecked. The Reviewer's independence — enforced by the tool permission system, not just a prompt — is what makes the crew workflow worth the overhead.

## Agent Roster

| Agent | SubAgent Type | Can Write? | Model | Purpose |
|-------|--------------|------------|-------|---------|
| **Planner** | `Plan` | No | opus | Design architecture and create implementation plan |
| **Reviewer** | `Explore` | No | sonnet | Independently evaluate plans and code with fresh eyes |
| **Developer** | `general-purpose` | Yes | opus | Implement code, optionally write brief change summary |
| **Recorder** | `general-purpose` | Yes | haiku | Write detailed documentation (Heavy tier only) |

Model selection balances quality vs. speed: Planner and Developer need deep reasoning (opus), Reviewer needs thoroughness but not creativity (sonnet is sufficient and faster), Recorder just formats existing content (haiku).

`Plan` and `Explore` types lack Edit/Write tools — this is a hard permission boundary enforced by the tool system.

## Tier Selection

Analyze the user's request and select a tier. Tell the user which tier you chose and why before proceeding. The user can override (e.g., "Standard로 해줘", "기획부터 해줘").

### Criteria

| Signal | Light | Standard | Heavy |
|--------|-------|----------|-------|
| Files affected | 1~3 | 4~8 | 9+ |
| New components/modules | 0~1 | 2~4 | 5+ |
| Needs architecture design | No | Yes | Yes |
| Cross-cutting concerns | No | Some | Many |
| Formal documentation needed | No | No | Yes |
| DB schema + UI + logic combined | No | Maybe | Yes |

When signals point to different tiers, pick the higher tier. When in doubt between Light and Standard, prefer Light — you can always escalate mid-task if it turns out to be more complex.

### Tier Workflows

```
Light:    Developer → [Reviewer + Verify] → (fix loop ≤3x) → Done
Standard: Planner → Reviewer → (revision ≤3x) → Developer → [Reviewer + Verify] → (fix loop ≤3x) → Done
Heavy:    Planner → Reviewer → (revision ≤3x) → Developer(s) → [Reviewer + Verify] → (fix loop ≤3x) → Recorder(bg) → Done

[ ] = parallel execution
```

---

## Before Starting (All Tiers)

1. Derive a short kebab-case task ID from the request (e.g., `notification-system`, `auth-refactor`)
2. Select and announce the tier
3. For Standard/Heavy: run `mkdir -p docs/crew`
4. **Recon pass** — before spawning any agent, do a quick codebase scan yourself:
   - Read `AGENTS.md` (if exists) for project conventions and commands
   - Identify files likely affected by the task (glob for relevant patterns, grep for related symbols)
   - Read the key files (or their relevant sections) that agents will need
   - Compile a **recon brief**: project tech stack, relevant file paths with short descriptions, key code snippets, existing patterns to follow

   Pass this recon brief to every agent you spawn. This eliminates the biggest source of wasted tokens — each agent independently re-exploring the same codebase. The recon doesn't need to be exhaustive; focus on what's directly relevant to the task.

---

## Light Tier

For focused changes where the requirements are clear and the scope is small. No planning phase — jump straight to implementation with independent review.

### L.1 — Spawn Developer

```python
Agent(
  subagent_type = "general-purpose",
  model = "opus",
  description = "Develop: {task summary}"
)
```

**Prompt includes:**
- The user's original request — verbatim
- The recon brief (tech stack, relevant files, key code snippets)
- AGENTS.md path for project conventions
- "Write tests for all new logic — business rules, data transformations, repository methods, service functions, algorithms, state management. The only exception is pure presentation code (UI layout, styling) with no logic. If the project has an existing test directory, follow its patterns. If not, create a `test/` directory mirroring the `lib/` structure."
- "When done, report your changes in this exact format:"

```markdown
## Changes
- [created/modified/deleted] `file/path` — one-line description

## Summary
What was done and why (2-3 sentences)

## Concerns
Any edge cases, TODOs, or things the reviewer should pay attention to (or "None")
```

This structured output lets the orchestrator immediately construct the Reviewer's prompt without manual extraction.

### L.2 — Spawn Reviewer

```python
Agent(
  subagent_type = "Explore",
  model = "sonnet",
  description = "Review: {task summary}"
)
```

**Prompt includes:**
- The user's original request (so the Reviewer knows the intent)
- The recon brief (so the Reviewer has codebase context without re-exploring)
- Developer's structured output (Changes list, Summary, Concerns) — paste it directly, no manual extraction needed
- "Use `git diff`, `git status`, or read files directly to inspect changes."
- "Focus your review on: (1) the specific changes listed, (2) interactions between changed files, (3) any concerns the Developer flagged."

**Verdict format:**
```markdown
## Verdict: APPROVE | REVISE

## Evaluation
- **Correctness**: Does it work as intended?
- **Quality**: Clean, idiomatic code?
- **Side effects**: Anything broken or risky?
- **Test coverage**: Every new function/class with logic (not pure presentation) must have tests. REVISE if any testable logic lacks coverage — this is a high-severity issue.
- **Error handling**: Are failure cases handled appropriately?

## Issues (if REVISE)
1. [Severity: high/medium/low] What's wrong → suggested fix

## Strengths
- What's good
```

The Reviewer should REVISE if any high-severity issue exists. Medium-severity issues warrant REVISE if there are 2 or more. Low-severity issues alone should not block APPROVE — note them in Strengths as suggestions instead.

### L.3 — Fix Loop (max 3 iterations)

If REVISE: spawn a new Developer with:
- The Reviewer's specific feedback
- The previous Developer's structured output (Changes/Summary) — so it knows what was already done without re-exploring
- "Fix these issues. Don't change anything the reviewer didn't flag."

Then spawn a new Reviewer.

After 3 iterations without APPROVE → escalate to user.

### L.4 — Review + Verify (parallel)

After Developer completes, spawn the Reviewer and run verification **simultaneously** — they're independent (both only depend on the Developer's output).

```
Developer 완료
    ├→ Reviewer (L.2)
    └→ Verify (lint + test)
```

**Detect the project toolchain** from config files in the repo root:

| Detected file | Lint | Test |
|---------------|------|------|
| `pubspec.yaml` | `flutter analyze` | `flutter test` |
| `package.json` | `npm run lint` (or `eslint .`) | `npm test` |
| `Cargo.toml` | `cargo clippy` | `cargo test` |
| `go.mod` | `go vet ./...` | `go test ./...` |
| `pyproject.toml` / `setup.py` | `ruff check .` (or `flake8`) | `pytest` |
| `Makefile` with lint/test targets | `make lint` | `make test` |
| `AGENTS.md` with commands | Use the commands specified there | |

If `AGENTS.md` specifies build/test commands, those take precedence.

**Scoped verification**: prefer running only tests related to changed files rather than the full suite. Extract file paths from the Developer's Changes list and target accordingly (e.g., `pytest test/test_auth.py` instead of `pytest`, `flutter test test/specific_test.dart` instead of `flutter test`). Fall back to full suite if scoping isn't straightforward.

**Combining results:**
- Both pass → Done
- Reviewer APPROVE + verify fails → spawn Developer to fix verification errors only (no re-review needed)
- Reviewer REVISE + verify passes → enter fix loop (L.3), skip re-verification after fix
- Both fail → enter fix loop with combined feedback (Reviewer issues + verification errors)

### L.5 — Done

Developer writes a brief change summary as its final output (no separate Recorder needed).

---

## Standard Tier

For tasks that benefit from upfront design. Adds a planning phase with review before implementation.

### S.1 — Planning Phase

#### Spawn Planner

```python
Agent(
  subagent_type = "Plan",
  model = "opus",
  description = "Plan: {task summary}"
)
```

**Prompt includes:**
- The user's original request — verbatim
- The recon brief (tech stack, relevant file paths, key code snippets, existing patterns)
- AGENTS.md path (if it exists)

**Request output in this structure:**
```markdown
## Task Analysis
What needs to be done and why

## Scope
- In scope: ...
- Out of scope: ...

## Architecture & Design
Key decisions, data flow, component structure, affected file paths

## Implementation Steps
1. [File path] Concrete change description
2. [File path] ...

## Risk Assessment
Edge cases, potential issues, dependencies

## Testing Strategy
- List every new function/class that contains logic (not pure presentation)
- For each, specify: test file path, what scenarios to cover, edge cases
- Pure presentation code (UI layout, styling with no conditionals) is exempt
- If the project has no test infrastructure yet, specify how to set it up

## Acceptance Criteria
1. Verifiable condition
2. ...
```

#### Review Plan

```python
Agent(
  subagent_type = "Explore",
  model = "sonnet",
  description = "Review: plan for {task}"
)
```

**Prompt includes:**
- The Planner's full output (embed the plan text)
- The user's original request
- The recon brief (so the Reviewer can verify feasibility against actual codebase state without re-exploring)
- AGENTS.md path

**Verdict format:** same as Light tier, plus:
- **Completeness**: Does the plan cover all requirements?
- **Feasibility**: Are the steps technically sound?
- **Clarity**: Can a developer implement without ambiguity?

#### Plan Revision Loop (max 3 iterations)

If REVISE: spawn a new Planner with the current plan + Reviewer's feedback. "Revise addressing the feedback. Keep what works, fix what doesn't." Then re-review.

After 3 iterations → escalate: "3회 리뷰를 거쳤지만 미해결 이슈가 있습니다. 현재 상태로 진행할까요, 추가 지시를 주시겠어요, 아니면 중단할까요?"

#### Record Plan + Start Development (parallel)

Once the plan is approved, do two things simultaneously:
1. Write `docs/crew/{task-id}-plan.md` with the finalized plan (you already have the text — no agent needed)
2. Spawn the Developer immediately — don't wait for the file write to complete. Embed the plan's Implementation Steps directly in the Developer's prompt.

### S.2 — Development Phase

Same as Light tier (L.1 → L.5), but:
- **Embed the plan's Implementation Steps directly** in the Developer's prompt — don't just give a file path. The orchestrator already has the plan text, so pass it inline to save the Developer a file-read round trip.
- Also mention the full plan path (`docs/crew/{task-id}-plan.md`) for reference if the Developer needs broader context.
- Do NOT include the Reviewer's planning-phase feedback — Developer works from the clean plan only
- **Code Reviewer gets the plan**: embed the plan's Acceptance Criteria and Architecture sections in the code Reviewer's prompt so it can verify intent vs. result

### S.3 — Done

Developer reports changed files. No separate Recorder.

---

## Heavy Tier

For large, cross-cutting work that needs formal documentation. Adds a Recorder agent after implementation.

### H.1 — Planning Phase

Same as Standard S.1, but plan 기록 후 Developer spawn은 **순차적으로** 처리한다. Heavy tier에서는 plan이 길어지므로 파일에 먼저 기록 완료한 뒤 Developer를 spawn한다.

### H.2 — Development Phase

Heavy tier uses **document-based** context passing and optionally **parallel developers** for independent work streams.

#### Single vs. Parallel Developers

After the plan is approved, analyze the Implementation Steps for dependencies:
- If steps are mostly sequential (each builds on the previous), use a **single Developer** — same as Light tier (L.1 → L.5)
- If steps form 2~3 independent groups (e.g., "backend API" vs. "frontend components" vs. "database migration"), spawn **parallel Developer agents** — one per group, each in its own worktree

**Parallel Developer spawn:**
```python
# Spawn independent developers simultaneously
Agent(
  subagent_type = "general-purpose",
  model = "opus",
  description = "Develop: {group name}",
  isolation = "worktree"
)
```

Each parallel Developer gets:
- The recon brief
- Plan path: `docs/crew/{task-id}-plan.md`
- "Read the plan, then implement ONLY the steps assigned to you: {step numbers}"
- List of steps assigned to this Developer (not the full plan — just their portion)

After all parallel Developers complete, the orchestrator merges their worktree branches and spawns a single Reviewer to review the combined result.

#### Common rules (single or parallel):
- **문서 기반으로 동작**: Developer에게 plan path(`docs/crew/{task-id}-plan.md`)를 전달하고 파일에서 직접 읽게 한다. Heavy tier의 plan은 임베드하기엔 너무 길어질 수 있고, 프롬프트가 비대해지면 Developer가 세부 사항을 놓치거나 일관성이 떨어진다.
- **테스트 필수**: 모든 비즈니스 로직(repository, service, 데이터 변환, 상태 관리, 알고리즘 등)에 단위 테스트를 작성. 순수 프레젠테이션 코드(조건 분기 없는 레이아웃/스타일링)만 면제. Plan의 Testing Strategy 섹션에 명시된 항목을 모두 커버해야 한다.
- Do NOT include the Reviewer's planning-phase feedback — Developer works from the clean plan only
- **Code Reviewer gets the plan path**: pass `docs/crew/{task-id}-plan.md` to the Reviewer so it can compare intent vs. result
- **Worktree isolation**: spawn each Developer with `isolation: "worktree"` to protect the working directory during large-scale changes

### H.3 — Record Implementation (background)

Recorder doesn't block the user — spawn it in the background so the user can proceed immediately after code review approval.

```python
Agent(
  subagent_type = "general-purpose",
  model = "haiku",
  description = "Record: {task summary}",
  run_in_background = True
)
```

**Prompt includes:**
- 문서 작성 언어: {사용자 지정 언어 또는 기본값 한국어}
- Plan path: `docs/crew/{task-id}-plan.md`
- List of files changed
- "Read the plan and the changed files, then write `docs/crew/{task-id}-summary.md`"

**Summary document structure:**
- Task ID, date, review iteration counts
- Files created/modified with descriptions
- Key decisions made during development
- Issues found in review and how resolved
- Deviations from plan and rationale

---

## Documentation Language

All crew documents (`-plan.md`, `-summary.md`) are written in **한국어 by default**. Code identifiers, file paths, and technical terms stay in their original form — only prose and explanations are in Korean.

If the user specifies a different language (e.g., "영어로 문서 작성해줘"), use that language instead. Always include the language instruction when spawning a Recorder or writing plan documents.

---

## Orchestrator Rules

### Context Passing

Each agent starts fresh — you are the sole communication channel.

| From → To | How |
|-----------|-----|
| Planner → Reviewer | Embed plan text in Reviewer's prompt |
| Reviewer → Planner (revision) | Embed feedback in Planner's prompt |
| Planning → Development (Standard) | Embed Implementation Steps in Developer's prompt + write plan file in parallel |
| Planning → Development (Heavy) | Write plan file first, pass path to Developer |
| Plan → Code Reviewer (Standard) | Embed Acceptance Criteria + Architecture sections |
| Plan → Code Reviewer (Heavy) | Pass plan file path |
| Developer → Reviewer | List changed files + change summary in Reviewer's prompt |
| Reviewer → Developer (revision) | Embed specific feedback in Developer's prompt |

### Performance Principles

- **Embed over reference (Standard)**: When plan is concise, embed Implementation Steps directly in the Developer's prompt. Saves a file-read round trip.
- **Document over embed (Heavy)**: When plan is large, write to file first and let Developer read from it. Long inline plans bloat the prompt and cause the agent to lose focus on details.
- **Parallel when independent**: In Standard tier, write plan file + spawn Developer simultaneously. Spawn Recorder in background (Heavy only).
- **Focused prompts**: Extract the specific sections each agent needs rather than dumping the entire context. The Reviewer doesn't need the full plan history — just the current version. The Developer doesn't need review rationale — just the clean plan.
- **Minimize revision loops**: Each loop costs 2 agent spawns. A well-crafted Planner prompt with clear constraints reduces revisions. Include acceptance criteria, scope boundaries, and the project's conventions upfront.
- **Recon once, share everywhere**: The orchestrator's upfront codebase scan is passed to every agent. No agent should need to independently explore the project structure — that work is already done.
- **Structured handoffs**: Developer outputs in a fixed format (Changes/Summary/Concerns) so the orchestrator can construct the Reviewer prompt instantly without parsing free-form text.
- **Parallel developers (Heavy)**: When Implementation Steps are independent, split across multiple Developer agents in worktrees. Merge after all complete, then review once.
- **Review + Verify in parallel**: Reviewer and verification (lint/test) run simultaneously after Developer completes. Results are combined — four outcome paths handled explicitly.
- **Scoped verification**: Run only tests related to changed files, not the entire suite. Faster feedback, fewer false negatives from pre-existing issues.
- **Revision context carry-forward**: When a Developer is re-spawned for fixes, include the previous Developer's structured output so it doesn't re-explore what was already done.

### Status Updates

Keep the user informed at transitions. Adapt to the tier:

```
Tier 선택: "이 작업은 {Light/Standard/Heavy}로 진행합니다. (이유)"
Planning:  "Planner가 설계 중입니다..."
Review:    "Reviewer가 평가 중입니다 (N/3)"
Approved:  "승인되었습니다. 구현을 시작합니다."
Dev:       "Developer가 구현 중입니다..."
Code review: "Reviewer가 코드를 검토 중입니다 (N/3)"
Done:      "완료되었습니다."
```

### Escalation

- 3 review iterations without approval → surface to user with options
- Agent needs user input → relay immediately
- Blocking error → report to user, don't silently retry

### Mid-Task Tier Upgrade

If during Light tier you realize the task is more complex than expected (e.g., Developer reports needing to touch many more files), pause and tell the user: "예상보다 복잡합니다. Standard로 전환하여 설계부터 진행할까요?" Respect the user's decision.

---

## Edge Cases

| Situation | Response |
|-----------|----------|
| User specifies a tier | Use the requested tier regardless of analysis |
| User wants to skip planning | Use Light tier |
| User provides their own plan | Skip to Development phase, still run Reviewer |
| User interrupts with new requirements | Restart the current phase with updated context |
| Task needs user decisions mid-way | Relay agent questions immediately |
| Task turns out simpler than expected | Finish current tier — don't downgrade mid-task |
