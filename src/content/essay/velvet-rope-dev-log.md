---
title: "Velvet Rope — 开发流程记录"
date: 2026-02-25
draft: false
archive: true
tags: ["chrome-extension", "AI-development", "cursor", "vibe-coding", "velvet-rope"]
---

> GitHub 仓库：[WayneVoyager/velvet-rope](https://github.com/WayneVoyager/velvet-rope)

**日期**：2026-02-24 ~ 2026-02-25
**参与者**：用户（产品负责人）+ AI（Cursor 开发执行）
**产出**：完整可用的 Chrome 扩展 v1.0.0，含文档和 GitHub 仓库

---

## 一、需求确认期

### 用户初始需求

用户提出 5 个核心功能点：
1. 输入域名和访问次数限制，超限屏蔽
2. 域名冗余识别（子域名自动匹配）
3. 标签页粒度计次（开到关算一次）
4. Popup 快速添加规则
5. 超限后允许临时解除限制

附加要求：插件名 **Velvet Rope**，UI 风格「拟物与扁平混合，高冷抽象，现代化交互」。

### 关键确认项

AI 主动提出 2 个需要确认的问题：

| 问题 | 用户选择 |
|------|---------|
| 同标签页内跳转到另一受限域名如何计次 | 只有新标签页打开或刷新才计次，同标签页内跳转不重复计 |
| 超限后临时解锁的行为 | **质数挑战**：展示两个质数之积，用户需反算出两个因子；质数位数 1-4 位可配置 |

质数挑战是用户自己提出的创意，不是 AI 建议的选项之一。

### 产品边界共识

用户主动确认：这是一个本地自律工具，卸载即可绕过所有限制，双方对此无异议。

> 用户原话："理论上说用户只要把插件卸载掉就可以绕过所有拦截。这点我们有异议吗？"
> AI 回复："没有任何异议。就像日记本锁扣——它拦截的是冲动，不是意志。"

### 经验总结

- **AI 设置了置信度门槛**（85%）才开始开发，通过 2 个精准问题补齐了缺失信息
- **用户的创意输入**（质数挑战）远优于 AI 预设的选项（按时长解锁、单次解锁等），说明开放式提问比封闭选项更能激发好设计
- 提前确认产品边界（卸载即失效）避免了后续在防绕过上浪费精力

---

## 二、开发期

### 第一轮：核心功能实现

按 TODO 清单顺序完成 8 个任务：

1. 项目脚手架（manifest.json、目录结构）
2. Service Worker 数据层（storage 读写、域名匹配、日期重置）
3. 核心拦截逻辑（webNavigation 监听、计次、重定向）
4. 拦截页（全屏视觉 + 质数挑战交互）
5. Popup（当前站点识别 + 快速添加 + 今日状态）
6. 设置页（域名规则 CRUD + 质数难度配置）
7. UI 精修（Velvet Rope 风格配色 + 图标生成）
8. docs 文档归档

技术决策：
- 选择 `webNavigation.onCommitted` 而非 `declarativeNetRequest`（后者无法动态条件判断）
- `tabRegistry` 存在 `chrome.storage.session` 而非 `local`（会话级数据无需持久化）
- 质数验证用试除法而非 Miller-Rabin（4 位数以内完全够用）
- 图标用 Python 脚本生成 PNG（避免引入图片编辑工具依赖）

### 第二轮：多语言 & 主题

用户提出 3 点改进：
1. 支持中英文切换
2. 深色模式对比度太差（灰色字在黑底上看不清）
3. 新增日光白底主题

实施方案：
- 新建 `shared/i18n.js`（完整中英文字符串表 + DOM 注入）和 `shared/prefs.js`（偏好加载器）
- 三套 CSS 均修复对比度：`--text-2` 从 `#5a5a7a`（3:1）提升到 `#8585a8`（6:1）
- 添加 `[data-theme="light"]` CSS 变量块：冷白底 `#f2f2f8`、加深金色 `#a87c28`
- 设置页新增语言/主题下拉，切换后即时生效

### 经验总结

- **TODO 驱动开发**效果好：8 个 task 逐个推进，每完成一个标记为 completed，保持节奏清晰
- **并行写文件**提速：独立的 HTML/CSS/JS 可以同时创建
- **对比度问题应在第一轮就考虑**：初始配色过度追求"高冷"导致 text-2/text-3 几乎不可见，后来不得不修复。教训是设计时先确保 WCAG AA 对比度（4.5:1），再追求氛围
- **shell heredoc 写大文件**比 Write 工具更稳定，不容易遇到编码/转义问题

---

## 三、反馈期

### 用户截图反馈（共 2 轮）

**第一轮截图（Popup 白天模式）**：

| 问题 | 修复 |
|------|------|
| 设置图标像太阳，应该是齿轮 | 替换为 Feather Icons 齿轮 SVG |
| 「全部规则 — 今日」字号 8.5px 太小 | 改为 12px + 金色竖条 accent bar |
| 绳索门柱分割线在弹窗里太拥挤 | 替换为 1px 渐变分隔线 |

**第二轮截图（Popup 深色 + Blocked 白天）**：

| 问题 | 修复 |
|------|------|
| blocked.html 白天模式下 header 到绳索过渡不自然 | 移除 border-bottom 和渐变背景，改用 box-shadow 柔和过渡 |
| Popup 扩展页面状态显示两行重复信息 | 移除冗余的 stateExtensionPage 块，只保留一行 |
| "当前页面"标签字号 9px 太小 | 提升到 11px，颜色改为 --text-1 |

### 经验总结

- **截图标注 > 文字描述**：用户用红框标注问题区域，AI 能精确定位需要修改的 CSS 类和 DOM 元素，比纯文字沟通效率高很多
- **Plan 模式先确认再执行**：两轮反馈都是先在 Plan 模式下讨论方案，用户确认后再切 Agent 模式执行，避免返工
- **小问题可以批量修**：一轮截图反馈包含 2-3 个小问题，在一次 Plan 里统一处理比逐个处理更高效

---

## 四、发布与文档期

### README 撰写

- AI 联网搜索了优秀 README 范例（Best-README-Template、LobeChat 中文版）
- 生成 banner 图和工作原理图（通过 GenerateImage 工具）
- 初版为中文 README
- 用户后续要求改为英文默认 + 中文切换，最终采用 `README.md`（EN）+ `README.zh-CN.md`（ZH）双文件方案

### Git 与 GitHub

首次推送到 GitHub 时遇到几个实际问题：

| 问题 | 解法 |
|------|------|
| 想推送到个人 GitHub 而非默认配置 | `git config user.name/email`（不加 --global，只影响当前仓库） + Personal Access Token |
| `git push` 报 `src refspec master does not match any` | 原因：跳过了 `git add` 和 `git commit`，仓库里没有提交 |
| `git reset HEAD~1` 报 `unknown revision` | 首次提交无法用 `HEAD~1`，需要 `git update-ref -d HEAD` |
| 不想提交 `.cursor` 目录 | 创建 `.gitignore`，加入 `.cursor/`，重新 add + commit |

### 补充知识问答

| 用户提问 | 要点 |
|---------|------|
| crx 和 pem 文件怎么用 | crx 是安装包，pem 是签名私钥；GitHub 开源分发不需要打包，Load unpacked 即可 |
| commit 信息是否符合 Conventional Commits | `init` 和 `ship` 不合规，修正为 `feat:` 前缀 |
| 想用 master 不用 main | `git checkout -b master` 或 `git branch -M master` |

### 经验总结

- **README 写作应参考业界标杆**：搜索 + 参考 > 凭感觉写，shields.io 徽章、项目结构树、双语切换链接都是标准做法
- **Git 操作对非开发者有门槛**：add → commit → push 的三步流程、首次提交的特殊性、.gitignore 的时机，这些对非日常使用 git 的用户需要逐步引导
- **pem/crx 等打包知识提前普及**可以帮用户少走弯路

---

## 五、协作模式复盘

### 什么做得好

1. **置信度门槛**：AI 没有拿到需求就直接动手，而是先提问到 85% 确定度再开始
2. **阶段性交付**：核心功能完成后才征求用户意见，中间过程自主决策，没有过度频繁地打断
3. **Plan → Agent 切换**：所有非平凡改动都先在 Plan 模式讨论方案、对齐预期，确认后再执行
4. **截图驱动反馈**：用户用截图 + 红框标注问题，AI 能快速精确定位

### 什么可以改进

1. **初始配色应先验证对比度**：第一轮开发的深色配色对比度不达标，导致后续必须修复
2. **`.gitignore` 应在 `git init` 后立即创建**：拖到提交后才想起来，导致需要撤回首次提交
3. **Write 工具对含特殊字符的大文件不稳定**：`popup.js` 首次写入失败，改用 shell heredoc 才成功，后续大文件可以直接用 heredoc

### 沟通效率数据

| 阶段 | 用户消息数 | AI 执行轮次 | 返工次数 |
|------|-----------|------------|---------|
| 需求确认 | 3 | 0（纯对话） | 0 |
| 核心开发 | 1（"开始工作"） | 1 | 0 |
| 多语言 & 主题 | 1 | 1 | 0 |
| UI 反馈修复 | 2（两轮截图） | 2 | 0 |
| 文档 & 发布 | 5 | 3 | 1（首次提交含 .cursor） |

---

## 附录：最终文件清单

```
velvet-rope/
├── manifest.json
├── README.md                        # 英文（默认）
├── README.zh-CN.md                  # 中文
├── LICENSE                          # MIT
├── .gitignore
├── background/
│   └── service-worker.js
├── popup/
│   ├── popup.html / js / css
├── pages/
│   ├── blocked.html / js / css
│   └── options.html / js / css
├── shared/
│   ├── i18n.js
│   └── prefs.js
├── icons/
│   └── icon16.png / icon48.png / icon128.png
└── docs/
    ├── assets/
    │   ├── banner.png
    │   └── how-it-works.png
    ├── research/
    │   └── 2026-02-24-velvet-rope-requirements.md
    ├── design/
    │   └── 2026-02-24-velvet-rope-architecture.md
    └── dev/
        ├── 2026-02-24-velvet-rope-implementation.md
        └── 2026-02-25-session-log.md
```
