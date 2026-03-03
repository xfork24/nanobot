# Agent API 文档

> **Agent 核心类 API 参考** - 详细介绍 AgentLoop 类的所有方法和属性

---

## 📑 目录

1. [AgentLoop 类](#agentloop-类)
2. [构造函数](#构造函数)
3. [核心方法](#核心方法)
4. [消息处理](#消息处理)
5. [会话管理](#会话管理)
6. [配置选项](#配置选项)
7. [使用示例](#使用示例)

---

## AgentLoop 类

`AgentLoop` 是 nanobot 的核心处理引擎，负责协调所有组件来完成 AI 代理的对话流程。

### 类定义

```python
class AgentLoop:
    """Nanobot 的核心处理引擎"""

    def __init__(
        self,
        bus: MessageBus,
        provider: LLMProvider,
        model: str,
        context: ContextBuilder,
        tools: ToolRegistry,
        sessions: SessionManager,
        memory: MemoryStore,
        skills: SkillsLoader,
        max_iterations: int = 10,
        memory_window: int = 50,
        consolidate_threshold: int = 100,
        temperature: float = 0.7,
        max_tokens: int | None = None,
        timeout: float = 30.0,
        **kwargs
    ):
        """初始化 AgentLoop"""
```

---

## 构造函数

### 参数说明

| 参数 | 类型 | 必需 | 默认值 | 描述 |
|------|------|------|--------|------|
| `bus` | MessageBus | ✅ | - | 消息总线，用于组件间通信 |
| `provider` | LLMProvider | ✅ | - | LLM 提供商实例 |
| `model` | str | ✅ | - | 使用的模型名称 |
| `context` | ContextBuilder | ✅ | - | 上下文构建器 |
| `tools` | ToolRegistry | ✅ | - | 工具注册表 |
| `sessions` | SessionManager | ✅ | - | 会话管理器 |
| `memory` | MemoryStore | ✅ | - | 记忆存储 |
| `skills` | SkillsLoader | ✅ | - | 技能加载器 |
| `max_iterations` | int | ❌ | 10 | 最大工具调用次数 |
| `memory_window` | int | ❌ | 50 | 对话记忆窗口大小 |
| `consolidate_threshold` | int | ❌ | 100 | 触发记忆整合的消息阈值 |
| `temperature` | float | ❌ | 0.7 | LLM 温度参数 |
| `max_tokens` | int | ❌ | None | LLM 最大令牌数 |
| `timeout` | float | ❌ | 30.0 | 操作超时时间（秒） |

### 使用示例

```python
from nanobot.agent.loop import AgentLoop
from nanobot.bus.queue import MessageBus

# 创建 agent
agent = AgentLoop(
    bus=bus,
    provider=provider,
    model="claude-3-5-sonnet-20241022",
    context=context,
    tools=tools,
    sessions=sessions,
    memory=memory,
    skills=skills,
    max_iterations=15,
    temperature=0.5,
)
```

---

## 核心方法

### run()

启动主循环，持续处理消息。

```python
async def run(self) -> None:
    """启动主循环：持续处理消息"""

    # 示例
    await agent.run()
```

**特点：**
- 异步运行，不会阻塞
- 自动处理消息队列
- 支持并发消息处理
- 异常时自动恢复

---

### _process_message()

处理单条消息的核心方法。

```python
async def _process_message(
    self,
    msg: InboundMessage,
    on_progress: Callable | None = None,
) -> OutboundMessage | None:
    """处理单条消息的完整流程"""

    # 返回 OutboundMessage 或 None（如果需要丢弃）
```

**处理流程：**
1. 特殊命令处理（/new, /help, /stop）
2. 获取或创建会话
3. 构建对话上下文
4. 运行 Agent 迭代循环
5. 保存到会话历史
6. 检查记忆整合
7. 返回响应

---

### _run_agent_loop()

运行 Agent 迭代循环，处理 LLM 响应和工具调用。

```python
async def _run_agent_loop(
    self,
    messages: list[dict],
    on_progress: Callable | None = None,
) -> tuple[str, list[str], list[dict]]:
    """Agent 迭代循环：处理 LLM 响应和工具调用

    返回:
        - 最终内容（str）
        - 使用的工具列表（list[str]）
        - 所有消息（list[dict]）
    """
```

---

## 消息处理

### 发布消息

```python
# 发布入站消息
await bus.publish_inbound(InboundMessage(
    channel="telegram",
    sender_id="123456789",
    chat_id="123456789",
    content="Hello, world!",
))

# 消费出站消息
response = await bus.consume_outbound()
```

### 消息类型

#### InboundMessage

```python
@dataclass
class InboundMessage:
    channel: str                      # 渠道名称
    sender_id: str                    # 发送者 ID
    chat_id: str                      # 聊天 ID
    content: str                      # 消息内容
    timestamp: datetime               # 时间戳
    media: list[str] = field(default_factory=list)  # 媒体文件路径
    metadata: dict = field(default_factory=dict)    # 元数据
```

#### OutboundMessage

```python
@dataclass
class OutboundMessage:
    channel: str
    chat_id: str
    content: str
    reply_to: str | None = None       # 回复消息 ID
    media: list[str] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)
```

---

## 会话管理

### 获取会话

```python
# 获取或创建会话
session = agent.sessions.get_or_create("telegram:123456789")

# 获取会话历史
history = session.get_history(max_messages=50)

# 保存消息
session.save("user", "Hello!")
session.save_many([
    {"role": "user", "content": "Hi"},
    {"role": "assistant", "content": "Hello!"},
])
```

### 会话隔离

每个会话使用唯一的键：`"{channel}:{chat_id}"`

```python
# 示例会话键
telegram:123456789
discord:987654321
email:user@example.com
```

---

## 配置选项

### Agent 配置

```python
# ~/.nanobot/config.yaml
agents:
  default_model: claude-3-5-sonnet-20241022
  default_provider: auto
  max_iterations: 10              # 最大工具调用次数
  memory_window: 50               # 对话记忆窗口
  consolidate_threshold: 100      # 触发记忆整合的阈值
  temperature: 0.7                # LLM 温度
  max_tokens: 4096                # LLM 最大令牌
  timeout: 30                    # 操作超时（秒）
```

### 高级配置

```python
# 使用自定义配置
agent = AgentLoop(
    bus=bus,
    provider=provider,
    model=model,
    context=context,
    tools=tools,
    sessions=sessions,
    memory=memory,
    skills=skills,
    max_iterations=20,              # 允许更多工具调用
    memory_window=100,              # 更大的记忆窗口
    consolidate_threshold=50,       # 更频繁的记忆整合
    temperature=0.3,               # 更确定的输出
    max_tokens=8192,               # 更长的响应
    timeout=60.0,                  # 更长的超时
)
```

---

## 使用示例

### 基础使用

```python
import asyncio
from pathlib import Path
from nanobot.agent.loop import AgentLoop
from nanobot.bus.queue import MessageBus
from nanobot.bus.events import InboundMessage, OutboundMessage

async def basic_example():
    """基本的 agent 使用示例"""

    # 1. 创建组件
    bus = MessageBus()

    # 2. 发送消息
    test_msg = InboundMessage(
        channel="test",
        sender_id="user123",
        chat_id="test",
        content="你好！",
    )

    await bus.publish_inbound(test_msg)

    # 3. 创建并运行 agent
    agent = AgentLoop(
        bus=bus,
        provider=provider,
        model="claude-3-5-sonnet-20241022",
        context=context,
        tools=tools,
        sessions=sessions,
        memory=memory,
        skills=skills,
    )

    # 4. 运行 agent
    await agent.run()

    # 5. 获取响应
    response = await bus.consume_outbound()
    print(f"响应: {response.content}")

# 运行示例
asyncio.run(basic_example())
```

### 高级使用

```python
class CustomAgent(AgentLoop):
    """自定义 Agent 类"""

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.custom_handlers = {}

    def register_handler(self, command: str, handler):
        """注册自定义命令处理器"""
        self.custom_handlers[command] = handler

    async def _handle_system_message(self, msg: InboundMessage):
        """处理系统消息"""
        cmd = msg.content.strip().lower()

        # 检查自定义处理器
        if cmd in self.custom_handlers:
            return await self.custom_handlers[cmd](msg)

        # 默认处理
        return await super()._handle_system_message(msg)

# 使用自定义 agent
custom_agent = CustomAgent(
    bus=bus,
    provider=provider,
    model=model,
    context=context,
    tools=tools,
    sessions=sessions,
    memory=memory,
    skills=skills,
)

# 注册自定义处理器
async def custom_handler(msg: InboundMessage):
    """自定义命令处理"""
    return OutboundMessage(
        channel=msg.channel,
        chat_id=msg.chat_id,
        content="自定义命令处理！",
    )

custom_agent.register_handler("/custom", custom_handler)
```

### 错误处理

```python
async def robust_agent_example():
    """带错误处理的 agent 示例"""

    try:
        agent = AgentLoop(
            bus=bus,
            provider=provider,
            model=model,
            context=context,
            tools=tools,
            sessions=sessions,
            memory=memory,
            skills=skills,
            timeout=30.0,
        )

        await agent.run()

    except Exception as e:
        print(f"Agent 运行错误: {e}")

    finally:
        # 清理资源
        await bus.close()
```

---

## 最佳实践

### 1. 配置管理

```python
# 使用配置文件
import yaml
from pathlib import Path

def load_agent_config(config_path: str) -> dict:
    """加载配置文件"""
    with open(config_path) as f:
        return yaml.safe_load(f)

# 使用配置
config = load_agent_config("config.yaml")
agent = AgentLoop(**config)
```

### 2. 资源清理

```python
async def cleanup_example():
    """资源清理示例"""

    bus = MessageBus()
    agent = AgentLoop(...)

    try:
        await agent.run()
    except KeyboardInterrupt:
        print("正在停止 agent...")
    finally:
        await bus.close()
        # 其他清理...
```

### 3. 日志记录

```python
import logging

# 配置日志
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("AgentLoop")

# 在 agent 中使用日志
logger.info("Agent 启动")
logger.debug("处理消息: %s", msg.content)
```

---

## 故障排除

### 常见问题

#### 1. Agent 无法启动

**问题**：AgentLoop 构造失败

**解决方案**：
- 检查所有必需参数
- 确保 provider 正确初始化
- 验证配置文件格式

#### 2. 消息处理超时

**问题**：消息处理时间过长

**解决方案**：
- 调整 timeout 参数
- 减少 max_iterations
- 检查工具执行时间

#### 3. 记忆整合失败

**问题**：记忆整合触发但未执行

**解决方案**：
- 检查 consolidate_threshold 设置
- 验证 LLM 提供商状态
- 查看 MEMORY.md 权限

### 调试技巧

```python
# 启用调试模式
import logging
logging.basicConfig(level=logging.DEBUG)

# 创建调试 agent
debug_agent = AgentLoop(
    bus=bus,
    provider=provider,
    model=model,
    context=context,
    tools=tools,
    sessions=sessions,
    memory=memory,
    skills=skills,
    # 可以添加更多调试选项
)

# 手动触发调试
async def debug_message():
    """调试消息处理"""
    msg = InboundMessage(
        channel="debug",
        sender_id="debug_user",
        chat_id="debug_session",
        content="DEBUG: 测试消息",
    )

    response = await agent._process_message(msg)
    print(f"调试响应: {response}")
```

---

## 总结

`AgentLoop` 是 nanobot 的核心组件，提供了完整的 AI 代理处理能力。通过正确配置和使用，可以实现：

- 🔥 高效的消息处理
- 🧠 智能的上下文管理
- 🔧 灵活的工具调用
- 💾 持久的会话存储
- 🎯 精确的记忆整合

### 关键要点

1. **初始化**：确保所有必需组件正确初始化
2. **配置**：根据需求调整配置参数
3. **并发**：支持高并发消息处理
4. **错误处理**：实现适当的错误处理机制
5. **资源管理**：正确清理和释放资源

### 下一步

- 阅读相关的 [Tool API](./API_TOOL.md)
- 了解 [Channel API](./API_CHANNEL.md)
- 查看 [Provider API](./API_PROVIDER.md)

---

**文档版本**: 1.0.0
**最后更新**: 2026-03-03