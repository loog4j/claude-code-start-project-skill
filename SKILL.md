---
name: start-project
description: Set up and build AI-assisted coding projects with structured planning, task breakdown, and automated execution. Supports both new projects and feature additions to existing codebases. Uses sub-agents for parallel plan generation, task breakdown, scaffolding, and wave-based parallel task execution with atomic git commits.
---

# AI-Assisted Project Setup

You are helping the user set up and build a structured, AI-assisted coding project. This skill creates a complete project management framework optimized for agentic coding — with lean task definitions, tiered context, and milestone-based checkpoints — then executes all tasks automatically using wave-based parallel agents with fresh context windows and atomic git commits.

## Phase 0: Resume Detection

Before starting anything, check if this is a resumption of a previously started project.

If the user provided a target directory (via arguments or prior context), check if `{target_directory}/STATE.md` exists. If no directory is known yet, ask the user for the target directory first, then check.

**If STATE.md exists**, read it and present the user with options using AskUserQuestion:

"I found an existing project state at `{target_directory}/STATE.md`. It was last at Phase {N} ({phase_name}). How would you like to proceed?"
- **"Resume from where I left off" (Recommended)** — Skip to the phase/wave/task indicated in STATE.md. Read PLAN.md, CONTEXT.md, and DECISIONS.md to restore context, then jump directly to the saved phase.
- **"Start fresh (overwrite)"** — Delete STATE.md and all planning files, then start from Phase 1.

When resuming:
- If resuming into Phase 7 (execution), pick up at the saved wave number — skip already-completed waves
- If resuming into Phase 9 (verification), re-run the verifier from scratch
- For any other phase, re-enter at the start of that phase
- Read the Handoff Notes section for critical context about what was happening when the session ended

**If STATE.md does not exist**, proceed normally to Phase 0.5.

## Phase 0.5: Environment Validation

Before gathering requirements, validate the development environment:

1. **Git check**: Run `git rev-parse --git-dir 2>/dev/null` to check if a git repo exists
   - For **new projects**: If no git repo, run `git init` in the target directory after scaffold (Phase 4) creates it, then `git add -A && git commit -m "chore: initial project scaffold"`
   - For **feature additions**: If the parent project has no git repo, warn the user: "No git repository found. Atomic commits require git. Initialize one with `git init`?"
   - If `git` is not installed, fail fast: "Git is required for this skill. Install git and retry."

2. **Complexity assessment** (Quick Mode detection): Analyze the user's request. If the project is clearly simple (≤3 files to create/modify, no external dependencies, no infrastructure), offer quick mode:
   "This looks like a straightforward task. Would you like quick mode?"
   - **"Quick mode" (Recommended for simple tasks)** — Skip Phases 2, 3, 5 (discuss, research, plan verification). Go directly: Requirements → Plan → Execute → Verify.
   - **"Full mode"** — Run all phases with research, discussion, and plan verification.

   Store the mode in STATE.md. If quick mode, skip the indicated phases during orchestration.

## Phase 1: Gather Requirements

Use the AskUserQuestion tool to collect project information. Ask these in a single call:

**Question 1 - Project Type:**
"Are you starting a new project or adding a feature to an existing codebase?"
- Options: "New project", "Feature addition"

**Question 2 - Target Directory:**
"What directory should the project/feature files be created in?"
- Options: Let the user provide a path via "Other"
- Header: "Directory"

**Question 3 - Project/Feature Name:**
"What is the name of your project or feature?"
- Options: Let the user provide a name via "Other"
- Header: "Name"

Then ask a second round of questions based on context:

**Question 4 - Tech Stack:**
"What is the primary technology stack?"
- Options: Common stacks like "Python", "Node.js/TypeScript", "React + Node.js", plus "Other"

**Question 5 - Project Goal:**
"In 1-2 sentences, what is the main goal? What problem does this solve?"
- Let user provide via "Other"

**Question 6 - Key Constraints:**
"Are there any key constraints or requirements to be aware of? (Select all that apply)"
- Options: "Authentication/Auth required", "Database/data storage", "API integrations", "Performance-critical"
- multiSelect: true

**Question 7 - Model Profile (optional):**
"Which model profile should sub-agents use? This controls cost vs. quality tradeoffs."
- **"Balanced" (Recommended)** — Sonnet for planning & execution, Haiku for research & verification. Best cost/quality ratio.
- **"Quality"** — Sonnet for all agents. Highest quality, higher cost.
- **"Budget"** — Haiku for everything except executors (Sonnet). Lowest cost.

Store the selected profile and use it to set the `model` parameter on every Agent tool call throughout the skill:

| Agent Type | Quality | Balanced | Budget |
|-----------|---------|----------|--------|
| Research (Phase 3) | sonnet | haiku | haiku |
| Planning (Phase 4) | sonnet | sonnet | haiku |
| Plan Checker (Phase 5) | sonnet | haiku | haiku |
| Executors (Phase 7) | sonnet | sonnet | sonnet |
| Verifier (Phase 9) | sonnet | haiku | haiku |
| Debug (Phase 10) | sonnet | sonnet | haiku |

## Phase 2: Discuss & Clarify

Before planning, identify implementation decisions that matter and lock them down. This prevents rework by ensuring agents have clear guidance on ambiguous areas.

### Step 1: Analyze Gray Areas

Based on the gathered requirements, identify 3-6 "gray areas" — decisions where multiple valid approaches exist and the user's preference matters. Tailor questions to the project type:

**For projects with UI**: layout approach, component library, styling method, routing strategy, responsive vs. desktop-first
**For API/backend projects**: authentication method, database choice, API style (REST vs. GraphQL), error handling strategy, logging approach
**For CLI tools**: argument parsing library, output formatting, configuration file format, plugin architecture
**For all projects**: testing strategy, code organization pattern, deployment target, environment management

### Step 2: Present Decisions

Use AskUserQuestion to present the gray areas. Ask 2-4 questions per round (max 2 rounds) to avoid overwhelming the user. Each question should:
- Present 2-4 concrete options (not abstract choices)
- Include a recommended option with brief rationale
- Frame options in terms of tradeoffs, not just names

Example:
```
"How should authentication be handled?"
- "JWT with httpOnly cookies (Recommended)" — Stateless, secure against XSS
- "Session-based with server store" — Simpler, but requires session storage
- "OAuth2 only (Google/GitHub)" — No password management, but requires third-party dependency
```

### Step 3: Record Decisions

After the user responds, classify each decision:
- **Locked** — User made a clear choice → agents MUST follow this
- **Agent discretion** — User said "you decide" or "whatever works" → agent chooses, documents rationale
- **Deferred** — User wants to decide later → note it, don't block on it

Write these to `{target_directory}/DECISIONS.md` (or `{target_directory}/features/{feature-slug}/DECISIONS.md` for feature additions) with this format:

```
| Date | Decision | Rationale | Impact | Status |
|------|----------|-----------|--------|--------|
| {today} | Use JWT with httpOnly cookies | User preference, stateless auth | Auth implementation | Locked |
| {today} | Choose between Vitest and Jest for testing | User deferred | Test setup | Agent discretion |
```

Pass ALL locked decisions and agent-discretion notes to the planning agents in Phase 4 so they incorporate them into PLAN.md and task files.

**Auto-save**: Update STATE.md — set Phase to `2/10 (Discuss & Clarify) — complete`, record decisions locked in Session History.

## Phase 3: Domain Research

Before planning, research the chosen tech stack and domain to produce better-informed plans. Spawn 2 parallel research agents.

**Agent A: Stack & Patterns** (subagent_type: "general-purpose")
```
You are a technical researcher. Research the {tech_stack} technology stack given these constraints: {constraints}.

Focus on:
1. Current best practices and recommended project structure for {tech_stack}
2. Recommended libraries/frameworks for the constraints (auth, database, API, etc.) with current stable versions
3. Common architectural patterns and design approaches for this type of project
4. Project layout conventions and naming standards

Use WebSearch to find current information. Keep research focused — no more than 5 minutes.

Return a concise summary with: recommended directory structure, key library recommendations with versions, and patterns to follow.
```

**Agent B: Pitfalls & Architecture** (subagent_type: "general-purpose")
```
You are a technical researcher specializing in architectural risks and performance.

Research the {tech_stack} technology stack given these constraints: {constraints}.

Focus on:
1. Common pitfalls and anti-patterns when building with {tech_stack}
2. Architectural decisions that are difficult or expensive to change later
3. Performance considerations relevant to the constraints
4. Integration gotchas between components in this stack

Use WebSearch to find current information. Keep research focused — no more than 5 minutes.

Return a concise summary with: major pitfalls to avoid, architectural recommendations, and performance tips.
```

### Synthesize Research

After both agents complete, write `{target_directory}/RESEARCH.md` (or `{target_directory}/features/{feature-slug}/RESEARCH.md` for feature additions) combining both agents' findings into two sections:
- **Stack & Patterns**: recommended structure, libraries, patterns
- **Pitfalls & Architecture**: risks, guidance, performance considerations

Pass these research findings to ALL planning agents in Phase 4 alongside the locked decisions from Phase 2. Append this to each planning agent prompt:
```
## Research Findings
{contents of RESEARCH.md}
```

## Phase 4: Dispatch Sub-Agents

After research and clarifying decisions, dispatch sub-agents using the Agent tool. The parallelization depends on project type.

**Path resolution**: All phases use `{working_directory}` to reference where project files live:
- **New projects**: `{working_directory}` = `{target_directory}` (the directory the user specified)
- **Feature additions**: `{working_directory}` = `{target_directory}/features/{feature-slug}/`

Set this once here and use `{working_directory}` consistently in ALL subsequent phases (5-10). This ensures research, plans, tasks, execution, verification, and UAT all operate on the correct directory.

**IMPORTANT**: Include the locked decisions from Phase 2 in every planning agent's prompt. Append this block to each agent prompt:

```
## Locked Decisions (from user discussion)
{list each locked decision and its rationale}

## Agent Discretion Areas
{list areas where the agent should choose and document rationale}

## Deferred Decisions
{list items to leave flexible — do not hard-code these}
```

### For NEW PROJECTS — dispatch 3 agents in parallel:

**Agent A: Plan Generation** (subagent_type: "general-purpose")
```
Prompt: "You are generating a project plan. Write a file called PLAN.md at {target_directory}/PLAN.md.

Project Name: {name}
Tech Stack: {tech_stack}
Goal: {goal}
Constraints: {constraints}

Generate PLAN.md with these sections (fill in real content, not placeholders):

# {Project Name} - Project Plan

## Goal
[1-2 paragraph description based on user's goal]

## Components
[3-5 components with brief descriptions of what each does]

## Tech Stack
[Specific technologies, frameworks, and tools]

## Requirements
### Functional
[3-5 concrete functional requirements]
### Technical
[Performance, security, compatibility requirements based on constraints]

## Success Criteria
[4-6 specific, measurable outcomes]

## Risk Assessment
[2-3 realistic risks with mitigations]

## Architecture Notes
[Key architectural decisions and patterns to follow]

Keep it concise and actionable. No placeholder brackets — fill everything in based on the provided context."
```

**Agent B: Task Breakdown** (subagent_type: "general-purpose")
```
Prompt: "You are generating task tickets for an AI-assisted coding project. Create individual task files in {target_directory}/tasks/.

Project Name: {name}
Tech Stack: {tech_stack}
Goal: {goal}
Constraints: {constraints}

Create 6-10 task files named TASK-01.md through TASK-XX.md. Each task should use this LEAN format (do NOT use step-by-step runbook style):

# TASK-XX: {Task Title}

## Goal
[1-2 sentences: what this task accomplishes]

## Constraints
[Specific technical constraints, patterns to follow, libraries to use]

## Acceptance Criteria
- [ ] [Specific, testable criterion 1]
- [ ] [Specific, testable criterion 2]
- [ ] [Specific, testable criterion 3]
- [ ] All existing tests still pass (if applicable)

## Dependencies
- Requires: [TASK-XX] (or 'None' for first tasks)

## Autonomy Guide
- Proceed independently: [what the agent can decide on its own]
- Ask before: [decisions that need user input]

## Notes
[Any important context, gotchas, or architectural considerations]

IMPORTANT RULES:
- Phase 1 tasks (first 2-3) should be detailed. Later phase tasks should be higher-level and will be refined when reached.
- Focus on WHAT and WHY, not HOW. The executing agent will determine implementation.
- Each acceptance criterion must be verifiable (testable command, observable behavior, or file existence).
- Keep each task file under 40 lines."
```

**Agent C: Scaffold & Config** (subagent_type: "general-purpose")
```
Prompt: "You are scaffolding the directory structure and configuration files for an AI-assisted coding project.

Project Name: {name}
Tech Stack: {tech_stack}
Target Directory: {target_directory}

Do the following:

1. Create the directory structure:
   {target_directory}/
   ├── tasks/          (task files go here — Agent B will create them)
   ├── src/            (source code)
   ├── tests/          (test files)
   └── docs/           (documentation)

2. Create {target_directory}/CONTEXT.md (static project reference — NOT updated during execution):

# {Project Name}
- **Tech Stack**: {stack}
- **Directory**: {target_directory}
- **Source**: src/ | **Tests**: tests/ | **Tasks**: tasks/ | **Plan**: PLAN.md

3. Create {target_directory}/DECISIONS.md:

# Decision Log
Decisions are logged here as they are made during development.

| Date | Decision | Rationale | Impact | Status |
|------|----------|-----------|--------|--------|
| {today} | Project initialized with {tech_stack} | {brief rationale from goal} | Foundation for all tasks | Locked |

4. Create {target_directory}/.gitignore appropriate for {tech_stack}.

5. Create {target_directory}/STATE.md:

# {Project Name} — Project State

## Current Position
- **Phase**: 4/10 (Dispatch Sub-Agents)
- **Wave**: —
- **Active Task**: —

## Task Status
### Completed (0)
### Pending (all tasks — not yet created by task agent)
### Blocked (0)

## Session History
| Session | Date | Phase | Accomplished |
|---------|------|-------|--------------|
| 1 | {today} | 1-3 | Requirements gathered, decisions locked, planning dispatched |

### Velocity
- Tasks/Session: 0 (planning phase)

## Handoff Notes
- Next immediate action: Review plan output, then execute tasks
- Known issues: None
- Blockers: None

---
*Last updated: {today}*

6. Do NOT create a README.md — that will be a task for the agent to create as part of development."
```

### For FEATURE ADDITIONS — dispatch in two waves:

**Wave 1 (parallel):**

**Agent 1: Codebase Analysis** (subagent_type: "Explore")
```
Prompt: "Thoroughly explore the existing codebase at {target_directory}. I need you to understand:

1. **Architecture**: What patterns are used (MVC, component-based, etc.)? How is the code organized?
2. **Tech stack**: What languages, frameworks, libraries, and tools are in use? Check package files, configs, imports.
3. **Entry points**: Where are the main entry points? (main files, route definitions, app bootstrapping)
4. **Testing**: What test framework is used? Where do tests live? How are they run?
5. **Code style**: Naming conventions, file organization patterns, comment style.
6. **Integration points**: Where would a new feature likely hook in? (routing, navigation, state management, API layer)

Return a structured summary covering all 6 areas. Be specific — reference actual file paths and patterns you observe."
```

**Agent 2: Scaffold** (subagent_type: "general-purpose")
```
Prompt: "Create a feature development directory within an existing project.

Feature Name: {name}
Project Directory: {target_directory}

Create:
{target_directory}/features/{feature-slug}/
├── tasks/
└── docs/

Create {target_directory}/features/{feature-slug}/CONTEXT.md:

# Feature: {Feature Name}
- **Project**: {target_directory}
- **Status**: Planning — awaiting task generation
- **Current Task**: Not started
- **Active Blockers**: None

## Agent Instructions
- Read this CONTEXT.md and the project's main code before starting any feature task
- Update CONTEXT.md when: completing a task, hitting a blocker, or ending a session
- Ensure all existing tests pass before and after each task
- If integration with existing code creates conflicts, document and resolve before proceeding
- You may split or merge tasks if needed — update CONTEXT.md with rationale

Create {target_directory}/features/{feature-slug}/DECISIONS.md:

# Feature Decision Log: {Feature Name}

| Date | Decision | Rationale | Impact | Status |
|------|----------|-----------|--------|--------|

Create {target_directory}/features/{feature-slug}/INTEGRATION_CHECKLIST.md:

# Integration Checklist: {Feature Name}

## Before Merging
- [ ] All existing tests pass
- [ ] New functionality has test coverage
- [ ] No security regressions introduced
- [ ] API contracts maintained (or versioned if changed)
- [ ] Documentation updated for any changed interfaces
- [ ] Performance acceptable (no regressions beyond 10%)
- [ ] Code follows existing project conventions and patterns
- [ ] Feature works end-to-end in development environment

## After Merging
- [ ] Smoke test in staging/preview environment
- [ ] Monitor error rates for 24 hours
- [ ] Update project README if user-facing changes

Create {target_directory}/features/{feature-slug}/STATE.md:

# Feature: {Feature Name} — Project State

## Current Position
- **Phase**: 4/10 (Dispatch Sub-Agents)
- **Wave**: —
- **Active Task**: —

## Task Status
### Completed (0)
### Pending (all tasks — not yet created by task agent)
### Blocked (0)

## Session History
| Session | Date | Phase | Accomplished |
|---------|------|-------|--------------|
| 1 | {today} | 1-3 | Requirements gathered, decisions locked, planning dispatched |

### Velocity
- Tasks/Session: 0 (planning phase)

## Handoff Notes
- Next immediate action: Review plan output, then execute tasks
- Known issues: None
- Blockers: None

---
*Last updated: {today}*"
```

**Wave 2 (after Agent 1 returns — dispatch both in parallel):**

Use the codebase analysis results from Agent 1 to inform Agents 2 and 3:

**Agent 2: Feature Plan** (subagent_type: "general-purpose")
```
Prompt: "Generate a feature plan. Write it to {target_directory}/features/{feature-slug}/PLAN.md.

Feature Name: {name}
Goal: {goal}
Constraints: {constraints}
Tech Stack: {tech_stack}

Codebase Analysis:
{paste Agent 1 results here}

Generate PLAN.md with:

# Feature: {Feature Name}

## Goal
[What this feature does and why it's valuable]

## Integration Points
[Based on codebase analysis — specific files/modules that need modification]

## New Components
[What new files/modules need to be created]

## Requirements
### Functional
[3-5 requirements]
### Technical
[Compatibility, performance, security requirements]

## Success Criteria
[4-6 measurable outcomes, including 'all existing tests pass']

## Risk Assessment
[2-3 risks specific to integrating with the existing codebase]

## Rollback Plan
[How to cleanly remove this feature if needed]

Fill in real content based on the codebase analysis. No placeholder brackets."
```

**Agent 3: Feature Tasks** (subagent_type: "general-purpose")
```
Prompt: "Generate task tickets for a feature addition. Create files in {target_directory}/features/{feature-slug}/tasks/.

Feature Name: {name}
Goal: {goal}
Constraints: {constraints}

Codebase Analysis:
{paste Agent 1 results here}

Create 5-8 task files named TASK-01.md through TASK-XX.md using this lean format:

# TASK-XX: {Task Title}

## Goal
[1-2 sentences]

## Constraints
- Must follow existing code patterns identified in codebase analysis
- [Feature-specific constraints]

## Acceptance Criteria
- [ ] [Criterion 1]
- [ ] [Criterion 2]
- [ ] All existing tests still pass
- [ ] New code has test coverage

## Dependencies
- Requires: [TASK-XX or None]

## Integration Notes
[Which existing files/modules this task touches, based on codebase analysis]

## Autonomy Guide
- Proceed independently: [what agent can decide]
- Ask before: [what needs user input]

RULES:
- First task should always be 'Analyze codebase and set up feature branch'
- Last task should always be 'Integration testing and cleanup'
- Phase 1 tasks detailed, later tasks high-level
- Each task under 40 lines
- Reference specific files from the codebase analysis in Integration Notes"
```

## Phase 5: Plan Verification

After planning agents complete, spawn a **plan-checker agent** to validate the plans before presenting to the user. This catches bad plans before they reach execution.

**Plan Checker Agent** (subagent_type: "general-purpose"):

```
You are a plan verification specialist for the "{project_name}" project.

Read `{target_directory}/PLAN.md` and all task files in `{target_directory}/tasks/`. Validate against these 6 dimensions:

1. **Goal Coverage**: Do the tasks, taken together, cover ALL success criteria from PLAN.md? Map each criterion to the task(s) that address it. Flag any unaddressed criteria.

2. **Dependency Correctness**: Are task dependencies valid? Check for: circular dependencies, references to non-existent tasks, missing dependencies where order clearly matters.

3. **Completeness**: Does the first task set up the foundation (project init, dependencies)? Does the last task tie things together (integration, cleanup)? Are there gaps in the middle?

4. **Feasibility**: Is any single task trying to do too much? Tasks should be completable in a single focused session. Flag tasks that combine unrelated concerns.

5. **Consistency**: Do tasks use consistent tech stack references, naming conventions, and patterns? Do they align with the locked decisions and research findings?

6. **Testability**: Does each acceptance criterion have a clear verification method (test command, observable behavior, or file existence check)?

## Actions
- If issues are found: fix them directly by editing the files. Add missing tasks, fix dependencies, split oversized tasks, fill in gaps.
- Max 2 check-and-fix passes to avoid infinite loops.

## Return
A brief validation report:
- Dimensions checked with pass/fail
- Issues found and fixes applied
- Overall quality: **Good** (no issues) | **Acceptable** (minor fixes applied) | **Needs Manual Review** (significant issues the user should see)
```

If the plan checker returns **Needs Manual Review**, flag the specific issues in the Phase 6 presentation so the user can address them before execution.

### Nyquist Test Mapping

As part of plan verification, create a test coverage map that ensures every requirement has an automated verification path. This is done by the plan checker agent (add to its prompt) or as a follow-up step:

1. Extract each functional requirement and success criterion from PLAN.md
2. For each, define a specific automated test command, expected output, and which task creates the test
3. Write the mapping to `{target_directory}/TEST_MAP.md`:

```
# Test Coverage Map

| Requirement | Test Command | Expected Output | Task |
|------------|-------------|-----------------|------|
| User can log in | `npm test -- --grep "login"` | Tests pass | TASK-03 |
| API returns JSON | `curl -s localhost:3000/api/health \| jq .` | Valid JSON with status field | TASK-02 |
| Data persists | `pytest tests/test_db.py -k "test_persistence"` | All tests pass | TASK-04 |

## Unmapped Requirements
- {requirement}: {why it can't be automated — needs manual testing}
```

4. Pass TEST_MAP.md to executor agents so they know what tests to create alongside their implementation
5. Flag any unmapped requirements for the user during Phase 6 presentation

## Phase 6: Present Results

After all sub-agents complete and plans are verified, present a summary to the user. Do NOT dump all file contents. Instead:

1. List the files that were created
2. Show the task sequence (task names and dependencies only)
3. Highlight any key architectural decisions from the plan
4. Ask: "Does this plan look good? Should I adjust anything before you start executing?"

Format:
```
## Project Setup Complete: {name}

### Files Created
- `PLAN.md` — project plan and architecture
- `CONTEXT.md` — agent context and session state
- `DECISIONS.md` — decision log
- `tasks/TASK-01.md` through `tasks/TASK-XX.md` — task tickets

### Task Sequence
1. **TASK-01**: {title}
2. **TASK-02**: {title} (requires TASK-01)
3. ...

### Key Decisions
- {decision 1 from plan}
- {decision 2 from plan}

### Next Step
Ready to execute tasks automatically with parallel agents and atomic commits.
```

Ask the user using AskUserQuestion:
"Plan is ready. How would you like to proceed?"
- **"Execute all tasks automatically" (Recommended)** — Spawn executor agents to build the project with atomic commits per task
- **"Let me review first"** — Pause here so you can review the plan and task files before execution

If the user chooses to execute, proceed to Phase 7. If they choose to review, end the skill here and tell them to run `/start-project` again or manually begin with TASK-01.

## Phase 7: Execute Tasks

After user approval, execute all tasks using wave-based parallel execution with fresh-context executor agents.

### Step 1: Wave Analysis

Read all task files from the `tasks/` directory. Parse each task's `## Dependencies` section to extract required task IDs.

Group tasks into waves using topological ordering:
- **Wave 1**: All tasks with no dependencies (Requires: None)
- **Wave 2**: Tasks whose dependencies are ALL in Wave 1
- **Wave 3**: Tasks whose dependencies are ALL in Waves 1-2
- Continue until all tasks are assigned

If a task's dependencies create a cycle, report the issue to the user and skip those tasks.

**Context budget**: Limit each wave to a maximum of **5 parallel executors**. If a wave has more than 5 independent tasks, split it into sub-waves of 5. This prevents context window pressure and keeps each executor's work focused. Total project tasks should not exceed 30 — if the planner created more, consolidate related tasks before execution.

Present the wave plan to the user before executing:
```
### Execution Plan
- **Wave 1** (parallel): TASK-01, TASK-02
- **Wave 2** (parallel, after Wave 1): TASK-03, TASK-04
- **Wave 3** (after Wave 2): TASK-05
```

### Step 2: Execute Waves Sequentially

For each wave, spawn one executor agent per task **in parallel** using the Agent tool (subagent_type: "general-purpose"). All agents in a wave are dispatched in a single tool-call block.

Wait for all agents in the current wave to complete before starting the next wave.

**Executor Agent Prompt** (use for each task, filling in the placeholders):

```
You are an autonomous executor agent for the {project_name} project.

## Your Assignment
Execute **TASK-{task_number}** as defined in `{task_file_path}`.

## Preparation
1. Read your task file at `{task_file_path}` — understand Goal, Constraints, Acceptance Criteria
2. Read `{target_directory}/PLAN.md` for architectural context and tech stack
3. Read `{target_directory}/CONTEXT.md` for current project state
4. Read `{target_directory}/TEST_MAP.md` to understand what tests your task should create or satisfy
5. If working in an existing codebase, explore the relevant source files to understand patterns before writing code

## Implementation Rules
- Write real, functional code in `{target_directory}` — no stubs, no TODOs, no placeholder implementations
- Follow existing code patterns, naming conventions, and project structure
- Handle error cases and edge cases appropriately
- Write tests if test infrastructure exists (match the existing test framework and patterns)
- Run tests after implementation to verify nothing is broken

## Deviation Handling
You may autonomously fix without asking:
- Bugs encountered during implementation
- Missing input validation, error handling, or logging
- Blocking issues: broken imports, missing dependencies, syntax errors

You MUST STOP and report a deviation for:
- Architectural changes (new database tables, major design pattern shifts)
- Adding new libraries or major dependencies not in the plan
- Significant deviations from the task specification
When you detect any of these, STOP immediately. Include `deviation_required: true` in your return summary with: (1) what you were trying to do, (2) what change is needed, (3) your recommended approach. The orchestrator will ask the user and may re-dispatch you with guidance.

Max 3 fix attempts per auto-fixable issue — if unresolved, report as blocked.

## Anti-Stall Rule
If you perform 5+ consecutive read/search operations without writing any code, you must either start writing code immediately or report that you are blocked with specific reasons.

## Git Commit
After completing the task successfully:
1. Stage only the files you created or modified — NEVER use `git add .` or `git add -A`
2. Commit with conventional format:
   - Features: `feat(TASK-{task_number}): {brief description}`
   - Bug fixes: `fix(TASK-{task_number}): {brief description}`
   - Refactoring: `refactor(TASK-{task_number}): {brief description}`
   - Tests: `test(TASK-{task_number}): {brief description}`
   Include `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>` in the commit body

## When Done
Do NOT write to CONTEXT.md or STATE.md — the orchestrator handles all state updates to prevent race conditions with parallel executors.

Return a structured summary (the orchestrator will merge these):
### TASK-{task_number} Execution Summary
- **Status**: Complete | Blocked | Partial
- **deviation_required**: true/false (if true, include deviation details below)
- **What was done**: [1-3 sentences]
- **Files created/modified**: [list]
- **Tests**: [passed/failed/skipped + details]
- **Issues**: [any problems encountered or deferred decisions]
- **Commit**: [commit hash or "no commit" if blocked]
- **Deviation** (if applicable): [what change is needed, why, recommended approach]
```

### Step 3: Track Progress (Orchestrator Responsibilities)

The orchestrator (you) is the **sole writer** of CONTEXT.md and STATE.md. Executors never write to these files directly — they return structured summaries that you merge.

After each wave completes:
1. Collect the execution summaries from all agents in the wave
2. **Check for deviations**: If any executor returned `deviation_required: true`, pause execution. Present the deviation to the user via AskUserQuestion with the executor's recommended approach. After the user decides, re-dispatch that executor with the guidance, or mark it as blocked.
3. Check for any blocked or failed tasks
4. If a task in this wave failed and later tasks depend on it, skip those dependent tasks and report them as blocked
5. **Merge state** (you do this, not executors): Update CONTEXT.md with completed tasks, current task, any new blockers. Update STATE.md with wave number, task status changes, velocity, and handoff notes.
6. Display a brief progress update to the user:

```
### Wave {N} Complete
- TASK-01: ✓ Complete — {brief summary}
- TASK-02: ✓ Complete — {brief summary}
- TASK-03: ✗ Blocked — {reason}

Proceeding to Wave {N+1}...
```

## Phase 8: Execution Summary

After all waves complete, present a final summary:

```
## Execution Complete: {project_name}

### Results
| Task | Status | Summary |
|------|--------|---------|
| TASK-01 | ✓ | {what was done} |
| TASK-02 | ✓ | {what was done} |
| TASK-03 | ✗ | {why it failed} |

### Git Log
{Show the commits created during execution — run `git log --oneline` for the relevant commits}

### Files Changed
{List of all files created or modified across all tasks}

### Issues & Follow-ups
{Any blocked tasks, deferred decisions, or recommended next steps}

### Next Steps
- Review the generated code and run the application
- Address any blocked tasks listed above
- Run the full test suite: {test command based on tech stack}
```

If any tasks were blocked or failed, ask the user if they'd like to retry those tasks or address the blockers manually.

**Auto-save**: Update STATE.md — set Phase to `8/10 (Execution Summary) — complete`, finalize task counts, update session history with tasks completed and velocity.

## Phase 9: Goal-Backward Verification

After execution completes (Phase 8), spawn a **verifier agent** that independently checks whether the project goals were actually achieved. This agent does NOT trust execution summaries — it inspects the codebase directly.

**Verifier Agent** (subagent_type: "general-purpose"):

```
You are a Goal-Backward Verification Agent for the "{project_name}" project.

Your job: verify that project goals were ACTUALLY achieved by working backward from observable outcomes — not by checking if tasks were marked complete.

## Process

### 1. Extract Goals
Read `{target_directory}/PLAN.md` and extract:
- All success criteria
- Functional and technical requirements
- Key components described in the plan

### 2. Derive Three Verification Layers

From the goals, derive:

**Truths** — Observable behaviors that must be true for the project to work:
- Example: "User can create an account", "API returns paginated results", "CLI accepts --verbose flag"
- Derive 4-8 truths from the success criteria

**Artifacts** — Files that must exist with REAL implementation:
- Example: "src/auth/login.ts contains credential validation logic", "tests/api.test.js has passing test cases"
- Each artifact must contain substantive code, not stubs

**Key Links** — Critical connections between artifacts that prove the system is wired together:
- Example: "The route handler imports and calls the service function", "The component dispatches the Redux action"
- These verify the pieces actually work together, not just exist in isolation

### 3. Inspect the Codebase

For each artifact, read the actual file and verify:
- The file exists
- It contains real, functional code (not stubs)
- It implements what the plan requires

**Stub Detection** — Flag as NOT REAL: empty bodies/`pass`, TODO/FIXME/HACK comments, hardcoded mock data, log-only functions, placeholder text ("Coming soon"), silent catch blocks, hardcoded return values, assertion-less tests.

For each key link, trace the connection:
- Verify imports exist and reference real exports
- Verify function calls pass correct arguments
- Verify data flows from source to destination

### 4. Produce Verification Report

Return a structured report:

#### Truths Verification
| Truth | Status | Evidence |
|-------|--------|----------|
| {behavior} | PASS/FAIL | {file:line or test result} |

#### Artifact Gaps
- {file_path}: {what's missing or stubbed — cite specific lines}

#### Broken Links
- {source} → {destination}: {what's wrong}

#### Stubs Detected
- {file_path}:{line}: {description of placeholder code}

#### Overall Verdict: VERIFIED | NEEDS WORK

#### Recommended Fixes (if NEEDS WORK)
1. {specific fix with file path and what needs to change}
2. ...
```

### After Verification

Present the verification report to the user. If the verdict is **VERIFIED**, the project is complete.

If the verdict is **NEEDS WORK**, ask the user:
"The verifier found gaps. Would you like me to fix these issues automatically?"
- **"Fix automatically" (Recommended)** — Spawn executor agents for each recommended fix, then re-verify
- **"Show me the details"** — Display the full verification report for manual review

If fixing automatically: create a task file per fix, execute them using the Phase 7 wave executor, then re-run the verifier (max 2 fix-and-verify cycles to avoid infinite loops).

**Auto-save**: Update STATE.md — set Phase to `9/10 (Verification) — complete` if VERIFIED, or note the fix-and-verify cycle count if NEEDS WORK. Record final verdict and session summary.

## Phase 10: User Acceptance Testing (UAT)

After automated verification passes, guide the user through hands-on testing. This closes the feedback loop with real human validation.

### Step 1: Extract Test Scenarios

Read PLAN.md success criteria and TEST_MAP.md. Derive 3-6 testable behaviors that require human verification — things the automated verifier can't fully confirm (UI appearance, UX flow, subjective quality).

### Step 2: Conversational Testing

Present each test scenario one at a time using AskUserQuestion:

"**Test {N}/{total}: {behavior description}**
- **Action**: {what to do — specific command, URL, or interaction}
- **Expected**: {what should happen}
- **How**: {exact steps or command to run}

What was the result?"
- **"Pass"** — Behavior works as expected
- **"Fail"** — Something is wrong (describe the issue via "Other")
- **"Skip"** — Can't test right now

### Step 3: Debug Failures

If any tests fail, spawn one **debug agent per failure** in parallel (subagent_type: "general-purpose", use model profile):

```
You are a debug agent for the "{project_name}" project.

## Failure Report
- Test: {test description}
- Expected: {expected behavior}
- Actual: {user's failure description}

## Process
1. Read the relevant source files in {target_directory} based on the failure description
2. Identify the root cause using the scientific method:
   - Form a hypothesis about what's wrong
   - Find evidence in the code (trace the code path)
   - Confirm or reject the hypothesis
3. Propose a specific fix: which file, which lines, what change

## Return
- **Root cause**: [1-2 sentences]
- **File(s)**: [paths to files that need changes]
- **Fix**: [specific code changes needed]
- **Confidence**: High | Medium | Low
```

### Step 4: Fix and Re-test

After debug agents return, present findings and ask:
"Found {N} issue(s). Would you like me to fix them?"
- **"Fix all automatically" (Recommended)** — Apply fixes, re-run the failed tests
- **"Show details first"** — Display root causes and proposed fixes for review

If fixing: create fix tasks, execute via Phase 7 executor, then re-present only the previously-failed test scenarios (max 2 fix cycles).

### Step 5: Record Results

Write `{target_directory}/UAT.md`:
```
# User Acceptance Testing — {project_name}

| # | Test | Result | Notes |
|---|------|--------|-------|
| 1 | {behavior} | Pass/Fail/Skip | {details} |

## Failures & Fixes
- {test}: {root cause} → {fix applied}

## Sign-off
- Date: {today}
- Status: All tests passed | {N} issues remaining
```

**Auto-save**: Update STATE.md — set Phase to `10/10 (UAT) — complete`.

## File Ownership Rules

To prevent confusion between overlapping files:

| File | Purpose | Written by | When updated |
|------|---------|-----------|--------------|
| **CONTEXT.md** | Project context: architecture, patterns, quick reference, agent instructions | Scaffold agent (Phase 4), then read-only | Once during setup — static reference |
| **STATE.md** | Execution state: current phase, task status, velocity, handoff notes | Orchestrator only (never executors) | After each wave, phase transition, or pause |
| **DECISIONS.md** | Decision log: locked choices, rationale, impact | Phase 2 (discuss), then append-only | When new decisions are made |
| **TEST_MAP.md** | Test coverage: requirement → test command mapping | Plan checker (Phase 5) | During plan verification |
| **PLAN.md** | Project plan: goals, components, requirements, success criteria | Planning agents (Phase 4) | Once during planning |

**Key rule**: Executors NEVER write to CONTEXT.md, STATE.md, or any shared state file. They return structured summaries to the orchestrator, which performs all state merges.

## Session Management

### Pause Handling
If the user says "pause", "stop", "save progress", or ends the session mid-execution:
1. Immediately update STATE.md with the current phase, wave, and task
2. Write concise handoff notes capturing: what was just completed, what's next, any in-flight issues
3. Confirm to the user: "Progress saved to STATE.md. Run `/start-project` to resume."

### Auto-Save Summary
STATE.md is automatically updated at these checkpoints:
- After Phase 2 (decisions locked)
- After each wave in Phase 7 (execution progress)
- After Phase 8 (execution complete)
- After Phase 9 (verification verdict)
- After Phase 10 (UAT complete)
- On pause/session end (wherever you are)
