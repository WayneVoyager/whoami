---
title: "AgentDrop — 开发流程记录"
date: 2026-02-28
draft: false
archive: true
tags: ["AgentDrop", "file-transfer", "claude-code", "skill", "python"]
---

> GitHub 仓库：[WayneVoyager/AgentDrop](https://github.com/WayneVoyager/AgentDrop)

**日期**：2026-02-28
**参与者**：用户（产品负责人）+ AI（Claude Code 执行开发）
**产出**：AgentDrop Skill — 远程服务器文件传输工具

---

## 一、想法的诞生

### 触发 brainstorming

在 Claude Code 中输入 `/brainstorm`，触发了 superpowers:brainstorming skill。Claude 问："你想做什么类型的新项目？"

### 用户原始 Prompt（一字未改）

> 我一般使用claude code将他部署在云上服务器，我自己在pc上通过shell工具和claude code交流。这种时候会面临一个问题。当我们需要互传文件的时候很不方便，只能使用rz或者sz命令。所以我想封装一个通用工具解决这个问题。起码需要具备一下功能
> 1. 我认为以网页的形式交互是很好的。
> 1.1 可以自动生成一次性随机链接，链接具备一定的安全性保证
> 1.2 可以通过网页的易用性交互，直接上传和下载文件。
> 2. 对话完成后需要主动回收该链接
> 3. 允许共享目录或者单个文件。支持仅上传、仅下载、上传下载都支持。
> 4. 你帮我判断下最终怎么封装他比较好，是skill还是plugins。主要的使用场景就是在claude code
>
> 调研下市场上，claude code商店或者github是否存在已有的项目

这就是一切的起点。一个远程开发中真实遇到的痛点——rz/sz 不好用。

### brainstorming 中的追问

Claude 提出了三个技术方案（A/B/C），用户中途追问：

> 封装成skills

> 方案ABC的区别是什么

> stdlib是啥意思。flask是啥意思。

当时对 Python 的 stdlib（标准库）和 Flask 框架的区别并不清楚，Claude 解释后选择了 stdlib 方案——零依赖，远程服务器上不需要 pip install。

---

## 二、实现方案的演进

### 第一版方案

brainstorming 产出设计文档后，发出第一条 implement 指令：

> Implement the following plan:
>
> \# File Transfer Skill 设计方案
>
> 方案：stdlib 双文件架构 — Python 后端（纯标准库，零依赖）+ 独立 HTML 前端文件。

方案定义了目录结构、SKILL.md frontmatter、服务器模块划分（约 500 行）、前端设计、安全机制（URL Token + 可选密码 + Session Cookie + 路径遍历防护 + 速率限制 + 自动过期）、端口规则（32000+），以及 9 项验证计划。

### 安全机制的即时调整

第一版方案提交后，用户马上补了一条修改：

> 回到上一步的设计文档中。
> 1. 密码不要太复杂，4位纯数字就行 url中的链接地址可以复杂一点
> 2. 严格控制用户可以下载的文件目录。一旦发现是风险目录需要提醒用户二次确认。不允许把根目录。root目录 home目录 暴露出去.

这条 prompt 定义了两个关键设计决策：PIN 保持简单（4 位数字），安全由 URL Token 承担；危险目录直接拦截，不给用户 `--force` 覆盖的机会。

### 第二版方案（正式开发版）

在前一轮实现的基础上，整理了更完整的方案，在新会话中发出，补充了：
- 多线程并发处理（ThreadingHTTPServer）
- 优雅超时机制（活跃传输计数器 + 硬限制 10 分钟）
- Multipart 解析器的流式处理设计
- 前端并发上传队列（最多 3 个 XHR 并行）

---

## 三、开发过程

### 核心实现

Claude 并行创建了三个文件：

1. **file_transfer_server.py**（634 行）
   - 模块结构：`MultipartParser` → `FileTransferHandler` → `ThreadedHTTPServer` → `ServerManager` → CLI 入口
   - 路由：`GET /{token}`, `GET /{token}/list`, `GET /{token}/download/{path}`, `GET /{token}/zip/{path}`, `POST /{token}/upload`, `POST /{token}/auth`
   - 安全：`secrets.token_urlsafe(16)` 生成 token，`DANGEROUS_DIRS` 拦截系统目录，速率限制 5 次/IP

2. **index.html**（单文件前端）
   - 暗色主题，拖拽上传区域 + 文件浏览器 + PIN 输入弹窗
   - 模板变量 `{{TOKEN}}`, `{{MODE}}`, `{{HAS_PASSWORD}}`, `{{SHARED_PATH}}`
   - 多文件并发上传（最多 3 个 XHR 并行）

3. **SKILL.md**
   - frontmatter: `name: file-transfer`
   - 触发词：传文件、发文件、上传、下载、共享文件

### 测试中的翻车

第一次测试时 timeout 设了 10 秒，服务在 curl 测试跑完之前就自动关闭了。改成 120 秒后重新测试，所有路由（主页、文件列表、下载、ZIP 打包、上传、密码保护、速率限制、危险目录拦截、路径遍历防护）全部通过。

### UI/UX 优化

用户第一次在浏览器中打开 AgentDrop 后，经过两轮反馈优化了以下内容：

- 浏览器标签页 favicon — 使用内联 SVG（深色圆角方块 + 蓝色渐变下载箭头）
- 文件列表字号偏小 — 逐步调大至合适尺寸
- 根目录显示孤零零的 "Root" 面包屑没有意义 — 改为根目录时隐藏面包屑，进入子目录后才显示
- 左上角标题加设计感 — 斜体 + "Drop" 使用蓝色渐变 + `background-clip: text`

有一个需要注意的点：改完 HTML 后页面没变化，原因是 `ServerManager.get_index_html()` 有 `_index_html_cache`，需要重启服务才能生效。

### 文本预览功能

用户提出需求：对 .md .txt 等基础文本文件提供查看功能。

> 你调研一下，如果对于.md .txt基础的文本文件提供一个查看功能。你评估一下这个需求是否复杂，复杂的话我们走superpowers流程开发。

评估复杂度中低，不需要走 brainstorming + writing-plans 流程。

交互设计确认后实现了：
- 后端新增 `GET /{token}/preview/{path}` 路由，30+ 种扩展名可预览，200KB 大小限制
- 前端预览弹窗：文件名 + Download 按钮 + 关闭按钮
- .md 文件有 Markdown 渲染，其他用等宽字体 `<pre>` 展示

---

## 四、发布前整理

### 收尾任务

用户提出四项收尾任务：

> 1. 整理交互对话内容做存档
> 2. 为安装 skill 写说明加入 README，参考 GitHub 上高星独立 skill
> 3. 代码安全审计
> 4. 写中文版 README

Claude 尝试并行执行，用户两次纠正：

> 这些任务是有依赖顺序的，第4个任务必须在第2个之后

> 任务1根本就不要放在AgentDrop这个目录下，丢去ftp目录 内容要尽量真实 别并行了就按照1234的顺序做

### 安全审计结果

代码审计发现了 7 个安全问题并全部修复：

| 级别 | 问题 | 修复 |
|------|------|------|
| Critical | Markdown 预览 XSS（`javascript:` URL 未过滤） | 添加 `isSafeUrl()` 过滤函数 |
| Critical | 上传无大小限制 | 添加 `UPLOAD_MAX_SIZE = 500MB`，超限返回 413 |
| High | 速率限制无衰减，永久锁定 | 10 分钟后自动重置 |
| High | X-Forwarded-For 可伪造 | 移除，直接用 `client_address[0]` |
| High | Session Cookie 缺 HttpOnly | 改由服务端 Set-Cookie 设置，含 HttpOnly 标志 |
| Medium | Upload 路径用 `startswith` 检查 | 改用 `Path.relative_to()` |
| Medium | SHARED_PATH 直接注入 HTML 模板 | `html.escape()` 转义 |

额外修复了 Python 3.9 兼容性（`X | None` → `Optional[X]`）、从预览列表移除 `.env`、ZIP 按钮 dead code 等。

---

## 五、关键设计决策

| 决策 | 理由 |
|------|------|
| 纯 Python 标准库 | 远程服务器上可能无法 pip install，零依赖保证可用性 |
| 封装为 Skill 而非 Plugin | 使用场景是 Claude Code，Skill 是最自然的集成方式 |
| 4 位数字 PIN | 够用且易输入，URL token 已提供主要安全性 |
| 危险目录全部直接拒绝 | 无 `--force` 覆盖，避免误操作暴露系统文件 |
| ThreadingHTTPServer | 原生支持并发请求，无需引入 asyncio |
| 单文件前端 | 无构建步骤，服务端直接读取模板注入变量 |

---

## 六、协作模式复盘

### 什么做得好

1. **brainstorming 阶段的充分沟通**：AI 没有拿到需求就直接动手，而是通过 brainstorming 提出多个方案让用户选择，用户不懂的地方（stdlib vs flask）也得到了解释
2. **方案迭代再开发**：经历了两版方案才正式开发，第二版补充了并发、超时、流式解析等关键设计，避免了开发中途大改
3. **安全设计前置**：密码策略和危险目录拦截在方案阶段就确定了，不是事后补丁
4. **复杂度评估**：文本预览功能评估为中低复杂度，跳过了 superpowers 流程，判断准确
5. **安全审计有实际产出**：7 个真实安全问题被发现并修复，包括 2 个 Critical 级别

### 什么可以改进

1. **AI 对任务依赖关系判断不够**：收尾的四项任务有明确的串行依赖，但 AI 两次尝试并行执行，用户不得不两次纠正。AI 应该主动分析任务间的依赖关系
2. **timeout 参数的默认值设置不当**：第一次测试用了 10 秒超时导致服务提前关闭，这种参数应该在测试时给一个宽松的默认值
3. **HTML 缓存机制没有提前告知**：用户改完 HTML 后刷新浏览器没看到变化，原因是服务端缓存。AI 应该在修改 HTML 后主动提示需要重启服务
4. **UI 优化需要多轮调整**：字号、面包屑等问题需要实际看到页面才能判断，这是 CLI 开发 Web UI 的固有局限，未来可以考虑让 AI 先给出截图预览

### 沟通效率数据

| 阶段 | 用户消息数 | 返工次数 |
|------|-----------|---------|
| brainstorming + 方案设计 | ~6 | 0 |
| 核心开发 + 测试 | 1 | 1（timeout 参数） |
| UI/UX 优化 | ~5 | 0（但需多轮微调） |
| 文本预览功能 | 3 | 0 |
| 发布前整理 | 4 | 2（任务顺序被纠正两次） |

---

## 附录：最终文件结构

```
AgentDrop/
├── README.md
├── README_CN.md
├── SKILL.md
├── LICENSE
├── .gitignore
├── assets/
│   └── index.html        (~750 行，含预览功能)
└── scripts/
    └── file_transfer_server.py  (~680 行，含 preview 路由)
```

## 附录：项目数据

- **开发时间跨度**：2026-02-28（单日完成）
- **代码量**：~1430 行（Python 690 行 + HTML 750 行）
- **会话数**：4 个 Claude Code 会话
- **用户 prompt 数**：约 44 条
- **安全问题修复**：7 个（2 Critical + 3 High + 2 Medium）
- **最终仓库**：https://github.com/WayneVoyager/AgentDrop
