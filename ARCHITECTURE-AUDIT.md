# Architecture Audit: "ohno" Personal Agent App vs. Existing Repos

> Audit date: 2026-04-15
> Scope: All 28 repos under `hansraj316/` mapped against the ohno architecture diagram

---

## Executive Summary

The ohno architecture diagram describes a **full-featured personal AI agent platform** with ~25 major subsystems. After auditing all 28 of your repositories, **~6 repos have partial overlap** with specific components, but **no single repo implements the ohno framework itself**. The biggest opportunity is to use your existing projects as building blocks and reference implementations for a unified agent platform.

---

## Component-by-Component Mapping

### 1. Channel Manager (Telegram, Slack, Discord, Feishu, DingTalk, Email, Matrix)

| Status | **Not implemented** |
|--------|---------------------|
| Closest repo | None |
| What to build | A unified message bus that normalizes inbound/outbound messages across IM platforms into a common `MessageBus` format |
| Recommended repo | `voice-agent-carnival` already has voice routing; extend it, or create a new `ohno-channels` repo |
| Effort | Medium-High -- each platform has its own SDK/webhook format |

---

### 2. CLI (Typer) + React TUI (Ink + TS)

| Status | **Not implemented** |
|--------|---------------------|
| Closest repo | None (your repos are mostly web apps) |
| What to build | A Typer-based Python CLI (`oh` / `ohno` commands) for setup, provider config, auth, and MCP management. Optionally a React Ink TUI for richer terminal UI |
| Recommended repo | New `ohno-cli` repo or add to a monorepo |
| Effort | Medium -- Typer CLI is straightforward; Ink TUI is a separate effort |

---

### 3. Backend Host (JSON-Lines, OHJSON protocol)

| Status | **Not implemented** |
|--------|---------------------|
| Closest repo | `voice-agent-carnival` has `src/routes/voice-api-endpoints.js` (Express routes with streaming) |
| What to build | A lightweight JSON-Lines streaming host that bridges the CLI/TUI to the agent runtime |
| Effort | Low-Medium |

---

### 4. ohno Gateway (Bridge, Router, Service, MessageBus, Runtime Pool)

| Status | **Partially exists** |
|--------|----------------------|
| Closest repo | **`voice-agent-carnival`** -- has a voice API router (`public/voice-router-client.js`, `src/routes/voice-api-endpoints.js`) with WebSocket/WebRTC streaming |
| What exists | Voice-specific routing and session management |
| What's missing | Generalized gateway that routes any message type (text, voice, file) to the correct agent session; Runtime Pool for managing multiple concurrent sessions |
| Effort | Medium -- refactor voice router into a generic gateway |

---

### 5. Session Pool (Per-chat RuntimeBundle, auto-compact, resume)

| Status | **Not implemented** |
|--------|---------------------|
| Closest repo | `PMChat` (Rust) -- likely has chat session concepts but details unavailable |
| What to build | Session lifecycle management: create, persist, resume, auto-compact long conversations, garbage collection |
| Can implement in | `PMChat` (Rust is great for this), `agentmesh`, or new dedicated module |
| Effort | Medium |

---

### 6. API Client Layer (Anthropic, OpenAI, Codex, Copilot, retry + backoff)

| Status | **Partially exists (scattered)** |
|--------|----------------------------------|
| Repos with LLM API usage | **`OpenAI`** -- OpenAI API testing |
| | **`Projects/InterviewAgent`** -- Claude/MCP agent calls |
| | **`PMChat`** -- Claude integration |
| | **`voice-agent-carnival`** -- GPT-4o Realtime API, WebSocket streaming |
| | **`claude-agent-sdk-learning`** -- Anthropic SDK patterns |
| What's missing | A **unified client layer** with consistent retry/backoff, streaming support, and provider abstraction |
| Effort | Medium -- consolidate existing API call patterns into one module |

---

### 7. LLM Provider Registry (20+ providers)

| Status | **Partially exists (1 provider)** |
|--------|-----------------------------------|
| Closest repo | **`OllamaBar`** (Swift) -- macOS menu bar for local Ollama models |
| What exists | Ollama model switching/management on macOS |
| What's missing | A registry that manages 20+ providers (Anthropic, OpenAI, DeepSeek, Kimi, Groq, OpenRouter, SiliconFlow, vLLM, etc.) with unified config, API key management, and model routing |
| Can implement in | `OllamaBar` for local models; new `ohno-providers` module for the full registry |
| Effort | High -- each provider has unique API patterns |

---

### 8. QueryEngine (Agent Loop: stream -> tool_use -> loop)

| Status | **Partially exists** |
|--------|----------------------|
| Closest repo | **`Projects/InterviewAgent`** -- multi-agent pipeline with specialized agents (`application_submitter.py`, `email_notification.py`), MCP tool use |
| | **`claude-agent-sdk-learning`** -- 15 modules teaching agent loop patterns |
| What exists | Pipeline-style agent execution, MCP config (`mcp_agent.config.yaml`), iframe browser automation |
| What's missing | A generic agent loop with streaming, parallel tool execution, auto-compaction, and cost tracking |
| Effort | Medium -- extract patterns from InterviewAgent into a reusable engine |

---

### 9. Tool Registry (43+ Tools, Pydantic schemas)

| Status | **Partially exists (MCP configs)** |
|--------|-------------------------------------|
| Closest repo | **`Projects`** -- has `.claude/mcp.servers.json`, MCP tool configs |
| | **`Projects/InterviewAgent`** -- `iframe_browser_server.py` (custom MCP tool) |
| What exists | MCP server configuration, one custom browser automation tool |
| What's missing | A formal tool registry with Pydantic schemas, dynamic loading, and categories (Bash, FileIO, Search, Web, MCP) |
| Effort | Medium |

---

### 10. Skills System (On-demand .md loading, anthropics/skills compat)

| Status | **Not implemented** |
|--------|---------------------|
| Closest repo | None directly, but CLAUDE.md files exist in `Projects/InterviewAgent` |
| What to build | A system that discovers and loads `.md` skill files on demand, similar to Claude Code's skills system |
| Effort | Low-Medium |

---

### 11. Memory (MEMORY.md persistent, CLAUDE.md discovery)

| Status | **Partially exists** |
|--------|----------------------|
| Closest repo | **`Projects`** -- has `.claude/README.md`, CLAUDE.md files |
| | **`Projects/InterviewAgent`** -- has `CLAUDE.md` with project context |
| | **`voice-agent-carnival`** -- has `.claude/project.json`, `docs/CLAUDE.md` |
| What exists | Static CLAUDE.md files for project context |
| What's missing | Persistent MEMORY.md that accumulates across sessions, automatic discovery, read/write memory lifecycle |
| Effort | Low -- the file-based pattern is simple; the discovery/auto-update logic is the main work |

---

### 12. Swarm Coordinator (Subagent, Team Registry, Mailbox, Permission Sync)

| Status | **Partially exists** |
|--------|----------------------|
| Closest repo | **`agentmesh`** -- "Real-time visibility and coordination for multi-agent AI systems" |
| | **`mission-control-openclaw`** -- dashboard for 25+ agents, cron jobs |
| | **`agentdate`** -- agent social graph, agent discovery/registration |
| What exists | Agent visibility dashboards, agent discovery concepts, multi-agent monitoring |
| What's missing | Actual subagent spawning, inter-agent mailbox/messaging, permission sync, worktree isolation, team lifecycle management |
| Can implement in | **`agentmesh`** is the natural home for this |
| Effort | High -- this is the most complex subsystem |

---

### 13. Plugin System (Commands, Hooks, Agents)

| Status | **Not implemented** |
|--------|---------------------|
| What to build | Extensibility layer: custom commands, pre/post hooks, pluggable agent types |
| Effort | Medium |

---

### 14. Prompts (System prompt assembly, Environment, Context)

| Status | **Not implemented** |
|--------|---------------------|
| Closest repo | `claude-agent-sdk-learning` likely covers prompt patterns |
| What to build | Dynamic system prompt construction from environment context, user preferences, and active tools |
| Effort | Low-Medium |

---

### 15. Permission Checker (default, auto, plan modes)

| Status | **Not implemented** |
|--------|---------------------|
| What to build | Permission system controlling which tools can run, path-based file access rules, denied command lists, approval modes (default, auto, plan) |
| Effort | Medium |

---

### 16. Hook Executor (PreToolUse / PostToolUse)

| Status | **Not implemented** |
|--------|---------------------|
| What to build | Lifecycle hooks that fire before/after every tool call for logging, validation, or transformation |
| Effort | Low |

---

### 17. MCP Servers (stdio, HTTP, WebSocket)

| Status | **Partially exists** |
|--------|----------------------|
| Closest repo | **`Projects`** -- `.claude/mcp.servers.json` with server definitions |
| | **`Projects/InterviewAgent`** -- `iframe_browser_server.py` (stdio MCP server) |
| | **`Projects/CoachAI`** -- `scripts/start_mcp.sh` (MCP server startup) |
| What exists | MCP server configurations and at least one custom stdio server |
| What's missing | HTTP and WebSocket MCP transports, dynamic server registration |
| Effort | Medium |

---

### 18. Docker Sandbox (Isolated execution)

| Status | **Not implemented** |
|--------|---------------------|
| What to build | Containerized tool execution for untrusted code, with resource limits and network isolation |
| Effort | Medium |

---

### 19. Services (Auto-Compact, Session Storage, Cron, LSP, Token Estimation)

| Status | **Partially exists** |
|--------|----------------------|
| Closest repo | **`mission-control-openclaw`** -- cron jobs, agent scheduling |
| What exists | Cron-based scheduling for agents |
| What's missing | Auto-compaction service, persistent session storage, LSP integration, token counting/cost estimation |
| Effort | Medium-High |

---

### 20. Built-in Tools (File I/O, Shell & Search, Web, Agent/Task, Schedule, Mode, Meta)

| Status | **Partially exists** |
|--------|----------------------|
| Closest repo | **`Projects/InterviewAgent`** -- browser automation, email, file operations |
| What exists | Specific tool implementations (browser, email) |
| What's missing | A standard library of 43+ tools with consistent interfaces |
| Effort | High |

---

## Repo-to-Architecture Mapping (Summary Table)

| Repo | Relevant Architecture Components | Overlap Level |
|------|----------------------------------|---------------|
| **agentmesh** | Swarm Coordinator, Services | High |
| **InterviewAgent** (in Projects) | QueryEngine, Tool Registry, MCP Servers | High |
| **voice-agent-carnival** | Gateway, Backend Host, API Client Layer | High |
| **mission-control-openclaw** | Services (Cron), Swarm Coordinator | Medium |
| **PMChat** | Session Pool, API Client Layer | Medium |
| **agentdate** | Swarm Coordinator (Agent Discovery) | Medium |
| **OllamaBar** | LLM Provider Registry (Ollama) | Medium |
| **claude-agent-sdk-learning** | QueryEngine patterns, Prompts | Low-Medium |
| **OpenAI** | API Client Layer | Low |
| **v0-ai-chat-interface** | React TUI concepts | Low |
| **Projects/CoachAI** | MCP Servers, Tool Registry | Low |
| **Startup** | Frontend patterns (Next.js/TS) | Low |
| **phishshield-drill-engine** | (Security, not directly relevant) | None |
| **pannas-com** | (Website, not relevant) | None |
| **hansraj-portfolio** | (Portfolio, not relevant) | None |
| Legacy repos (Windows8, Kinect, etc.) | (Historical, not relevant) | None |

---

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1-3)
**Goal: Unified API client + basic agent loop**

1. **API Client Layer** -- Consolidate LLM API calls from `OpenAI`, `voice-agent-carnival`, and `InterviewAgent` into a single module with retry/backoff and streaming
2. **LLM Provider Registry** -- Extend `OllamaBar` patterns; add Anthropic, OpenAI, and 2-3 more providers
3. **QueryEngine v1** -- Extract the agent loop from `InterviewAgent` into a standalone stream -> tool_use -> loop engine

### Phase 2: Infrastructure (Weeks 4-6)
**Goal: Session management + tool framework**

4. **Session Pool** -- Build on `PMChat`'s chat concepts; add persistence, auto-compact, resume
5. **Tool Registry** -- Formalize MCP tools from `Projects` into a Pydantic-schema registry
6. **Memory System** -- Implement MEMORY.md read/write with automatic discovery
7. **MCP Servers** -- Add HTTP/WebSocket transports to existing stdio servers

### Phase 3: Coordination (Weeks 7-10)
**Goal: Multi-agent orchestration**

8. **Swarm Coordinator** -- Build on `agentmesh` for subagent spawning, mailbox, team lifecycle
9. **Gateway** -- Generalize `voice-agent-carnival`'s router into a universal message gateway
10. **Permission Checker + Hook Executor** -- Add safety rails

### Phase 4: User Interface (Weeks 11-14)
**Goal: CLI + channels**

11. **CLI (Typer)** -- Build `oh`/`ohno` CLI commands
12. **Channel Manager** -- Start with Telegram + Slack adapters using the Gateway
13. **Plugin System + Skills** -- Extensibility layer

### Phase 5: Hardening (Weeks 15+)
**Goal: Production-ready**

14. **Docker Sandbox** -- Isolated execution for untrusted tools
15. **Services** -- Auto-compact, token estimation, LSP
16. **React TUI** -- Rich terminal interface with Ink

---

## Quick Wins (Can implement immediately in existing repos)

| What | Where | Effort |
|------|-------|--------|
| Add MEMORY.md persistence | All active repos | 1-2 days |
| Unify .claude/mcp.servers.json configs | `Projects`, `voice-agent-carnival` | 1 day |
| Add retry/backoff wrapper to API calls | `InterviewAgent`, `voice-agent-carnival` | 1-2 days |
| Add cost tracking to existing agent loops | `InterviewAgent` | 1 day |
| Create agent discovery registry in `agentdate` | `agentdate` | 2-3 days |
| Add WebSocket transport to MCP servers | `Projects/InterviewAgent` | 2-3 days |
| Add hook system (pre/post tool use) | `InterviewAgent` | 1-2 days |

---

## Repos Not Relevant to ohno Architecture

These repos don't map to the architecture and can be left as-is:

- `Windows-8-Apps`, `Windows8`, `WinApps` -- Legacy Windows/C# projects
- `KinectDevelopment`, `KinectSample` -- Legacy Kinect projects
- `Home-Surveillance-System` -- Legacy IoT project
- `Code` -- General code snippets
- `hello-world-web` -- Demo
- `hansraj-portfolio`, `pannas-com` -- Websites
- `ai-security-quickstart-avengers` -- Security training
- `phishshield-drill-engine` -- Security tool
- `vercel-nano-banana` -- Vercel experiment
- `claude-automatic-sniffle` -- Experiment
- `openclaw-setup-tracker` -- Setup utility

---
---

# Deep Dive: agentmesh Implementation Plan

> Target repo: [`hansraj316/agentmesh`](https://github.com/hansraj316/agentmesh)
> Stack: Python + SQLite + OpenClaw
> Status: Early-stage / greenfield

The `agentmesh` repo is the natural home for the **coordination layer** of the ohno architecture. It maps to 7 architecture components from the diagram and can serve as the central nervous system connecting your other repos.

---

## Architecture Components to Implement in agentmesh

### Overview

```
┌─────────────────────────────────────────────────────────┐
│                      agentmesh                          │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │    Swarm      │  │   Session    │  │   Services   │  │
│  │ Coordinator   │  │    Pool      │  │              │  │
│  │              │  │              │  │ - Cron       │  │
│  │ - Team Reg.  │  │ - Create     │  │ - Storage    │  │
│  │ - Mailbox    │  │ - Persist    │  │ - Compact    │  │
│  │ - Subagents  │  │ - Resume     │  │ - Tokens     │  │
│  │ - Lifecycle  │  │ - Compact    │  │              │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                 │           │
│  ┌──────┴─────────────────┴─────────────────┴───────┐  │
│  │              QueryEngine (Agent Loop)              │  │
│  │         stream → tool_use → loop → respond         │  │
│  └──────────────────────┬────────────────────────────┘  │
│                         │                               │
│  ┌──────────────┐  ┌────┴─────────┐  ┌──────────────┐  │
│  │    Memory     │  │    Tool      │  │  Permission  │  │
│  │              │  │  Registry    │  │   Checker    │  │
│  │ - MEMORY.md  │  │ - Pydantic   │  │ - Modes      │  │
│  │ - Discovery  │  │ - MCP        │  │ - Path rules │  │
│  │ - R/W cycle  │  │ - Dynamic    │  │ - Hooks      │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 1. Swarm Coordinator

The most complex and valuable component. This is what makes agentmesh unique.

### 1.1 Team Registry

```python
# agentmesh/coordinator/registry.py

"""
Central registry where agents register themselves and discover peers.
Backed by SQLite for persistence.

Tables:
  agents       - id, name, type, capabilities, status, registered_at
  teams        - id, name, purpose, created_at
  team_members - team_id, agent_id, role, joined_at
"""
```

**What it does:**
- Agents register on startup with their capabilities (tools, skills, models)
- Teams group agents for coordinated tasks
- Registry is queryable -- agents can discover peers by capability
- Connects to `agentdate` concepts (agent social graph)

**Schema (SQLite):**

```sql
CREATE TABLE agents (
    id          TEXT PRIMARY KEY,
    name        TEXT NOT NULL,
    type        TEXT NOT NULL,        -- 'subprocess', 'in_process', 'remote'
    capabilities TEXT,                -- JSON array of capability strings
    endpoint    TEXT,                 -- URL or stdio path
    status      TEXT DEFAULT 'idle',  -- 'idle', 'busy', 'offline'
    last_seen   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    registered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE teams (
    id       TEXT PRIMARY KEY,
    name     TEXT NOT NULL,
    purpose  TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE team_members (
    team_id  TEXT REFERENCES teams(id),
    agent_id TEXT REFERENCES agents(id),
    role     TEXT DEFAULT 'member',   -- 'lead', 'member', 'observer'
    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (team_id, agent_id)
);
```

### 1.2 Mailbox (Inter-Agent Messaging)

```python
# agentmesh/coordinator/mailbox.py

"""
Async message passing between agents.
Agents send messages to a named mailbox; recipients poll or subscribe.

Message types:
  - task_request   : "Please do X"
  - task_result    : "Here's the result of X"
  - status_update  : "I'm now doing Y"
  - broadcast      : "FYI to all team members"
"""
```

**Schema:**

```sql
CREATE TABLE messages (
    id          TEXT PRIMARY KEY,
    from_agent  TEXT REFERENCES agents(id),
    to_agent    TEXT,                     -- NULL = broadcast
    team_id     TEXT REFERENCES teams(id),
    type        TEXT NOT NULL,
    payload     TEXT NOT NULL,            -- JSON
    status      TEXT DEFAULT 'pending',   -- 'pending', 'delivered', 'read'
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 1.3 Subagent Spawning

```python
# agentmesh/coordinator/spawner.py

"""
Spawn subagents as:
  - subprocess  : isolated OS process (safest)
  - in_process  : thread/coroutine (fastest)
  - worktree    : git worktree for code-editing agents

Each subagent gets its own RuntimeBundle with:
  - Scoped permissions
  - Dedicated mailbox
  - Resource limits (timeout, max tokens)
"""
```

### 1.4 Team Lifecycle

```python
# agentmesh/coordinator/lifecycle.py

"""
Team lifecycle: create → assign → execute → report → dissolve

States:
  FORMING   - team created, members joining
  ACTIVE    - executing coordinated task
  REPORTING - aggregating results
  DISSOLVED - task complete, team disbanded
"""
```

---

## 2. QueryEngine (Agent Loop)

The core execution engine. Extract patterns from `InterviewAgent`.

### Proposed Module Structure

```python
# agentmesh/engine/query_engine.py

"""
Agent loop: stream → tool_use → loop

Core loop:
  1. Send message + tools to LLM
  2. Stream response tokens
  3. If tool_use in response → execute tool → append result → goto 1
  4. If text response → return to user
  5. Auto-compact if context exceeds threshold

Features:
  - Parallel tool execution (multiple tool calls in one turn)
  - Cost tracking (input/output tokens per turn)
  - Streaming output
  - Auto-compaction (summarize old messages when context fills)
"""
```

### Key Classes

```python
class QueryEngine:
    """Main agent loop executor."""
    
    def __init__(self, provider, tool_registry, memory, permissions):
        ...
    
    async def run(self, messages, stream=True):
        """Execute the agent loop until completion."""
        ...
    
    async def _execute_tools(self, tool_calls):
        """Execute tool calls, optionally in parallel."""
        ...
    
    def _should_compact(self, messages) -> bool:
        """Check if messages exceed context threshold."""
        ...
    
    def _compact(self, messages) -> list:
        """Summarize older messages to free context."""
        ...


class CostTracker:
    """Track token usage and estimated cost per provider."""
    
    def record(self, provider, model, input_tokens, output_tokens): ...
    def total_cost(self) -> float: ...
    def summary(self) -> dict: ...
```

---

## 3. Session Pool

Manages per-chat agent sessions with persistence.

```python
# agentmesh/sessions/pool.py

"""
Session lifecycle: create → active → suspended → resumed → closed

Features:
  - SQLite-backed persistence
  - Auto-compact long conversations
  - Resume from last checkpoint
  - Garbage collection for expired sessions
"""
```

**Schema:**

```sql
CREATE TABLE sessions (
    id          TEXT PRIMARY KEY,
    agent_id    TEXT REFERENCES agents(id),
    status      TEXT DEFAULT 'active',    -- 'active', 'suspended', 'closed'
    messages    TEXT,                      -- JSON array (compacted)
    metadata    TEXT,                      -- JSON (model, provider, tools)
    token_count INTEGER DEFAULT 0,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at  TIMESTAMP
);

CREATE TABLE checkpoints (
    id          TEXT PRIMARY KEY,
    session_id  TEXT REFERENCES sessions(id),
    messages    TEXT NOT NULL,             -- JSON snapshot
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 4. Memory System

Persistent memory across sessions.

```python
# agentmesh/memory/manager.py

"""
Two-tier memory:
  1. CLAUDE.md  - Project context, discovered automatically (read-only)
  2. MEMORY.md  - Persistent facts, preferences, decisions (read-write)

Discovery:
  - Walk up from cwd looking for CLAUDE.md files
  - Merge all found files into context
  - MEMORY.md lives in ~/.ohno/memory/ per-project
"""
```

---

## 5. Tool Registry

Dynamic tool management with Pydantic schemas.

```python
# agentmesh/tools/registry.py

"""
Tool categories:
  - file_io    : Read, Write, Edit, Glob
  - shell      : Bash, Grep
  - web        : WebFetch, WebSearch
  - agent      : Agent (spawn subagent), SendMessage
  - mcp        : Dynamically loaded from MCP servers
  - custom     : User-defined via plugins

Each tool has:
  - name, description
  - Pydantic input schema
  - execute() method
  - Required permissions
"""
```

### MCP Integration

```python
# agentmesh/tools/mcp_loader.py

"""
Load tools dynamically from MCP servers.
Supports three transports:
  - stdio   : spawn a subprocess, communicate via stdin/stdout
  - HTTP    : REST API calls
  - WebSocket : persistent connection for streaming
  
Config format (.claude/mcp.servers.json):
{
  "servers": {
    "browser": {
      "command": "python",
      "args": ["iframe_browser_server.py"],
      "transport": "stdio"
    }
  }
}
"""
```

---

## 6. Permission Checker

Safety rails for tool execution.

```python
# agentmesh/permissions/checker.py

"""
Three modes:
  - default  : ask user before risky operations
  - auto     : allow all (for trusted contexts)
  - plan     : propose actions, wait for approval

Rules:
  - Path-based file access (allow/deny patterns)
  - Command denylist (rm -rf, DROP TABLE, etc.)
  - Network access control
  - Per-tool permission overrides
"""
```

---

## 7. Services

Background services that support the platform.

```python
# agentmesh/services/

"""
auto_compact.py   - Summarize old messages when context fills
session_store.py  - SQLite persistence for sessions
cron.py           - Scheduled agent tasks (from mission-control patterns)
token_counter.py  - Estimate tokens before sending to LLM
health.py         - Agent health checks and heartbeat
"""
```

---

## Proposed Directory Structure

```
agentmesh/
├── README.md
├── pyproject.toml
├── agentmesh/
│   ├── __init__.py
│   ├── coordinator/
│   │   ├── __init__.py
│   │   ├── registry.py          # Team & Agent Registry
│   │   ├── mailbox.py           # Inter-agent messaging
│   │   ├── spawner.py           # Subagent spawning
│   │   └── lifecycle.py         # Team lifecycle management
│   ├── engine/
│   │   ├── __init__.py
│   │   ├── query_engine.py      # Core agent loop
│   │   ├── cost_tracker.py      # Token & cost tracking
│   │   └── compactor.py         # Auto-compaction
│   ├── sessions/
│   │   ├── __init__.py
│   │   └── pool.py              # Session pool manager
│   ├── memory/
│   │   ├── __init__.py
│   │   └── manager.py           # MEMORY.md + CLAUDE.md
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── registry.py          # Tool registry
│   │   ├── mcp_loader.py        # MCP server integration
│   │   └── builtins/
│   │       ├── __init__.py
│   │       ├── file_io.py       # Read, Write, Edit, Glob
│   │       ├── shell.py         # Bash, Grep
│   │       └── web.py           # WebFetch, WebSearch
│   ├── permissions/
│   │   ├── __init__.py
│   │   └── checker.py           # Permission checker
│   ├── services/
│   │   ├── __init__.py
│   │   ├── auto_compact.py
│   │   ├── session_store.py
│   │   ├── cron.py
│   │   ├── token_counter.py
│   │   └── health.py
│   └── db/
│       ├── __init__.py
│       ├── schema.sql            # All table definitions
│       └── migrations/
├── tests/
│   ├── test_coordinator.py
│   ├── test_engine.py
│   ├── test_sessions.py
│   └── test_tools.py
└── .claude/
    └── mcp.servers.json
```

---

## Implementation Priority (for agentmesh)

| Priority | Component | Effort | Why First |
|----------|-----------|--------|-----------|
| **P0** | SQLite schema + DB layer | 1-2 days | Foundation for everything else |
| **P0** | Agent Registry | 2-3 days | Core identity — agents must register before coordinating |
| **P1** | QueryEngine (basic loop) | 3-4 days | The execution engine that makes agents actually work |
| **P1** | Tool Registry + 3 builtins | 2-3 days | Agents need tools to be useful |
| **P1** | Session Pool | 2-3 days | Persistence for conversations |
| **P2** | Mailbox | 2-3 days | Multi-agent communication |
| **P2** | Memory Manager | 1-2 days | Cross-session learning |
| **P2** | Permission Checker | 2 days | Safety before going multi-user |
| **P3** | MCP Loader | 2-3 days | Extensibility via MCP servers |
| **P3** | Subagent Spawner | 3-4 days | Advanced orchestration |
| **P3** | Team Lifecycle | 2 days | Structured multi-agent workflows |
| **P3** | Services (cron, compact, tokens) | 3-4 days | Production hardening |

**Total estimated: ~4-6 weeks for full implementation**

---

## Cross-Repo Integration Points

Once agentmesh is built, it connects to your other repos:

```
InterviewAgent ──registers──▶ agentmesh Registry
                              (resume agent, cover letter agent, submitter agent)

voice-agent-carnival ──routes via──▶ agentmesh Gateway
                                     (voice sessions managed by Session Pool)

mission-control ──monitors──▶ agentmesh Registry + Services
                              (dashboard reads agent status, cron jobs)

agentdate ──discovers via──▶ agentmesh Registry
                             (external agent discovery feeds into local registry)

OllamaBar ──provides models to──▶ agentmesh QueryEngine
                                   (local LLM as a provider option)

PMChat ──uses──▶ agentmesh QueryEngine + Memory
                  (chat sessions with persistent context)
```
