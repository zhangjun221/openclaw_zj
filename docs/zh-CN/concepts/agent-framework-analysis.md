---
summary: "OpenClaw 智能体框架综合分析：架构、系统提示词、上下文引擎、记忆、技能与工具"
read_when:
  - 想从整体上理解智能体运行框架时
  - 在研究智能体运行时、上下文管理、记忆、技能或工具时
  - 需要一个汇总所有智能体子系统的单一入口时
title: "智能体框架分析"
x-i18n:
  generated_at: "2026-03-09T03:23:35Z"
  model: claude-sonnet-4-5
  provider: pi
  source_hash: 3217c9a86993e9147f203a9df88175e6ae44efaa257babf3a132b66c83a8b890
  source_path: concepts/agent-framework-analysis.md
  workflow: manual
---

# 智能体框架分析

本文对 OpenClaw 智能体运行时及其各子系统提供统一的技术解读，涵盖：**系统提示词**、**上下文引擎**、**记忆**、**技能**与**工具**。每个章节均附有对应的专项概念文档链接，供深入阅读。

---

## 1. 整体架构

OpenClaw 运行一个基于 **pi-mono**（`@mariozechner/pi-coding-agent` 库）的嵌入式智能体运行时。完整的智能体工作轮称为**智能体循环（agent loop）**——一个按会话序列化的运行流程，负责接收消息、组装上下文、调用模型、执行工具、流式输出并持久化会话。

```
用户消息
     │
     ▼
┌─────────────────────────────────┐
│         智能体入口点             │
│  Gateway RPC (agent / agent.wait│
│  CLI (openclaw agent)           │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│                  runEmbeddedPiAgent()                    │
│  src/agents/pi-embedded-runner/run.ts                   │
│                                                         │
│  1. resolveSessionAgentIds()   → 智能体 ID 与配置       │
│  2. resolveAgentConfig()       → 工作区、模型等          │
│  3. resolveBootstrapContextForRun() → 引导文件          │
│  4. buildAgentSystemPrompt()   → 完整系统提示词          │
│  5. resolveContextEngine()     → 上下文引擎插件          │
│  6. createOpenClawCodingTools()→ 工具数组               │
│  7. runEmbeddedAttempt()       → 模型推理 + 工具执行    │
│  8. context.afterTurn()        → 压缩 / 持久化          │
└─────────────────────────────────────────────────────────┘
               │
               ▼
         EmbeddedPiRunResult
         { success, messages, usage, meta }
```

### 智能体配置（`src/agents/agent-scope.ts`）

每次运行都会从配置中解析出一个**智能体作用域**：

```typescript
type ResolvedAgentConfig = {
  name?: string;
  workspace?: string;       // 工作目录（所有工具的 cwd）
  agentDir?: string;        // 智能体状态目录
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

**解析优先级**：显式 `agentId` 参数 → 会话键对应的智能体 ID → 配置默认值 → `"default"`。

### 智能体路径（`src/agents/agent-paths.ts`）

关键路径辅助函数：

- `resolveOpenClawAgentDir()` — 解析智能体状态目录
- `ensureOpenClawAgentEnv()` — 确保工作区和智能体目录在磁盘上存在

---

## 2. 系统提示词

**来源**：`src/agents/system-prompt.ts`  
**概念文档**：[系统提示词](/zh-CN/concepts/system-prompt)

系统提示词**完全由 OpenClaw 管理**（不使用 pi-coding-agent 的默认提示词）。每次运行时，系统从多个模块化区块中组装提示词。

### 组装函数

```typescript
buildAgentSystemPrompt({
  workspaceDir,
  toolNames,
  toolSummaries,
  skillsPrompt,       // 来自技能子系统
  contextFiles,       // 引导文件（AGENTS.md、SOUL.md 等）
  runtimeInfo,        // 主机、OS、模型、节点、频道、能力
  userTimezone,
  memoryCitationsMode,
  promptMode,         // "full" | "minimal" | "none"
  sandboxInfo,
  // ...
}): string
```

### 各区块说明（提示词模式 `full`）

| 区块 | 内容 |
|------|------|
| **工具列表（Tooling）** | 工具列表 + 简短描述 |
| **安全（Safety）** | 防止权力寻求行为的建议性护栏 |
| **技能（Skills）** | 包含名称/描述/位置的 `<available_skills>` XML |
| **记忆召回（Memory Recall）** | 指示模型在回答前先搜索记忆 |
| **OpenClaw 自更新** | 如何运行 `config.apply` / `update.run` |
| **工作区（Workspace）** | 工作目录路径 |
| **文档（Documentation）** | 本地文档路径 + 公开镜像 + ClawHub |
| **项目上下文（Project Context）** | 注入的引导文件（AGENTS.md、SOUL.md 等） |
| **沙盒（Sandbox）**（条件性） | 沙盒路径 + 高权限执行可用性 |
| **当前日期与时间** | 时区信息（不包含动态时钟，以保持缓存稳定性） |
| **回复标签（Reply Tags）** | 支持频道的引用/回复语法 |
| **消息（Messaging）** | 会话消息命令、频道路由、子智能体编排 |
| **语音/TTS**（条件性） | TTS 格式化提示 |
| **运行时（Runtime）** | 主机、OS、节点、模型、代码库根目录、思考级别 |
| **推理（Reasoning）** | 可见级别 + `/reasoning` 切换提示 |

### 提示词模式

- **`full`**（默认）— 包含以上所有区块。
- **`minimal`** — 用于子智能体；省略技能、记忆召回、自更新、模型别名、用户身份、回复标签、消息、心跳。
- **`none`** — 仅基础身份行。

### 引导文件注入

引导文件经裁剪后注入到 **Project Context（项目上下文）** 中，让模型每轮都能看到身份和配置，而无需显式读取。主会话注入的文件包括：

```
AGENTS.md  SOUL.md  TOOLS.md  IDENTITY.md  USER.md
HEARTBEAT.md  BOOTSTRAP.md（仅新工作区）
MEMORY.md / memory.md（存在时）
```

子智能体会话仅注入 `AGENTS.md` 和 `TOOLS.md`，以保持上下文精简。

单文件上限：`agents.defaults.bootstrapMaxChars`（默认 20 000 字符）。  
总上限：`agents.defaults.bootstrapTotalMaxChars`（默认 150 000 字符）。

---

## 3. 上下文引擎

**来源**：`src/context-engine/`  
**概念文档**：[上下文](/zh-CN/concepts/context)

上下文引擎管理模型**上下文窗口**内的所有内容：发送哪些消息、如何压缩、子智能体上下文共享，以及系统提示词补充。

### 接口（`src/context-engine/types.ts`）

```typescript
interface ContextEngine {
  info: ContextEngineInfo;

  // 会话开启时调用一次
  bootstrap?(params: { sessionId: string; sessionFile: string }): Promise<BootstrapResult>;

  // 每条消息到来时在本轮之前调用
  ingest(params: { sessionId: string; message: AgentMessage; isHeartbeat?: boolean }): Promise<IngestResult>;
  ingestBatch?(params: { sessionId: string; messages: AgentMessage[]; isHeartbeat?: boolean }): Promise<IngestBatchResult>;

  // 为模型调用组装上下文窗口
  assemble(params: { sessionId: string; messages: AgentMessage[]; tokenBudget?: number }): Promise<AssembleResult>;

  // 压缩旧历史以释放窗口空间
  compact(params: {
    sessionId: string; sessionFile: string;
    tokenBudget?: number; force?: boolean;
    currentTokenCount?: number;
    compactionTarget?: "budget" | "threshold";
    customInstructions?: string;
    runtimeContext?: ContextEngineRuntimeContext;
  }): Promise<CompactResult>;

  // 轮次结束后的持久化钩子
  afterTurn?(params: {
    sessionId: string; sessionFile: string;
    messages: AgentMessage[];
    prePromptMessageCount: number;
    autoCompactionSummary?: string;
    isHeartbeat?: boolean;
    tokenBudget?: number;
    runtimeContext?: ContextEngineRuntimeContext;
  }): Promise<void>;

  // 子智能体上下文交接
  prepareSubagentSpawn?(params: { parentSessionKey: string; childSessionKey: string; ttlMs?: number }): Promise<SubagentSpawnPreparation | undefined>;
  onSubagentEnded?(params: { childSessionKey: string; reason: SubagentEndReason }): Promise<void>;

  dispose?(): Promise<void>;
}
```

### 关键结果类型

```typescript
type AssembleResult = {
  messages: AgentMessage[];        // 为模型调用排序后的消息列表
  estimatedTokens: number;         // 估算的总 Token 数
  systemPromptAddition?: string;   // 引擎可追加的系统提示词补充
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

### 注册表与插件槽

上下文引擎按 ID 注册，并从配置中解析：

```typescript
// src/context-engine/registry.ts
registerContextEngine(id: string, factory: () => ContextEngine | Promise<ContextEngine>): void
resolveContextEngine(config?: OpenClawConfig): Promise<ContextEngine>
```

**默认引擎**：`"legacy"`（内置）。  
**覆盖**：将 `config.plugins.slots.contextEngine` 设置为已注册的引擎 ID。  
插件通过 `api.registerContextEngine()` 注册额外引擎。

### 一次轮次内的生命周期

```
ingest(新消息)
       │
       ▼
assemble() → 有序消息列表 + Token 估算
       │
       ▼
   模型调用
       │
       ▼
afterTurn() → 检查是否需要压缩，持久化会话
```

当 Token 使用量接近窗口上限时自动触发压缩。手动压缩：`/compact`。  
详见：[压缩](/zh-CN/concepts/compaction)。

---

## 4. 记忆

**来源**：`src/memory/`  
**概念文档**：[记忆](/zh-CN/concepts/memory)

OpenClaw 的记忆是**磁盘上的 Markdown 文件**。模型只"记住"写入磁盘的内容。记忆子系统为这些文件提供语义搜索和精准检索能力。

### 记忆文件布局

```
~/.openclaw/workspace/
  MEMORY.md                  ← 精心整理的长期记忆（仅主会话）
  memory/
    2026-01-15.md            ← 每日日志（仅追加）
    2026-01-16.md
    ...
```

当天和前一天的日志文件在会话开始时注入。  
`MEMORY.md` 仅在私密主会话中加载（不在群组上下文中加载）。

### MemorySearchManager 接口（`src/memory/types.ts`）

```typescript
interface MemorySearchManager {
  // 语义 + 全文搜索
  search(query: string, opts?: { maxResults?: number; minScore?: number; sessionKey?: string }): Promise<MemorySearchResult[]>;

  // 精准读取特定文件/行范围
  readFile(params: { relPath: string; from?: number; lines?: number }): Promise<{ text: string; path: string }>;

  // 状态与诊断
  status(): MemoryProviderStatus;

  // 同步/重建
  sync?(params?: { reason?: string; force?: boolean; progress?: (update: MemorySyncProgressUpdate) => void }): Promise<void>;

  // 探测嵌入/向量可用性
  probeEmbeddingAvailability(): Promise<MemoryEmbeddingProbeResult>;
  probeVectorAvailability(): Promise<boolean>;

  close?(): Promise<void>;
}
```

### 搜索结果

```typescript
type MemorySearchResult = {
  path: string;       // 文件路径（相对于工作区）
  startLine: number;
  endLine: number;
  score: number;      // 相关性评分
  snippet: string;    // 匹配的文本
  source: "memory" | "sessions";
  citation?: string;
};
```

### 后端选项

| 后端 | 描述 |
|------|------|
| `builtin` | 默认：SQLite + FTS5 + 可选 `sqlite-vec` 向量索引 |
| `qmd` | QMD（外部进程，支持高级向量搜索） |

配置方式：

```json5
{
  plugins: { slots: { memory: "memory-core" } }   // 或 "none" 以禁用
}
```

### 面向智能体的记忆工具

| 工具 | 描述 |
|------|------|
| `memory_search` | 对记忆 + 会话文件进行语义召回 |
| `memory_get` | 精准读取记忆文件的特定行 |

系统提示词指示模型：**在回答任何关于先前工作、决策、偏好或待办事项的问题之前，先运行 `memory_search`，然后用 `memory_get` 获取所需的具体行**。

当文件不存在时，`memory_get` 会优雅降级（返回 `{ text: "", path }`，而不是抛出异常）。

### 嵌入模型

支持多种嵌入提供商：

- Voyage AI（`embeddings-voyage.ts`）
- OpenAI（`embeddings-openai.ts`）
- Gemini（`embeddings-gemini.ts`）
- Mistral（`embeddings-mistral.ts`）
- Ollama（`embeddings-ollama.ts`）
- 远程 HTTP（`embeddings-remote-fetch.ts`）

批量嵌入由 `batch-runner.ts` 管理，支持可配置的并发度和错误恢复。

---

## 5. 技能

**来源**：`src/agents/skills/`  
**概念文档**：[智能体运行时 — 技能](/zh-CN/concepts/agent#技能)

技能是**按需加载的指令文件**（Markdown），模型在运行时读取后执行特定工作流。这样可以保持基础系统提示词的精简——系统只注入紧凑的技能列表；完整指令在需要时通过 `read` 工具获取。

### 技能条目类型（`src/agents/skills/types.ts`）

```typescript
type SkillEntry = {
  skill: Skill;                        // pi-coding-agent 库的 Skill 对象
  frontmatter: ParsedSkillFrontmatter; // 解析后的 YAML 元数据
  metadata?: OpenClawSkillMetadata;    // OpenClaw 特定元数据
  invocation?: SkillInvocationPolicy;  // 调用控制
};

type OpenClawSkillMetadata = {
  always?: boolean;                    // 始终可用（即使不在智能体技能列表中）
  skillKey?: string;                   // 唯一标识符
  primaryEnv?: string;                 // 所需环境变量
  emoji?: string;
  homepage?: string;
  os?: string[];                       // OS 限制（"darwin" | "linux" | "win32"）
  requires?: {
    bins?: string[];                   // 所有这些二进制文件都必须存在
    anyBins?: string[];                // 至少一个二进制文件必须存在
    env?: string[];                    // 所需环境变量
    config?: string[];                 // 所需配置键
  };
  install?: SkillInstallSpec[];        // 安装规范
};

type SkillInvocationPolicy = {
  userInvocable: boolean;              // 用户是否可以直接调用此技能？
  disableModelInvocation: boolean;     // 阻止模型自主调用？
};
```

### 安装规范

```typescript
type SkillInstallSpec = {
  kind: "brew" | "node" | "go" | "uv" | "download";
  formula?: string;    // brew 公式
  package?: string;    // npm/pip/uv 包名
  module?: string;     // node 模块
  url?: string;        // 下载 URL
  bins?: string[];     // 安装后的预期二进制文件
  os?: string[];
};
```

### 技能来源（优先级顺序）

1. **工作区**（`<workspace>/skills/`）— 同名冲突时优先。
2. **已安装**（通过 `openclaw skills install` 安装）。
3. **内置**（随 OpenClaw 一起发布）。
4. **插件**（由插件贡献）。

### 技能在系统提示词中的体现

符合条件的技能以紧凑 XML 格式注入：

```xml
<available_skills>
  <skill>
    <name>frontend-design</name>
    <description>设计并实现前端 UI 组件</description>
    <location>/home/user/.openclaw/workspace/skills/frontend-design/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

提示词指示：
- 若**恰好一个**技能明显适用 → 用 `read` 读取其 SKILL.md，然后按照执行。
- 若**多个**可能适用 → 选择最具体的一个。
- 若**没有**明显适用的 → 不读取任何 SKILL.md。

### 技能加载（`src/agents/skills/workspace.ts`）

```typescript
loadWorkspaceSkillEntries(workspaceDir: string): Promise<SkillEntry[]>
buildWorkspaceSkillsPrompt(skillEntries: SkillEntry[], context?: SkillEligibilityContext): string
```

资格筛选会在注入前检查 OS 兼容性、所需二进制文件和所需环境变量。

---

## 6. 工具

**来源**：`src/agents/pi-tools.ts`、`src/agents/bash-tools.ts`、`src/agents/channel-tools.ts`  
**概念文档**：[系统提示词 — 工具摘要](/zh-CN/concepts/system-prompt)

所有工具通过以下函数创建：

```typescript
createOpenClawCodingTools(options?: {
  config?: OpenClawConfig;
  capabilities?: ToolCapabilities;
  toolPolicy?: ToolPolicy;
  sessionKey?: string;
  // ...
}): AnyAgentTool[]
```

### 核心文件工具

| 工具 | 描述 |
|------|------|
| `read` | 读取文件内容 |
| `write` | 创建或覆盖文件 |
| `edit` | 对文件进行精准编辑 |
| `apply_patch` | 应用多文件统一差异补丁 |
| `grep` | 按模式搜索文件内容 |
| `find` | 按 glob 模式查找文件 |
| `ls` | 列出目录内容 |

### Shell 执行工具（`src/agents/bash-tools.ts`）

**`exec`** — 运行 shell 命令（支持 PTY 用于交互式 CLI）：

```typescript
createExecTool(options: ExecToolDefaults): AnyAgentTool
```

- 执行文件沙盒边界检查。
- 支持高权限提示。
- 针对需要 TTY 的 CLI 的 PTY 模式。

**`process`** — 管理后台进程：

```typescript
createProcessTool(defaults: ProcessToolDefaults): AnyAgentTool
```

- 跟踪运行中的进程。
- 发送信号（SIGTERM、SIGKILL）。
- 轮询进程状态。

### Web 工具

| 工具 | 描述 |
|------|------|
| `web_search` | 通过 Brave API 进行 Web 搜索 |
| `web_fetch` | 获取并提取 URL 的可读内容 |

### 记忆工具

| 工具 | 描述 |
|------|------|
| `memory_search` | 对已索引的记忆 + 会话文件进行语义召回 |
| `memory_get` | 精准读取记忆文件的特定行 |

### 消息与编排工具

| 工具 | 描述 |
|------|------|
| `message` | 发送消息和频道动作 |
| `sessions_spawn` | 生成隔离的子智能体会话（或 ACP 会话） |
| `sessions_send` | 向另一会话发送消息 |
| `sessions_list` | 列出活跃会话 |
| `subagents` | 列出/引导/终止子智能体运行 |
| `agents_list` | 列出可用的智能体 ID |

### 媒体与 UI 工具

| 工具 | 描述 |
|------|------|
| `browser` | Web 浏览器控制 |
| `canvas` | 画布演示与求值 |
| `image` | 图像分析 |
| `pdf` | PDF 分析（支持向量/FTS） |
| `tts` | 文字转语音输出 |

### 系统与基础设施工具

| 工具 | 描述 |
|------|------|
| `cron` | 调度提醒和唤醒事件 |
| `gateway` | 重启/配置/更新 OpenClaw |
| `nodes` | 控制配对节点（摄像头、屏幕等） |
| `session_status` | 显示会话状态卡（时间、使用量等） |

### 频道工具（`src/agents/channel-tools.ts`）

频道专用工具按会话解析：

```typescript
listChannelAgentTools(params: { cfg?: OpenClawConfig }): ChannelAgentTool[]
resolveChannelMessageToolHints(params: { cfg?: OpenClawConfig; channel?: string | null; accountId?: string | null }): string[]
```

示例：WhatsApp 登录工具、平台特定的消息动作。

### 工具策略

`ToolPolicy` 控制每个会话可用哪些工具，以及执行命令是否需要审批。  
`ToolCapabilities` 声明运行时实际支持的能力（例如，浏览器工具需要显示器）。

---

## 7. 引导上下文系统

**来源**：`src/agents/bootstrap-files.ts`、`src/agents/bootstrap-hooks.ts`、`src/agents/bootstrap-budget.ts`

引导（Bootstrap）是在第一次模型调用之前将工作区文件加载到上下文窗口中的过程。

### 解析

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

### 预算管理（`src/agents/bootstrap-budget.ts`）

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
    bootstrapMaxChars: number;       // 单文件上限
    bootstrapTotalMaxChars: number;  // 总上限
    nearLimitRatio: number;          // 默认 0.85
  };
};
```

截断警告模式：`agents.defaults.bootstrapPromptTruncationWarning` → `"off" | "once" | "always"`（默认 `"once"`）。

### 插件钩子（`src/agents/bootstrap-hooks.ts`）

```typescript
applyBootstrapHookOverrides(params: {
  files: WorkspaceBootstrapFile[];
  workspaceDir: string;
  config?: OpenClawConfig;
  sessionKey?: string;
  agentId?: string;
}): Promise<WorkspaceBootstrapFile[]>
```

内部钩子 `agent:bootstrap` 在此处触发，允许插件添加、移除或替换引导上下文文件（例如，将 `SOUL.md` 替换为备用人设）。

---

## 8. 综合运行流程

```
1.  resolveSessionAgentIds()
    → 从会话/配置确定 agentId

2.  resolveAgentConfig()
    → 工作区目录、模型链、技能过滤器、
      记忆配置、沙盒、工具策略等

3.  resolveBootstrapContextForRun()
    → 加载 + 裁剪工作区文件：
      AGENTS.md、SOUL.md、TOOLS.md、IDENTITY.md、
      USER.md、HEARTBEAT.md、BOOTSTRAP.md、MEMORY.md
    → applyBootstrapHookOverrides()（插件拦截）

4.  buildAgentSystemPrompt()
    → 组装模块化区块：
      工具列表、安全、技能、记忆、工作区、时间、
      回复标签、消息、运行时、项目上下文
    → 注入引导文件和技能列表

5.  resolveContextEngine()
    → 加载 "legacy" 引擎（或插件覆盖）
    → 新会话调用 engine.bootstrap()

6.  createOpenClawCodingTools()
    → 文件工具（read/write/edit/grep/find/exec/process）
    → 记忆工具（memory_search / memory_get）
    → Web 工具（web_search / web_fetch）
    → 消息工具（message / sessions_spawn / …）
    → 频道专用工具
    → 媒体工具（browser / canvas / image / pdf）

7.  engine.ingest(用户消息)
    engine.assemble() → 有序消息列表 + Token 估算

8.  模型推理（pi-coding-agent）
    → 流式推送助手增量 + 工具调用
    → 工具调用分发到工具处理器
    → 工具结果反馈回上下文
    → 重复直到 stop_reason = "end_turn"

9.  engine.afterTurn()
    → 若接近 Token 上限则自动压缩
    → 会话记录持久化

10. EmbeddedPiRunResult
    { success, messages, usage, meta }
```

---

## 9. 关键文件参考

| 文件 | 用途 |
|------|------|
| `src/agents/pi-embedded-runner/run.ts` | 主智能体运行器（`runEmbeddedPiAgent`） |
| `src/agents/agent-scope.ts` | 智能体 ID + 配置解析 |
| `src/agents/agent-paths.ts` | 工作区/智能体目录解析 |
| `src/agents/system-prompt.ts` | 系统提示词组装（`buildAgentSystemPrompt`） |
| `src/agents/bootstrap-files.ts` | 引导上下文文件加载 |
| `src/agents/bootstrap-hooks.ts` | 引导阶段的插件钩子拦截 |
| `src/agents/bootstrap-budget.ts` | 引导文件的 Token 预算执行 |
| `src/agents/bash-tools.ts` | Shell 执行工具（exec、process） |
| `src/agents/channel-tools.ts` | 频道专用工具连接 |
| `src/agents/pi-tools.ts` | 工具工厂（`createOpenClawCodingTools`） |
| `src/agents/skills/types.ts` | 技能类型定义 |
| `src/agents/skills/workspace.ts` | 技能加载和提示词构建 |
| `src/context-engine/types.ts` | 上下文引擎接口 |
| `src/context-engine/registry.ts` | 引擎注册与解析 |
| `src/context-engine/init.ts` | 引擎初始化 |
| `src/memory/types.ts` | 记忆管理器接口 |
| `src/memory/manager.ts` | 记忆管理器实现 |
| `src/memory/embeddings.ts` | 嵌入提供商抽象 |

---

## 10. 相关概念文档

- [智能体运行时](/zh-CN/concepts/agent) — 工作区、引导文件、工具概览
- [智能体循环](/zh-CN/concepts/agent-loop) — 详细生命周期、钩子、流、压缩
- [系统提示词](/zh-CN/concepts/system-prompt) — 完整区块列表、提示词模式、引导注入
- [上下文](/zh-CN/concepts/context) — 上下文窗口、`/context` 命令、Token 使用
- [压缩](/zh-CN/concepts/compaction) — 上下文压缩流程
- [记忆](/zh-CN/concepts/memory) — 记忆文件、工具、写入工作流、刷新策略
- [多智能体](/zh-CN/concepts/multi-agent) — 子智能体生成与编排
- [智能体工作区](/zh-CN/concepts/agent-workspace) — 目录布局与文件指南
