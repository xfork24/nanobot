# Nanobot AI Agent 教学方案

## 背景与目标

本教学方案面向 AI 工程新手，帮助学生从零开始理解 AI Agent 的核心概念与工程实现。通过 nanobot 这个轻量级（~4000行代码）但功能完整的项目，学生可以：

- 理解 AI Agent 的基本工作原理
- 掌握消息处理、工具调用、记忆系统等核心模块
- 学习如何扩展一个生产级 Agent 系统

---

## 教学阶段划分

### 第一阶段：概念入门（1-2小时）

**目标**：建立基本认知，了解 AI Agent 是什么

#### 1.1 什么是 AI Agent（30分钟）

**教学要点**：

- AI Agent vs 传统程序：被动响应 vs 主动推理
- Agent 的核心能力：感知 → 思考 → 行动 → 反馈
- Agent 的组成要素：LLM + 工具 + 记忆 + 规划

**生活化类比**：把 Agent 想象成一个「有手有脑的助手」

- LLM = 大脑（思考能力）
- 工具 = 手（执行能力）
- 记忆 = 笔记本（记住上下文）
- 消息系统 = 耳朵和嘴巴（接收和发送信息）

**推荐阅读**：

- `docs/PROJECT_OVERVIEW.md` - 项目概览
- `docs/QUICK_START.md` - 快速入门

---

### 第二阶段：快速上手（1小时）

**目标**：安装运行 nanobot，感受 Agent 的实际行为

#### 2.1 安装与配置（30分钟）

**实践操作**：

```bash
# 1. 克隆项目
git clone https://github.com/nanobot-ai/nanobot.git
cd nanobot

# 2. 安装依赖
pip install -e .

# 3. 初始化配置
nanobot onboard

# 4. 启动服务
nanobot gateway
```

**体验内容**：

- 与 Agent 对话（通过 Telegram 或命令行）
- 让 Agent 执行简单任务（读取文件、执行代码）
- 观察 Agent 如何使用工具

#### 2.2 核心文件结构认知（30分钟）

**目录结构概览**：

```
nanobot/
├── agent/          # 核心 Agent 逻辑（大脑）
│   ├── loop.py     # 主循环
│   ├── context.py  # 上下文构建
│   ├── memory.py   # 记忆系统
│   └── tools/      # 工具箱
├── channels/       # 消息渠道（耳朵/嘴巴）
├── bus/            # 消息总线（神经系统）
├── providers/      # LLM 接口（认知能力）
└── skills/         # 技能扩展
```

**推荐阅读**：

- `docs/QUICK_START.md` 全文

---

### 第三阶段：架构深入（3-4小时）

**目标**：理解核心模块的设计与实现

#### 3.1 消息总线（MessageBus）- 1小时

**核心概念**：

- 发布-订阅模式
- 入站消息（InboundMessage）与出站消息（OutboundMessage）
- 消息队列与异步处理

**关键代码**：

- `nanobot/bus/events.py` - 消息事件定义
- `nanobot/bus/queue.py` - 消息队列实现

**教学类比**：MessageBus 就像「公司的前台接待」

- 接收所有外部来件（用户消息）
- 分发给对应部门（AgentLoop）
- 收集回复并寄出（发送响应）

---

#### 3.2 Agent 核心循环（AgentLoop）- 1.5小时

**核心概念**：

- 主循环的工作流程
- 与 LLM 的交互方式
- 工具调用的触发与执行

**关键代码**：

- `nanobot/agent/loop.py` - 主循环实现（核心中的核心）

**处理流程**：

```
接收消息 → 构建上下文 → 调用 LLM → 执行工具 → 生成回复 → 保存记忆
```

**教学重点**：

1. 消息如何进入系统
2. 上下文如何构建（系统提示 + 记忆 + 技能）
3. LLM 如何决定调用工具
4. 工具结果如何反馈给 LLM

---

#### 3.3 上下文构建（ContextBuilder）- 30分钟

**核心概念**：

- 系统提示（System Prompt）
- 对话历史
- 技能注入
- 长期记忆

**关键代码**：

- `nanobot/agent/context.py`

**教学要点**：

- 为什么需要构建上下文？（让 LLM 知道「你是谁」+「之前发生了什么」）
- 上下文有限制怎么办？（记忆窗口、总结压缩）

---

#### 3.4 工具系统（ToolRegistry）- 1小时

**核心概念**：

- 工具的定义与注册
- 参数验证（Pydantic）
- 工具执行与结果返回

**关键代码**：

- `nanobot/agent/tools/base.py` - 工具基类
- `nanobot/agent/tools/registry.py` - 工具注册表
- `nanobot/agent/tools/filesystem.py` - 文件工具示例

**内置工具示例**：

- `read_file` / `write_file` - 文件操作
- `exec` - 执行命令
- `web_search` / `web_fetch` - 网络搜索
- `mcp` - 调用 MCP 服务器

---

#### 3.5 记忆系统（MemoryStore）- 30分钟

**核心概念**：

- 两层记忆：Session（短期）+ MemoryStore（长期）
- 记忆整合（consolidation）
- MEMORY.md 与 HISTORY.md

**关键代码**：

- `nanobot/agent/memory.py`

**记忆整合流程**：

1. 积累一定数量消息后触发
2. LLM 提取关键信息
3. 更新长期记忆文件
4. 清理短期历史

---

### 第四阶段：扩展开发（2-3小时）

**目标**：学会扩展 Agent 功能

#### 4.1 添加新工具（1小时）

**教学路径**：

- 阅读 `docs/TOOL_DEVELOPMENT.md`
- 参考现有工具实现（`nanobot/agent/tools/shell.py`）
- 实践：创建一个自定义工具

**工具开发模板**：

```python
from nanobot.agent.tools.base import Tool, ToolResult

class MyTool(Tool):
    name = "my_tool"
    description = "工具功能描述"

    async def execute(self, param: str) -> str:
        # 实现逻辑
        return result
```

---

#### 4.2 添加新渠道（1小时）

**教学路径**：

- 阅读 `docs/CHANNEL_DEVELOPMENT.md`
- 参考现有渠道实现（`nanobot/channels/telegram.py`）

**渠道开发要点**：

- 实现 `BaseChannel` 接口
- 处理消息格式转换
- 实现权限检查

---

#### 4.3 技能系统（30分钟）

**教学路径**：

- 阅读 `docs/SKILL_DEVELOPMENT.md`
- 查看内置技能示例（`nanobot/skills/github/`）

**技能定义**：Markdown 格式的提示工程

- 元数据（name, description）
- 系统提示（角色定义、能力描述）
- 使用示例

---

### 第五阶段：综合实践（2小时）

**目标**：完成一个完整的扩展功能

#### 实践项目建议

**项目1：天气查询技能**

- 创建天气技能（调用外部 API）
- 集成到 Agent 中

**项目2：自定义工具**

- 实现一个文件搜索工具
- 添加参数验证

**项目3：新渠道支持**

- 添加一个简单渠道（如 console）

---

## 教学方法建议

### 1. 渐进式学习

- 先跑通，再理解
- 先看整体，再看细节
- 先模仿，再创新

### 2. 代码驱动

- 每讲一个模块，带着学生读核心代码
- 不要只是讲概念，要看实际实现

### 3. 对比学习

- 对比其他框架（LangChain、AutoGen）
- 理解 nanobot 的设计取舍

### 4. 实践为主

- 每个阶段都有动手练习
- 最终要有完整作品

---

## 关键文档索引

| 教学阶段 | 推荐文档 | 重点章节 |
|---------|---------|---------|
| 入门 | `QUICK_START.md` | 快速安装、基础使用 |
| 架构 | `NANOBOT_ARCHITECTURE_GUIDE.md` | 整体架构、核心模块 |
| 流程 | `WORKFLOW_GUIDE.md` | 消息处理流程 |
| 实践 | `CODE_EXAMPLES.md` | 代码示例 |
| 扩展 | `TOOL_DEVELOPMENT.md` | 工具开发 |
| 扩展 | `CHANNEL_DEVELOPMENT.md` | 渠道开发 |

---

## 验证方式

### 理解验证

- [ ] 能画出整体架构图
- [ ] 能描述消息处理流程
- [ ] 能解释工具调用机制

### 能力验证

- [ ] 能安装运行 nanobot
- [ ] 能创建自定义工具
- [ ] 能添加新渠道
- [ ] 能创建自定义技能

---

## 预计学习时间

| 阶段 | 内容 | 预计时间 |
|-----|------|---------|
| 1 | 概念入门 | 1-2小时 |
| 2 | 快速上手 | 1小时 |
| 3 | 架构深入 | 3-4小时 |
| 4 | 扩展开发 | 2-3小时 |
| 5 | 综合实践 | 2小时 |
| **总计** | | **9-12小时** |

---

## 后续深入方向

完成基础学习后，学生可以探索：

- 多 Agent 协作
- 更复杂的记忆策略
- Agent 安全性（对抗 prompt injection）
- 性能优化
- 与其他系统集成
