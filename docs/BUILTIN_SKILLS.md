# 内置 Skills 汇总

> **内置技能指南** - Nanobot 自带的技能一览

---

## 概述

Nanobot 预置了 8 个内置 Skills，涵盖 GitHub 交互、天气查询、终端管理、记忆系统、定时任务、URL 摘要、技能市场等功能。每个技能都包含详细的 `SKILL.md` 文档。

---

## 技能总览

| 技能 | 描述 | 依赖 |
|------|------|------|
| [github](#github) | 使用 `gh` CLI 交互 GitHub | `gh` CLI |
| [weather](#weather) | 获取天气信息 (wttr.in, Open-Meteo) | `curl` |
| [tmux](#tmux) | 远程控制 tmux 会话 | `tmux` |
| [memory](#memory) | 双重记忆系统 (MEMORY.md + HISTORY.md) | - |
| [summarize](#summarize) | 总结 URL/文件/YouTube | `summarize` CLI |
| [cron](#cron) | 定时任务管理 | - |
| [clawhub](#clawhub) | 公开技能库搜索和安装 | `npx`, Node.js |
| [skill-creator](#skill-creator) | 技能创建指南 | - |

---

## 详细说明

### github

> 使用 `gh` CLI 与 GitHub 交互

**功能描述**：
- 管理 GitHub Issues、PRs、Actions
- 查看 CI/CD 状态
- 执行高级 API 查询
- 使用 JSON/JQ 过滤输出

**触发条件**：
- 用户提到 "GitHub"、"issue"、"PR"、"pull request"
- 需要查询仓库状态
- 需要检查 CI 运行结果

**依赖**：
```bash
# macOS
brew install gh

# Ubuntu/Debian
apt install gh
```

**使用示例**：
```bash
# 查看 PR 的 CI 状态
gh pr checks 55 --repo owner/repo

# 列出最近的 workflow 运行
gh run list --repo owner/repo --limit 10

# 查看失败的步骤日志
gh run view <run-id> --repo owner/repo --log-failed

# 使用 JSON 输出
gh issue list --repo owner/repo --json number,title --jq '.[] | "\(.number): \(.title)"'
```

---

### weather

> 获取天气信息和预报

**功能描述**：
- 查询当前位置或指定城市的天气
- 支持多种格式输出
- 兼容 wttr.in 和 Open-Meteo API
- 无需 API 密钥

**触发条件**：
- 用户询问天气
- 提到 "天气"、"temperature"、"forecast"

**依赖**：`curl`

**使用示例**：
```bash
# 简洁格式
curl -s "wttr.in/London?format=3"
# 输出: London: ⛅️ +8°C

# 完整预报
curl -s "wttr.in/London?T"

# 使用 Open-Meteo (JSON 格式)
curl -s "https://api.open-meteo.com/v1/forecast?latitude=51.5&longitude=-0.12&current_weather=true"
```

**格式代码**：
- `%c` - 天气状况
- `%t` - 温度
- `%h` - 湿度
- `%w` - 风速
- `%l` - 地点

---

### tmux

> 远程控制 tmux 会话

**功能描述**：
- 创建和管理 tmux 会话
- 发送按键和捕获输出
- 支持交互式 TTY 操作
- 适合运行需要终端的 CLI 工具

**触发条件**：
- 需要交互式终端
- 需要长时间运行的进程
- 需要多任务并行处理

**依赖**：
- `tmux` (macOS/Linux)
- 支持 macOS 和 Linux（Windows 需使用 WSL）

**使用示例**：
```bash
# 创建新会话
SOCKET_DIR="${NANOBOT_TMUX_SOCKET_DIR:-${TMPDIR:-/tmp}/nanobot-tmux-sockets}"
tmux -S "$SOCKET_DIR/nanobot.sock" new -d -s nanobot

# 发送命令
tmux -S "$SOCKET" send-keys -t nanobot:0.0 -- 'python3' Enter

# 捕获输出
tmux -S "$SOCKET" capture-pane -p -J -t nanobot:0.0 -S -200

# 杀死会话
tmux -S "$SOCKET" kill-session -t nanobot
```

**注意事项**：
- 使用独立 socket 避免干扰用户会话
- 设置 `PYTHON_BASIC_REPL=1` 用于 Python REPL
- 检查 `❯` 或 `$` 提示符判断命令完成

---

### memory

> 双重记忆系统

**功能描述**：
- **MEMORY.md** - 长期记忆（用户偏好、项目上下文、关系信息）
- **HISTORY.md** - 事件日志（会话历史、重要事件）
- 自动会话摘要和记忆提取

**触发条件**：
- 默认加载（`always: true`）
- 每次会话都会自动加载

**文件结构**：
```
memory/
├── MEMORY.md    # 长期记忆，会话开始时加载
└── HISTORY.md   # 事件日志，搜索使用
```

**使用示例**：
```bash
# 搜索历史
grep -i "keyword" memory/HISTORY.md

# 更新长期记忆
# 直接编辑 memory/MEMORY.md
```

**写入时机**：
- 用户偏好（"I prefer dark mode"）
- 项目上下文（"The API uses OAuth2"）
- 重要关系（"Alice is the project lead"）

---

### summarize

> 总结 URL/文件/YouTube

**功能描述**：
- 快速总结网页内容
- 提取视频/YouTube 字幕
- 支持本地文件摘要
- 支持多种 LLM 提供商

**触发条件**：
- 用户提到 "summarize"、"总结"
- 提供 URL 要求摘要
- 提到 "transcribe"

**依赖**：
```bash
# 安装
brew install steipete/tap/summarize
```

**使用示例**：
```bash
# 总结网页
summarize "https://example.com" --model google/gemini-3-flash-preview

# 总结 PDF
summarize "/path/to/file.pdf"

# YouTube 摘要
summarize "https://youtu.be/dQw4w9WgXcQ" --youtube auto

# 仅提取字幕
summarize "https://youtu.be/dQw4w9WgXcQ" --youtube auto --extract-only
```

**环境变量**：
- `OPENAI_API_KEY`
- `ANTHROPIC_API_KEY`
- `XAI_API_KEY`
- `GEMINI_API_KEY`

---

### cron

> 定时任务管理

**功能描述**：
- 安排提醒和周期性任务
- 支持三种调度模式（at/every/cron）
- 时区支持

**触发条件**：
- 用户提到 "remind"、"提醒"、"schedule"、"定时"

**使用示例**：
```python
# 每20分钟提醒
cron(action="add", message="Time to take a break!", every_seconds=1200)

# 一次性任务
cron(action="add", message="Meeting reminder", at="2024-03-15T14:00:00")

# Cron 表达式
cron(action="add", message="Morning standup", cron_expr="0 9 * * 1-5")

# 带时区
cron(action="add", message="NY standup", cron_expr="0 9 * * *", tz="America/New_York")

# 列出任务
cron(action="list")

# 删除任务
cron(action="remove", job_id="abc123")
```

**时间表达式对照**：

| 需求 | 参数 |
|------|------|
| 每 20 分钟 | `every_seconds: 1200` |
| 每小时 | `every_seconds: 3600` |
| 每天 8 点 | `cron_expr: "0 8 * * *"` |
| 工作日 5 点 | `cron_expr: "0 17 * * 1-5"` |

---

### clawhub

> 公开技能库搜索和安装

**功能描述**：
- 搜索 ClawHub 公开技能
- 安装新技能到 workspace
- 更新已安装的技能

**触发条件**：
- 用户提到 "find skill"、"search skill"、"install skill"
- 询问可用的技能

**依赖**：`npx`（Node.js）

**使用示例**：
```bash
# 搜索技能
npx --yes clawhub@latest search "web scraping" --limit 5

# 安装技能
npx --yes clawhub@latest install <slug> --workdir ~/.nanobot/workspace

# 更新所有技能
npx --yes clawhub@latest update --all --workdir ~/.nanobot/workspace

# 列出已安装
npx --yes clawhub@latest list --workdir ~/.nanobot/workspace
```

**注意事项**：
- `--workdir` 参数是必须的
- 安装后需要新会话才能加载新技能

---

### skill-creator

> 技能创建指南

**功能描述**：
- 提供完整的技能开发指南
- 包含最佳实践和设计模式
- 技能打包和验证工具

**触发条件**：
- 用户提到 "create skill"、"new skill"
- 需要创建自定义技能

**技能结构**：
```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name, description)
│   └── Markdown body
├── scripts/ (optional)
├── references/ (optional)
└── assets/ (optional)
```

**创建流程**：
```bash
# 1. 初始化技能
scripts/init_skill.py <skill-name> --path <output-dir>

# 2. 编辑 SKILL.md 和添加资源

# 3. 打包技能
scripts/package_skill.py <path/to/skill-folder>
```

**核心原则**：
- **简洁优先**：只添加 agent 真正需要的信息
- **渐进披露**：元数据 → SKILL.md → references
- **适当自由度**：根据任务特性选择指导级别

---

## 技能加载机制

### 加载优先级

1. **Workspace Skills** - `~/.nanobot/workspace/skills/`
2. **Built-in Skills** - `nanobot/skills/`

### 渐进披露

1. **元数据** (name + description) - 始终在上下文中
2. **SKILL.md 正文** - 技能触发后加载
3. **Bundled Resources** - 需要时由 agent 决定加载

### Always-Load 技能

`memory` 技能设置了 `always: true`，会在每次会话开始时自动加载。

---

## 自定义技能

参考 [SKILL_DEVELOPMENT.md](./SKILL_DEVELOPMENT.md) 了解如何创建自定义技能。

使用 ClawHub 可以发现和安装社区创建的技能：
```bash
npx --yes clawhub@latest search "<功能描述>"
```