# 渠道开发指南

> **如何创建自定义渠道** - 为 Nanobot 添加新平台支持

---

## 📑 目录

1. [渠道系统概述](#渠道系统概述)
2. [快速入门](#快速入门)
3. [BaseChannel 基类](#basechannel-基类)
4. [消息处理](#消息处理)
5. [权限管理](#权限管理)
6. [错误处理](#错误处理)
7. [渠道管理器](#渠道管理器)
8. [测试渠道](#测试渠道)
9. [高级功能](#高级功能)
10. [最佳实践](#最佳实践)

---

## 渠道系统概述

Nanobot 的渠道系统支持在多个平台上运行 AI 代理，包括 Telegram、Discord、WhatsApp 等。

### 核心概念

```mermaid
graph LR
    A[平台 API] --> B[Channel Adapter]
    B --> C[MessageBus]
    C --> D[AgentLoop]
    D --> C
    C --> B
    B --> A
```

### 渠道类型

| 类型 | 平台 | 特点 |
|------|------|------|
| **即时消息** | Telegram, Discord | 实时对话，支持多媒体 |
| **社交媒体** | Twitter, Facebook | 公开互动，关注机制 |
| **企业通讯** | Slack, Microsoft Teams | 群组管理，集成能力 |
| **邮件系统** | SMTP, IMAP | 异步通信，附件支持 |
| **自定义平台** | 任何 API | 完全可控的集成 |

---

## 快速入门

### 创建第一个渠道

```python
from nanobot.channels.base import BaseChannel, InboundMessage, OutboundMessage
from typing import List, Optional
import asyncio

class EchoChannel(BaseChannel):
    """回声渠道 - 简单的测试渠道"""

    def __init__(self, config: dict):
        super().__init__(
            name="echo",
            description="回声测试渠道",
            config=config
        )

    async def send(self, message: OutboundMessage) -> bool:
        """发送消息"""
        print(f"[ECHO] {message.content}")
        return True

    async def receive(self) -> Optional[InboundMessage]:
        """接收消息"""
        # 模拟接收消息
        await asyncio.sleep(1)
        return InboundMessage(
            channel="echo",
            sender_id="user123",
            chat_id="test",
            content="你好！",
            timestamp=datetime.now()
        )

    async def start(self) -> None:
        """启动渠道"""
        print("Echo渠道已启动")

    async def stop(self) -> None:
        """停止渠道"""
        print("Echo渠道已停止")
```

### 注册和使用

```python
from nanobot.channels.manager import ChannelManager

# 创建渠道管理器
manager = ChannelManager()

# 注册渠道
manager.register(EchoChannel({
    "enabled": True
}))

# 启动渠道
await manager.start_all()

# 发送消息
message = OutboundMessage(
    channel="echo",
    chat_id="test",
    content="Hello, World!"
)
await manager.send(message)
```

---

## BaseChannel 基类

### 基类结构

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, List, Optional
from datetime import datetime
from pydantic import BaseModel

class BaseChannel(ABC):
    """渠道基类"""

    def __init__(
        self,
        name: str,
        description: str,
        config: Dict[str, Any],
        working_dir: Optional[str] = None
    ):
        self.name = name
        self.description = description
        self.config = config
        self.working_dir = working_dir
        self._running = False
        self._should_stop = False
```

### 核心方法

#### send()

```python
@abstractmethod
async def send(self, message: OutboundMessage) -> bool:
    """发送消息

    Args:
        message: 要发送的消息

    Returns:
        bool: 是否发送成功
    """
    pass
```

#### receive()

```python
@abstractmethod
async def receive(self) -> Optional[InboundMessage]:
    """接收消息

    Returns:
        InboundMessage 或 None（如果没有消息）
    """
    pass
```

#### start()

```python
async def start(self) -> None:
    """启动渠道"""
    self._running = True
    self._should_stop = False
    await self._on_start()

async def _on_start(self) -> None:
    """启动回调（子类可重写）"""
    pass
```

#### stop()

```python
async def stop(self) -> None:
    """停止渠道"""
    self._should_stop = True
    await self._on_stop()
    self._running = False

async def _on_stop(self) -> None:
    """停止回调（子类可重写）"""
    pass
```

### 完整示例

```python
import asyncio
import json
from typing import Dict, Any, Optional
from datetime import datetime
from nanobot.channels.base import BaseChannel, InboundMessage, OutboundMessage

class MockChannel(BaseChannel):
    """模拟渠道 - 用于测试"""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(
            name="mock",
            description="模拟测试渠道",
            config=config
        )
        self.message_queue = asyncio.Queue()
        self.sent_messages = []

    async def send(self, message: OutboundMessage) -> bool:
        """发送消息"""
        # 记录发送的消息
        self.sent_messages.append(message)
        print(f"[MOCK SENT] {message.content}")
        return True

    async def receive(self) -> Optional[InboundMessage]:
        """接收消息"""
        try:
            # 从队列获取消息，设置超时
            message = await asyncio.wait_for(
                self.message_queue.get(),
                timeout=1.0
            )
            return message
        except asyncio.TimeoutError:
            return None

    async def send_message(self, content: str, chat_id: str = "test") -> None:
        """向队列添加消息（测试用）"""
        message = InboundMessage(
            channel="mock",
            sender_id="test_user",
            chat_id=chat_id,
            content=content,
            timestamp=datetime.now()
        )
        await self.message_queue.put(message)

    async def _on_start(self) -> None:
        """启动时的初始化"""
        print("Mock渠道已启动")
        self._running = True

    async def _on_stop(self) -> None:
        """停止时的清理"""
        print("Mock渠道已停止")
        self._running = False

    async def run(self) -> None:
        """运行接收循环"""
        while not self._should_stop:
            message = await self.receive()
            if message:
                print(f"[MOCK RECEIVED] {message.content}")
            await asyncio.sleep(0.1)
```

---

## 消息处理

### 消息格式

```python
@dataclass
class InboundMessage:
    channel: str                    # 渠道名称
    sender_id: str                  # 发送者 ID
    chat_id: str                    # 聊天 ID
    content: str                    # 消息内容
    timestamp: datetime              # 时间戳
    media: List[str] = field(default_factory=list)  # 媒体文件
    metadata: Dict[str, Any] = field(default_factory=dict)  # 元数据

@dataclass
class OutboundMessage:
    channel: str                    # 渠道名称
    chat_id: str                    # 聊天 ID
    content: str                    # 消息内容
    reply_to: Optional[str] = None   # 回复消息 ID
    media: List[str] = field(default_factory=list)  # 媒体文件
    metadata: Dict[str, Any] = field(default_factory=dict)  # 元数据
```

### 消息处理示例

```python
class AdvancedChannel(BaseChannel):
    """高级渠道示例"""

    async def send(self, message: OutboundMessage) -> bool:
        """发送消息"""
        try:
            # 构建发送数据
            data = {
                "chat_id": message.chat_id,
                "text": message.content,
                "reply_to_message_id": message.reply_to
            }

            # 添加媒体文件
            if message.media:
                data["media"] = message.media

            # 调用平台 API
            response = await self._call_api("send_message", data)

            # 记录消息 ID
            if response and "message_id" in response:
                message.metadata["platform_message_id"] = response["message_id"]

            return True

        except Exception as e:
            print(f"发送消息失败: {e}")
            return False

    async def receive(self) -> Optional[InboundMessage]:
        """接收消息"""
        try:
            # 获取平台消息
            platform_message = await self._get_platform_updates()

            if not platform_message:
                return None

            # 转换为 InboundMessage
            return self._convert_from_platform(platform_message)

        except Exception as e:
            print(f"接收消息失败: {e}")
            return None

    def _convert_from_platform(self, platform_message: Dict) -> InboundMessage:
        """转换平台消息格式"""
        return InboundMessage(
            channel=self.name,
            sender_id=platform_message["from"]["id"],
            chat_id=platform_message["chat"]["id"],
            content=platform_message.get("text", ""),
            timestamp=datetime.fromtimestamp(platform_message["date"]),
            media=platform_message.get("media", []),
            metadata={
                "platform_data": platform_message
            }
        )

    async def _call_api(self, method: str, data: Dict) -> Optional[Dict]:
        """调用平台 API"""
        # 实现 API 调用逻辑
        pass

    async def _get_platform_updates(self) -> Optional[Dict]:
        """获取平台更新"""
        # 实现获取更新逻辑
        pass
```

---

## 权限管理

### 基础权限

```python
class SecureChannel(BaseChannel):
    """带权限管理的渠道"""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(
            name="secure",
            description="安全渠道",
            config=config
        )
        self.allowed_users = config.get("allowed_users", [])
        self.admin_users = config.get("admin_users", [])

    async def send(self, message: OutboundMessage) -> bool:
        """带权限检查的发送"""
        if not self._check_permission(message.chat_id, "send"):
            return False
        return await super().send(message)

    async def receive(self) -> Optional[InboundMessage]:
        """带权限检查的接收"""
        message = await super().receive()
        if message and not self._check_permission(message.sender_id, "receive"):
            return None
        return message

    def _check_permission(self, user_id: str, action: str) -> bool:
        """检查用户权限"""
        if self.allowed_users and user_id not in self.allowed_users:
            return False

        # 管理员允许所有操作
        if user_id in self.admin_users:
            return True

        # 基本权限检查
        if action == "receive":
            return True  # 默认允许接收

        return False
```

### 基于角色的访问控制

```python
class RoleBasedChannel(BaseChannel):
    """基于角色的访问控制渠道"""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(
            name="rbac",
            description="基于角色的访问控制渠道",
            config=config
        )
        self.roles = config.get("roles", {})
        self.user_roles = config.get("user_roles", {})

    def has_permission(self, user_id: str, permission: str) -> bool:
        """检查用户是否有权限"""
        user_id_str = str(user_id)

        # 获取用户角色
        user_role = self.user_roles.get(user_id_str)
        if not user_role:
            return False

        # 获取角色权限
        role_permissions = self.roles.get(user_role, {})

        # 检查权限
        return permission in role_permissions

    async def send(self, message: OutboundMessage) -> bool:
        """基于权限的发送"""
        if not self.has_permission(message.chat_id, "send_messages"):
            return False
        return await super().send(message)

    async def receive(self) -> Optional[InboundMessage]:
        """基于权限的接收"""
        message = await super().receive()
        if message and not self.has_permission(message.sender_id, "receive_messages"):
            return None
        return message
```

---

## 错误处理

### 错误处理策略

```python
class ResilientChannel(BaseChannel):
    """弹性渠道 - 带重试和错误恢复"""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(
            name="resilient",
            description="弹性渠道",
            config=config
        )
        self.max_retries = config.get("max_retries", 3)
        self.retry_delay = config.get("retry_delay", 1.0)
        self.circuit_breaker_open = False
        self.failure_count = 0

    async def send(self, message: OutboundMessage) -> bool:
        """带重试机制的发送"""
        if self.circuit_breaker_open:
            return False

        for attempt in range(self.max_retries):
            try:
                result = await self._send_with_retry(message, attempt)
                if result:
                    self._reset_circuit_breaker()
                    return True
            except Exception as e:
                print(f"发送失败（尝试 {attempt + 1}/{self.max_retries}）: {e}")
                await asyncio.sleep(self.retry_delay * (attempt + 1))

        self._record_failure()
        return False

    async def _send_with_retry(self, message: OutboundMessage, attempt: int) -> bool:
        """带重试的发送实现"""
        # 实现 API 调用
        pass

    def _record_failure(self) -> None:
        """记录失败"""
        self.failure_count += 1
        if self.failure_count >= 5:
            self.circuit_breaker_open = True

    def _reset_circuit_breaker(self) -> None:
        """重置断路器"""
        self.failure_count = 0
        self.circuit_breaker_open = False
```

### 日志和监控

```python
import logging
from datetime import datetime

class MonitoredChannel(BaseChannel):
    """带监控的渠道"""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(
            name="monitored",
            description="带监控的渠道",
            config=config
        )
        self.logger = logging.getLogger(f"channel.{self.name}")
        self.message_count = 0
        self.error_count = 0
        self.start_time = datetime.now()

    async def send(self, message: OutboundMessage) -> bool:
        """带监控的发送"""
        start_time = datetime.now()
        self.message_count += 1

        try:
            result = await super().send(message)
            duration = (datetime.now() - start_time).total_seconds()
            self.logger.info(
                f"发送消息成功 - 耗时: {duration:.2f}s, "
                f"消息ID: {message.chat_id}"
            )
            return result
        except Exception as e:
            self.error_count += 1
            duration = (datetime.now() - start_time).total_seconds()
            self.logger.error(
                f"发送消息失败 - 耗时: {duration:.2f}s, "
                f"错误: {str(e)}"
            )
            raise

    async def receive(self) -> Optional[InboundMessage]:
        """带监控的接收"""
        try:
            result = await super().receive()
            if result:
                self.message_count += 1
                self.logger.debug(f"接收消息: {result.content}")
            return result
        except Exception as e:
            self.error_count += 1
            self.logger.error(f"接收消息失败: {str(e)}")
            raise

    def get_stats(self) -> Dict[str, Any]:
        """获取统计信息"""
        uptime = datetime.now() - self.start_time
        return {
            "name": self.name,
            "uptime_seconds": uptime.total_seconds(),
            "messages_processed": self.message_count,
            "errors": self.error_count,
            "messages_per_second": self.message_count / uptime.total_seconds()
        }
```

---

## 渠道管理器

### ChannelManager 基类

```python
from nanobot.channels.manager import ChannelManager

class EnhancedChannelManager(ChannelManager):
    """增强的渠道管理器"""

    def __init__(self):
        super().__init__()
        self.channel_stats = {}

    async def start_all(self) -> None:
        """启动所有渠道"""
        await super().start_all()
        # 启动监控任务
        asyncio.create_task(self._monitor_channels())

    async def stop_all(self) -> None:
        """停止所有渠道"""
        # 停止监控
        await self._stop_monitoring()
        # 停止渠道
        await super().stop_all()

    async def _monitor_channels(self) -> None:
        """监控渠道状态"""
        while True:
            stats = {}
            for channel_name, channel in self.channels.items():
                if hasattr(channel, 'get_stats'):
                    stats[channel_name] = channel.get_stats()

            # 存储统计信息
            self.channel_stats = stats
            await asyncio.sleep(60)  # 每分钟更新一次

    async def _stop_monitoring(self) -> None:
        """停止监控"""
        pass

    def get_channel_status(self) -> Dict[str, Dict[str, Any]]:
        """获取所有渠道状态"""
        status = {}
        for name, channel in self.channels.items():
            status[name] = {
                "name": channel.name,
                "description": channel.description,
                "running": getattr(channel, '_running', False),
                "stats": self.channel_stats.get(name, {})
            }
        return status
```

---

## 测试渠道

### 单元测试

```python
import pytest
import asyncio
from unittest.mock import Mock, patch
from mock_channel import MockChannel

class TestMockChannel:
    """MockChannel 测试"""

    @pytest.fixture
    def channel(self):
        return MockChannel({"enabled": True})

    @pytest.mark.asyncio
    async def test_send_message(self, channel):
        """测试发送消息"""
        message = OutboundMessage(
            channel="mock",
            chat_id="test",
            content="Hello"
        )

        result = await channel.send(message)
        assert result is True
        assert len(channel.sent_messages) == 1
        assert channel.sent_messages[0].content == "Hello"

    @pytest.mark.asyncio
    async def test_receive_message(self, channel):
        """测试接收消息"""
        # 添加消息到队列
        await channel.send_message("Test message")

        message = await channel.receive()
        assert message is not None
        assert message.content == "Test message"

    @pytest.mark.asyncio
    async def test_channel_lifecycle(self, channel):
        """测试渠道生命周期"""
        # 启动
        await channel.start()
        assert channel._running is True

        # 运行
        async def send_test_message():
            await asyncio.sleep(0.1)
            await channel.send_message("Lifecycle test")

        # 并发运行接收循环和发送测试消息
        receive_task = asyncio.create_task(channel.run())
        send_task = asyncio.create_task(send_test_message())

        await asyncio.sleep(0.2)

        # 停止
        await channel.stop()
        await receive_task
        await send_task

        assert channel._running is False
```

### 集成测试

```python
import pytest
from nanobot.channels.manager import ChannelManager

class TestChannelIntegration:
    """渠道集成测试"""

    @pytest.fixture
    def manager(self):
        manager = ChannelManager()
        manager.register(MockChannel({"enabled": True}))
        return manager

    @pytest.mark.asyncio
    async def test_channel_manager_integration(self, manager):
        """测试渠道管理器集成"""
        # 启动所有渠道
        await manager.start_all()

        # 发送消息
        message = OutboundMessage(
            channel="mock",
            chat_id="test",
            content="Integration test"
        )

        result = await manager.send(message)
        assert result is True

        # 停止所有渠道
        await manager.stop_all()

    @pytest.mark.asyncio
    async def test_channel_communication(self, manager):
        """测试渠道间通信"""
        # 添加测试消息
        mock_channel = manager.get_channel("mock")
        await mock_channel.send_message("Communication test")

        # 通过消息总线发送消息
        from nanobot.bus.events import InboundMessage

        test_message = InboundMessage(
            channel="mock",
            sender_id="test",
            chat_id="test",
            content="Bus test"
        )

        # 这里应该是测试消息总线的处理
        pass
```

---

## 高级功能

### 事件订阅

```python
from typing import Callable, Dict
from dataclasses import dataclass

@dataclass
class ChannelEvent:
    type: str
    data: Dict[str, Any]
    timestamp: datetime

class EventfulChannel(BaseChannel):
    """支持事件订阅的渠道"""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(name="eventful", description="", config=config)
        self.event_listeners: Dict[str, List[Callable]] = {}

    def on_event(self, event_type: str):
        """事件装饰器"""
        def decorator(func):
            if event_type not in self.event_listeners:
                self.event_listeners[event_type] = []
            self.event_listeners[event_type].append(func)
            return func
        return decorator

    async def emit_event(self, event_type: str, data: Dict[str, Any]) -> None:
        """触发事件"""
        event = ChannelEvent(
            type=event_type,
            data=data,
            timestamp=datetime.now()
        )

        if event_type in self.event_listeners:
            for listener in self.event_listeners[event_type]:
                await listener(event)

    async def send(self, message: OutboundMessage) -> bool:
        """发送时触发事件"""
        result = await super().send(message)

        await self.emit_event("message_sent", {
            "message": message,
            "success": result
        })

        return result
```

### 中间件支持

```python
from typing import List, Optional

class MiddlewareChannel(BaseChannel):
    """支持中间件的渠道"""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(name="middleware", description="", config=config)
        self.send_middleware: List[Callable] = []
        self.receive_middleware: List[Callable] = []

    def use_send_middleware(self, middleware: Callable):
        """添加发送中间件"""
        self.send_middleware.append(middleware)

    def use_receive_middleware(self, middleware: Callable):
        """添加接收中间件"""
        self.receive_middleware.append(middleware)

    async def send(self, message: OutboundMessage) -> bool:
        """处理发送中间件"""
        processed_message = message

        # 应用发送中间件
        for middleware in self.send_middleware:
            processed_message = await middleware(processed_message)

        return await super().send(processed_message)

    async def receive(self) -> Optional[InboundMessage]:
        """处理接收中间件"""
        message = await super().receive()

        if message:
            # 应用接收中间件
            for middleware in self.receive_middleware:
                message = await middleware(message)

        return message

# 中间件示例
async def logging_middleware(message):
    """日志中间件"""
    print(f"[MIDDLEWARE] {message.content}")
    return message

async def text_cleaning_middleware(message):
    """文本清理中间件"""
    if hasattr(message, 'content'):
        message.content = message.content.strip()
    return message
```

---

## 最佳实践

### 1. 渠道设计原则

```python
class WellDesignedChannel(BaseChannel):
    """设计良好的渠道示例"""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(
            name="telegram",
            description="Telegram 渠道 - 支持群组和私聊",
            config=config
        )

        # 验证配置
        self._validate_config()

        # 初始化资源
        self.api_client = None
        self.update_offset = 0

    def _validate_config(self) -> None:
        """验证配置"""
        required = ['api_token']
        for key in required:
            if key not in self.config:
                raise ValueError(f"缺少必需配置: {key}")

    async def start(self) -> None:
        """启动渠道"""
        try:
            # 初始化 API 客户端
            self.api_client = self._create_api_client()

            # 初始化渠道状态
            self._running = True

            # 启动接收循环
            asyncio.create_task(self._receive_loop())

        except Exception as e:
            self.logger.error(f"启动失败: {e}")
            raise

    async def stop(self) -> None:
        """停止渠道"""
        self._running = False

        # 清理资源
        if self.api_client:
            await self.api_client.close()
```

### 2. 性能优化

```python
import aiohttp
from typing import Optional

class OptimizedChannel(BaseChannel):
    """性能优化的渠道"""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(name="optimized", description="", config=config)
        self.session: Optional[aiohttp.ClientSession] = None
        self.rate_limiter = asyncio.Semaphore(config.get("rate_limit", 10))

    async def start(self) -> None:
        """启动渠道"""
        self.session = aiohttp.ClientSession()
        await super().start()

    async def stop(self) -> None:
        """停止渠道"""
        if self.session:
            await self.session.close()
        await super().stop()

    async def send(self, message: OutboundMessage) -> bool:
        """带速率限制的发送"""
        async with self.rate_limiter:
            try:
                async with self.session.post(
                    self.config['api_url'],
                    json=self._build_message_data(message)
                ) as response:
                    return response.status == 200
            except Exception as e:
                self.logger.error(f"发送失败: {e}")
                return False

    async def receive(self) -> Optional[InboundMessage]:
        """批处理的接收"""
        messages = await self._batch_receive()

        # 处理批量消息
        for msg in messages:
            yield self._convert_message(msg)
```

### 3. 配置管理

```python
from pydantic import BaseModel, validator

class ChannelConfig(BaseModel):
    """渠道配置模型"""
    enabled: bool = True
    api_token: str
    rate_limit: int = 10
    timeout: float = 30.0
    allowed_users: List[str] = []
    admin_users: List[str] = []

    @validator('rate_limit')
    def validate_rate_limit(cls, v):
        if v <= 0:
            raise ValueError('速率限制必须大于0')
        return v

class ConfigurableChannel(BaseChannel):
    """配置驱动的渠道"""

    def __init__(self, name: str, description: str, config: Dict[str, Any]):
        # 使用 Pydantic 验证配置
        validated_config = ChannelConfig(**config)

        super().__init__(
            name=name,
            description=description,
            config=validated_config.dict()
        )

        # 使用验证后的配置
        self.rate_limit = validated_config.rate_limit
        self.timeout = validated_config.timeout
```

### 4. 错误恢复

```python
class ResilientChannel(BaseChannel):
    """错误恢复渠道"""

    def __init__(self, config: Dict[str, Any]):
        super().__init__(name="resilient", description="", config=config)
        self.retry_strategy = {
            'max_attempts': 3,
            'base_delay': 1.0,
            'max_delay': 60.0,
            'backoff_factor': 2.0
        }

    async def send(self, message: OutboundMessage) -> bool:
        """带指数退避的重试"""
        last_error = None

        for attempt in range(self.retry_strategy['max_attempts']):
            try:
                return await super().send(message)
            except Exception as e:
                last_error = e
                delay = min(
                    self.retry_strategy['base_delay'] *
                    (self.retry_strategy['backoff_factor'] ** attempt),
                    self.retry_strategy['max_delay']
                )

                self.logger.warning(
                    f"发送失败（尝试 {attempt + 1}），"
                    f"{delay}秒后重试: {e}"
                )

                await asyncio.sleep(delay)

        self.logger.error(f"发送最终失败: {last_error}")
        return False
```

---

## 总结

通过本指南，你已经学会了如何创建和使用 Nanobot 的渠道：

### 核心要点

1. **继承 BaseChannel**：实现平台特定的消息处理
2. **实现核心方法**：send() 和 receive() 是必需的
3. **管理生命周期**：start() 和 stop() 处理初始化和清理
4. **错误处理**：实现健壮的错误处理和重试机制
5. **性能优化**：使用异步模式和适当的并发控制

### 进阶学习

- 查看 [API 文档](./API_CHANNEL.md) 了解渠道系统 API
- 阅读 [架构指南](./NANOBOT_ARCHITECTURE_GUIDE.md) 了解设计原理
- 查看 [代码示例](./CODE_EXAMPLES.md) 获取更多实例

---

**文档版本**: 1.0.0
**最后更新**: 2026-03-03