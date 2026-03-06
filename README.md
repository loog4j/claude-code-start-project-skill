# start-project

An AI-assisted project builder skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Takes you from idea to working codebase with structured planning, parallel execution, automated verification, and user acceptance testing.

## What It Does

Run `/start-project` and the skill handles everything:

1. **Gathers requirements** — project type, stack, goals, constraints
2. **Discusses gray areas** — locks down ambiguous decisions before planning
3. **Researches your stack** — best practices, pitfalls, recommended libraries
4. **Generates plans & tasks** — PLAN.md + lean task tickets with acceptance criteria
5. **Verifies the plan** — 6-dimension validation + test coverage mapping
6. **Executes tasks in parallel** — wave-based execution with fresh-context agents and atomic git commits
7. **Verifies the result** — goal-backward codebase inspection with stub detection
8. **Runs UAT** — conversational testing with auto-debug on failures

### Key Features

- **Wave-based parallel execution** — independent tasks run simultaneously, dependent tasks wait
- **Atomic git commits** — one conventional commit per task (`feat(TASK-01): ...`)
- **Goal-backward verification** — checks if goals were achieved, not just if tasks ran
- **Stub detection** — catches placeholder code, empty functions, TODO comments
- **Session persistence** — pause anytime, resume later via STATE.md
- **Quick mode** — skip research/discussion for simple tasks
- **Model profiles** — Quality / Balanced / Budget to control cost
- **Deviation handling** — executors flag architectural decisions for your approval
- **Self-healing** — auto-fix loop when verification finds gaps

## Installation

### Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed and authenticated
- Git installed (`git --version` to verify)

### Steps

1. Create the skills directory if it doesn't exist:

```bash
mkdir -p ~/.claude/skills/start-project
```

2. Copy the skill file:

```bash
cp SKILL.md ~/.claude/skills/start-project/SKILL.md
```

3. Verify it's installed — open Claude Code and type:

```
/start-project
```

You should see the skill activate and begin asking for project requirements.

### One-Liner Install

```bash
mkdir -p ~/.claude/skills/start-project && cp SKILL.md ~/.claude/skills/start-project/SKILL.md
```

## Usage

### New Project

```
/start-project
```

Select "New project", provide a directory, name, tech stack, and goal. The skill handles the rest.

### Feature Addition

```
/start-project
```

Select "Feature addition" and point it at an existing codebase. It will analyze the codebase before planning.

### Resume a Paused Session

```
/start-project
```

If STATE.md exists from a prior run, it will detect it and offer to resume.

## Model Profiles

Choose during setup to control cost vs. quality:

| Profile | Planning | Execution | Research/Verification | Best For |
|---------|----------|-----------|----------------------|----------|
| Quality | Sonnet | Sonnet | Sonnet | Production projects |
| Balanced | Sonnet | Sonnet | Haiku | Most projects (default) |
| Budget | Haiku | Sonnet | Haiku | Prototypes, experiments |

## Generated Files

The skill creates these files in your project directory:

| File | Purpose |
|------|---------|
| `PLAN.md` | Project plan, goals, components, requirements, success criteria |
| `CONTEXT.md` | Static project reference (stack, directory structure) |
| `STATE.md` | Execution state, progress tracking, session handoff |
| `DECISIONS.md` | Decision log with rationale and status |
| `RESEARCH.md` | Stack research findings and architectural guidance |
| `TEST_MAP.md` | Requirement-to-test coverage mapping |
| `UAT.md` | User acceptance testing results |
| `tasks/TASK-XX.md` | Individual task tickets with acceptance criteria |

## Phase Reference

| Phase | Name | Skipped in Quick Mode |
|-------|------|-----------------------|
| 0 | Resume Detection | No |
| 0.5 | Environment Validation | No |
| 1 | Gather Requirements | No |
| 2 | Discuss & Clarify | Yes |
| 3 | Domain Research | Yes |
| 4 | Dispatch Sub-Agents | No |
| 5 | Plan Verification | Yes |
| 6 | Present Results | No |
| 7 | Execute Tasks | No |
| 8 | Execution Summary | No |
| 9 | Goal-Backward Verification | No |
| 10 | User Acceptance Testing | No |

## License

MIT
