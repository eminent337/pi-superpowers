# pi-superpowers Design

Lightweight adaptation of [obra/superpowers](https://github.com/obra/superpowers) as a standalone pi package. Skills + one small extension. No dependency on aery-superteam.

## Background

Superpowers is a complete software development workflow for coding agents, built on composable "skills" — structured instructions for brainstorming, TDD, debugging, planning, code review, and more. The original targets Claude Code and uses its plugin system (hooks, commands, `Skill` tool, `TodoWrite` tool).

Pi's native skill system handles discovery, system prompt injection, and `/skill:name` commands out of the box. Most of the Claude-specific infrastructure becomes unnecessary. What remains is the skill content itself — the actual methodology — plus one small extension to replace `TodoWrite` with something better.

## Goals

- Faithful port of superpowers skill content to pi
- Self-contained package — works without aery-superteam
- Lighter than the original — no bootstrap injection, no command wrappers, no JS skill resolver
- One extension providing a plan tracker tool with TUI widget

## Non-Goals

- Automated subagent dispatch (that's aery-superteam's domain)
- Upstream tracking / fork synchronization
- Reproducing the `using-superpowers` bootstrap hook

---

## Package Structure

```
pi-superpowers/
├── package.json
├── README.md
├── LICENSE
├── docs/
│   └── plans/
│       └── 2026-02-08-pi-superpowers-design.md
├── extensions/
│   └── plan-tracker.ts
└── skills/
    ├── brainstorming/
    │   └── SKILL.md
    ├── dispatching-parallel-agents/
    │   └── SKILL.md
    ├── executing-plans/
    │   └── SKILL.md
    ├── finishing-a-development-branch/
    │   └── SKILL.md
    ├── receiving-code-review/
    │   └── SKILL.md
    ├── requesting-code-review/
    │   ├── SKILL.md
    │   └── code-reviewer.md
    ├── subagent-driven-development/
    │   ├── SKILL.md
    │   ├── implementer-prompt.md
    │   ├── spec-reviewer-prompt.md
    │   └── code-quality-reviewer-prompt.md
    ├── systematic-debugging/
    │   ├── SKILL.md
    │   ├── condition-based-waiting.md
    │   ├── condition-based-waiting-example.ts
    │   ├── defense-in-depth.md
    │   ├── find-polluter.sh
    │   └── root-cause-tracing.md
    ├── test-driven-development/
    │   ├── SKILL.md
    │   └── testing-anti-patterns.md
    ├── using-git-worktrees/
    │   └── SKILL.md
    ├── verification-before-completion/
    │   └── SKILL.md
    └── writing-plans/
        └── SKILL.md
```

### Excluded from upstream

| Upstream file | Reason excluded |
|---------------|-----------------|
| `using-superpowers/` | Replaced by distributed cross-references in each skill |
| `writing-skills/` | Meta-skill for contributing to superpowers itself |
| `commands/*.md` | Pi's `/skill:name` handles this natively |
| `hooks/` | Replaced by skill cross-references |
| `lib/skills-core.js` | Pi handles skill discovery natively |
| `.claude-plugin/` | Claude Code plugin metadata |
| `.opencode/`, `.codex/` | Other platform adapters |
| `agents/code-reviewer.md` | Agent dispatch is a aery-superteam concept; the prompt template at `requesting-code-review/code-reviewer.md` remains |
| `tests/` | Claude Code test harness, not portable |

### package.json

```json
{
  "name": "pi-superpowers",
  "version": "0.1.0",
  "description": "Superpowers workflow skills adapted for pi",
  "keywords": ["pi-package"],
  "license": "MIT",
  "author": "coctostan",
  "repository": {
    "type": "git",
    "url": "https://github.com/coctostan/pi-superpowers.git"
  },
  "pi": {
    "extensions": ["extensions/plan-tracker.ts"],
    "skills": ["skills"]
  },
  "peerDependencies": {
    "@mariozechner/pi-ai": "*",
    "@mariozechner/pi-coding-agent": "*",
    "@mariozechner/pi-tui": "*",
    "@sinclair/typebox": "*"
  }
}
```

---

## Skill Adaptation Rules

### Mechanical transforms

Every skill gets these search-and-replace transformations:

| Find | Replace | Rationale |
|------|---------|-----------|
| `superpowers:skill-name` | `skill-name` | No namespace prefix in pi |
| `Skill` tool / `Invoke Skill tool` | `/skill:name` or `read` tool | Pi's native mechanisms |
| `TodoWrite` / `Create TodoWrite` | `plan_tracker` tool | Extension provides this |
| `CLAUDE.md` | Project config (`.aery/settings.json`, `README.md`) | Pi equivalent |
| `Task tool` / `Task tool (general-purpose)` | "dispatch a subagent" (generic) | No assumption about team tool |
| `superpowers:code-reviewer` agent type | Reference to `code-reviewer.md` prompt template | Prompt template, not agent |

### Distributed skill awareness

Instead of `using-superpowers` injecting "always check skills" into every session, each skill gets a **Related Skills** blockquote after the frontmatter:

```markdown
> **Related skills:** Consider `/skill:brainstorming` before starting implementation.
```

Cross-references use `/skill:name` syntax (not relative file paths) so they work regardless of install location.

This costs ~50 bytes per skill instead of 3.8KB injected every turn. It appears only when the model has already loaded a skill, so it's contextually relevant.

Specific cross-references per skill:

| Skill | Related skills nudge |
|-------|---------------------|
| brainstorming | → using-git-worktrees, writing-plans |
| writing-plans | → brainstorming (designed first?), executing-plans / subagent-driven-development (ready to implement?) |
| executing-plans | → using-git-worktrees (isolated workspace?), verification-before-completion (verify each task), finishing-a-development-branch (done?) |
| subagent-driven-development | → using-git-worktrees, writing-plans, finishing-a-development-branch |
| test-driven-development | → verification-before-completion (run tests before claiming done) |
| systematic-debugging | → test-driven-development (write failing test for bug), verification-before-completion (actually fixed?) |
| verification-before-completion | → (none — this is a leaf skill) |
| dispatching-parallel-agents | → (none — standalone pattern) |
| requesting-code-review | → verification-before-completion (run tests before requesting review) |
| receiving-code-review | → (none — triggered by review feedback) |
| finishing-a-development-branch | → verification-before-completion (tests pass?), requesting-code-review |
| using-git-worktrees | → (none — infrastructure skill) |

### Subagent dispatch without aery-superteam

Skills referencing subagent dispatch (`subagent-driven-development`, `requesting-code-review`, `dispatching-parallel-agents`) use generic language:

- Keep prompt templates (`implementer-prompt.md`, etc.) unchanged — they're agent-agnostic reference docs
- Replace `Task tool` references with: "dispatch a subagent with the following prompt"
- The model can use whatever dispatch mechanism is available, or the user can run it manually
- If aery-superteam is installed, the model will naturally use the `team` tool

Each skill that references subagent dispatch includes a **Dispatch options** recipe block:

```markdown
**How to dispatch:**
- If a dispatch tool is available (e.g. aery-superteam's `team` tool), use it
- Otherwise, run a second aery instance: `aery -p "prompt from template"`
- For parallel tasks, use multiple terminal panes (one per task)
```

The skills degrade gracefully: structured workflow with or without automation.

### Content left untouched

The actual methodology — TDD cycle, debugging phases, brainstorming question flow, plan structure, review checklists, anti-patterns references. That's the value of superpowers and it's agent-agnostic.

Supporting reference files are copied as-is:
- `systematic-debugging/`: condition-based-waiting.md, defense-in-depth.md, root-cause-tracing.md, find-polluter.sh, condition-based-waiting-example.ts
- `test-driven-development/testing-anti-patterns.md`
- `requesting-code-review/code-reviewer.md`
- `subagent-driven-development/`: implementer-prompt.md, spec-reviewer-prompt.md, code-quality-reviewer-prompt.md

---

## Plan Tracker Extension

### Purpose

Replace `TodoWrite` (Claude-specific) and the "write a markdown tracking file" workaround with a native pi tool that stores state in the session and shows progress in the TUI.

### Tool definition

One tool, three actions:

```
plan_tracker({ action: "init", tasks: ["Task 1: Hook installation", "Task 2: Recovery modes", ...] })
plan_tracker({ action: "update", index: 0, status: "complete" })
plan_tracker({ action: "status" })
plan_tracker({ action: "clear" })
```

**`init`** — Set task list from a plan. Clears any previous state. Returns confirmation with task count.

**`update`** — Set a task's status to `pending`, `in_progress`, or `complete`. Returns updated status summary.

**`status`** — Return current task list with statuses. Useful for the model to check progress without file I/O.

**`clear`** — Remove all state and dismiss the widget. Use when a plan is abandoned or complete.

### State management

- Stored via `aery.appendEntry("plan_tracker", { tasks, updated })` — persists in session, survives restarts, works with branching/forking
- Restored on `session_start`, `session_switch`, `session_fork`, and `session_tree` by walking `ctx.sessionManager.getBranch()` for the latest `plan_tracker` entry
- No file I/O, no tracking file to manage

### TUI widget

Persistent widget above the editor:

```
Tasks: ✓✓→○○ (2/5)  Task 3: Recovery modes
```

- `✓` complete, `→` in progress, `○` pending
- Shows current (in-progress or next pending) task name
- Updated on every `plan_tracker` call via `ctx.ui.setWidget()`
- Cleared when no plan is active (`ctx.ui.setWidget("plan_tracker", undefined)`)

### What it doesn't do

- No enforcement — doesn't block the model from skipping tasks
- No plan file parsing — model extracts tasks and calls `init`
- No integration with `team` tool — that's aery-superteam's domain
- No file output — state lives in session only

### Cost advantage

One `plan_tracker` tool call replaces three tool calls (read tracking file + edit + write) or the model trying to keep state in its context window. Cheaper in tokens, faster in execution.

---

## Attribution

Skill content adapted from [Superpowers](https://github.com/obra/superpowers) by Jesse Vincent, licensed under MIT.

LICENSE file carries dual copyright:

```
MIT License

Copyright (c) 2026 coctostan
Copyright (c) 2025 Jesse Vincent (original skill content from obra/superpowers)
```
