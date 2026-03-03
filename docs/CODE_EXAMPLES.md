# Nanobot 代码示例集

> **从入门到实践的代码示例** - 本文档提供大量可直接运行的代码示例，帮助快速理解 nanobot 的实现

---

## 📑 目录

1. [基础示例](#基础示例)
2. [工具开发示例](#工具开发示例)
3. [渠道开发示例](#渠道开发示例)
4. [技能开发示例](#技能开发示例)
5. [高级用法示例](#高级用法示例)
6. [测试示例](#测试示例)

---

## 基础示例

### 示例 1: 创建简单的 Agent

```python
# simple_agent.py
"""
最简单的 nanobot agent 示例
"""

import asyncio
from pathlib import Path
from nanobot.agent.loop import AgentLoop
from nanobot.agent.context import ContextBuilder
from nanobot.agent.memory import MemoryStore
from nanobot.agent.skills import SkillsLoader
from nanobot.agent.tools.registry import ToolRegistry
from nanobot.agent.tools.filesystem import ReadFileTool
from nanobot.bus.queue import MessageBus
from nanobot.session.manager import SessionManager
from nanobot.providers.registry import ProviderRegistry
from nanobot.config.loader import load_config

async def main():
    """创建并运行简单的agent"""

    # 1. 加载配置
    config = load_config()

    # 2. 创建工作空间
    workspace = Path("~/.nanobot").expanduser()
    workspace.mkdir(parents=True, exist_ok=True)

    # 3. 初始化消息总线
    bus = MessageBus()

    # 4. 初始化提供商
    provider_registry = ProviderRegistry(config.providers)
    provider, model = provider_registry.get_provider("claude-3-5-sonnet-20241022")

    # 5. 初始化核心组件
    sessions = SessionManager(workspace)
    memory = MemoryStore(workspace)
    skills = SkillsLoader(workspace)
    context = ContextBuilder(
        workspace=workspace,
        memory=memory,
        skills=skills,
    )

    # 6. 注册工具
    tools = ToolRegistry()
    tools.register(ReadFileTool(workspace=workspace))

    # 7. 创建 agent 循环
    agent = AgentLoop(
        bus=bus,
        provider=provider,
        model=model,
        context=context,
        tools=tools,
        sessions=sessions,
        memory=memory,
        skills=skills,
        max_iterations=10,
    )

    # 8. 发送测试消息
    from nanobot.bus.events import InboundMessage

    test_msg = InboundMessage(
        channel="test",
        sender_id="user123",
        chat_id="test",
        content="请帮我读取 README.md 文件的内容",
    )

    await bus.publish_inbound(test_msg)

    # 9. 运行 agent
    await asyncio.sleep(2)  # 等待处理完成

    # 10. 获取响应
    response = await bus.consume_outbound()
    print(f"Agent 响应: {response.content}")

if __name__ == "__main__":
    asyncio.run(main())
```

### 示例 2: 使用配置创建 Agent

```python
# configured_agent.py
"""
使用配置文件创建 agent
"""

from pathlib import Path
import yaml
from nanobot.config.schema import Config

def create_agent_from_config(config_path: str):
    """从配置文件创建 agent"""

    # 1. 加载 YAML 配置
    with open(config_path) as f:
        config_data = yaml.safe_load(f)

    # 2. 解析为 Config 对象
    config = Config(**config_data)

    # 3. 使用配置创建 agent
    from nanobot.cli.commands import _create_agent_from_config

    agent, channels = _create_agent_from_config(config)

    return agent, channels

# 配置文件示例 (config.yaml)
"""
agents:
  default_model: claude-3-5-sonnet-20241022
  default_provider: auto
  memory_window: 50
  max_iterations: 10
  consolidate_threshold: 100

channels:
  telegram:
    enabled: true
    token: ${TELEGRAM_BOT_TOKEN}
    allow_from: ["*"]

providers:
  anthropic:
    api_key: ${ANTHROPIC_API_KEY}

tools:
  exec:
    enabled: true
    working_dir: ~/projects
"""
```

---

## 工具开发示例

### 示例 3: 创建简单的计算工具

```python
# tools/calculator.py
"""
简单的计算器工具
"""

from nanobot.agent.tools.base import Tool

class CalculatorTool(Tool):
    """基本计算器工具"""

    name = "calculator"
    description = "执行基本数学运算（+、-、*、/）"

    parameters = {
        "type": "object",
        "properties": {
            "expression": {
                "type": "string",
                "description": "数学表达式，例如 '2 + 3 * 4'"
            }
        },
        "required": ["expression"]
    }

    async def execute(self, expression: str) -> str:
        """执行计算"""
        try:
            # 安全检查：只允许数字和基本运算符
            allowed_chars = set("0123456789+-*/(). ")
            if not set(expression).issubset(allowed_chars):
                return "错误：表达式包含非法字符"

            # 计算结果
            result = eval(expression)

            return f"计算结果：{expression} = {result}"

        except ZeroDivisionError:
            return "错误：除零错误"
        except Exception as e:
            return f"错误：{str(e)}"

# 注册工具
from nanobot.agent.tools.registry import ToolRegistry

tools = ToolRegistry()
tools.register(CalculatorTool())
```

### 示例 4: 创建数据库查询工具

```python
# tools/database.py
"""
数据库查询工具
"""

import sqlite3
from pathlib import Path
from nanobot.agent.tools.base import Tool

class DatabaseQueryTool(Tool):
    """SQLite 数据库查询工具"""

    name = "db_query"
    description = "执行 SQLite 数据库查询"

    parameters = {
        "type": "object",
        "properties": {
            "db_path": {
                "type": "string",
                "description": "数据库文件路径"
            },
            "query": {
                "type": "string",
                "description": "SQL 查询语句（只读）"
            }
        },
        "required": ["db_path", "query"]
    }

    def __init__(self, allowed_dbs: list[Path]):
        self.allowed_dbs = allowed_dbs

    def validate_params(self, params: dict) -> list[str]:
        """验证参数"""
        errors = []

        # 检查数据库路径
        if "db_path" in params:
            db_path = Path(params["db_path"])
            if db_path not in self.allowed_dbs:
                errors.append(f"数据库路径不在允许列表中")

        # 检查查询安全性
        if "query" in params:
            query = params["query"].strip().upper()
            dangerous = ["DROP", "DELETE", "UPDATE", "INSERT", "ALTER"]
            if any(word in query for word in dangerous):
                errors.append("只允许 SELECT 查询")

        return errors

    async def execute(self, db_path: str, query: str) -> str:
        """执行数据库查询"""
        try:
            conn = sqlite3.connect(db_path)
            cursor = conn.cursor()

            cursor.execute(query)
            rows = cursor.fetchall()

            # 格式化结果
            if not rows:
                return "查询结果：无数据"

            # 获取列名
            columns = [desc[0] for desc in cursor.description]
            result = [" | ".join(columns)]
            result.append("-" * len(result[0]))

            for row in rows:
                result.append(" | ".join(str(v) for v in row))

            conn.close()

            return "查询结果：\n" + "\n".join(result)

        except Exception as e:
            return f"查询失败：{str(e)}"
```

### 示例 5: 创建 HTTP API 工具

```python
# tools/api_tool.py
"""
HTTP API 调用工具
"""

import httpx
from nanobot.agent.tools.base import Tool

class HttpApiTool(Tool):
    """HTTP API 调用工具"""

    name = "http_request"
    description = "发送 HTTP 请求到 API 端点"

    parameters = {
        "type": "object",
        "properties": {
            "url": {
                "type": "string",
                "description": "API 端点 URL"
            },
            "method": {
                "type": "string",
                "enum": ["GET", "POST", "PUT", "DELETE"],
                "description": "HTTP 方法"
            },
            "headers": {
                "type": "object",
                "description": "请求头（可选）"
            },
            "body": {
                "type": "string",
                "description": "请求体（JSON 字符串，可选）"
            }
        },
        "required": ["url", "method"]
    }

    def __init__(self, timeout: int = 30):
        self.timeout = timeout

    async def execute(
        self,
        url: str,
        method: str = "GET",
        headers: dict | None = None,
        body: str | None = None,
    ) -> str:
        """执行 HTTP 请求"""
        try:
            async with httpx.AsyncClient(timeout=self.timeout) as client:
                # 准备请求
                kwargs = {}
                if headers:
                    kwargs["headers"] = headers
                if body and method in ["POST", "PUT"]:
                    kwargs["content"] = body

                # 发送请求
                response = await client.request(method, url, **kwargs)

                # 格式化响应
                result = [
                    f"状态码: {response.status_code}",
                    f"响应头: {dict(response.headers)}",
                    f"响应体:\n{response.text}",
                ]

                return "\n".join(result)

        except httpx.TimeoutError:
            return f"错误：请求超时（{self.timeout}秒）"
        except Exception as e:
            return f"错误：{str(e)}"
```

---

## 渠道开发示例

### 示例 6: 创建简单的命令行渠道

```python
# channels/cli_channel.py
"""
命令行交互渠道
"""

import asyncio
from nanobot.channels.base import BaseChannel
from nanobot.bus.events import InboundMessage, OutboundMessage

class CliChannel(BaseChannel):
    """命令行交互渠道"""

    name = "cli"

    def __init__(self, config, bus: MessageBus):
        super().__init__(config, bus)
        self.sender_id = "cli_user"
        self.chat_id = "cli_session"

    async def start(self) -> None:
        """启动命令行交互"""
        self._running = True
        print("CLI 渠道已启动。输入 'quit' 退出。")

        # 在后台监听用户输入
        asyncio.create_task(self._read_input())

    async def _read_input(self):
        """读取用户输入"""
        while self._running:
            try:
                # 异步读取输入
                loop = asyncio.get_event_loop()
                content = await loop.run_in_executor(
                    None,
                    input,
                    "You: "
                )

                if content.lower() == "quit":
                    self._running = False
                    break

                # 发送到消息总线
                await self._handle_message(
                    sender_id=self.sender_id,
                    chat_id=self.chat_id,
                    content=content,
                )

            except EOFError:
                self._running = False
                break

    async def send(self, msg: OutboundMessage) -> None:
        """发送消息到命令行"""
        print(f"\nBot: {msg.content}\n")

# 注册渠道
from nanobot.channels.manager import ChannelManager
from nanobot.config.schema import Base

class CliConfig(Base):
    enabled: bool = True

# 在 ChannelManager 中添加
channels = {
    "cli": CliChannel,
    # ... 其他渠道
}
```

### 示例 7: 创建邮件渠道

```python
# channels/email_channel.py
"""
邮件接收和发送渠道
"""

import asyncio
from email import policy
from email.parser import BytesParser
import aiosmtplib
from imap_tools import MailBox
from nanobot.channels.base import BaseChannel

class EmailChannel(BaseChannel):
    """邮件渠道"""

    name = "email"

    def __init__(self, config, bus: MessageBus):
        super().__init__(config, bus)
        self.imap_host = config.imap_host
        self.imap_port = config.imap_port
        self.smtp_host = config.smtp_host
        self.smtp_port = config.smtp_port
        self.username = config.username
        self.password = config.password

    async def start(self) -> None:
        """启动邮件监听"""
        self._running = True

        while self._running:
            try:
                # 连接到 IMAP 服务器
                async with MailBox(self.imap_host).login(
                    self.username,
                    self.password
                ) as mailbox:
                    # 监听收件箱
                    await mailbox.idle_start()

                    for msg in mailbox.idle_check(timeout=30):
                        if msg.is_new:
                            await self._process_email(msg)

            except Exception as e:
                print(f"Email error: {e}")
                await asyncio.sleep(60)  # 等待后重连

    async def _process_email(self, msg):
        """处理接收的邮件"""
        # 解析邮件内容
        sender = msg.from_
        subject = msg.subject
        content = msg.text or ""

        # 提取发件人 ID（邮箱地址）
        sender_id = sender.split("<")[-1].strip(">")

        # 发送到消息总线
        await self._handle_message(
            sender_id=sender_id,
            chat_id=sender_id,
            content=f"主题: {subject}\n\n{content}",
        )

    async def send(self, msg: OutboundMessage) -> None:
        """发送邮件"""
        # 构建邮件
        message = f"""
        From: {self.username}
        To: {msg.chat_id}
        Subject: Nanobot 回复

        {msg.content}
        """

        # 发送邮件
        await aiosmtplib.send(
            message,
            hostname=self.smtp_host,
            port=self.smtp_port,
            username=self.username,
            password=self.password,
        )
```

---

## 技能开发示例

### 示例 8: 创建翻译技能

```markdown
# skills/translator/SKILL.md
---
name: translator
description: 多语言翻译助手
requirements:
  bins: []
  env: ["TRANSLATION_API_KEY"]
---

# 翻译助手

你是一个专业的翻译助手，可以帮助用户在不同语言之间进行翻译。

## 可用语言

- 英语 (en)
- 中文 (zh)
- 日语 (ja)
- 韩语 (ko)
- 法语 (fr)
- 德语 (de)
- 西班牙语 (es)

## 使用方法

当用户请求翻译时：

1. 识别源语言和目标语言
2. 如果用户没有指定目标语言，询问用户
3. 调用 `translate_text` 工具进行翻译
4. 返回翻译结果

## 示例对话

**用户**: 请将 "Hello, world!" 翻译成中文
**助手**: [调用 translate_text] "你好，世界！"

**用户**: 这个日语是什么意思："こんにちは"
**助手**: [调用 translate_text] 这是日语的"你好"

## 注意事项

- 保持翻译的准确性和流畅性
- 如果遇到专业术语，提供多个翻译选项
- 保留原文的格式和结构
```

### 示例 9: 创建代码审查技能

```markdown
# skills/code_reviewer/SKILL.md
---
name: code_reviewer
description: 代码审查助手
requirements:
  bins: ["git"]
always: true
---

# 代码审查助手

你是一个经验丰富的代码审查员，可以帮助用户审查代码并提供改进建议。

## 审查要点

1. **代码质量**
   - 遵循语言最佳实践
   - 适当的命名和注释
   - 避免代码重复

2. **安全性**
   - 检查常见漏洞（SQL注入、XSS等）
   - 验证输入和输出
   - 安全的错误处理

3. **性能**
   - 识别性能瓶颈
   - 建议优化方案
   - 考虑时间和空间复杂度

4. **可维护性**
   - 代码结构清晰
   - 易于测试
   - 文档完善

## 使用工具

使用以下工具进行代码审查：

- `read_file`: 读取代码文件
- `exec`: 运行代码检查工具（如 pylint, eslint）
- `search_code`: 搜索代码模式

## 审查流程

1. 用户提交代码或文件路径
2. 使用 `read_file` 读取代码
3. 分析代码的各个方面
4. 提供详细的审查报告，包括：
   - 发现的问题
   - 严重程度（高/中/低）
   - 改进建议
   - 示例代码

## 输出格式

```markdown
## 代码审查报告

### 总体评估
[总体评价]

### 发现的问题

#### 1. [问题标题]
- **位置**: 文件:行号
- **严重程度**: 高/中/低
- **描述**: [问题描述]
- **建议**: [改进建议]

### 改进示例
```python
[改进后的代码]
```

### 总结
[总结和建议]
```

## 示例

**用户**: 请审查这个文件：src/auth.py
**助手**: [执行代码审查]

## 代码审查报告

### 总体评估
代码结构良好，但存在一些安全隐患。

### 发现的问题

#### 1. SQL 注入风险
- **位置**: src/auth.py:45
- **严重程度**: 高
- **描述**: 直接拼接 SQL 查询，存在注入风险
- **建议**: 使用参数化查询

### 改进示例
```python
# 不安全
query = f"SELECT * FROM users WHERE username='{username}'"

# 安全
query = "SELECT * FROM users WHERE username=?"
cursor.execute(query, (username,))
```

### 总结
建议优先修复高严重性问题，其他问题可以逐步改进。
```

---

## 高级用法示例

### 示例 10: 自定义记忆策略

```python
# custom_memory.py
"""
自定义记忆整合策略
"""

from nanobot.agent.memory import MemoryStore
from datetime import datetime, timedelta

class SmartMemoryStore(MemoryStore):
    """智能记忆存储"""

    async def consolidate(
        self,
        session,
        provider,
        model,
        **kwargs
    ) -> bool:
        """增强的记忆整合"""

        # 1. 提取不同类型的信息
        categories = {
            "preferences": [],  # 用户偏好
            "tasks": [],         # 待办任务
            "facts": [],         # 事实信息
            "decisions": [],     # 决策记录
        }

        # 2. 分类消息
        for msg in session.messages:
            content = msg.get("content", "")
            category = await self._classify_message(content, provider, model)
            if category in categories:
                categories[category].append(msg)

        # 3. 为每个类别生成摘要
        memory_updates = {}
        for category, messages in categories.items():
            if messages:
                summary = await self._summarize_category(
                    category,
                    messages,
                    provider,
                    model,
                )
                memory_updates[category] = summary

        # 4. 更新长期记忆
        current = self.read_long_term()
        updated = self._merge_memory(current, memory_updates)
        self.write_long_term(updated)

        return True

    async def _classify_message(
        self,
        content: str,
        provider,
        model: str,
    ) -> str:
        """使用 LLM 分类消息"""
        response = await provider.chat(
            messages=[
                {
                    "role": "system",
                    "content": (
                        "分类消息到以下类别之一："
                        "preferences（偏好）、tasks（任务）、"
                        "facts（事实）、decisions（决策）"
                    )
                },
                {"role": "user", "content": content},
            ],
            model=model,
        )
        return response.content.strip().lower()

    async def _summarize_category(
        self,
        category: str,
        messages: list,
        provider,
        model: str,
    ) -> str:
        """摘要特定类别的消息"""
        messages_text = "\n".join([
            f"- {m['content']}"
            for m in messages[-10:]  # 最近10条
        ])

        response = await provider.chat(
            messages=[
                {
                    "role": "system",
                    "content": f"总结以下{category}类别的关键信息"
                },
                {"role": "user", "content": messages_text},
            ],
            model=model,
        )

        return response.content

    def _merge_memory(
        self,
        current: str,
        updates: dict[str, str]
    ) -> str:
        """合并记忆更新"""
        parts = [current]

        for category, summary in updates.items():
            if summary:
                parts.append(f"\n## {category.title()}\n\n{summary}")

        return "\n".join(parts)
```

### 示例 11: 实现多 agent 协作

```python
# multi_agent.py
"""
多 agent 协作系统
"""

from nanobot.agent.loop import AgentLoop
from nanobot.agent.tools.spawn import SpawnTool
from nanobot.bus.queue import MessageBus

class MultiAgentSystem:
    """多 agent 协作系统"""

    def __init__(self, config):
        self.bus = MessageBus()
        self.agents = {}
        self.coordinator = None

    async def setup(self):
        """设置多 agent 系统"""

        # 1. 创建协调者 agent
        self.coordinator = await self._create_agent(
            name="coordinator",
            role="任务协调",
            instructions=(
                "你是任务协调者。"
                "将复杂任务分解为子任务，"
                "并分配给专门的 agent 执行。"
            ),
        )

        # 2. 创建专门 agent
        self.agents["researcher"] = await self._create_agent(
            name="researcher",
            role="研究助手",
            instructions="你是研究助手，负责收集和分析信息。",
        )

        self.agents["coder"] = await self._create_agent(
            name="coder",
            role="代码助手",
            instructions="你是代码助手，负责编写和审查代码。",
        )

        self.agents["tester"] = await self._create_agent(
            name="tester",
            role="测试助手",
            instructions="你是测试助手，负责编写和运行测试。",
        )

    async def _create_agent(
        self,
        name: str,
        role: str,
        instructions: str,
    ) -> AgentLoop:
        """创建专门的 agent"""

        # 创建自定义上下文构建器
        context = ContextBuilder(
            workspace=Path(f"~/.nanobot/agents/{name}").expanduser(),
            instructions=instructions,
        )

        # 创建 agent
        agent = AgentLoop(
            bus=self.bus,
            context=context,
            # ... 其他参数
        )

        return agent

    async def process_task(self, task: str):
        """处理复杂任务"""

        # 1. 协调者分析任务
        analysis = await self.coordinator.think(
            f"分析这个任务：{task}\n\n"
            "请分解为子任务，并指定执行每个子任务的 agent。"
        )

        # 2. 执行子任务
        results = []
        for subtask in analysis["subtasks"]:
            agent_name = subtask["agent"]
            agent = self.agents[agent_name]

            result = await agent.execute(subtask["instruction"])
            results.append({
                "agent": agent_name,
                "task": subtask["instruction"],
                "result": result,
            })

        # 3. 协调者整合结果
        final = await self.coordinator.think(
            "整合以下结果：\n\n" +
            "\n".join([
                f"{r['agent']}: {r['result']}"
                for r in results
            ])
        )

        return final

# 使用示例
async def main():
    system = MultiAgentSystem(config)
    await system.setup()

    result = await system.process_task(
        "创建一个待办事项应用，"
        "包括后端API、前端界面和测试"
    )

    print(result)

asyncio.run(main())
```

---

## 测试示例

### 示例 12: 测试工具

```python
# tests/test_tools.py
"""
工具测试示例
"""

import pytest
from pathlib import Path
from nanobot.agent.tools.filesystem import ReadFileTool, WriteFileTool

@pytest.fixture
def temp_workspace(tmp_path):
    """临时工作空间"""
    return tmp_path

@pytest.fixture
def read_tool(temp_workspace):
    """读取工具实例"""
    return ReadFileTool(workspace=temp_workspace)

@pytest.fixture
def write_tool(temp_workspace):
    """写入工具实例"""
    return WriteFileTool(workspace=temp_workspace)

class TestFileSystemTools:
    """文件系统工具测试"""

    @pytest.mark.asyncio
    async def test_read_file(self, read_tool, temp_workspace):
        """测试读取文件"""
        # 准备测试文件
        test_file = temp_workspace / "test.txt"
        test_file.write_text("Hello, world!")

        # 执行工具
        result = await read_tool.execute(path="test.txt")

        # 验证结果
        assert result == "Hello, world!"

    @pytest.mark.asyncio
    async def test_read_nonexistent_file(self, read_tool):
        """测试读取不存在的文件"""
        result = await read_tool.execute(path="nonexistent.txt")

        assert "错误" in result
        assert "找不到" in result

    @pytest.mark.asyncio
    async def test_write_and_read(self, write_tool, read_tool, temp_workspace):
        """测试写入后读取"""
        # 写入文件
        content = "Test content\nLine 2\nLine 3"
        write_result = await write_tool.execute(
            path="new_file.txt",
            content=content,
        )

        assert "成功" in write_result

        # 读取文件
        read_result = await read_tool.execute(path="new_file.txt")

        assert read_result == content

    def test_parameter_validation(self, read_tool):
        """测试参数验证"""
        # 缺少必需参数
        errors = read_tool.validate_params({})
        assert "path" in str(errors)

        # 空路径
        errors = read_tool.validate_params({"path": ""})
        assert len(errors) > 0
```

### 示例 13: 测试 Agent

```python
# tests/test_agent.py
"""
Agent 测试示例
"""

import pytest
from pathlib import Path
from nanobot.agent.loop import AgentLoop
from nanobot.bus.queue import MessageBus
from nanobot.bus.events import InboundMessage

@pytest.fixture
def mock_agent(tmp_path):
    """模拟 agent"""
    bus = MessageBus()

    # 使用模拟提供商
    class MockProvider:
        async def chat(self, messages, **kwargs):
            from nanobot.providers.base import LLMResponse
            return LLMResponse(
                content="测试响应",
                tool_calls=[],
            )

    agent = AgentLoop(
        bus=bus,
        provider=MockProvider(),
        model="test-model",
        # ... 其他参数
    )

    return agent, bus

@pytest.mark.asyncio
async def test_message_processing(mock_agent):
    """测试消息处理"""
    agent, bus = mock_agent

    # 发送测试消息
    test_msg = InboundMessage(
        channel="test",
        sender_id="user1",
        chat_id="chat1",
        content="测试消息",
    )

    await bus.publish_inbound(test_msg)

    # 等待处理
    await asyncio.sleep(0.1)

    # 检查响应
    response = await bus.consume_outbound()

    assert response is not None
    assert "测试响应" in response.content

@pytest.mark.asyncio
async def test_session_isolation(mock_agent):
    """测试会话隔离"""
    agent, bus = mock_agent

    # 发送不同会话的消息
    msg1 = InboundMessage(
        channel="test",
        sender_id="user1",
        chat_id="chat1",
        content="我是用户1",
    )

    msg2 = InboundMessage(
        channel="test",
        sender_id="user2",
        chat_id="chat2",
        content="我是用户2",
    )

    await bus.publish_inbound(msg1)
    await bus.publish_inbound(msg2)

    # 检查会话隔离
    session1 = agent.sessions.get("test:chat1")
    session2 = agent.sessions.get("test:chat2")

    assert session1 != session2
    assert "用户1" in str(session1.messages)
    assert "用户2" in str(session2.messages)
```

---

## 总结

### 示例分类

| 类别 | 示例数量 | 覆盖内容 |
|------|----------|----------|
| 基础示例 | 2 | 创建 agent、配置管理 |
| 工具开发 | 3 | 计算器、数据库、HTTP API |
| 渠道开发 | 2 | CLI、邮件 |
| 技能开发 | 2 | 翻译、代码审查 |
| 高级用法 | 2 | 自定义记忆、多 agent |
| 测试示例 | 2 | 工具测试、Agent 测试 |

### 学习建议

1. **从简单开始**: 先运行基础示例，理解整体流程
2. **逐步扩展**: 在基础示例上添加功能
3. **参考实现**: 查看现有工具和渠道的实现
4. **编写测试**: 为自己的代码编写测试
5. **调试技巧**: 使用 logger 查看详细日志

---

**文档版本**: 1.0.0
**最后更新**: 2026-03-03
