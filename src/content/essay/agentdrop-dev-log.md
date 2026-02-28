---
title: "AgentDrop — 开发流程记录"
date: 2026-02-28
draft: false
archive: true
tags: ["AgentDrop", "file-transfer", "claude-code", "skill", "python"]
---

> GitHub 仓库：[WayneVoyager/AgentDrop](https://github.com/WayneVoyager/AgentDrop)

日期：2026-02-28

参与者： 用户 + AI(Claude Code 执行开发)

产出： AgentDrop Skill。

---

## 一、项目背景

在远程服务器上使用 Claude Code 时，PC 与服务器之间传输文件非常不便（通常只能用 rz/sz）。AgentDrop 通过启动一个临时 Web 服务器，让用户在 PC 浏览器中完成文件的上传和下载。

## 二、用户初始 Prompt

用户提交了一份完整的实现方案文档，核心指令为：

> Implement the following plan:
>
> AgentDrop — File Transfer Skill 实现方案
>
> 方案要点包括：
>
> * 开发目录独立于 Skill 安装目录
>
> * Skill 安装到 `.claude/skills/file-transfer/`
>
> * 三个文件：`SKILL.md`、`file_transfer_server.py`（~500 行，纯标准库）、`index.html`（单文件前端）
>
> * 路由设计、CLI 参数、安全机制（URL Token、4 位 PIN、危险目录拦截、优雅超时）均已在方案中定义

## 三、开发过程

### Phase 1：核心实现

**并行创建三个文件：**

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

### Phase 2：功能测试

**启动测试 — 第一次尝试：**

```bash
python3 scripts/file_transfer_server.py --path /tmp/agentdrop-test --port 32900 --timeout 10 &
```

输出：
```json
{"status": "running", "url": "http://x.x.x.x:32900/{token}", "pid": 1150832, "mode": "both", "path": "/tmp/agentdrop-test", "timeout_seconds": 10}
```

**问题**：超时设为 10 秒太短，在跑测试之前服务就已经自动关闭了。改为 120 秒重启。

**第二次启动（timeout=120）后的 curl 测试结果：**

```
Main page: 200
File list: {"path": "", "items": [{"name": "subdir", "is_dir": true, ...}, {"name": "hello.txt", "is_dir": false, ...}]}
Download: test content
Subdir list: {"path": "subdir", "items": [{"name": "nested.txt", "is_dir": false, ...}]}
Invalid token: 403
Path traversal: 404
Zip download: 200
```

**上传测试：**
```
Upload response: {"ok": true, "files": ["hello.txt"]}
```

**危险目录拦截测试（全部通过）：**
```
--path /     → {"status": "error", "message": "Refusing to share '/' — it contains system directory '/root'"}
--path /root → {"status": "error", "message": "Refusing to share dangerous directory: /root"}
--path /home → {"status": "error", "message": "Refusing to share dangerous directory: /home"}
--path /etc  → {"status": "error", "message": "Refusing to share dangerous directory: /etc"}
--path /var  → {"status": "error", "message": "Refusing to share dangerous directory: /var"}
```

**密码保护测试（password=1234）：**

```
No auth:         {"error": "Authentication required"}
Wrong password:  {"ok": false, "error": "Wrong password"}
Correct password: {"ok": true, "session": "..."}
With session:    {"path": "", "items": [...]}（正常返回文件列表）
```

**速率限制测试（连续 6 次错误密码）：**
```
Attempt 1: {"ok": false, "error": "Wrong password"}
Attempt 2: {"ok": false, "error": "Wrong password"}
Attempt 3: {"ok": false, "error": "Wrong password"}
Attempt 4: {"ok": false, "error": "Wrong password"}
Attempt 5: {"ok": false, "error": "Wrong password"}
Attempt 6: {"ok": false, "error": "Too many failed attempts. Please try later."}
```

第 6 次被锁定，符合预期。

### Phase 3：Skill 安装

文件复制到 `.claude/skills/file-transfer/` 后，Claude Code skill 列表中成功出现 `file-transfer`。

### Phase 4：UI/UX 优化

用户启动服务后实际体验，提出了以下优化需求：

**第一轮：**
1. 浏览器标签页 favicon — 使用内联 SVG（深色圆角方块 + 蓝色渐变下载箭头）
2. 文件列表字号偏小 — 表头 12→13px，单元格 13→14px

**问题**：改完 HTML 后用户反馈页面没变化。原因是 `ServerManager.get_index_html()` 有 `_index_html_cache`，需要重启服务才能生效。重启后正常。

**第二轮：**
1. 字号再调大 — 表头 13→14px，单元格 14→15px
2. 根目录显示孤零零的 "Root" 面包屑没有意义 — 改为根目录时隐藏面包屑，进入子目录后才显示
3. 左上角标题加设计感 — 放大到 28px，加粗 700，斜体，"Drop" 使用蓝色渐变 `linear-gradient(135deg, #4f9cf7, #6db3ff)` + `background-clip: text`
4. 右上角 badge 放大 — 字号 12→14px

**插曲**：用户上传了一个中文名的 .md 文件，以为文件名被单引号包裹。检查磁盘文件后确认是误判，文件名正常。

### Phase 5：文本预览功能

用户提出需求：对 .md .txt 等基础文本文件提供查看功能。

**评估结论**：复杂度中低，不需要走 superpowers 流程。

**用户确认的交互设计**：
- 点击文本文件 → 弹出预览弹窗（非直接下载）
- 弹窗顶部：文件名 + Download 按钮 + 关闭按钮
- .md 做 markdown 渲染，其他用等宽字体 `<pre>` 展示
- 点击 Download 按钮 → 触发下载
- 支持 Esc 键和点击遮罩关闭

**实现**：

后端新增 `GET /{token}/preview/{path}` 路由，30+ 种扩展名可预览，200KB 大小限制。

前端新增 `renderMarkdown()` 和 `inlineMarkdown()` 实现简单 markdown 渲染（标题、列表、表格、代码块、链接、粗斜体、分隔线）。

**Preview API 测试：**
```
.md file:  {"name": "test.md", "content": "# Test Markdown\n\nHello **world**\n\n- item 1\n- item 2\n\n", "size": 53, "ext": ".md"}
.txt file: {"name": "test.txt", "content": "plain text file\n", "size": 16, "ext": ".txt"}
```

### Phase 6：发布前整理

用户要求：
1. 对话存档（本文档）
2. 代码安全审计 + 修复
3. README 重写（参考 GitHub 高星 skill 仓库）
4. 中文版 README

**安全审计发现的关键问题**：
- Critical: Markdown 预览 XSS（`javascript:` URL），上传无大小限制
- High: 速率限制无衰减，X-Forwarded-For 可伪造，Session Cookie 缺 HttpOnly
- Medium: Upload 路径检查用 `startswith` 不安全，SHARED_PATH 模板注入

## 四、关键设计决策

| 决策 | 理由 |
|------|------|
| 纯 Python 标准库 | 远程服务器上可能无法 pip install，零依赖保证可用性 |
| 4 位数字 PIN | 够用且易输入（手机/PC），URL token 已提供主要安全性 |
| 危险目录全部直接拒绝 | 无 `--force` 覆盖，避免误操作暴露系统文件 |
| ThreadingHTTPServer | 原生支持并发请求，无需引入 asyncio |
| 单文件前端 | 无构建步骤，服务端直接读取模板注入变量 |
| 内联 SVG favicon | 无需额外文件，通过 data URI 嵌入 |
| 面包屑根目录隐藏 | 根目录时显示无意义，减少视觉噪音 |

## 五、最终文件结构

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
