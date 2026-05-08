# Claude Code Internals — Reference Card
> Compiled from deep-dive architectural discussions. Fintech-oriented. Every insight retained.

---

## Table of Contents
1. [Primitive Determinism Spectrum](#1-primitive-determinism-spectrum)
2. [Hooks](#2-hooks)
3. [CLI Flags & Environment Variables](#3-cli-flags--environment-variables)
4. [Slash Commands](#4-slash-commands)
5. [Subagents](#5-subagents)
6. [Skills](#6-skills)
7. [MCP Tools](#7-mcp-tools)
8. [CLAUDE.md — The Persistence Anchor](#8-claudemd--the-persistence-anchor)
9. [append-system-prompt — Scope & Limits](#9-append-system-prompt--scope--limits)
10. [Model Selection & Routing](#10-model-selection--routing)
11. [Context & Compaction Mechanics](#11-context--compaction-mechanics)
12. [Fintech Architecture Patterns](#12-fintech-architecture-patterns)
13. [Open Bugs & Feature Gaps](#13-open-bugs--feature-gaps)

---

## 1. Primitive Determinism Spectrum

The most important architectural axis in Claude Code is **"inside vs. outside the LLM's reasoning chain"**. Everything outside is deterministic. Everything inside is probabilistic to varying degrees.

### Ordered from most → least deterministic

| Rank | Primitive | Invocation Determinism | Execution Determinism |
|------|-----------|----------------------|----------------------|
| 1 | **Hooks (shell type)** | ✅ 100% — event-driven, system-level | ✅ 100% — shell script |
| 2 | **CLI flags / env vars** | ✅ 100% — resolved at process startup | ✅ 100% |
| 3 | **Slash Commands** (`disable-model-invocation: true`) | ✅ User-triggered gesture | ✅ Deterministic prompt injection |
| 4 | **Subagents** | ⚠️ LLM description matching for invocation | ✅ Isolated, bounded return |
| 5 | **Skills (auto-invoked)** | ❌ LLM semantic match — no algorithmic trigger | ⚠️ LLM follows instructions |
| 6 | **MCP tool selection** | ❌ LLM semantic match | ✅ Deterministic tool execution |
| — | **Hooks (LLM/agent type)** | ✅ Event-driven | ❌ LLM reasoning inside hook |

### The key architectural rule
> **For anything that MUST happen — compliance gates, PII masking, audit logging — use shell hooks. Everything else is probabilistic to some degree.**

Prompt-based instructions (CLAUDE.md, system prompt, skills) achieve ~70–90% compliance. Shell hooks achieve 100%. The gap matters in regulated environments.

---

## 2. Hooks

### What they are
Shell commands that fire at specific lifecycle moments, **completely outside the LLM's reasoning chain**. Claude cannot skip, forget, or decide otherwise.

### Types
| Type | Execution | Use Case |
|------|-----------|----------|
| `type: command` | Shell script — fully deterministic | Linting, blocking, logging, notifications |
| `type: prompt` | LLM prompt — probabilistic | Soft evaluation, advisory checks |
| `type: agent` | Spawns a full Claude model with tools | Complex multi-turn verification before proceeding |

### Lifecycle events
- **PreToolUse** — runs before a tool call; can block it (exit code 2 cancels the call)
- **PostToolUse** — runs after a tool call; can inject `additionalContext` back into Claude's context; cannot block (after the fact)

### Key behaviors
- PreToolUse with exit code 2 = hard block on the tool call
- PostToolUse injects errors/results back as `additionalContext` — Claude sees and reacts to them
- Hooks fire per subagent **only if explicitly embedded** in that subagent's definition — parent hooks do NOT propagate to spawned subagents
- Agent-type hooks reintroduce LLM non-determinism inside the hook itself

### Config location
```json
// .claude/settings.local.json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Edit", "command": "npm run lint --fix $FILE" }
    ],
    "PostToolUse": [
      { "matcher": "Write", "command": "./scripts/audit-log.sh $FILE" }
    ]
  }
}
```

### Fintech use cases
- **PreToolUse**: Block SQL writes to PII tables without approved CDD flag
- **PostToolUse**: Inject SAR filing validation result into Claude's context after every report generation
- **PreToolUse**: Block `bash` tool from executing destructive commands (DROP, DELETE without WHERE)
- **PostToolUse**: Auto-log every MCP tool call to immutable audit trail

---

## 3. CLI Flags & Environment Variables

### Why CLI sits above Commands in determinism
CLI flags are resolved at **process startup** — before any LLM call is made. They govern model selection, system prompt assembly, permission modes, and session configuration. Nothing in prompt content or agent reasoning can override them.

### Key flags
| Flag | Effect |
|------|--------|
| `--model <name>` | Sets the model for the session |
| `--append-system-prompt <text>` | Appends text to the end of the assembled system prompt |
| `--permission-mode` | Controls what tools are accessible |
| `--agent <name>` | Launches session as a named subagent |
| `--no-tools` | Disables all tool use |

### Key environment variables
| Variable | Effect |
|----------|--------|
| `ANTHROPIC_MODEL` | Default model |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Opus model alias |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Sonnet model alias |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Haiku model alias |

---

## 4. Slash Commands

### What they are
Prompt templates stored as `.md` files in `.claude/commands/`. A file at `.claude/commands/review.md` creates the `/review` command. They have been merged into the Skills system — a slash command is effectively a skill with a user-invocable trigger.

### Determinism control: the critical flag
```yaml
---
disable-model-invocation: true   # Manual-only — user must type /command-name
                                  # Without this flag, Claude can auto-invoke = non-deterministic
---
```

### Invocation modes
| Mode | Trigger | Determinism |
|------|---------|-------------|
| `disable-model-invocation: true` | Only when user types `/name` | ✅ User-deterministic |
| Auto-invocation enabled | Claude decides based on task context | ❌ LLM semantic match |

### Argument substitution
- `$ARGUMENTS` — all arguments as string
- `$0`, `$1`, `$2` — positional args
- `${CLAUDE_SKILL_DIR}` — skill's directory path

### Rule for compliance-critical workflows
Always use `disable-model-invocation: true` for SAR filing triggers, CDD check invocations, and any workflow where the trigger must be explicit and auditable.

---

## 5. Subagents

### What they are
Isolated Claude instances spawned for specialized tasks. Defined in `.claude/agents/<name>.md`. Each invocation creates a **fresh context** — no memory of previous invocations unless explicitly resumed.

### Context isolation — the most important property
**Named subagents (spawned via Agent/Task tool mid-session) do NOT inherit:**
- Parent's system prompt
- Parent's `--append-system-prompt` content
- CLAUDE.md (this is a filed bug — not intended behavior)
- Parent's hooks (hooks must be embedded per-subagent)

**`context: fork` subagents DO inherit:**
- Full conversation history at the moment of forking
- Same system prompt
- Same tools and model

### Lifecycle
```
Spawned → Fresh context assembled from subagent .md → Task executes → 
Summary (1,000–2,000 tokens) returned to parent → Process terminates
```

- Subagent process terminates when task completes
- Transcript persisted to `~/.claude/projects/{project}/{sessionId}/subagents/`
- Resumed subagents retain their full conversation history including all previous tool calls

### Invocation determinism
- **Subagent definitions (descriptions)** — loaded into system prompt at startup → **never lost to compaction**
- **When Claude invokes a subagent** — LLM description matching → non-deterministic
- Result: invocation rules persist, but invocation timing does not

### Compaction behavior for subagent results
- Subagent summary (1–2k tokens) lands in **parent's conversation history**
- Conversation history IS subject to compaction
- Documented behavior: in long sessions with many subagents, only the 3–4 most recent subagent results survived compaction; earlier results were gone
- Subagent descriptions (in system prompt) — always survive compaction

### Subagent frontmatter
```yaml
---
name: datalake-agent
description: Text-to-SQL agent for internal datalake queries. Invoked when user asks to query financial data, run analytics, or generate SQL against the datalake.
model: claude-sonnet-4-6
skills:
  - sql-conventions       # preloaded at startup — never missed
  - pii-masking           # preloaded at startup
mcpServers:
  - datalake-mcp          # scoped to this subagent only
tools:
  - mcp__datalake__query
  - read_file
disallowedTools:
  - bash                  # prevent raw shell access
---
```

### Preloading skills in subagents
List skills in frontmatter `skills:` array → full skill content injected at startup, not lazy-loaded. Eliminates the "Claude forgot to invoke the skill mid-session" problem.

**Trade-off:** Each preloaded skill consumes tokens upfront. One documented case: 22k+ tokens (11% of context window) consumed at startup with full preloading. Be selective.

### Parallel execution
Subagents can be spawned in parallel. Parent orchestrator delegates to multiple specialists simultaneously. Each has isolated context — no cross-contamination.

---

## 6. Skills

### What they are
Reusable prompt/instruction bundles stored as `.md` files in `.claude/skills/`. Invoked either by Claude automatically (semantic matching) or explicitly by user (`/skill-name`) or by other agents.

### Invocation — the non-determinism problem
> "Claude decides using its own semantic understanding of your request — there's no algorithmic matching, it's LLM reasoning about whether your intent matches a skill description."

- No override mechanism to force invocation
- Cannot prevent invocation when Claude decides it matches
- No visibility into invocation decisions (silent)
- Unpredictable context consumption

**Practical consequence:** Even a perfectly written skill description has irreducible non-determinism. For compliance-critical skills (PII masking, SAR validation), use `disable-model-invocation: true` + explicit slash command invocation.

### Context permanence — the key limitation
> **Once a skill is loaded into context, it cannot be removed. There is no eviction mechanism.**

- Skill content stays in context for the entire remaining session
- Every line is a recurring token cost on every subsequent LLM call
- This is an **open feature request** — skill unloading does not exist yet
- `context: fork` does NOT help with this (see below)

### What `context: fork` actually does (and doesn't do)
**Does:** Heavy output generated *during skill execution* (bash results, file reads) stays in the subagent's context, not the parent's. Prevents verbose tool outputs from bloating the main thread.

**Does NOT:** Prevent the SKILL.md content itself from loading into the parent context. The skill body is still injected into the parent conversation when invoked. Fork only isolates execution side-effects, not the skill definition.

### Compaction behavior for skills
- Auto-compaction re-attaches skills after summary
- Keeps **first 5,000 tokens** of each skill
- All skills share a **combined budget of 25,000 tokens** after compaction
- Budget filled starting from most recently invoked skill
- Older skills can be **dropped entirely** after compaction if many skills were invoked
- Net effect: large skills get truncated to 5k, not evicted. Small skills survive intact.

### Skill design rules for context efficiency
1. Keep SKILL.md body under 500 lines — every line costs tokens for the rest of the session
2. State what to do, not how or why (imperative, not explanatory)
3. Move reference material (schemas, rule tables) into supporting files Claude reads on-demand during execution, not into the SKILL.md body itself
4. Use `user-invocable: false` for background knowledge skills (domain context, regulatory rules) — hides from menu but still injectable

### Frontmatter reference
```yaml
---
name: sql-pii-masker
description: >
  Invoked when generating SQL queries against tables containing PII fields.
  Triggers: customer_data, account_holder, ssn, date_of_birth, address fields.
  Applies column-level masking and ensures SELECT * is never used on sensitive tables.
disable-model-invocation: false    # true = manual-only via /skill-name
user-invocable: true               # false = background knowledge only, hidden from menu
context: fork                      # run in subagent, return summary only (isolates verbose output)
skills:                            # this skill depends on another skill
  - data-classification
---
```

### `skillOverrides` in settings
Controls skill visibility without editing the SKILL.md file directly. Useful for shared repo skills you don't own:
```json
// .claude/settings.local.json
{
  "skillOverrides": {
    "sql-pii-masker": "on",
    "legacy-reporter": "off"
  }
}
```

---

## 7. MCP Tools

### Two distinct layers — different determinism
| Layer | Determinism | Set by |
|-------|-------------|--------|
| Tool availability (which servers connected, which tools registered) | ✅ Deterministic | Config at session startup |
| Tool selection (which tool Claude calls for a task) | ❌ LLM semantic match | Claude's reasoning |

### Progressive disclosure (ToolSearch)
When ToolSearch is enabled, only tool *names* are loaded into the initial context. Full schemas are loaded on-demand when Claude decides to use a tool. This is the MCP equivalent of skill descriptions vs. full SKILL.md bodies.

**Key parallel with skills:** Once a full MCP tool schema is loaded, it cannot be evicted mid-session. Same append-only constraint as skills.

### Context permanence — same problem as skills
MCP tool definitions, once loaded, persist for the session. The mitigation strategy is identical:
- Scope MCP servers per-subagent via `mcpServers:` frontmatter rather than globally
- Don't connect every MCP server at session start — only what that session actually needs
- Use subagent isolation to prevent parent context from accumulating unused MCP schemas

### Scoping MCP to subagents
```yaml
# In subagent .md frontmatter
---
mcpServers:
  - datalake-mcp        # only this subagent gets datalake access
  - compliance-check-mcp
---
```
This prevents the parent session's context from being polluted with tool schemas irrelevant to the main thread.

---

## 8. CLAUDE.md — The Persistence Anchor

### What it is
The most reliable persistence layer in Claude Code. Content re-injected fresh after every compaction event by re-reading from disk. Survives: model switches, context compaction, session resumes.

### What it does NOT do
- Does **not** propagate to named subagents spawned via Agent/Task tool (filed bug — not intended)
- Does **not** propagate into `--agent` sessions in the same way (subagent system prompt replaces CC system prompt)
- Is **not** a hard enforcement mechanism — it's guidance, not a gate

### Hierarchy of CLAUDE.md files
Claude Code merges CLAUDE.md files from multiple levels:
```
~/.claude/CLAUDE.md          # user-global (always loaded)
<project-root>/CLAUDE.md     # project-level
<subdirectory>/CLAUDE.md     # directory-scoped (loaded when Claude works in that dir)
```

### What belongs in CLAUDE.md vs. where else
| Content type | Best location |
|--------------|---------------|
| Always-on coding conventions | CLAUDE.md |
| Naming standards, data classifications | CLAUDE.md |
| MCP server references | CLAUDE.md |
| Compliance rules that MUST fire | Shell hooks (not CLAUDE.md) |
| Session-specific tone/behavior | `--append-system-prompt` |
| Subagent-specific compliance rules | Subagent system prompt body (copy from CLAUDE.md) |

### Survival after compaction
- CLAUDE.md content: **always survives** (re-read from disk and re-injected)
- Instructions only in chat history: **vulnerable to compaction loss**
- Skills in context: **partially survives** (first 5k tokens, 25k shared budget)
- Subagent results in conversation history: **vulnerable to compaction loss**

---

## 9. `--append-system-prompt` — Scope & Limits

### What it does
Appends custom text to the very end of the assembled system prompt. Resolved at session startup. Because it's in the system prompt layer (not conversation history), it is **not subject to compaction loss**.

### Persistence across session events
| Event | append-system-prompt persists? |
|-------|-------------------------------|
| `/model` switch in main thread | ✅ Yes — system prompt unchanged |
| Skill invoked in main thread | ✅ Yes — skill rides on top |
| Context compaction | ✅ Yes — system prompt layer, not history |
| Named subagent spawned (Agent tool) | ❌ No — subagent has its own system prompt |
| `--agent` session | ❌ No — subagent prompt replaces CC system prompt |
| `context: fork` | ✅ Yes — fork inherits everything |

### What it IS good for
- Session-wide behavioral tone (e.g., output format, verbosity, domain conventions)
- Hard compliance statements for the main thread
- Any instruction that must outlast context compaction

### What it CANNOT do
- Control model selection (model is chosen before prompt is evaluated — wrong layer)
- Propagate instructions into spawned subagents
- Enforce anything (it's still guidance inside the LLM's reasoning chain)

### Model routing via prompt text — why it fails
Instructing Claude via prompt "use Haiku for research, Sonnet for coding" asks an already-instantiated model to swap itself out. Model selection happens at the infrastructure layer before any LLM call. The correct mechanisms:
- **During session**: `/model <name>` or `/model opusplan`
- **Session launch**: `--model` CLI flag
- **Environment**: `ANTHROPIC_MODEL`, `ANTHROPIC_DEFAULT_*` env vars
- **opusplan mode**: `/model opusplan` — Claude uses Opus for planning, auto-switches to Sonnet for execution (native plan/execute split)
- **Subagent spawn**: each subagent can specify its own `model:` in frontmatter — this is the correct pattern for task-aware model routing

---

## 10. Model Selection & Routing

### Correct mechanisms (in priority order)
1. `ANTHROPIC_MODEL` env var (session default)
2. `--model` CLI flag (session override)
3. `/model <name>` during session (immediate switch, persists for session)
4. `/model` with no args (opens interactive picker)
5. Subagent `model:` frontmatter (per-subagent routing)

### opusplan mode
`/model opusplan` activates a hybrid mode:
- Opus for complex reasoning, architectural decisions, planning
- Auto-switches to Sonnet for code generation and implementation
- Closest native equivalent to task-aware model routing without manual switching

### For fintech subagent architectures
The correct pattern for task-aware model routing is subagent `model:` frontmatter:
```yaml
# Planning subagent
---
name: architecture-planner
model: claude-opus-4-6
---

# Implementation subagent
---
name: sql-generator
model: claude-sonnet-4-6
---

# Research/exploration subagent
---
name: schema-explorer
model: claude-haiku-4-5-20251001
---
```
This is deterministic — model is set at subagent definition time, not at LLM reasoning time.

---

## 11. Context & Compaction Mechanics

### The fundamental constraint: context is append-only
Within a session, nothing is removed from context — only summarized or truncated by compaction. Skills, MCP schemas, subagent results, CLAUDE.md — all append. Design for this.

### What compaction does
- Triggered automatically when context approaches limit (or manually via `/compact`)
- Summarizes conversation history into a compressed summary
- Re-attaches CLAUDE.md fresh from disk
- Re-attaches invoked skills (first 5k tokens each, 25k total shared budget — most recent first)
- Subagent results in history may be dropped (older ones lost first)
- System prompt (including `--append-system-prompt`) is reassembled — survives

### Compaction survival table
| Content | Survives compaction? | Notes |
|---------|---------------------|-------|
| CLAUDE.md | ✅ Always | Re-read from disk and re-injected |
| `--append-system-prompt` | ✅ Always | System prompt layer |
| Subagent descriptions (in system prompt) | ✅ Always | System prompt layer |
| Skills (invoked) | ⚠️ Partial | First 5k tokens, 25k shared budget |
| Subagent results (in history) | ❌ Vulnerable | Older ones dropped first |
| Chat history | ❌ Summarized | Details can be lost |

### Custom compaction prompt
```json
// .claude/settings.local.json
{
  "compactPrompt": "Preserve: all compliance decisions, SQL query patterns, PII classifications, subagent outcomes. Summarize: exploration and reasoning steps."
}
```

### Context window pressure: practical mitigations
| Problem | Mitigation |
|---------|-----------|
| Skill content permanent after load | Keep SKILL.md under 500 lines; move reference data to on-demand files |
| MCP schemas permanent after load | Scope MCP servers to subagents, not globally |
| Verbose skill execution output | `context: fork` (isolates execution output in subagent, not the skill body itself) |
| Heavy one-shot skills | `context: fork` — subagent does the work, returns only summary |
| Subagent results lost to compaction | Custom `compactPrompt` to prioritize retention; design for re-invocability |

---

## 12. Fintech Architecture Patterns

### Recommended primitive sequencing
```
CLAUDE.md → MCP → Skills → PostToolUse Hooks → Scoped Subagents → Compliance Plugin → Agent Teams
```

### Hard enforcement vs. best-practice guidance
| Layer | Type | Example |
|-------|------|---------|
| Shell Hooks | Hard enforcement — 100% | Block SQL writes to PII tables; mandatory audit logging |
| CLI / env vars | Infrastructure control | Model selection, permission modes |
| CLAUDE.md | Persistent guidance — ~85% | Coding conventions, data classification labels |
| Subagent system prompt | Scoped guidance | Compliance rules for that specific agent's domain |
| Skills | Soft guidance — ~70% | Style preferences, non-mandatory workflow steps |

### Critical gap: CLAUDE.md does not propagate to spawned subagents
**Implication:** Compliance rules in CLAUDE.md are silently absent in dynamically spawned subagents. For a regulated environment, compliance-critical rules must be:
1. Embedded explicitly in each subagent's system prompt body
2. Enforced via shell hooks embedded in each subagent's hook config
3. Never assumed to propagate from the parent session

### Text-to-SQL agent blueprint
```
Session startup:
  --append-system-prompt: "Fintech compliance context. Never expose raw PII. All queries must include audit metadata."
  
CLAUDE.md:
  - Data classification labels (PII, PCI, public)
  - Always-on MCP server references
  - Naming conventions

Datalake subagent (.claude/agents/datalake-agent.md):
  model: claude-sonnet-4-6
  skills: [sql-conventions, pii-masking]   # preloaded, never missed
  mcpServers: [datalake-mcp]               # scoped, not global
  system prompt body: [copy of compliance rules — don't rely on CLAUDE.md propagation]

Hooks:
  PreToolUse (matcher: mcp__datalake__query):
    - Validate query does not SELECT * on sensitive tables
    - Block if CDD flag not set for account-level queries
  PostToolUse (matcher: mcp__datalake__query):
    - Log query + result metadata to immutable audit trail
    - Inject row count and sensitivity classification into Claude's context

Skills:
  sql-conventions: disable-model-invocation: false (auto-invoke — verbose description with table name triggers)
  pii-masking: disable-model-invocation: true (manual-only — must be explicitly called for auditability)
  regulatory-context (FATF, BSA): user-invocable: false (background knowledge, silent)
```

### Key fintech-specific architecture rules
1. **Hooks are the only true compliance enforcement layer** — everything else is advisory
2. **Each subagent must carry its own compliance rules** — parent context does not propagate
3. **Model routing belongs in subagent frontmatter** — not in prompt instructions
4. **`disable-model-invocation: true` for any skill whose invocation must be auditable** — LLM semantic matching is not auditable
5. **Scope MCP servers per subagent** — prevent context accumulation and enforce least-privilege access
6. **SAR filing, CDD/EDD checks, PEP screening** → shell hooks + explicit slash commands, never auto-invoked skills

---

## 13. Open Bugs & Feature Gaps

| Issue | Status | Workaround |
|-------|--------|-----------|
| CLAUDE.md does not propagate to named subagents spawned via Agent tool | Filed bug — not intended behavior | Explicitly embed rules in each subagent's system prompt body |
| Skills cannot be unloaded/evicted from context | Open feature request | Keep SKILL.md small; use `context: fork` for output isolation; selective preloading |
| Subagent hooks do not inherit from parent session | By design / known gap | Embed hooks in each subagent's definition |
| Older subagent results lost to compaction | Known behavior | Custom `compactPrompt`; design workflows to be re-invocable |
| Subagent processes not terminated on Linux when parent ends (orphaned processes, ~400MB RAM each) | Known bug | Manually monitor and kill orphaned Claude Code processes |
| Skill auto-invocation is non-deterministic — no algorithmic matching | By design | `disable-model-invocation: true` + explicit slash commands for critical skills |

---

## Quick Decision Tree

```
Does this MUST happen (compliance gate, audit log, quality gate)?
  YES → Shell Hook (PreToolUse or PostToolUse)
  NO  ↓

Is this a session-wide setting (model, permission mode)?
  YES → CLI flag or environment variable
  NO  ↓

Is this a frequently-reused prompt template that user triggers explicitly?
  YES → Slash Command with disable-model-invocation: true
  NO  ↓

Does this need isolated context, parallel execution, or specialized tooling?
  YES → Named Subagent (with skills preloaded in frontmatter)
  NO  ↓

Is this domain knowledge or workflow guidance Claude should apply contextually?
  YES → Skill
    Critical path? → disable-model-invocation: true
    Heavy output?  → context: fork
    Background knowledge? → user-invocable: false
  NO  ↓

Is this an external tool/data source?
  YES → MCP (scope to subagent if possible, not globally)
```

---

*Last updated: May 2026. Claude Code version ~2.1.x. Cross-check against https://code.claude.com/docs for latest changes.*
