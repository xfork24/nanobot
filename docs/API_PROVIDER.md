# Provider API 文档

> **LLM 提供商 API 参考** - 详细介绍 LLM 提供商系统

---

## 📑 目录

1. [LLMProvider 基类](#llmprovider-基类)
2. [ProviderRegistry 管理器](#providerregistry-管理器)
3. [内置提供商](#内置提供商)
4. [自定义提供商](#自定义提供商)
5. [使用示例](#使用示例)
6. [配置选项](#配置选项)
7. [最佳实践](#最佳实践)

---

## LLMProvider 基类

`LLMProvider` 是所有 LLM 提供商的基类，定义了统一接口。

### 类定义

```python
from abc import ABC, abstractmethod
from typing import Any, Dict, List, Optional
from nanobot.providers.base import LLMResponse

class LLMProvider(ABC):
    """LLM 提供商基类"""

    def __init__(self, provider_name: str, config: Dict[str, Any]):
        """初始化提供商

        Args:
            provider_name: 提供商名称
            config: 配置字典
        """
        self.provider_name = provider_name
        self.config = config
        self.timeout = config.get('timeout', 30.0)

    @abstractmethod
    async def chat(
        self,
        messages: List[Dict[str, str]],
        **kwargs
    ) -> LLMResponse:
        """发送聊天请求

        Args:
            messages: 消息列表
            **kwargs: 其他参数

        Returns:
            LLMResponse 响应对象
        """
        pass

    @abstractmethod
    async def validate_config(self) -> bool:
        """验证配置

        Returns:
            配置是否有效
        """
        pass
```

### 核心方法

#### chat()

```python
async def chat(
    self,
    messages: List[Dict[str, str]],
    temperature: float = 0.7,
    max_tokens: Optional[int] = None,
    **kwargs
) -> LLMResponse:
    """聊天请求

    Args:
        messages: 消息列表，格式为 [{"role": "user", "content": "..."}]
        temperature: 温度参数
        max_tokens: 最大令牌数
        **kwargs: 其他提供商特定参数

    Returns:
        LLMResponse: 包含响应内容、令牌使用等
    """
```

#### validate_config()

```python
async def validate_config(self) -> bool:
    """验证 API 密钥和配置

    Returns:
        bool: 配置是否有效
    """
```

---

## ProviderRegistry 管理器

`ProviderRegistry` 管理所有注册的 LLM 提供商。

### 类定义

```python
class ProviderRegistry:
    """LLM 提供商注册表"""

    def __init__(self):
        self.providers: Dict[str, LLMProvider] = {}
        self.default_provider = None

    def register(self, provider: LLMProvider) -> None:
        """注册提供商

        Args:
            provider: LLMProvider 实例
        """
        self.providers[provider.provider_name] = provider
        if provider.config.get('default', False):
            self.default_provider = provider.provider_name

    def get_provider(self, name: str) -> Optional[LLMProvider]:
        """获取提供商实例

        Args:
            name: 提供商名称

        Returns:
            LLMProvider 实例或 None
        """
        return self.providers.get(name)

    def list_providers(self) -> List[str]:
        """列出所有提供商

        Returns:
            提供商名称列表
        """
        return list(self.providers.keys())

    async def validate_all(self) -> Dict[str, bool]:
        """验证所有提供商配置

        Returns:
            {provider_name: is_valid}
        """
        results = {}
        for name, provider in self.providers.items():
            results[name] = await provider.validate_config()
        return results
```

### 使用示例

```python
# 创建注册表
registry = ProviderRegistry()

# 注册 Anthropic 提供商
anthropic_provider = AnthropicProvider(
    provider_name="anthropic",
    config={
        "api_key": "sk-ant-xxx",
        "default": True
    }
)
registry.register(anthropic_provider)

# 获取提供商
provider = registry.get_provider("anthropic")

# 列出所有提供商
providers = registry.list_providers()
```

---

## 内置提供商

### Anthropic Provider

```python
from nanobot.providers import AnthropicProvider

# 创建 Anthropic 提供商
provider = AnthropicProvider(
    provider_name="anthropic",
    config={
        "api_key": "sk-ant-xxx",
        "model": "claude-3-5-sonnet-20241022",
        "max_tokens": 4096,
        "temperature": 0.7,
        "default": True
    }
)

# 验证配置
is_valid = await provider.validate_config()

# 发送请求
response = await provider.chat([
    {"role": "user", "content": "你好！"}
])
```

### OpenAI Provider

```python
from nanobot.providers import OpenAIProvider

# 创建 OpenAI 提供商
provider = OpenAIProvider(
    provider_name="openai",
    config={
        "api_key": "sk-xxx",
        "model": "gpt-4-turbo-preview",
        "max_tokens": 4096,
        "temperature": 0.7
    }
)
```

### OpenRouter Provider

```python
from nanobot.providers import OpenRouterProvider

# 创建 OpenRouter 提供商
provider = OpenRouterProvider(
    provider_name="openrouter",
    config={
        "api_key": "sk-or-xxx",
        "model": "anthropic/claude-3.5-sonnet",
        "max_tokens": 4096,
        "temperature": 0.7
    }
)
```

---

## 自定义提供商

### 创建自定义提供商

```python
from nanobot.providers.base import LLMProvider, LLMResponse
from typing import List, Dict, Any

class CustomProvider(LLMProvider):
    """自定义 LLM 提供商示例"""

    def __init__(self, provider_name: str, config: Dict[str, Any]):
        super().__init__(provider_name, config)
        self.api_base = config.get('api_base', 'https://api.example.com/v1')
        self.api_key = config.get('api_key')

    async def chat(
        self,
        messages: List[Dict[str, str]],
        **kwargs
    ) -> LLMResponse:
        """实现聊天请求"""

        # 构建请求
        request_data = {
            "messages": messages,
            "temperature": kwargs.get('temperature', 0.7),
            "max_tokens": kwargs.get('max_tokens', 1000),
        }

        # 调用 API
        response = await self._make_api_request(request_data)

        # 解析响应
        return LLMResponse(
            content=response['content'],
            tool_calls=response.get('tool_calls', []),
            usage={
                'prompt_tokens': response.get('prompt_tokens', 0),
                'completion_tokens': response.get('completion_tokens', 0),
                'total_tokens': response.get('total_tokens', 0)
            }
        )

    async def validate_config(self) -> bool:
        """验证配置"""
        if not self.api_key:
            return False

        # 测试 API 连接
        try:
            response = await self._make_api_request({
                "messages": [{"role": "user", "content": "test"}]
            })
            return True
        except Exception:
            return False

    async def _make_api_request(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """发送 API 请求"""
        import httpx

        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.api_base}/chat/completions",
                json=data,
                headers={
                    "Authorization": f"Bearer {self.api_key}",
                    "Content-Type": "application/json"
                },
                timeout=self.timeout
            )

            if response.status_code != 200:
                raise Exception(f"API error: {response.status_code}")

            return response.json()
```

### 注册自定义提供商

```python
# 注册自定义提供商
custom_provider = CustomProvider(
    provider_name="custom",
    config={
        "api_key": "your-api-key",
        "api_base": "https://api.example.com/v1"
    }
)

registry.register(custom_provider)
```

---

## 使用示例

### 基础使用

```python
import asyncio
from nanobot.providers import ProviderRegistry, AnthropicProvider

async def basic_example():
    """基础使用示例"""

    # 创建注册表
    registry = ProviderRegistry()

    # 注册 Anthropic 提供商
    anthropic_provider = AnthropicProvider(
        provider_name="anthropic",
        config={
            "api_key": "sk-ant-xxx",
            "default": True
        }
    )
    registry.register(anthropic_provider)

    # 获取默认提供商
    provider = registry.get_default_provider()

    # 发送消息
    response = await provider.chat([
        {"role": "user", "content": "你好！"}
    ])

    print(f"响应: {response.content}")

# 运行示例
asyncio.run(basic_example())
```

### 多提供商切换

```python
async def multi_provider_example():
    """多提供商使用示例"""

    registry = ProviderRegistry()

    # 注册多个提供商
    registry.register(AnthropicProvider(
        provider_name="anthropic",
        config={"api_key": "sk-ant-xxx", "default": True}
    ))

    registry.register(OpenAIProvider(
        provider_name="openai",
        config={"api_key": "sk-xxx"}
    ))

    # 根据条件选择提供商
    def get_provider_for_task(task_type: str):
        if task_type == "code":
            return registry.get_provider("openai")
        else:
            return registry.get_provider("anthropic")

    # 使用
    provider = get_provider_for_task("general")
    response = await provider.chat([{"role": "user", "content": "解释什么是机器学习"}])
```

### 错误处理

```python
async def error_handling_example():
    """错误处理示例"""

    provider = registry.get_provider("anthropic")

    try:
        response = await provider.chat([
            {"role": "user", "content": "你好！"}
        ])
    except Exception as e:
        print(f"请求失败: {e}")
        # 尝试备用提供商
        try:
            fallback_provider = registry.get_provider("openai")
            response = await fallback_provider.chat([
                {"role": "user", "content": "你好！"}
            ])
        except Exception as e2:
            print(f"备用请求也失败: {e2}")
```

---

## 配置选项

### 基础配置

```yaml
# ~/.nanobot/config.yaml
providers:
  default: anthropic

  anthropic:
    api_key: ${ANTHROPIC_API_KEY}
    model: claude-3-5-sonnet-20241022
    max_tokens: 4096
    temperature: 0.7
    timeout: 30
    default: true

  openai:
    api_key: ${OPENAI_API_KEY}
    model: gpt-4-turbo-preview
    max_tokens: 4096
    temperature: 0.7

  custom_provider:
    api_key: ${CUSTOM_API_KEY}
    api_base: https://api.example.com/v1
    timeout: 60
    retry_attempts: 3
```

### 高级配置

```python
# 使用配置字典
config = {
    "default": "anthropic",
    "providers": {
        "anthropic": {
            "api_key": "sk-ant-xxx",
            "model": "claude-3-opus-20240229",
            "max_tokens": 8192,
            "temperature": 0.3,
            "timeout": 60
        }
    }
}

# 创建注册表
registry = ProviderRegistry()
registry.load_from_config(config)
```

---

## 最佳实践

### 1. 配置管理

```python
import os
from nanobot.providers import ProviderRegistry

def load_provider_config() -> Dict:
    """从环境变量加载配置"""
    return {
        "api_key": os.getenv("LLM_API_KEY"),
        "model": os.getenv("LLM_MODEL", "claude-3-5-sonnet-20241022"),
        "max_tokens": int(os.getenv("LLM_MAX_TOKENS", "4096")),
        "temperature": float(os.getenv("LLM_TEMPERATURE", "0.7"))
    }
```

### 2. 提供商验证

```python
async def validate_all_providers():
    """验证所有提供商配置"""

    registry = ProviderRegistry()
    registry.load_from_config(config)

    results = await registry.validate_all()

    for provider_name, is_valid in results.items():
        if is_valid:
            print(f"✅ {provider_name}: 配置有效")
        else:
            print(f"❌ {provider_name}: 配置无效")

    return results
```

### 3. 重试机制

```python
from tenacity import retry, stop_after_attempt, wait_exponential

class ResilientProvider(LLMProvider):
    """带重试机制的提供商"""

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=4, max=10)
    )
    async def chat(self, messages: List[Dict[str, str]], **kwargs) -> LLMResponse:
        """带重试的聊天请求"""
        try:
            return await self._chat_impl(messages, **kwargs)
        except Exception as e:
            print(f"请求失败，重试中: {e}")
            raise
```

### 4. 负载均衡

```python
class LoadBalancedProvider(LLMProvider):
    """负载均衡提供商"""

    def __init__(self, providers: List[LLMProvider]):
        self.providers = providers
        self.current_index = 0

    async def chat(self, messages: List[Dict[str, str]], **kwargs) -> LLMResponse:
        """轮询选择提供商"""
        provider = self.providers[self.current_index]
        self.current_index = (self.current_index + 1) % len(self.providers)

        return await provider.chat(messages, **kwargs)
```

---

## 故障排除

### 常见问题

#### 1. API 密钥错误

**问题**：`Invalid API key` 错误

**解决方案**：
- 检查 API 密钥是否正确
- 确认密钥权限是否足够
- 验证密钥格式

```python
# 验证 API 密钥
await provider.validate_config()
```

#### 2. 网络连接问题

**问题**：连接超时或网络错误

**解决方案**：
- 检查网络连接
- 增加 timeout 设置
- 使用代理或 VPN

```python
provider = AnthropicProvider(
    provider_name="anthropic",
    config={
        "api_key": "sk-ant-xxx",
        "timeout": 60.0  # 增加超时时间
    }
)
```

#### 3. 配置文件错误

**问题**：配置加载失败

**解决方案**：
- 检查 YAML 格式
- 验证环境变量
- 使用配置验证

```python
# 验证配置
try:
    registry.load_from_config(config)
except Exception as e:
    print(f"配置错误: {e}")
```

### 调试技巧

```python
# 启用详细日志
import logging
logging.basicConfig(level=logging.DEBUG)

# 创建调试提供商
debug_provider = DebugProvider(
    provider_name="debug",
    config={"api_key": "sk-ant-xxx"}
)

# 记录请求和响应
debug_provider.log_requests = True
```

---

## 总结

`Provider` 系统提供了灵活的 LLM 提供商管理能力：

### 关键特性

1. **统一接口**：所有提供商实现相同的接口
2. **自动切换**：支持多提供商和故障转移
3. **配置验证**：自动验证 API 密钥和配置
4. **扩展性强**：易于添加自定义提供商

### 主要组件

- **LLMProvider**: 基类，定义核心接口
- **ProviderRegistry**: 管理多个提供商
- **内置提供商**: Anthropic、OpenAI、OpenRouter 等
- **自定义提供商**: 支持任何 API 集成

### 下一步

- 阅读 [Agent API 文档](./API_AGENT.md)
- 了解 [Tool API](./API_TOOL.md)
- 查看 [Channel API](./API_CHANNEL.md)

---

**文档版本**: 1.0.0
**最后更新**: 2026-03-03