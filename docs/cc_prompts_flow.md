# Claude Code System Prompt 构造流程图

> 配套文字说明见对话记录 / `CLAUDE.md`。下列 Mermaid 图按 GitHub / 飞书 Mermaid 渲染器兼容写法编写。
> 颜色图例：🔵 调用点/入口　🟣 核心阶段　🟢 静态区(可缓存)　🟠 动态区　🔴 缓存边界/击穿点　🟡 分支决策

---

## 1. 总览：三阶段流水线

```mermaid
flowchart TD
    CALL["调用点<br/>REPL.tsx (每轮) · compact.ts<br/>AgentTool · analyzeContext"]:::entry

    S1["阶段一　getSystemPrompt()<br/><i>constants/prompts.ts:444</i><br/>产出默认 prompt 的各 section"]:::stage
    S2["阶段二　buildEffectiveSystemPrompt()<br/><i>utils/systemPrompt.ts:41</i><br/>override / agent / custom / default 选择"]:::stage
    S3["阶段三　splitSysPromptPrefix()<br/><i>utils/api.ts:321</i><br/>切成带 cacheScope 的块"]:::stage

    API["Claude API 请求<br/>system + cache_control"]:::api

    CALL --> S1 -->|"string[]"| S2 -->|"SystemPrompt"| S3 -->|"SystemPromptBlock[]"| API

    classDef entry fill:#1e3a8a,stroke:#3b82f6,color:#fff
    classDef stage fill:#6b21a8,stroke:#a855f7,color:#fff
    classDef api fill:#374151,stroke:#9ca3af,color:#fff
```

---

## 2. 阶段一：getSystemPrompt 内部分支

```mermaid
flowchart TD
    START(["getSystemPrompt()"]):::entry

    Q1{"CLAUDE_CODE_SIMPLE<br/>为真?"}:::decision
    A["路径 A · 短路<br/>单行：身份 + CWD + Date"]:::stage

    Q2{"PROACTIVE/KAIROS<br/>且已激活?"}:::decision
    B["路径 B · 自主 agent prompt<br/>身份 + reminders + memory + env<br/>+ language + MCP + scratchpad<br/>+ FRC + 摘要 + ProactiveSection"]:::stage

    C["路径 C · 标准"]:::stage

    START --> Q1
    Q1 -->|yes| A
    Q1 -->|no| Q2
    Q2 -->|yes| B
    Q2 -->|no| C

    subgraph STATIC["静态区　可跨组织缓存　顺序固定"]
        direction TB
        I1["getSimpleIntroSection　身份 + CyberRisk + 禁造URL"]:::static
        I2["getSimpleSystemSection　# System"]:::static
        I3["getSimpleDoingTasksSection　# Doing tasks"]:::static
        I4["getActionsSection　# Executing actions with care"]:::static
        I5["getUsingYourToolsSection　# Using your tools"]:::static
        I6["getSimpleToneAndStyleSection　# Tone and style"]:::static
        I7["getOutputEfficiencySection　# Output efficiency"]:::static
        I1 --> I2 --> I3 --> I4 --> I5 --> I6 --> I7
    end

    BOUND(["★ SYSTEM_PROMPT_DYNAMIC_BOUNDARY ★<br/>仅 shouldUseGlobalCacheScope() 时插入"]):::boundary

    subgraph DYN["动态区　resolveSystemPromptSections 解析"]
        direction TB
        D1["session_guidance"]:::dynamic
        D2["memory · ant_model_override · env_info_simple"]:::dynamic
        D3["language · output_style"]:::dynamic
        D4["mcp_instructions　★唯一 DANGEROUS_uncached"]:::danger
        D5["scratchpad · frc · summarize_tool_results"]:::dynamic
        D6["[ant] numeric_length_anchors　[feat] token_budget · brief"]:::dynamic
        D1 --> D2 --> D3 --> D4 --> D5 --> D6
    end

    C --> STATIC --> BOUND --> DYN
    DYN --> OUT["filter(null) → string[]"]:::api

    classDef entry fill:#1e3a8a,stroke:#3b82f6,color:#fff
    classDef stage fill:#6b21a8,stroke:#a855f7,color:#fff
    classDef decision fill:#854d0e,stroke:#eab308,color:#fff
    classDef static fill:#14532d,stroke:#22c55e,color:#fff
    classDef dynamic fill:#7c2d12,stroke:#f97316,color:#fff
    classDef danger fill:#7f1d1d,stroke:#ef4444,color:#fff
    classDef boundary fill:#7f1d1d,stroke:#ef4444,color:#fff
    classDef api fill:#374151,stroke:#9ca3af,color:#fff
```

---

## 3. 动态 section 的缓存解析

```mermaid
flowchart TD
    R(["resolveSystemPromptSections(sections)"]):::entry
    LOOP["对每个 section 并行处理"]:::stage

    Q{"cacheBreak == false<br/>且 cache.has(name)?"}:::decision
    HIT["返回缓存值"]:::static
    MISS["value = await compute()"]:::dynamic
    SET["setSystemPromptSectionCacheEntry(name, value)"]:::dynamic
    RET["返回 value"]:::api

    CLEAR["/clear 或 /compact<br/>→ clearSystemPromptSections()<br/>→ 清空全部缓存"]:::danger

    R --> LOOP --> Q
    Q -->|命中| HIT
    Q -->|"未命中 / cacheBreak=true(每轮重算)"| MISS --> SET --> RET
    CLEAR -.清空.-> Q

    classDef entry fill:#1e3a8a,stroke:#3b82f6,color:#fff
    classDef stage fill:#6b21a8,stroke:#a855f7,color:#fff
    classDef decision fill:#854d0e,stroke:#eab308,color:#fff
    classDef static fill:#14532d,stroke:#22c55e,color:#fff
    classDef dynamic fill:#7c2d12,stroke:#f97316,color:#fff
    classDef danger fill:#7f1d1d,stroke:#ef4444,color:#fff
    classDef api fill:#374151,stroke:#9ca3af,color:#fff
```

---

## 4. 阶段二：优先级决策树

```mermaid
flowchart TD
    START(["buildEffectiveSystemPrompt({...})"]):::entry

    Q0{"overrideSystemPrompt?"}:::decision
    R0["return [override]<br/><b>完全替换，不加 append</b>"]:::danger

    Q1{"COORDINATOR_MODE<br/>且无 agent?"}:::decision
    R1["[coordinator, ...append]"]:::stage

    Q2{"有 mainThreadAgentDefinition?"}:::decision
    R2A["[...default,<br/>'# Custom Agent Instructions'+agent,<br/>...append]　(proactive: 追加)"]:::dynamic
    R2B["[agent, ...append]　(agent 替换 default)"]:::dynamic

    Q3{"有 customSystemPrompt?"}:::decision
    R3["[custom, ...append]　(custom 替换 default)"]:::dynamic
    R4["[...default, ...append]　(用 default)"]:::static

    START --> Q0
    Q0 -->|yes| R0
    Q0 -->|no| Q1
    Q1 -->|yes| R1
    Q1 -->|no| Q2
    Q2 -->|"有 + proactive激活"| R2A
    Q2 -->|"有 (普通)"| R2B
    Q2 -->|无| Q3
    Q3 -->|yes| R3
    Q3 -->|no| R4

    classDef entry fill:#1e3a8a,stroke:#3b82f6,color:#fff
    classDef stage fill:#6b21a8,stroke:#a855f7,color:#fff
    classDef decision fill:#854d0e,stroke:#eab308,color:#fff
    classDef static fill:#14532d,stroke:#22c55e,color:#fff
    classDef dynamic fill:#7c2d12,stroke:#f97316,color:#fff
    classDef danger fill:#7f1d1d,stroke:#ef4444,color:#fff
```

---

## 5. 阶段三：切缓存块（三种模式）

```mermaid
flowchart TD
    START(["splitSysPromptPrefix(prompt, {skipGlobal})"]):::entry
    Q{"shouldUseGlobalCacheScope()?"}:::decision

    M1["模式① 有MCP工具(skip=true) · 3块<br/>attribution(null) | prefix(org) | rest(org)"]:::dynamic
    M2["模式② 找到边界标记 · ≤4块 (仅1P)<br/>attribution(null) | prefix(null)<br/>静态(global ◄跨组织复用) | 动态(null)"]:::static
    M3["模式③ false/无边界/3P · 3块<br/>attribution(null) | prefix(org) | rest(org)"]:::dynamic

    START --> Q
    Q -->|"true + 有MCP工具"| M1
    Q -->|"true + 找到边界"| M2
    Q -->|"false / 无边界 / 3P"| M3

    PFX["prefix 文本 ← getCLISyspromptPrefix() (system.ts)<br/>vertex/交互式 → 'You are Claude Code...CLI'<br/>非交互+无append → 'You are a Claude agent...SDK'<br/>非交互+有append → '...CLI...within Agent SDK'"]:::stage

    M1 -.-> PFX
    M2 -.-> PFX
    M3 -.-> PFX

    classDef entry fill:#1e3a8a,stroke:#3b82f6,color:#fff
    classDef stage fill:#6b21a8,stroke:#a855f7,color:#fff
    classDef decision fill:#854d0e,stroke:#eab308,color:#fff
    classDef static fill:#14532d,stroke:#22c55e,color:#fff
    classDef dynamic fill:#7c2d12,stroke:#f97316,color:#fff
```

---

## 6. Subagent 旁路（不走 getSystemPrompt）

```mermaid
flowchart TD
    START(["runAgent / AgentTool"]):::entry
    BASE["起始 = agent 自定义 prompt<br/>或 DEFAULT_AGENT_PROMPT"]:::stage
    ENH["enhanceSystemPromptWithEnvDetails()<br/><i>prompts.ts:763</i>"]:::stage

    N1["+ Notes<br/>绝对路径 / 回复带文件路径<br/>不用 emoji / tool 前不加冒号"]:::dynamic
    N2["+ DiscoverSkills 指引<br/>(EXPERIMENTAL_SKILL_SEARCH 且工具启用)"]:::dynamic
    N3["+ computeEnvInfo　# Environment"]:::dynamic

    START --> BASE --> ENH --> N1 --> N2 --> N3 --> OUT["string[]"]:::api

    classDef entry fill:#1e3a8a,stroke:#3b82f6,color:#fff
    classDef stage fill:#6b21a8,stroke:#a855f7,color:#fff
    classDef dynamic fill:#7c2d12,stroke:#f97316,color:#fff
    classDef api fill:#374151,stroke:#9ca3af,color:#fff
```
