# Claude Code × Codex Workflow — Technical Architecture Document
**Version:** 1.0 | **Date:** March 2026 | **Audience:** Engineering Team

---

## 1. The Problem This Solves

### Before This Pipeline

Modern software tasks outgrow what a single AI context window can handle reliably. A single Claude or Codex session working on a complex feature hits three fundamental limits:

1. **Context contamination** — Exploration, planning, and implementation all share the same context window. Early assumptions pollute later decisions.
2. **Optimism bias** — A single model plans and reviews its own work. It is unlikely to critique its own decisions with genuine adversarial depth.
3. **Sequential bottleneck** — All tasks run one at a time, even when they have no dependencies on each other.

### What This Pipeline Fixes

| Problem | Solution |
|---|---|
| One model plans and reviews itself | Claude plans, Codex reviews — adversarial pair |
| Context contamination | Sub-agents get isolated context windows per task |
| Sequential execution | Independent tasks delegated to Codex cloud sandboxes in parallel |
| Autonomous execution with no human gate | Stop hook enforces stakeholder approval before any code runs |
| No audit trail | PreToolUse HTTP hook logs every tool invocation |

---

## 2. Architecture Overview

```
User Task Input
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│                   PHASE 1: PLANNING                     │
│   Claude Code (Plan Mode)                               │
│   • Reads codebase via Explore sub-agent                │
│   • Decomposes task → plan.md                           │
│   • Identifies task clusters + dependencies             │
│   PostToolUse hook fires on Write(plan.md) ────────────►│
└─────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│                   PHASE 2: REVIEW                       │
│   Codex CLI (Review Mode)                               │
│   • Reads plan.md                                       │
│   • Outputs structured critique → codex-review.md      │
│   SubagentStop hook returns critique to Claude ◄────────│
└─────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│               PHASE 3: ITERATIVE DIALOGUE               │
│   Claude refines plan → PostToolUse fires → Codex       │
│   reviews again → SubagentStop returns → loop           │
│   Exit condition: all review items resolved             │
│   Max iterations: 3 (configurable)                      │
└─────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│               PHASE 4: STAKEHOLDER GATE                 │
│   Stop hook returns ok: false                           │
│   Checks for existence of plan.approved.md              │
│   /approve  →  writes plan.approved.md  →  unblocks     │
│   /reject   →  appends note  →  restarts loop           │
└─────────────────────────────────────────────────────────┘
      │  (only after /approve)
      ▼
┌─────────────────────────────────────────────────────────┐
│               PHASE 5: /go — EXECUTION                  │
│   Claude reads plan.approved.md                         │
│   Spawns N sub-agents (one per task cluster)            │
│   Each sub-agent: isolated context, scoped tools,       │
│   defined file-system boundaries                        │
└─────────────────────────────────────────────────────────┘
      │
      ├──────────────────────────────────────────────────┐
      ▼                                                  ▼
┌───────────────┐                         ┌──────────────────────┐
│  Claude       │                         │  Codex (optional)    │
│  Sub-agents   │                         │  Delegated tasks     │
│  (local ctx)  │                         │  (cloud sandbox)     │
│  auth / db /  │                         │  independent tasks   │
│  api clusters │                         │  user-approved       │
└───────────────┘                         └──────────────────────┘
      │                                          │
      └──────────────┬───────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────────┐
│               PHASE 6: /progress-report                 │
│   UserPromptSubmit hook intercepts command              │
│   progress-agg.sh reads .claude/tmp/progress/*.json     │
│   Renders TUI bar: ████████░░ 78% [3/4 tasks]           │
│   TaskCompleted → git checkpoint + optional Slack notify│
└─────────────────────────────────────────────────────────┘
```

---

## 3. Component Deep Dive

### 3.1 Claude Code — Planner + Orchestrator

Claude Code is the **orchestration layer**. Its key architectural properties:

- **Plan Mode**: Read-only reasoning phase. Cannot write files. Forces explicit planning before execution.
- **15 Hook Events**: Lifecycle events at every stage (SessionStart, PreToolUse, PostToolUse, SubagentStop, Stop, etc.). Hooks fire deterministically — they cannot be bypassed by model output.
- **Sub-agent spawning**: Each sub-agent runs in an isolated context window. Memory is not shared. Data must be passed explicitly via files.
- **Model routing**: Opus for planning and orchestration; Sonnet/Haiku for execution sub-agents. Cost-critical design decision.

**Critical property**: The `Stop` hook returning `ok: false` is **not a suggestion** — it is an enforced block at the execution layer. The model cannot override it.

### 3.2 Codex CLI — Reviewer + Parallel Worker

Codex CLI is the **adversarial reviewer and parallel execution engine**:

- **Review mode** (`--review`): Structured file-in / critique-out workflow. Designed for hook integration.
- **Sandboxing**: macOS Seatbelt / Linux Landlock OS-level process isolation. Network disabled by default. This is not a prompt-level restriction — it is OS-enforced.
- **MCP server mode** (`codex mcp start`): Codex exposes itself as an MCP server, allowing Claude to call it as a native tool rather than a shell script bridge. This is the preferred integration path for production.
- **Async cloud tasks**: Long-running tasks can run in Codex cloud sandboxes while the local terminal remains responsive.

### 3.3 Hook Bridge

The connection between Claude and Codex runs through shell scripts invoked by Claude's hook system:

```
Claude writes plan.md
      │
      ▼
PostToolUse hook fires
      │
      ▼
codex-review.sh called with JSON stdin
{
  "tool": "Write",
  "tool_input": { "file_path": "plan.md", "content": "..." }
}
      │
      ▼
Script extracts file_path, passes to Codex CLI
      │
      ▼
Codex outputs codex-review.md
      │
      ▼
Script returns {"continue": true} to Claude
      │
      ▼
SubagentStop hook injects codex-review.md as new agent prompt
```

### 3.4 Sub-agent Memory Architecture

Sub-agents in this pipeline use **two memory patterns**:

1. **Ephemeral context** (default): Sub-agent context is discarded on stop. Results must be written to files for the parent to consume.
2. **Persistent memory** (optional): Sub-agents can be given a `memory:` directory in their definition. Knowledge of codebase patterns, architectural decisions, and debugging insights persists across sessions.

**Critical isolation rule**: Sub-agents in a hub-and-spoke model do not share memory. If the orchestrator knows a user ID or session token, it must explicitly pass that value in the sub-agent's prompt. Implicit inheritance does not exist.

---

## 4. Fallback Architecture

The pipeline is designed with layered fallbacks for every failure mode:

### 4.1 Hook Failures

| Failure | Fallback |
|---|---|
| `codex-review.sh` exits non-zero | Hook returns `{"continue": true}` — Claude proceeds without review |
| Codex CLI not installed | Script checks for binary; if absent, writes "CODEX_UNAVAILABLE" to status file; Claude skips review phase |
| Codex takes > 120s | Hook timeout fires, returns continue — pipeline does not stall |
| HTTP audit hook unreachable | Fire-and-forget pattern; audit failure does not block execution |

### 4.2 Review Loop Fallbacks

| Failure | Fallback |
|---|---|
| Dialogue loop exceeds max iterations | Surface to human with summary of unresolved items |
| Claude and Codex in persistent disagreement | Human arbitration flag written to `.claude/tmp/review-status` |
| plan.md is malformed | Codex returns structured error; Claude re-attempts plan generation |

### 4.3 Sub-agent Failures

| Failure | Fallback |
|---|---|
| Sub-agent exits with error | Status JSON written as `{"status": "failed", "error": "..."}` — picked up by `/progress-report` |
| Sub-agent modifies out-of-scope files | PreToolUse hook validates file paths; blocks and returns error to sub-agent |
| Codex delegation fails | Task falls back to Claude local execution; user notified |
| Rate limit hit | Exponential backoff with jitter in bridge script; max 3 retries |

### 4.4 Gate Fallbacks

| Failure | Fallback |
|---|---|
| `plan.approved.md` corrupted | Stop hook validates file checksum; rejects corrupted approvals |
| Approval issued for wrong plan version | File includes plan hash; mismatch triggers re-approval request |

---

## 5. Security Architecture

### 5.1 Threat Model

This pipeline executes code autonomously. The threat surface includes:

- **Prompt injection via file content**: Malicious content in reviewed files could attempt to redirect agent behavior
- **Scope creep**: Sub-agents with broad tool access modifying files outside their scope
- **Credential exfiltration**: Agents reading `.env` files or SSH keys
- **RCE escalation**: Hook scripts calling shell commands that escalate privileges

### 5.2 Controls

```
Layer 1 — OS: Codex sandbox (Seatbelt/Landlock)
Layer 2 — Hook: PreToolUse blocks rm -rf, curl | bash, sudo, cat .env
Layer 3 — Prompt: Sub-agent system prompts explicitly scope file access
Layer 4 — Gate: Stop hook enforces human approval before any execution
Layer 5 — Audit: Every tool invocation logged via HTTP hook
Layer 6 — Git: TaskCompleted hook commits checkpoint for full rollback
```

### 5.3 Known Vulnerability (Feb 2026)

Security researchers identified an RCE risk in Claude Code agentic workflows where hook scripts could be manipulated via crafted file content. Anthropic issued patches. **Mitigation**: Always validate hook inputs; never pass file content directly to shell `eval`; use `--dangerously-skip-permissions` only in isolated CI sandboxes.

---

## 6. Cost Architecture

### 6.1 Model Routing Strategy

Running Opus for all operations is unsustainable at scale. Recommended routing:

| Phase | Model | Reason |
|---|---|---|
| Planning | claude-opus-4-6 | Long-horizon reasoning required |
| Review dialogue | claude-sonnet-4-6 | Sufficient for critique synthesis |
| Sub-agents (execution) | claude-haiku-4-5 | Fast, cheap, scoped tasks |
| Codex review | gpt-5.2-codex | Repo-scale review default |

### 6.2 Cost Controls

- **Max iterations**: Set to 3 on review loops. Beyond that, token cost outweighs marginal plan improvement.
- **Sub-agent tool scope**: Restrict to minimum tools needed. Read-only agents should never have `Write` or `Bash`.
- **Codex delegation**: Only delegate truly independent tasks. Context-passing overhead negates savings for short tasks.

---

## 7. Progress Tracking — Implementation Detail

### 7.1 Event-Driven Design (No Polling)

```
Agent completes task
      │
      ▼
SubagentStop hook fires
      │
      ▼
Agent writes .claude/tmp/progress/{agent-id}.json
{
  "agent": "auth",
  "status": "done",         // done | running | blocked | failed
  "tasks_total": 3,
  "tasks_done": 3,
  "blocked_on": null,
  "est_remaining_ms": 0,
  "last_updated": "2026-03-24T09:16:01Z"
}
      │
      ▼
/progress-report → UserPromptSubmit hook intercepts
      │
      ▼
progress-agg.sh reads all .json files, renders TUI
```

### 7.2 Why Event-Driven vs. Polling

- Zero overhead when agents are idle
- Instant update on completion (SubagentStop fires immediately)
- No shared mutable state — each agent owns its own status file
- Git checkpoint on TaskCompleted provides full rollback at each stage

---

## 8. Operational Runbook

### First Run Checklist
- [ ] `npm i -g @openai/codex`
- [ ] Claude Code installed (anthropic.com/claude-code)
- [ ] `.claude/settings.json` hook config in place
- [ ] `.claude/hooks/codex-review.sh` executable (`chmod +x`)
- [ ] `codex mcp start &` running (or add to project startup script)
- [ ] `claude mcp add codex -- npx @openai/codex mcp start`
- [ ] `.env` excluded from agent tool access via PreToolUse hook
- [ ] Audit HTTP endpoint running on `localhost:9000`

### Day-to-Day Workflow
```
/plan "<task description>"   → Claude decomposes, writes plan.md
                             → Codex review loop runs automatically
/approve                     → Review plan.refined.md, approve
/go                          → Sub-agents spawn, execution begins
/progress-report             → Check status any time
/delegate <task-id>          → Optionally move a task to Codex
```

---

*Document generated for internal team use. Review security section before production deployment.*
