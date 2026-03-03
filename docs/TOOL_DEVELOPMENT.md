# 工具开发指南

> **如何创建自定义工具** - 从入门到精通的工具开发教程

---

## 📑 目录

1. [工具系统概述](#工具系统概述)
2. [快速入门](#快速入门)
3. [Tool 基类详解](#tool-基类详解)
4. [参数验证](#参数验证)
5. [安全机制](#安全机制)
6. [高级工具](#高级工具)
7. [工具注册](#工具注册)
8. [测试工具](#测试工具)
9. [最佳实践](#最佳实践)

---

## 工具系统概述

Nanobot 的工具系统允许 AI 代理执行各种实际任务，包括文件操作、代码执行、网络交互等。

### 核心概念

```mermaid
graph LR
    A[LLM 请求] --> B[ToolRegistry]
    B --> C[工具匹配]
    C --> D[参数解析]
    D --> E[Tool.execute]
    E --> F[执行结果]
    F --> G[响应返回]
```

### 工具类型

| 类型 | 示例 | 用途 |
|------|------|------|
| **文件工具** | read_file, write_file | 读写文件 |
| **Shell 工具** | exec, bash | 执行命令 |
| **Web 工具** | web_search, fetch_url | 网络请求 |
| **代码工具** | python, code_review | 代码处理 |
| **AI 工具** | translate, summarize | AI 辅助 |

---

## 快速入门

### 创建第一个工具

```python
from nanobot.tools.base import Tool
from typing import Dict, Any

class HelloTool(Tool):
    """打招呼工具"""

    def __init__(self):
        super().__init__(
            name="hello",
            description="向用户打招呼",
            parameters={
                "name": {
                    "type": "string",
                    "description": "用户名",
                    "required": True
                }
            }
        )

    async def execute(self, **kwargs) -> str:
        """执行工具"""
        name = kwargs.get('name', '朋友')
        return f"你好，{name}！欢迎使用 Nanobot！"
```

### 注册和使用

```python
from nanobot.tools.registry import ToolRegistry

# 创建工具注册表
tools = ToolRegistry()

# 注册工具
tools.register(HelloTool())

# 检查工具
print(tools.list_tools())  # ['hello']

# 调用工具
result = await tools.execute('hello', name='Alice')
print(result)  # 你好，Alice！欢迎使用 Nanobot！
```

---

## Tool 基类详解

### 基类结构

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, Optional, List
from pydantic import BaseModel, Field

class Tool(ABC):
    """工具基类"""

    def __init__(
        self,
        name: str,
        description: str,
        parameters: Optional[Dict[str, Any]] = None,
        working_dir: Optional[str] = None,
        enabled: bool = True
    ):
        self.name = name
        self.description = description
        self.parameters = parameters or {}
        self.working_dir = working_dir
        self.enabled = enabled
```

### 核心方法

#### execute()

```python
@abstractmethod
async def execute(self, **kwargs) -> str:
    """执行工具的核心方法

    Args:
        **kwargs: 解析后的参数

    Returns:
        str: 执行结果
    """
    pass
```

#### validate_params()

```python
async def validate_params(self, params: Dict[str, Any]) -> bool:
    """验证参数

    Args:
        params: 参数字典

    Returns:
        bool: 参数是否有效
    """
    # 检查必需参数
    required_params = [k for k, v in self.parameters.items() if v.get('required', False)]
    for param in required_params:
        if param not in params:
            return False

    # 检查参数类型
    for param_name, param_value in params.items():
        if param_name in self.parameters:
            expected_type = self.parameters[param_name].get('type')
            if not self._check_type(param_value, expected_type):
                return False

    return True
```

#### get_info()

```python
def get_info(self) -> Dict[str, Any]:
    """获取工具信息"""
    return {
        "name": self.name,
        "description": self.description,
        "parameters": self.parameters,
        "enabled": self.enabled
    }
```

### 完整示例

```python
import asyncio
import json
from pathlib import Path
from typing import Dict, Any
from nanobot.tools.base import Tool

class JsonTool(Tool):
    """JSON 处理工具"""

    def __init__(self, working_dir: str = "."):
        super().__init__(
            name="json",
            description="读取和操作 JSON 文件",
            parameters={
                "action": {
                    "type": "string",
                    "description": "操作类型：read, write, format",
                    "required": True,
                    "enum": ["read", "write", "format"]
                },
                "file": {
                    "type": "string",
                    "description": "文件路径",
                    "required": True
                },
                "content": {
                    "type": "string",
                    "description": "JSON 内容（用于 write）",
                    "required": False
                }
            },
            working_dir=working_dir
        )

    async def execute(self, **kwargs) -> str:
        """执行 JSON 操作"""
        action = kwargs['action']
        file_path = Path(self.working_dir) / kwargs['file']

        if action == "read":
            return await self._read_json(file_path)
        elif action == "write":
            return await self._write_json(file_path, kwargs.get('content', '{}'))
        elif action == "format":
            return await self._format_json(file_path)
        else:
            raise ValueError(f"不支持的操作: {action}")

    async def _read_json(self, file_path: Path) -> str:
        """读取 JSON 文件"""
        try:
            with open(file_path, 'r', encoding='utf-8') as f:
                data = json.load(f)
            return json.dumps(data, indent=2, ensure_ascii=False)
        except FileNotFoundError:
            return f"文件不存在: {file_path}"
        except json.JSONDecodeError:
            return f"JSON 格式错误: {file_path}"

    async def _write_json(self, file_path: Path, content: str) -> str:
        """写入 JSON 文件"""
        try:
            data = json.loads(content)
            with open(file_path, 'w', encoding='utf-8') as f:
                json.dump(data, f, indent=2, ensure_ascii=False)
            return f"成功写入 JSON: {file_path}"
        except json.JSONDecodeError:
            return "无效的 JSON 格式"

    async def _format_json(self, file_path: Path) -> str:
        """格式化 JSON 文件"""
        content = await self._read_json(file_path)
        if content.startswith("文件不存在") or content.startswith("JSON 格式错误"):
            return content

        try:
            data = json.loads(content)
            formatted = json.dumps(data, indent=2, ensure_ascii=False)
            with open(file_path, 'w', encoding='utf-8') as f:
                f.write(formatted)
            return f"已格式化 JSON: {file_path}"
        except:
            return f"格式化失败: {file_path}"
```

---

## 参数验证

### 参数类型定义

```python
from pydantic import BaseModel, Field

class ToolParameter(BaseModel):
    type: str  # string, number, boolean, array, object
    description: str
    required: bool = False
    default: Any = None
    enum: Optional[List[str]] = None
    pattern: Optional[str] = None  # 正则表达式
    minimum: Optional[float] = None
    maximum: Optional[float] = None
```

### 类型检查

```python
def _check_type(self, value: Any, expected_type: str) -> bool:
    """检查参数类型"""
    type_map = {
        'string': str,
        'number': (int, float),
        'integer': int,
        'boolean': bool,
        'array': list,
        'object': dict
    }

    if expected_type not in type_map:
        return True  # 未知类型，跳过检查

    return isinstance(value, type_map[expected_type])
```

### 验证示例

```python
class CalculatorTool(Tool):
    """计算器工具"""

    def __init__(self):
        super().__init__(
            name="calculator",
            description="执行数学计算",
            parameters={
                "operation": {
                    "type": "string",
                    "description": "运算类型",
                    "required": True,
                    "enum": ["add", "subtract", "multiply", "divide"]
                },
                "a": {
                    "type": "number",
                    "description": "第一个数字",
                    "required": True
                },
                "b": {
                    "type": "number",
                    "description": "第二个数字",
                    "required": True
                }
            }
        )

    async def execute(self, **kwargs) -> str:
        """执行计算"""
        operation = kwargs['operation']
        a = float(kwargs['a'])
        b = float(kwargs['b'])

        if operation == "add":
            result = a + b
        elif operation == "subtract":
            result = a - b
        elif operation == "multiply":
            result = a * b
        elif operation == "divide":
            if b == 0:
                return "错误：除数不能为零"
            result = a / b
        else:
            return f"未知运算：{operation}"

        return f"计算结果：{result}"
```

---

## 安全机制

### 权限检查

```python
class SecureTool(Tool):
    """带安全检查的工具"""

    def __init__(self, allowed_commands: List[str]):
        super().__init__(
            name="secure_exec",
            description="安全执行命令",
            parameters={
                "command": {
                    "type": "string",
                    "description": "要执行的命令",
                    "required": True
                }
            }
        )
        self.allowed_commands = allowed_commands

    async def execute(self, **kwargs) -> str:
        """执行安全的命令"""
        command = kwargs['command']

        # 检查命令是否安全
        if not self._is_safe_command(command):
            return "错误：不允许执行此命令"

        # 执行命令
        try:
            process = await asyncio.create_subprocess_shell(
                command,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE
            )
            stdout, stderr = await process.communicate()

            if process.returncode == 0:
                return stdout.decode().strip()
            else:
                return f"错误: {stderr.decode().strip()}"
        except Exception as e:
            return f"执行失败: {str(e)}"

    def _is_safe_command(self, command: str) -> bool:
        """检查命令是否安全"""
        # 检查黑名单
        blacklist = ['rm -rf', 'del', 'format', 'mkfs', 'dd']
        for blacklisted in blacklist:
            if blacklisted in command.lower():
                return False

        # 检查是否在允许列表中
        for allowed in self.allowed_commands:
            if command.startswith(allowed):
                return True

        return False
```

### 沙箱执行

```python
import tempfile
import os
from contextlib import contextmanager

class SandboxTool(Tool):
    """沙箱执行工具"""

    def __init__(self):
        super().__init__(
            name="sandbox",
            description="在沙箱中执行代码",
            parameters={
                "code": {
                    "type": "string",
                    "description": "要执行的代码",
                    "required": True
                },
                "language": {
                    "type": "string",
                    "description": "编程语言",
                    "required": True,
                    "enum": ["python", "bash"]
                }
            }
        )

    @contextmanager
    def _create_sandbox(self) -> str:
        """创建临时沙箱目录"""
        temp_dir = tempfile.mkdtemp(prefix="nanobot_sandbox_")
        try:
            yield temp_dir
        finally:
            # 清理沙箱
            import shutil
            shutil.rmtree(temp_dir, ignore_errors=True)

    async def execute(self, **kwargs) -> str:
        """在沙箱中执行代码"""
        code = kwargs['code']
        language = kwargs['language']

        with self._create_sandbox() as temp_dir:
            if language == "python":
                return await self._execute_python(code, temp_dir)
            elif language == "bash":
                return await self._execute_bash(code, temp_dir)
            else:
                return f"不支持的语言: {language}"

    async def _execute_python(self, code: str, temp_dir: str) -> str:
        """执行 Python 代码"""
        # 创建临时文件
        script_path = os.path.join(temp_dir, "script.py")
        with open(script_path, 'w') as f:
            f.write(code)

        # 设置受限环境
        env = os.environ.copy()
        env.update({
            'PYTHONPATH': temp_dir,
            'HOME': temp_dir,
            'TMPDIR': temp_dir
        })

        # 执行代码
        try:
            process = await asyncio.create_subprocess_exec(
                'python', script_path,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
                env=env
            )
            stdout, stderr = await process.communicate()

            if process.returncode == 0:
                return stdout.decode()
            else:
                return f"错误:\n{stderr.decode()}"
        except Exception as e:
            return f"执行失败: {str(e)}"
```

---

## 高级工具

### 带缓存的工具

```python
import time
from functools import lru_cache

class CachedTool(Tool):
    """带缓存的工具"""

    def __init__(self, cache_ttl: int = 300):
        super().__init__(
            name="cached_search",
            description="带缓存的搜索工具",
            parameters={
                "query": {
                    "type": "string",
                    "description": "搜索查询",
                    "required": True
                }
            }
        )
        self.cache_ttl = cache_ttl
        self.cache = {}

    async def execute(self, **kwargs) -> str:
        """执行带缓存的搜索"""
        query = kwargs['query']

        # 检查缓存
        cache_key = f"search:{query}"
        if cache_key in self.cache:
            cached_result, timestamp = self.cache[cache_key]
            if time.time() - timestamp < self.cache_ttl:
                return cached_result

        # 执行搜索
        result = await self._perform_search(query)

        # 缓存结果
        self.cache[cache_key] = (result, time.time())

        return result

    async def _perform_search(self, query: str) -> str:
        """执行实际的搜索"""
        # 这里应该是实际的搜索逻辑
        return f"搜索结果：{query}"
```

### 异步流式工具

```python
import asyncio
from typing import AsyncIterator

class StreamingTool(Tool):
    """流式输出工具"""

    def __init__(self):
        super().__init__(
            name="stream_download",
            description="流式下载文件",
            parameters={
                "url": {
                    "type": "string",
                    "description": "下载 URL",
                    "required": True
                }
            }
        )

    async def execute(self, **kwargs) -> str:
        """执行流式下载"""
        url = kwargs['url']

        # 模拟流式下载
        chunks = []
        total_size = 1000
        chunk_size = 100

        for i in range(0, total_size, chunk_size):
            # 模拟下载延迟
            await asyncio.sleep(0.1)
            progress = min(100, (i + chunk_size) / total_size * 100)
            chunks.append(f"下载进度: {progress:.1f}%")

        return "\n".join(chunks)
```

---

## 工具注册

### 基本注册

```python
from nanobot.tools.registry import ToolRegistry

# 创建工具注册表
tools = ToolRegistry()

# 注册单个工具
tools.register(HelloTool())

# 批量注册
tools.register_all([
    CalculatorTool(),
    JsonTool(),
    SecureTool(['ls', 'cat', 'echo'])
])

# 检查工具
print(tools.list_tools())
# ['hello', 'calculator', 'json', 'secure_exec']
```

### 动态注册

```python
class DynamicToolRegistry(ToolRegistry):
    """动态工具注册表"""

    async def register_from_directory(self, directory: str) -> None:
        """从目录注册工具"""
        from pathlib import Path
        import importlib.util

        tool_dir = Path(directory)

        for py_file in tool_dir.glob("*.py"):
            if py_file.name.startswith("_"):
                continue

            # 动态导入模块
            spec = importlib.util.spec_from_file_location(
                py_file.stem, py_file
            )
            module = importlib.util.module_from_spec(spec)
            spec.loader.exec_module(module)

            # 查找工具类
            for name in dir(module):
                obj = getattr(module, name)
                if isinstance(obj, type) and issubclass(obj, Tool) and obj != Tool:
                    tool_instance = obj()
                    self.register(tool_instance)

# 使用动态注册
registry = DynamicToolRegistry()
await registry.register_from_directory("./my_tools")
```

---

## 测试工具

### 单元测试

```python
import pytest
import asyncio
from hello_tool import HelloTool

class TestHelloTool:
    """HelloTool 测试"""

    @pytest.fixture
    def tool(self):
        return HelloTool()

    @pytest.mark.asyncio
    async def test_execute_with_name(self, tool):
        """测试带参数的执行"""
        result = await tool.execute(name="Alice")
        assert "Alice" in result
        assert "Nanobot" in result

    @pytest.mark.asyncio
    async def test_execute_without_name(self, tool):
        """测试默认参数"""
        result = await tool.execute()
        assert "朋友" in result

    @pytest.mark.asyncio
    async def test_validate_params(self, tool):
        """测试参数验证"""
        # 有效参数
        valid_params = {"name": "Alice"}
        assert await tool.validate_params(valid_params)

        # 缺少必需参数
        invalid_params = {}
        assert not await tool.validate_params(invalid_params)
```

### 集成测试

```python
import pytest
from nanobot.tools.registry import ToolRegistry

class TestToolIntegration:
    """工具集成测试"""

    @pytest.fixture
    def registry(self):
        tools = ToolRegistry()
        tools.register_all([
            HelloTool(),
            CalculatorTool()
        ])
        return tools

    @pytest.mark.asyncio
    async def test_tool_execution(self, registry):
        """测试工具执行流程"""
        # 测试 hello 工具
        result = await registry.execute('hello', name="Bob")
        assert "Bob" in result

        # 测试 calculator 工具
        result = await registry.execute('calculator',
                                      operation="add",
                                      a=10,
                                      b=20)
        assert "30" in result

    @pytest.mark.asyncio
    async def test_tool_not_found(self, registry):
        """测试工具不存在的情况"""
        with pytest.raises(Exception):
            await registry.execute('nonexistent_tool', arg="test")
```

---

## 最佳实践

### 1. 工具设计原则

```python
class WellDesignedTool(Tool):
    """设计良好的工具示例"""

    def __init__(self):
        super().__init__(
            name="file_manager",
            description="文件管理工具，支持读取、创建、删除文件",
            parameters={
                "action": {
                    "type": "string",
                    "description": "操作类型",
                    "required": True,
                    "enum": ["read", "create", "delete", "list"]
                },
                "path": {
                    "type": "string",
                    "description": "文件或目录路径",
                    "required": True
                },
                "content": {
                    "type": "string",
                    "description": "文件内容（用于创建文件）",
                    "required": False
                }
            }
        )

    async def execute(self, **kwargs) -> str:
        """执行文件操作"""
        action = kwargs['action']
        path = kwargs['path']

        try:
            if action == "read":
                return await self._read_file(path)
            elif action == "create":
                return await self._create_file(path, kwargs.get('content', ''))
            elif action == "delete":
                return await self._delete_file(path)
            elif action == "list":
                return await self._list_directory(path)
        except Exception as e:
            return f"操作失败: {str(e)}"
```

### 2. 错误处理

```python
class RobustTool(Tool):
    """健壮的工具实现"""

    async def execute(self, **kwargs) -> str:
        """带错误处理的执行"""
        try:
            # 参数验证
            if not await self.validate_params(kwargs):
                return "错误：参数验证失败"

            # 执行主逻辑
            result = await self._safe_execute(**kwargs)
            return result

        except PermissionError:
            return "错误：权限不足"
        except FileNotFoundError:
            return "错误：文件不存在"
        except ValueError as e:
            return f"错误：参数错误 - {str(e)}"
        except Exception as e:
            return f"错误：执行失败 - {str(e)}"

    async def _safe_execute(self, **kwargs) -> str:
        """安全的执行逻辑"""
        # 子类实现具体逻辑
        pass
```

### 3. 性能优化

```python
class OptimizedTool(Tool):
    """性能优化的工具"""

    def __init__(self):
        super().__init__(
            name="fast_search",
            description="快速搜索工具",
            parameters={
                "query": {
                    "type": "string",
                    "description": "搜索关键词",
                    "required": True
                },
                "max_results": {
                    "type": "integer",
                    "description": "最大结果数",
                    "required": False,
                    "default": 10
                }
            }
        )
        self._cache = {}

    @lru_cache(maxsize=1000)
    def _search_cached(self, query: str, max_results: int) -> str:
        """带缓存的搜索"""
        # 实际搜索逻辑
        return f"搜索 '{query}' 的结果（最多 {max_results} 条）"

    async def execute(self, **kwargs) -> str:
        """执行搜索"""
        query = kwargs['query']
        max_results = kwargs.get('max_results', 10)

        # 使用缓存
        cache_key = (query, max_results)
        if cache_key in self._cache:
            return self._cache[cache_key]

        # 执行搜索
        result = self._search_cached(query, max_results)

        # 缓存结果
        self._cache[cache_key] = result

        return result
```

### 4. 文档化

```python
class DocumentedTool(Tool):
    """文档完善的工具"""

    def __init__(self):
        super().__init__(
            name="data_processor",
            description="""
            数据处理工具，支持 CSV、JSON、XML 格式的数据处理。

            功能包括：
            - 数据格式转换
            - 数据清洗
            - 数据分析
            - 数据导出

            使用示例：
            - 转换格式: data_processor action=convert from=data.csv to=data.json
            - 数据清洗: data_processor action=clean data='{"name": "test"}'
            """,
            parameters={
                "action": {
                    "type": "string",
                    "description": "操作类型：convert（转换）、clean（清洗）、analyze（分析）",
                    "required": True,
                    "enum": ["convert", "clean", "analyze"]
                },
                "data": {
                    "type": "string",
                    "description": "要处理的数据",
                    "required": True
                },
                "format": {
                    "type": "string",
                    "description": "数据格式：csv、json、xml",
                    "required": False,
                    "default": "json"
                }
            }
        )

    async def execute(self, **kwargs) -> str:
        """执行数据处理"""
        action = kwargs['action']
        data = kwargs['data']
        format_type = kwargs.get('format', 'json')

        # 实现...
        pass
```

---

## 总结

通过本指南，你已经学会了如何创建和使用 Nanobot 的工具：

### 核心要点

1. **继承 Tool 基类**：实现自己的工具逻辑
2. **定义参数**：使用 JSON Schema 定义工具参数
3. **实现 execute**：编写具体的执行逻辑
4. **注册工具**：使用 ToolRegistry 注册工具
5. **测试验证**：编写单元测试确保工具质量

### 进阶学习

- 查看 [API 文档](./API_TOOL.md) 了解工具系统 API
- 阅读 [架构指南](./NANOBOT_ARCHITECTURE_GUIDE.md) 了解设计原理
- 查看 [代码示例](./CODE_EXAMPLES.md) 获取更多实例

---

**文档版本**: 1.0.0
**最后更新**: 2026-03-03