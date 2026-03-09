---
summary: "Comprehensive analysis of the OpenClaw agent framework: architecture, system prompt, context engine, memory, skills, and tools"
read_when:
  - You want to understand how the agent is structured end-to-end
  - You are working on agent runtime, context management, memory, skills, or tools
  - You want a single entry point that connects all agent subsystems
title: "Agent Framework Analysis"
---

# Agent Framework Analysis

This document gives a unified technical walkthrough of the OpenClaw agent runtime and its subsystems: **system prompt**, **context engine**, **memory**, **skills**, and **tools**. Each section links to the dedicated concept doc for deeper detail.

---

## 1. Overall Architecture

OpenClaw runs a single embedded agent runtime built on **pi-mono** (the `@mariozechner/pi-coding-agent` library). A full agent turn is called an **agent loop**—a serialized, per-session run that ingests a message, assembles context, calls the model, executes tools, streams output, and persists the session.

```
User message
     │
     ▼
┌─────────────────────────────────┐
│        Agent Entry Points       │
│  Gateway RPC (agent / agent.wait│
│  CLI (openclaw agent)           │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│                  runEmbeddedPiAgent()                    │
│  src/agents/pi-embedded-runner/run.ts                   │
│                                                         │
│  1. resolveSessionAgentIds()   → Agent ID & config      │
│  2. resolveAgentConfig()       → Workspace, model, etc. │
│  3. resolveBootstrapContextForRun() → Bootstrap files   │
│  4. buildAgentSystemPrompt()   → Full system prompt     │
│  5. resolveContextEngine()     → Context engine plugin  │
│  6. createOpenClawCodingTools()→ Tool array             │
│  7. runEmbeddedAttempt()       → Model inference + tools│
│  8. context.afterTurn()        → Compaction / persist   │
└─────────────────────────────────────────────────────────┘
               │
               ▼
         EmbeddedPiRunResult
         { success, messages, usage, meta }
```

### Agent Configuration (`src/agents/agent-scope.ts`)

Every run resolves an **agent scope** from the config:

```typescript
type ResolvedAgentConfig = {
  name?: string;
  workspace?: string;       // Working directory (cwd for all tools)
  agentDir?: string;        // Agent state directory
  model?: AgentEntry["model"];
  skills?: AgentEntry["skills"];
  memorySearch?: AgentEntry["memorySearch"];
  humanDelay?: AgentEntry["humanDelay"];
  heartbeat?: AgentEntry["heartbeat"];
  identity?: AgentEntry["identity"];
  groupChat?: AgentEntry["groupChat"];
  subagents?: AgentEntry["subagents"];
  sandbox?: AgentEntry["sandbox"];
  tools?: AgentEntry["tools"];
};
```

**Resolution order**: explicit `agentId` param → session-key agent ID → config default → `"default"`.

### Agent Paths (`src/agents/agent-paths.ts`)

Key path helpers:

- `resolveOpenClawAgentDir()` — resolves the agent state directory
- `ensureOpenClawAgentEnv()` — ensures workspace and agent directory exist on disk

---

## 2. System Prompt

**Source**: `src/agents/system-prompt.ts`  
**Concept doc**: [System Prompt](/concepts/system-prompt)

The system prompt is **fully OpenClaw-owned** (not the pi-coding-agent default). It is assembled fresh on every run from modular sections.

### Assembly Function

```typescript
buildAgentSystemPrompt({
  workspaceDir,
  toolNames,
  toolSummaries,
  skillsPrompt,       // injected from skills subsystem
  contextFiles,       // bootstrap files (AGENTS.md, SOUL.md, etc.)
  runtimeInfo,        // host, OS, model, node, channel, caps
  userTimezone,
  memoryCitationsMode,
  promptMode,         // "full" | "minimal" | "none"
  sandboxInfo,
  // ... more
}): string
```

### Sections (prompt mode `full`)

| Section | Contents |
|---------|----------|
| **Tooling** | Tool list + short descriptions |
| **Safety** | Advisory guardrails against power-seeking |
| **Skills** | `<available_skills>` XML with name/description/location |
| **Memory Recall** | Instructions to search memory before answering |
| **OpenClaw Self-Update** | How to run `config.apply` / `update.run` |
| **Workspace** | Working directory path |
| **Documentation** | Local docs path + public mirror + ClawHub |
| **Project Context** | Injected bootstrap files (AGENTS.md, SOUL.md, …) |
| **Sandbox** (conditional) | Sandbox paths + elevated-exec availability |
| **Current Date & Time** | Timezone (no dynamic clock for cache stability) |
| **Reply Tags** | Quote/reply syntax for supported channels |
| **Messaging** | Session messaging commands, channel routing, sub-agent orchestration |
| **Voice/TTS** (conditional) | TTS formatting hints |
| **Runtime** | Host, OS, node, model, repo root, thinking level |
| **Reasoning** | Visibility level + `/reasoning` toggle hint |

### Prompt Modes

- **`full`** (default) — all sections above.
- **`minimal`** — for sub-agents; drops Skills, Memory Recall, Self-Update, Model Aliases, User Identity, Reply Tags, Messaging, Heartbeats.
- **`none`** — base identity line only.

### Bootstrap File Injection

Bootstrap files are trimmed and injected under **Project Context** on every turn.
Files injected for a main session:

```
AGENTS.md  SOUL.md  TOOLS.md  IDENTITY.md  USER.md
HEARTBEAT.md  BOOTSTRAP.md (new workspaces only)
MEMORY.md / memory.md (when present)
```

Sub-agent sessions only inject `AGENTS.md` and `TOOLS.md` to keep context small.

Per-file limit: `agents.defaults.bootstrapMaxChars` (default 20 000 chars).  
Total limit: `agents.defaults.bootstrapTotalMaxChars` (default 150 000 chars).

---

## 3. Context Engine

**Source**: `src/context-engine/`  
**Concept doc**: [Context](/concepts/context)

The context engine manages everything inside the model's **context window**: what messages to send, compaction, sub-agent context sharing, and system prompt supplements.

### Interface (`src/context-engine/types.ts`)

```typescript
interface ContextEngine {
  info: ContextEngineInfo;

  // Called once at session open
  bootstrap?(params: { sessionId: string; sessionFile: string }): Promise<BootstrapResult>;

  // Called per-message before the turn
  ingest(params: { sessionId: string; message: AgentMessage; isHeartbeat?: boolean }): Promise<IngestResult>;
  ingestBatch?(params: { sessionId: string; messages: AgentMessage[]; isHeartbeat?: boolean }): Promise<IngestBatchResult>;

  // Assemble the context window for a model call
  assemble(params: { sessionId: string; messages: AgentMessage[]; tokenBudget?: number }): Promise<AssembleResult>;

  // Compact older history to free window space
  compact(params: {
    sessionId: string; sessionFile: string;
    tokenBudget?: number; force?: boolean;
    currentTokenCount?: number;
    compactionTarget?: "budget" | "threshold";
    customInstructions?: string;
    runtimeContext?: ContextEngineRuntimeContext;
  }): Promise<CompactResult>;

  // Post-turn persistence hook
  afterTurn?(params: {
    sessionId: string; sessionFile: string;
    messages: AgentMessage[];
    prePromptMessageCount: number;
    autoCompactionSummary?: string;
    isHeartbeat?: boolean;
    tokenBudget?: number;
    runtimeContext?: ContextEngineRuntimeContext;
  }): Promise<void>;

  // Sub-agent context handoff
  prepareSubagentSpawn?(params: { parentSessionKey: string; childSessionKey: string; ttlMs?: number }): Promise<SubagentSpawnPreparation | undefined>;
  onSubagentEnded?(params: { childSessionKey: string; reason: SubagentEndReason }): Promise<void>;

  dispose?(): Promise<void>;
}
```

### Key Result Types

```typescript
type AssembleResult = {
  messages: AgentMessage[];        // Messages ordered for the model call
  estimatedTokens: number;         // Total estimated tokens
  systemPromptAddition?: string;   // Optional system prompt supplement from engine
};

type CompactResult = {
  ok: boolean;
  compacted: boolean;
  reason?: string;
  result?: {
    summary?: string;
    firstKeptEntryId?: string;
    tokensBefore: number;
    tokensAfter?: number;
  };
};
```

### Registry and Plugin Slot

Context engines are registered by ID and resolved from config:

```typescript
// src/context-engine/registry.ts
registerContextEngine(id: string, factory: () => ContextEngine | Promise<ContextEngine>): void
resolveContextEngine(config?: OpenClawConfig): Promise<ContextEngine>
```

**Default engine**: `"legacy"` (built-in).  
**Override**: set `config.plugins.slots.contextEngine` to a registered engine ID.  
Plugins register additional engines via `api.registerContextEngine()`.

### Lifecycle within a Turn

```
ingest(new message)
       │
       ▼
assemble() → ordered messages + token estimate
       │
       ▼
   model call
       │
       ▼
afterTurn() → compaction check, persist session
```

Compaction is triggered automatically when token usage approaches the window limit. Manual compaction: `/compact`. See [Compaction](/concepts/compaction).

---

## 4. Memory

**Source**: `src/memory/`  
**Concept doc**: [Memory](/concepts/memory)

Memory in OpenClaw is **Markdown files on disk**. The model only "remembers" what gets written. The memory subsystem provides semantic search and targeted retrieval over those files.

### Memory File Layout

```
~/.openclaw/workspace/
  MEMORY.md                  ← curated long-term memory (main session only)
  memory/
    2026-01-15.md            ← daily log (append-only)
    2026-01-16.md
    ...
```

Daily files for today and yesterday are injected at session start.  
`MEMORY.md` is only loaded in private main sessions (never in group contexts).

### MemorySearchManager Interface (`src/memory/types.ts`)

```typescript
interface MemorySearchManager {
  // Semantic + full-text search
  search(query: string, opts?: { maxResults?: number; minScore?: number; sessionKey?: string }): Promise<MemorySearchResult[]>;

  // Targeted read of a specific file/range
  readFile(params: { relPath: string; from?: number; lines?: number }): Promise<{ text: string; path: string }>;

  // Status and diagnostics
  status(): MemoryProviderStatus;

  // Sync / rebuild
  sync?(params?: { reason?: string; force?: boolean; progress?: (update: MemorySyncProgressUpdate) => void }): Promise<void>;

  // Probe embedding / vector availability
  probeEmbeddingAvailability(): Promise<MemoryEmbeddingProbeResult>;
  probeVectorAvailability(): Promise<boolean>;

  close?(): Promise<void>;
}
```

### Search Result

```typescript
type MemorySearchResult = {
  path: string;       // File path (relative to workspace)
  startLine: number;
  endLine: number;
  score: number;      // Relevance score
  snippet: string;    // Matched text
  source: "memory" | "sessions";
  citation?: string;
};
```

### Backend Options

| Backend | Description |
|---------|-------------|
| `builtin` | Default SQLite + FTS5 + optional `sqlite-vec` vector index |
| `qmd` | QMD (external process, supports advanced vector search) |

Configure with:

```json5
{
  plugins: { slots: { memory: "memory-core" } }   // or "none" to disable
}
```

### Agent-Facing Memory Tools

| Tool | Description |
|------|-------------|
| `memory_search` | Semantic recall over memory + session files |
| `memory_get` | Targeted read of specific lines in a memory file |

The system prompt instructs the model: **before answering anything about prior work, decisions, preferences, or todos — run `memory_search`, then use `memory_get` for specific lines**.

`memory_get` degrades gracefully when a file does not exist (returns `{ text: "", path }` instead of throwing).

### Embeddings

Multiple embedding providers are supported:

- Voyage AI (`embeddings-voyage.ts`)
- OpenAI (`embeddings-openai.ts`)
- Gemini (`embeddings-gemini.ts`)
- Mistral (`embeddings-mistral.ts`)
- Ollama (`embeddings-ollama.ts`)
- Remote HTTP (`embeddings-remote-fetch.ts`)

Batch embedding is managed by `batch-runner.ts` with configurable concurrency and error recovery.

---

## 5. Skills

**Source**: `src/agents/skills/`  
**Concept doc**: [Agent Runtime — Skills](/concepts/agent#skills)

Skills are **on-demand instruction files** (Markdown) that the model reads at runtime to execute specific workflows. They keep the base system prompt small—only a compact skills list is injected; the full instructions are fetched by the `read` tool when needed.

### Skill Entry Type (`src/agents/skills/types.ts`)

```typescript
type SkillEntry = {
  skill: Skill;                        // pi-coding-agent library Skill object
  frontmatter: ParsedSkillFrontmatter; // Parsed YAML metadata
  metadata?: OpenClawSkillMetadata;    // OpenClaw-specific metadata
  invocation?: SkillInvocationPolicy;  // Invocation control
};

type OpenClawSkillMetadata = {
  always?: boolean;                    // Always available (even if not in agent's skill list)
  skillKey?: string;                   // Unique identifier
  primaryEnv?: string;                 // Required environment variable
  emoji?: string;
  homepage?: string;
  os?: string[];                       // OS restriction ("darwin" | "linux" | "win32")
  requires?: {
    bins?: string[];                   // ALL these binaries must be present
    anyBins?: string[];                // AT LEAST ONE of these binaries must be present
    env?: string[];                    // Required env vars
    config?: string[];                 // Required config keys
  };
  install?: SkillInstallSpec[];        // Installation guidance
};

type SkillInvocationPolicy = {
  userInvocable: boolean;              // Can the user call this skill directly?
  disableModelInvocation: boolean;     // Prevent model from invoking it autonomously?
};
```

### Install Specs

```typescript
type SkillInstallSpec = {
  kind: "brew" | "node" | "go" | "uv" | "download";
  formula?: string;    // brew formula
  package?: string;    // npm/pip/uv package
  module?: string;     // node module
  url?: string;        // download URL
  bins?: string[];     // expected binaries after install
  os?: string[];
};
```

### Skill Sources (priority order)

1. **Workspace** (`<workspace>/skills/`) — wins on name conflict.
2. **Managed** (installed via `openclaw skills install`).
3. **Bundled** (shipped with OpenClaw).
4. **Plugin** (contributed by plugins).

### Skills in System Prompt

Eligible skills are injected as compact XML:

```xml
<available_skills>
  <skill>
    <name>frontend-design</name>
    <description>Design and implement frontend UI components</description>
    <location>/home/user/.openclaw/workspace/skills/frontend-design/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

The prompt instructs:
- If exactly **one** skill clearly applies → `read` its SKILL.md, then follow it.
- If **multiple** could apply → choose the most specific one.
- If **none** apply → do not read any SKILL.md.

### Skill Loading (`src/agents/skills/workspace.ts`)

```typescript
loadWorkspaceSkillEntries(workspaceDir: string): Promise<SkillEntry[]>
buildWorkspaceSkillsPrompt(skillEntries: SkillEntry[], context?: SkillEligibilityContext): string
```

Eligibility filtering checks OS compatibility, required binaries, and required environment variables before injecting a skill into the prompt.

---

## 6. Tools

**Source**: `src/agents/pi-tools.ts`, `src/agents/bash-tools.ts`, `src/agents/channel-tools.ts`  
**Concept doc**: [System Prompt — Tool summaries](/concepts/system-prompt)

All tools are created via:

```typescript
createOpenClawCodingTools(options?: {
  config?: OpenClawConfig;
  capabilities?: ToolCapabilities;
  toolPolicy?: ToolPolicy;
  sessionKey?: string;
  // ...
}): AnyAgentTool[]
```

### Core File Tools

| Tool | Description |
|------|-------------|
| `read` | Read file contents |
| `write` | Create or overwrite files |
| `edit` | Make precise edits to files |
| `apply_patch` | Apply multi-file unified diffs |
| `grep` | Search file contents for patterns |
| `find` | Find files by glob pattern |
| `ls` | List directory contents |

### Shell Execution Tools (`src/agents/bash-tools.ts`)

**`exec`** — Run shell commands (supports PTY for interactive CLIs):

```typescript
createExecTool(options: ExecToolDefaults): AnyAgentTool
```

- Enforces file sandbox boundaries.
- Supports elevated permission prompts.
- PTY mode for TTY-dependent CLIs.

**`process`** — Manage background processes:

```typescript
createProcessTool(defaults: ProcessToolDefaults): AnyAgentTool
```

- Track running processes.
- Send signals (SIGTERM, SIGKILL).
- Poll process status.

### Web Tools

| Tool | Description |
|------|-------------|
| `web_search` | Web search via Brave API |
| `web_fetch` | Fetch and extract readable content from a URL |

### Memory Tools

| Tool | Description |
|------|-------------|
| `memory_search` | Semantic recall over indexed memory + session files |
| `memory_get` | Targeted read of specific lines in a memory file |

### Messaging & Orchestration Tools

| Tool | Description |
|------|-------------|
| `message` | Send messages and channel actions |
| `sessions_spawn` | Spawn an isolated sub-agent session (or ACP session) |
| `sessions_send` | Send a message to another session |
| `sessions_list` | List active sessions |
| `subagents` | List / steer / kill subagent runs |
| `agents_list` | List available agent IDs |

### Media & UI Tools

| Tool | Description |
|------|-------------|
| `browser` | Web browser control |
| `canvas` | Canvas presentation and evaluation |
| `image` | Image analysis |
| `pdf` | PDF analysis (with vector/FTS support) |
| `tts` | Text-to-speech output |

### System & Infrastructure Tools

| Tool | Description |
|------|-------------|
| `cron` | Schedule reminders and wake events |
| `gateway` | Restart / configure / update OpenClaw |
| `nodes` | Control paired nodes (camera, screen, etc.) |
| `session_status` | Show session status card (time, usage, etc.) |

### Channel Tools (`src/agents/channel-tools.ts`)

Channel-specific tools are resolved per-session:

```typescript
listChannelAgentTools(params: { cfg?: OpenClawConfig }): ChannelAgentTool[]
resolveChannelMessageToolHints(params: { cfg?: OpenClawConfig; channel?: string | null; accountId?: string | null }): string[]
```

Examples: WhatsApp login tool, platform-specific message actions.

### Tool Policy

`ToolPolicy` gates which tools are available per session and whether exec requires approval.  
`ToolCapabilities` declares what the runtime can actually support (e.g., browser requires a display).

---

## 7. Bootstrap Context System

**Source**: `src/agents/bootstrap-files.ts`, `src/agents/bootstrap-hooks.ts`, `src/agents/bootstrap-budget.ts`

Bootstrap is the process of loading workspace files into the context window before the first model call.

### Resolution

```typescript
resolveBootstrapContextForRun(params: {
  workspaceDir: string;
  config?: OpenClawConfig;
  sessionKey?: string;
  sessionId?: string;
  agentId?: string;
  contextMode?: "full" | "lightweight";
  runKind?: "default" | "heartbeat" | "cron";
}): Promise<{ bootstrapFiles: WorkspaceBootstrapFile[]; contextFiles: EmbeddedContextFile[] }>
```

### Budget Management (`src/agents/bootstrap-budget.ts`)

```typescript
type BootstrapBudgetAnalysis = {
  files: BootstrapAnalyzedFile[];
  truncatedFiles: BootstrapAnalyzedFile[];
  nearLimitFiles: BootstrapAnalyzedFile[];
  totalNearLimit: boolean;
  hasTruncation: boolean;
  totals: {
    rawChars: number;
    injectedChars: number;
    truncatedChars: number;
    bootstrapMaxChars: number;       // per-file limit
    bootstrapTotalMaxChars: number;  // total limit
    nearLimitRatio: number;          // default 0.85
  };
};
```

Truncation warning mode: `agents.defaults.bootstrapPromptTruncationWarning` → `"off" | "once" | "always"` (default `"once"`).

### Plugin Hooks (`src/agents/bootstrap-hooks.ts`)

```typescript
applyBootstrapHookOverrides(params: {
  files: WorkspaceBootstrapFile[];
  workspaceDir: string;
  config?: OpenClawConfig;
  sessionKey?: string;
  agentId?: string;
}): Promise<WorkspaceBootstrapFile[]>
```

The internal hook `agent:bootstrap` fires here, allowing plugins to add, remove, or replace bootstrap context files (e.g., swap `SOUL.md` for an alternate persona).

---

## 8. Putting It All Together: Run Flow

```
1.  resolveSessionAgentIds()
    → Determines agentId from session/config

2.  resolveAgentConfig()
    → Workspace dir, model chain, skills filter,
      memory config, sandbox, tools policy, etc.

3.  resolveBootstrapContextForRun()
    → Loads + trims workspace files:
      AGENTS.md, SOUL.md, TOOLS.md, IDENTITY.md,
      USER.md, HEARTBEAT.md, BOOTSTRAP.md, MEMORY.md
    → applyBootstrapHookOverrides() (plugin interception)

4.  buildAgentSystemPrompt()
    → Assembles modular sections:
      Tooling, Safety, Skills, Memory, Workspace, Time,
      Reply Tags, Messaging, Runtime, Project Context
    → Injects bootstrap files and skills list

5.  resolveContextEngine()
    → Loads "legacy" engine (or plugin override)
    → engine.bootstrap() for new sessions

6.  createOpenClawCodingTools()
    → File tools (read/write/edit/grep/find/exec/process)
    → Memory tools (memory_search / memory_get)
    → Web tools (web_search / web_fetch)
    → Messaging tools (message / sessions_spawn / …)
    → Channel-specific tools
    → Media tools (browser / canvas / image / pdf)

7.  engine.ingest(userMessage)
    engine.assemble() → ordered messages + token estimate

8.  Model inference (pi-coding-agent)
    → Streams assistant deltas + tool calls
    → Tool calls dispatched to tool handlers
    → Tool results fed back into context
    → Repeat until stop_reason = "end_turn"

9.  engine.afterTurn()
    → Auto-compaction if near token limit
    → Session transcript persistence

10. EmbeddedPiRunResult
    { success, messages, usage, meta }
```

---

## 9. Key Files Reference

| File | Purpose |
|------|---------|
| `src/agents/pi-embedded-runner/run.ts` | Main agent runner (`runEmbeddedPiAgent`) |
| `src/agents/agent-scope.ts` | Agent ID + config resolution |
| `src/agents/agent-paths.ts` | Workspace / agent directory resolution |
| `src/agents/system-prompt.ts` | System prompt assembly (`buildAgentSystemPrompt`) |
| `src/agents/bootstrap-files.ts` | Bootstrap context file loading |
| `src/agents/bootstrap-hooks.ts` | Plugin hook interception for bootstrap |
| `src/agents/bootstrap-budget.ts` | Token budget enforcement for bootstrap files |
| `src/agents/bash-tools.ts` | Shell execution tools (exec, process) |
| `src/agents/channel-tools.ts` | Channel-specific tool wiring |
| `src/agents/pi-tools.ts` | Tool factory (`createOpenClawCodingTools`) |
| `src/agents/skills/types.ts` | Skill type definitions |
| `src/agents/skills/workspace.ts` | Skill loading and prompt building |
| `src/context-engine/types.ts` | Context engine interface |
| `src/context-engine/registry.ts` | Engine registration and resolution |
| `src/context-engine/init.ts` | Engine initialization |
| `src/memory/types.ts` | Memory manager interface |
| `src/memory/manager.ts` | Memory manager implementation |
| `src/memory/embeddings.ts` | Embedding provider abstraction |

---

## 10. Related Concept Docs

- [Agent Runtime](/concepts/agent) — workspace, bootstrap files, tools overview
- [Agent Loop](/concepts/agent-loop) — detailed lifecycle, hooks, streams, compaction
- [System Prompt](/concepts/system-prompt) — full section list, prompt modes, bootstrap injection
- [Context](/concepts/context) — context window, `/context` commands, token usage
- [Compaction](/concepts/compaction) — context compaction pipeline
- [Memory](/concepts/memory) — memory files, tools, write workflow, flush policy
- [Multi-Agent](/concepts/multi-agent) — sub-agent spawning and orchestration
- [Agent Workspace](/concepts/agent-workspace) — directory layout and file guide
