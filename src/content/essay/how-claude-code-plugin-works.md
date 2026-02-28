---
title: "Claude Code的Plugin是怎么运行的"
date: 2026-02-28
draft: false
archive: true
tags: ["claude-code", "plugin", "prompt-engineering", "AI-tools", "superpowers"]
---

> 基于对 superpowers v4.3.1 插件实际文件的完整审计编写。

## 1. 安装plugin后发生了什么

### 1.1 标记插件启用

通过 `/plugin install superpowers@claude-plugins-official` 在CLI中交互可以直接安装，会自动解析插件标识完成下载。

```
步骤 1：解析插件标识
         "superpowers@claude-plugins-official"
              ↓                    ↓
           插件名              市场名（官方市场，自动可用）

步骤 2：从市场源拉取插件
         市场注册在 $CLAUDE_CONFIG_DIR/plugins/marketplaces/ 下
         官方市场源：github.com/anthropics/claude-plugins-official
                    ↓
         读取市场的 .claude-plugin/marketplace.json
         找到 superpowers 的 source 路径
                    ↓
         下载插件文件

步骤 3：缓存到本地
         存储路径：$CLAUDE_CONFIG_DIR/plugins/cache/claude-plugins-official/superpowers/4.3.1/
         （注意：不是 ~/.claude/，而是 $CLAUDE_CONFIG_DIR 指向的目录）
```

完成后写入安装记录和项目配置，标记插件开启。

```json
步骤 4：写入安装记录
         记录到 $CLAUDE_CONFIG_DIR/plugins/installed_plugins.json：
         {
           "superpowers@claude-plugins-official": {
             "scope": "project",          ← 可以是 project 或 user
             "version": "4.3.1",
             "projectPath": "/res/dev/claude-working"
           }
         }

步骤 5：写入项目配置
         写入 .claude/settings.json：
         {
           "enabledPlugins": {
             "superpowers@claude-plugins-official": true
           }
         }
```

### 实际目录结构

插件文件被下载并缓存到 `~/.claude/plugins/cache/` 目录。**不会**出现在项目的 `.claude/skills/` 中。但是依然是可以被系统扫到的  `superpowers:*` skill。

```
superpowers/4.3.1/
├── .claude-plugin/
│   ├── plugin.json                  ← 插件清单（必需）
│   └── marketplace.json             ← 市场清单
├── skills/                          ← 15 个 Skills
│   ├── brainstorming/SKILL.md
│   ├── test-driven-development/SKILL.md
│   ├── systematic-debugging/SKILL.md
│   ├── writing-plans/SKILL.md
│   ├── executing-plans/SKILL.md
│   ├── subagent-driven-development/SKILL.md
│   ├── dispatching-parallel-agents/SKILL.md
│   ├── requesting-code-review/SKILL.md
│   ├── receiving-code-review/SKILL.md
│   ├── verification-before-completion/SKILL.md
│   ├── finishing-a-development-branch/SKILL.md
│   ├── using-git-worktrees/SKILL.md
│   ├── using-superpowers/SKILL.md
│   ├── writing-skills/SKILL.md
│   └── code-quality-reviewer-prompt/SKILL.md
├── agents/                          ← 子代理定义
│   └── code-reviewer.md             ← 代码审查代理
├── hooks/                           ← 事件钩子
│   ├── hooks.json                   ← 钩子配置
│   ├── session-start                ← SessionStart 钩子脚本（bash）
│   └── run-hook.cmd                 ← 跨平台 polyglot 包装器
├── commands/                        ← 斜杠命令（目录存在但当前为空）
├── lib/
│   └── skills-core.js               ← Skill 发现/解析工具库
├── docs/                            ← 文档
├── tests/                           ← 测试套件
├── .cursor-plugin/plugin.json       ← Cursor 编辑器兼容
├── .opencode/                       ← OpenCode 兼容
├── .codex/                          ← Codex 兼容
└── .mcp.json                        ← MCP 服务器配置（可选）
```

### plugin.json 实际内容

```
{
  "name": "superpowers",
  "description": "Core skills library for Claude Code: TDD, debugging, collaboration patterns, and proven techniques",
  "version": "4.3.1",
  "author": { "name": "Jesse Vincent", "email": "jesse@fsck.com" },
  "homepage": "https://github.com/obra/superpowers",
  "repository": "https://github.com/obra/superpowers",
  "license": "MIT"
}
```



## 2. 启动时插件是如何注入上下文的

### 2.1 完整注入流程图

```
Claude Code 启动（或 resume/clear/compact）
    │
    │ ══════════ 阶段 A：静态组装 ══════════
    │
    ├─① 读取 .claude/settings.json
    │      └─ enabledPlugins → 确定激活哪些插件
    │
    ├─② 从插件缓存加载已启用插件
    │      └─ $CLAUDE_CONFIG_DIR/plugins/cache/<市场>/<插件>/<版本>/
    │             │
    │             ├─ 读取 .claude-plugin/plugin.json → 插件元信息
    │             ├─ 扫描 skills/*/SKILL.md → 提取 YAML frontmatter (name + description)
    │             ├─ 扫描 agents/*.md → 注册可用子代理
    │             └─ 读取 hooks/hooks.json → 注册事件钩子
    │
    ├─③ 扫描项目 .claude/skills/*/SKILL.md → 提取独立 skill 的 frontmatter
    │
    ├─④ 读取 CLAUDE.md → 项目规则
    │
    ├─⑤ 组装系统提示（System Prompt）
    │      │
    │      ├─ [常驻] 基础 System Prompt（Claude Code 内置指令）
    │      ├─ [常驻] CLAUDE.md 内容（项目规则）
    │      └─ [常驻] 所有 skill 的 name + description 列表 ← Level 1 元数据
    │
    │ ══════════ 阶段 B：Hook 注入 ══════════
    │
    ├─⑥ 执行 SessionStart Hook
    │      │
    │      │  superpowers 的 hooks.json 定义：
    │      │  {
    │      │    "hooks": {
    │      │      "SessionStart": [{
    │      │        "matcher": "startup|resume|clear|compact",
    │      │        "hooks": [{
    │      │          "type": "command",
    │      │          "command": "'${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd' session-start"
    │      │        }]
    │      │      }]
    │      │    }
    │      │  }
    │      │
    │      └─ session-start 脚本执行：
    │             ├─ 读取 skills/using-superpowers/SKILL.md 完整内容
    │             ├─ 包裹在 <EXTREMELY_IMPORTANT> 标签中
    │             └─ 输出 JSON: { "additional_context": "..." }
    │                    │
    │                    └─ Claude Code 将其注入为 <system-reminder>  ← Level 0 行为规则
    │
    │ ══════════ 阶段 C：对话中按需加载 ══════════
    │
    ├─⑦ 用户发消息 / Claude 判断需要某个 skill
    │      └─ 调用 Skill 工具 → 加载 SKILL.md 正文              ← Level 2 完整指导
    │
    └─⑧ Skill 执行过程中按需引用
           ├─ Read references/*.md → 参考文档                   ← Level 3 捆绑资源
           ├─ Bash scripts/*.py → 执行脚本                     ← Level 3 捆绑资源
           └─ 引用 assets/* → 输出资源                          ← Level 3 捆绑资源
```



### 2.2 启动时自发现

Claude Code 启动时：

1. 读取 `enabledPlugins`，找到所有启用的插件
2. 从缓存目录读取插件的 `.claude-plugin/plugin.json` 清单
3. 扫描插件内部的标准目录结构：

```
superpowers/                          ← 缓存中的插件目录
├── .claude-plugin/
│   └── plugin.json                   ← 插件清单（必需）
├── skills/                           ← Skills（就是你看到的那些）
│   ├── brainstorming/
│   │   └── SKILL.md
│   ├── test-driven-development/
│   │   └── SKILL.md
│   ├── systematic-debugging/
│   │   └── SKILL.md
│   └── ... (其他 skill)
├── commands/                         ← 斜杠命令（可选）
├── agents/                           ← 子代理定义（可选）
├── hooks/
│   └── hooks.json                    ← 事件钩子（可选）
└── .mcp.json                         ← MCP 服务器（可选）
```



### 2.3 四级注入详解

superpowers 通过 SessionStart Hook 在**每次会话启动时**注入一段行为规则到 `<system-reminder>` 中。这段内容来自 `using-superpowers` skill 的完整正文，包含：

| 注入内容                                                     | 作用                                 |
| ------------------------------------------------------------ | ------------------------------------ |
| "If you think there is even a 1% chance a skill might apply, you ABSOLUTELY MUST invoke the skill" | 强制 Claude 主动检查并触发 skill     |
| Red Flags 表格（12 条思维陷阱）                              | 阻止 Claude 找理由跳过 skill         |
| Skill Priority 规则                                          | 多 skill 冲突时的优先级决策          |
| "Invoke relevant skills BEFORE any response"                 | 要求 skill 在回复之前触发            |
| Skill Types 分类（Rigid / Flexible）                         | 指导 Claude 如何执行不同类型的 skill |

**效果等同于在 CLAUDE.md 中写了一条最高优先级的规则**，但用户不直接可见、不可编辑。

触发时机：`startup`（新会话）、`resume`（恢复会话）、`clear`（清除）、`compact`（压缩上下文）——确保**任何情况下都会注入**。

#### Level 1：Skill 元数据（始终在系统提示中）

所有已启用 skill 的 `name` + `description`（来自 SKILL.md 的 YAML frontmatter）被拼接到系统提示：

```
The following skills are available for use with the Skill tool:
- superpowers:brainstorming: You MUST use this before any creative work...
- superpowers:test-driven-development: Use when implementing any feature...
- pdf: Use this skill whenever the user wants to do anything with PDF files...
（共 24 条，包括 15 个 superpowers skill + 9 个独立 skill）
```

每条约 ~100 tokens，总计约 ~2400 tokens 常驻开销。

#### Level 2：Skill 正文（Skill 工具调用时）

当 Claude 通过 `Skill` 工具调用某个 skill 时，SKILL.md 的 Markdown 正文被加载到上下文。

例如调用 `superpowers:brainstorming` → 加载完整的头脑风暴流程指导。

通常 <5000 词。

#### Level 3：捆绑资源（Skill 执行过程中按需）

- `references/*.md` → 通过 Read 工具加载（API 文档、最佳实践等）
- `scripts/*.py` → 通过 Bash 工具执行（代码生成、数据处理等）
- `assets/*` → 被引用到输出中





### 2.4 其他注入：子代理注册

superpowers 的 `agents/code-reviewer.md` 被注册为可用子代理。当 Claude 使用 Task 工具时，`code-reviewer` 出现在可选的子代理类型中。

代理定义格式（markdown + frontmatter）：

```yaml
---
name: code-reviewer
description: Use this agent when a major project step has been completed...
model: inherit
---

（代理的详细行为指导）
```

## 三、依赖了 Claude Code 的什么基础功能

### 3.1 核心依赖

| 基础功能                 | 插件如何利用                                               | 必需/可选 |
| ------------------------ | ---------------------------------------------------------- | --------- |
| **Skill 系统**           | `skills/*/SKILL.md` 被注册为可用 skill，元数据注入系统提示 | 核心      |
| **Skill 工具**           | Claude 通过 `Skill` tool 调用触发 skill 正文加载           | 核心      |
| **Hooks 系统**           | `hooks/hooks.json` 注册 SessionStart 钩子，注入行为规则    | 核心      |
| **Agent 系统**           | `agents/*.md` 注册子代理，在 Task 工具中可选               | 可选      |
| **settings.json**        | `enabledPlugins` 字段控制插件启用/禁用                     | 核心      |
| **`$CLAUDE_CONFIG_DIR`** | 插件缓存目录的根路径                                       | 核心      |

### 3.2 核心依赖详解：Hooks 系统如何注入行为规则

这是 superpowers 插件最关键、也最容易被忽视的机制。普通 skill 需要 Claude 主动调用 `Skill` 工具才能加载，但 superpowers 通过 Hook 在会话启动时**强制注入**了一段行为规则，Claude 没有"选择不加载"的机会。

以下是基于实际源码（`hooks/hooks.json`、`hooks/session-start`）的逐步拆解：

#### 第 1 步：Claude Code 触发事件

当你启动会话（`startup`）、恢复会话（`resume`）、清除（`clear`）或压缩上下文（`compact`）时，Claude Code 内部触发 `SessionStart` 事件。

Claude Code 查找所有已启用插件的 `hooks/hooks.json`，发现 superpowers 注册了该事件：

```json
{
  "hooks": {
    "SessionStart": [{
      "matcher": "startup|resume|clear|compact",   // ← 匹配哪些触发场景
      "hooks": [{
        "type": "command",                          // ← 类型：执行一条 shell 命令
        "command": "'${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd' session-start",
        "async": false                              // ← 同步执行，必须等它跑完才继续
      }]
    }]
  }
}
```

#### 第 2 步：执行 shell 脚本

Claude Code 执行该 command。`run-hook.cmd` 是跨平台 polyglot 包装器（Windows 走 batch 找 bash.exe，Unix 直接 `exec bash`），最终调用同目录下的 `session-start` 脚本。

`session-start` 脚本做了 3 件事：

```bash
# 1. 读取 using-superpowers skill 的完整 SKILL.md 文件内容
using_superpowers_content=$(cat "${PLUGIN_ROOT}/skills/using-superpowers/SKILL.md")

# 2. 用 <EXTREMELY_IMPORTANT> 标签包裹，加强语气
session_context="<EXTREMELY_IMPORTANT>
You have superpowers.
**Below is the full content of your 'superpowers:using-superpowers' skill...**
（SKILL.md 完整内容）
</EXTREMELY_IMPORTANT>"

# 3. JSON 转义后输出到 stdout
{
  "additional_context": "（上面那段内容）",
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "（同样的内容，兼容不同平台）"
  }
}
```

#### 第 3 步：Claude Code 接收并解析

Claude Code 捕获脚本的 stdout，解析 JSON，提取 `additional_context` 字段的值。

这是 Hook 的**协议约定**：脚本输出的 JSON 中包含 `additional_context` 字段 → 其内容会被注入到 LLM 上下文中。

#### 第 4 步：注入为 `<system-reminder>`

Claude Code 将 `additional_context` 的内容包裹在 `<system-reminder>` 标签中，插入到对话的系统消息里。最终 LLM 看到的是：

```xml
<system-reminder>
SessionStart:resume hook success: Success
</system-reminder>
<system-reminder>
SessionStart hook additional context: <EXTREMELY_IMPORTANT>
You have superpowers.

**Below is the full content of your 'superpowers:using-superpowers' skill...**

（一大段行为规则：强制检查 skill、12 条思维陷阱表、优先级规则等）

</EXTREMELY_IMPORTANT>
</system-reminder>
```

#### 完整管道图

```
Claude Code 启动/恢复/清除/压缩
       │
       ▼
  触发 SessionStart 事件
       │
       ▼
  查找匹配的 hooks.json         ← "matcher": "startup|resume|clear|compact"
       │
       ▼
  执行 shell 命令               ← run-hook.cmd session-start
       │
       ▼
  session-start 脚本
       │
       ├─ cat SKILL.md                            ← 读取 skill 文件
       │
       ├─ 包裹 <EXTREMELY_IMPORTANT> 标签          ← 加强语气标记
       │
       └─ 输出 JSON 到 stdout                      ← { "additional_context": "..." }
              │
              ▼
  Claude Code 解析 stdout JSON
       │
       ├─ 提取 additional_context 字段
       │
       └─ 包裹为 <system-reminder> 注入对话上下文
              │
              ▼
  LLM 收到的上下文中多了一段 system-reminder
  内容 = using-superpowers 的完整行为规则
```

#### 为什么这是推荐的最佳实践

1. **解决鸡和蛋问题**：如果 `using-superpowers` 只是普通 skill，Claude 需要先"知道要用它"才会加载它——但它怎么知道呢？Hook 在会话开始时强制注入，绕过了这个循环依赖。

2. **不会丢失**：matcher 包含 `resume|clear|compact`，所以即使上下文被压缩或会话被恢复，行为规则都会重新注入。

3. **对用户不透明**：注入内容在 `<system-reminder>` 中，普通用户不会直接看到，但 LLM 会完全遵循。这既是优点（无缝体验）也是风险点（第三方插件可以偷偷注入指令）。

4. **效果等同于 CLAUDE.md**：最终注入的内容和在 CLAUDE.md 中写规则的效果完全一样——都是往 LLM 上下文里塞指令文本。只是入口不同：一个走文件读取，一个走 Hook 注入。

### 3.3 间接依赖（Skill 执行时用到）

| 基础功能                  | 场景                                                  |
| ------------------------- | ----------------------------------------------------- |
| **Bash 工具**             | 执行 `scripts/` 下的 Python/JS 脚本                   |
| **Read 工具**             | 读取 `references/` 下的参考文档                       |
| **Task 工具**             | `dispatching-parallel-agents` 等 skill 调度并行子代理 |
| **TaskCreate/TaskUpdate** | `writing-plans` 等 skill 要求用 todo 跟踪进度         |
| **EnterPlanMode**         | `brainstorming` skill 在实现前先进入计划模式          |
| **CLAUDE.md**             | 项目规则，与 plugin 注入的规则共同生效                |
| **settings.local.json**   | 权限白名单决定 skill 中的工具调用能否执行             |

### 3.4 依赖关系图

```
┌─────────────────────────────────────────────────────────┐
│                    Claude Code 基础设施                    │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │ Skill 系统   │  │ Hooks 系统   │  │  Agent 系统    │  │
│  │ (发现+加载)  │  │ (事件驱动)   │  │  (子代理注册)  │  │
│  └──────┬──────┘  └──────┬──────┘  └───────┬────────┘  │
│         │                │                  │           │
│  ┌──────┴────────────────┴──────────────────┴────────┐  │
│  │              Plugin 管理器                          │  │
│  │  (marketplace → download → cache → register)      │  │
│  └──────┬────────────────────────────────────────────┘  │
│         │                                               │
│  ┌──────┴──────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ settings.json│  │ CLAUDE.md│  │ settings.local    │   │
│  │ (启用控制)   │  │ (规则)    │  │ (权限白名单)     │   │
│  └─────────────┘  └──────────┘  └──────────────────┘   │
│                                                         │
│  ┌─────────┐ ┌──────┐ ┌──────┐ ┌────────────────────┐  │
│  │ Bash    │ │ Read │ │ Task │ │ TaskCreate/Update   │  │
│  │ (脚本)  │ │(引用) │ │(代理)│ │ (任务跟踪)         │  │
│  └─────────┘ └──────┘ └──────┘ └────────────────────┘  │
└─────────────────────────────────────────────────────────┘
          ↑          ↑          ↑          ↑
          │          │          │          │
    ┌─────┴──────────┴──────────┴──────────┴─────┐
    │           superpowers 插件                   │
    │  skills/ + agents/ + hooks/ + lib/          │
    └─────────────────────────────────────────────┘
```

---

## 四、superpowers 插件都实现了什么

### 4.1 组件清单

| 组件类型    | 数量 | 内容                           |
| ----------- | ---- | ------------------------------ |
| Skills      | 15   | 覆盖开发全生命周期的工作流指导 |
| Agents      | 1    | code-reviewer 代码审查子代理   |
| Hooks       | 1    | SessionStart 钩子注入行为规则  |
| Lib         | 1    | skills-core.js 技能发现工具    |
| Commands    | 0    | 目录存在但当前为空             |
| MCP Servers | 0    | 无                             |

### 4.2 15 个 Skill 按开发阶段分类

```
┌─────────────────────────────────────────────────────────────┐
│                     软件开发生命周期                           │
│                                                             │
│  ┌─────────┐   ┌──────────┐   ┌───────────────────────┐    │
│  │ 构思     │   │ 设计      │   │ 开发                   │    │
│  │         │   │          │   │                       │    │
│  │ brain-  │──▶│ writing- │──▶│ test-driven-dev       │    │
│  │ storming│   │ plans    │   │ subagent-driven-dev   │    │
│  │         │   │          │   │ dispatching-parallel   │    │
│  │         │   │          │   │ using-git-worktrees    │    │
│  │         │   │          │   │ executing-plans        │    │
│  └─────────┘   └──────────┘   └───────────┬───────────┘    │
│                                            │                │
│  ┌─────────┐   ┌──────────┐   ┌───────────▼───────────┐    │
│  │ 收尾     │   │ 审查      │   │ 验证/调试              │    │
│  │         │   │          │   │                       │    │
│  │ finish- │◀──│ request- │◀──│ verification-before-  │    │
│  │ ing-a-  │   │ ing-code-│   │ completion            │    │
│  │ dev-    │   │ review   │   │ systematic-debugging  │    │
│  │ branch  │   │ receiv-  │   │                       │    │
│  │         │   │ ing-code-│   │                       │    │
│  │         │   │ review   │   │                       │    │
│  └─────────┘   └──────────┘   └───────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────┐                │
│  │ 元技能                                   │                │
│  │ using-superpowers / writing-skills       │                │
│  │ code-quality-reviewer-prompt             │                │
│  └─────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 每个 Skill 的作用

| Skill                            | 类型     | 作用                                                     |
| -------------------------------- | -------- | -------------------------------------------------------- |
| `brainstorming`                  | Rigid    | 任何创造性工作之前必须执行。探索用户意图、需求、设计方案 |
| `writing-plans`                  | Rigid    | 有需求规格后，编写多步骤实施计划                         |
| `executing-plans`                | Rigid    | 在独立会话中执行实施计划，带审查检查点                   |
| `test-driven-development`        | Rigid    | 先写测试再写实现，强制 TDD 流程                          |
| `subagent-driven-development`    | Flexible | 在当前会话中用子代理并行执行计划                         |
| `dispatching-parallel-agents`    | Flexible | 分派 2+ 个无依赖的独立任务并行执行                       |
| `using-git-worktrees`            | Flexible | 创建隔离的 git worktree 进行功能开发                     |
| `systematic-debugging`           | Rigid    | 遇到 bug/测试失败时的系统化调试流程                      |
| `verification-before-completion` | Rigid    | 声称完成之前必须运行验证命令，证据先于断言               |
| `requesting-code-review`         | Flexible | 完成任务后请求代码审查                                   |
| `receiving-code-review`          | Rigid    | 收到审查反馈后的处理流程，防止盲目同意                   |
| `finishing-a-development-branch` | Flexible | 开发完成后的分支收尾：合并/PR/清理                       |
| `using-superpowers`              | Rigid    | 元技能：如何发现和使用其他 skill                         |
| `writing-skills`                 | Flexible | 创建、编辑、验证新 skill                                 |
| `code-quality-reviewer-prompt`   | Flexible | 代码质量审查的评估标准                                   |

### 4.4 Agent 的作用

| Agent           | 作用                                                         |
| --------------- | ------------------------------------------------------------ |
| `code-reviewer` | 子代理，在 Task 工具中可选。用于完成重大步骤后对照计划和编码规范进行代码审查 |

### 4.5 Hook 的作用

| Hook            | 事件                                        | 作用                                                         |
| --------------- | ------------------------------------------- | ------------------------------------------------------------ |
| `session-start` | SessionStart (startup/resume/clear/compact) | 注入 `using-superpowers` 完整内容作为行为规则，确保 Claude 在每个会话中都主动检查和触发 skill |

### 4.6 核心设计思想

superpowers 的本质是**通过 prompt engineering 将软件工程最佳实践编码为可复用的工作流指导**：

1. **Hook 注入行为规则** → 让 Claude 主动寻找并使用 skill（解决"不知道要用"的问题）
2. **Skill 元数据描述触发条件** → 让 Claude 知道什么时候该用哪个 skill（解决"不知道什么时候用"的问题）
3. **Skill 正文提供详细指导** → 让 Claude 知道具体怎么做（解决"不知道怎么用"的问题）
4. **Agent 提供专业子角色** → 让特定任务由专门的代理处理（解决"一个角色不够"的问题）

---

## 五、开发自己的插件：必做 vs 可选

### 5.1 必做项

#### (1) `.claude-plugin/plugin.json`（必需）

插件身份证明，没有这个文件就不是合法插件：

```json
{
  "name": "my-plugin",              // 必填，kebab-case
  "version": "1.0.0",               // 必填，语义化版本
  "description": "插件做什么",        // 必填
  "author": { "name": "你的名字" },  // 建议填写
  "license": "MIT"                   // 建议填写
}
```

#### (2) 至少一个功能组件

插件必须包含以下至少一项，否则没有意义：

- `skills/` — Skill 定义
- `agents/` — 子代理定义
- `hooks/` — 事件钩子
- `commands/` — 斜杠命令
- `.mcp.json` — MCP 服务器

#### (3) 最小可用结构

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          ← 必需
└── skills/
    └── my-skill/
        └── SKILL.md         ← 至少一个 skill
```

### 5.2 可选项（按实用性排序）

| 组件                      | 用途                 | 何时需要                         |
| ------------------------- | -------------------- | -------------------------------- |
| `hooks/hooks.json` + 脚本 | 在事件发生时自动执行 | 需要注入行为规则、自动化检查     |
| `agents/*.md`             | 注册专用子代理       | 需要独立的专业角色执行子任务     |
| `commands/*.md`           | 注册斜杠命令         | 需要用户通过 `/command` 显式触发 |
| `.mcp.json`               | 声明 MCP 服务器      | 需要集成外部 API/服务            |
| `lib/`                    | 工具库               | 有复杂的脚本逻辑需要复用         |
| `references/`             | 参考文档             | skill 正文太长，需要拆分按需加载 |
| `scripts/`                | 可执行脚本           | 有重复性的代码生成/数据处理      |
| `tests/`                  | 测试                 | 正式发布前验证质量               |
| `.cursor-plugin/`         | Cursor 兼容          | 需要跨编辑器支持                 |
| `.opencode/`              | OpenCode 兼容        | 需要跨平台支持                   |
| `.codex/`                 | Codex 兼容           | 需要跨平台支持                   |

### 5.3 SKILL.md 编写要点

```yaml
---
name: my-plugin:my-skill              # 建议用 "插件名:技能名" 格式
description: >
  精确描述触发条件。这段文字决定 Claude 是否会使用这个 skill。
  要写清楚：(1) 做什么 (2) 什么时候触发 (3) 关键词列表
---

# Skill 标题

## 前置条件（可选）
触发此 skill 前需要确认什么

## 工作流程（核心）
1. 步骤一
2. 步骤二
3. 步骤三

## 检查清单（可选）
- [ ] 验证项 1
- [ ] 验证项 2

## 注意事项（可选）
禁止做什么、边界条件等
```

**关键原则**：

- `description` 是唯一的触发机制——写得好不好直接决定 skill 会不会被调用
- 正文保持精简（<500 行），大段参考资料拆到 `references/`
- 区分 Rigid（必须严格遵循）和 Flexible（可适当调整）skill
- 重复性代码写成 `scripts/`，不要让 Claude 每次重写

### 5.4 hooks.json 编写要点

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "'${CLAUDE_PLUGIN_ROOT}/hooks/my-hook-script'",
            "async": false
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "'${CLAUDE_PLUGIN_ROOT}/hooks/after-write'"
          }
        ]
      }
    ]
  }
}
```

可用的 Hook 事件：

| 事件               | 触发时机                |
| ------------------ | ----------------------- |
| `SessionStart`     | 会话启动/恢复/清除/压缩 |
| `PreToolUse`       | 工具调用之前            |
| `PostToolUse`      | 工具调用之后            |
| `UserPromptSubmit` | 用户提交消息后          |

Hook 脚本输出 JSON，可包含 `additional_context` 字段注入上下文。

### 5.5 Agent 编写要点

```markdown
---
name: my-agent
description: 描述什么时候应该使用这个代理
model: inherit
---

# 代理行为指导

（详细描述代理的职责、工作流程、输出格式）
```

### 5.6 发布方式

| 方式       | 操作                                            | 适用场景                     |
| ---------- | ----------------------------------------------- | ---------------------------- |
| 官方市场   | 提交 PR 到 `anthropics/claude-plugins-official` | 公开发布                     |
| 第三方市场 | 创建自己的 marketplace 仓库，用户手动添加       | 团队/组织内部                |
| 本地安装   | 放到 `.claude/skills/`                          | 仅当前项目，绕过 plugin 系统 |
| skills.sh  | `npx skills add`                                | 快速分享单个 skill           |

### 5.7 本地开发/测试最简路径

不想走 plugin 发布流程：

```bash
# 方式 1：作为独立 skill（最简单）
mkdir -p .claude/skills/my-skill
# 编写 SKILL.md → 重启会话即可

# 方式 2：作为完整 plugin（需要 hook/agent 等能力时）
# 创建完整的 plugin 目录结构
# 通过 /plugin install 从本地路径安装
```

---

## 六、Plugin 与独立 Skill 对比

| 维度                 | 独立 Skill                         | Plugin                                      |
| -------------------- | ---------------------------------- | ------------------------------------------- |
| **存放位置**         | `.claude/skills/`（项目内）        | `$CLAUDE_CONFIG_DIR/plugins/cache/`（全局） |
| **安装方式**         | 手动复制 / `npx skills add`        | `/plugin install`                           |
| **配置**             | 自动发现，无需配置                 | `settings.json` 的 `enabledPlugins`         |
| **作用域**           | 仅当前项目                         | 可 project 级或 user 级                     |
| **包含内容**         | 仅 SKILL.md + scripts + references | skill + agents + hooks + commands + MCP     |
| **能否注入行为规则** | 不能（无 hook 能力）               | 能（通过 SessionStart hook）                |
| **能否注册子代理**   | 不能                               | 能（agents/ 目录）                          |
| **能否注册斜杠命令** | 不能                               | 能（commands/ 目录）                        |
| **能否集成外部服务** | 不能                               | 能（.mcp.json）                             |
| **版本管理**         | 无                                 | 有（语义化版本）                            |
| **分发**             | Git 仓库 / skills.sh               | 市场（marketplace）                         |
| **命名空间**         | 平级名称（`pdf`）                  | 带前缀（`superpowers:brainstorming`）       |
| **启用/禁用**        | 删文件                             | `settings.json` 中设为 `false`              |
| **跨平台支持**       | 仅 Claude Code                     | 可兼容 Cursor / OpenCode / Codex            |
| **适用场景**         | 单一功能、快速迭代                 | 完整工作流、团队分发                        |

**总结**：Plugin = 独立 Skill 的超集 + 管理机制。如果你只需要一个 skill，直接放 `.claude/skills/` 就够了。如果需要 hook 注入行为规则、注册子代理、集成外部服务，或者要分发给团队，那就做成 plugin。

---

## 附录 A：Marketplace 机制

### 官方市场

`claude-plugins-official`（GitHub: `anthropics/claude-plugins-official`），自动可用。

包含 28+ 官方插件 + 12 个外部 MCP 集成插件（github、gitlab、slack、playwright 等）。

### 第三方市场

```bash
/plugin marketplace add <github-user>/<repo>
```

### Marketplace 清单格式

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "marketplace-name",
  "version": "1.0.0",
  "description": "Description",
  "owner": { "name": "...", "email": "..." },
  "plugins": [
    {
      "name": "plugin-name",
      "description": "...",
      "source": "./plugins/plugin-name",
      "category": "development"
    }
  ]
}
```

---

## 附录 B：环境变量

| 变量                  | 作用                                   |
| --------------------- | -------------------------------------- |
| `CLAUDE_CONFIG_DIR`   | Claude 配置根目录（插件缓存在此下）    |
| `CLAUDE_PLUGIN_ROOT`  | 当前插件的根目录（在 hook 脚本中使用） |
| `CLAUDE_SETTINGS_DIR` | 设置文件目录                           |

---

## 附录 C：安全注意事项

1. **Hook 可以注入任意上下文**：SessionStart hook 能注入 `<EXTREMELY_IMPORTANT>` 级别的指令，效果等同于 system prompt。安装第三方插件前务必审计 hook 脚本。
2. **Script 可以执行任意代码**：`scripts/` 目录下的脚本通过 Bash 工具执行，拥有完整的 shell 权限。
3. **插件对用户不完全透明**：hook 注入的内容在 `<system-reminder>` 中，普通用户不一定会注意到。
4. **建议审计流程**：安装前检查 `hooks/`、`scripts/`、`lib/` 中的所有可执行文件，确认无恶意行为。
