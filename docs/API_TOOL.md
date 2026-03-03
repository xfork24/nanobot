# Tool API 文档

> **工具系统 API 参考** - 详细介绍工具基类、工具注册表和所有内置工具的使用方法

---

## 📑 目录

1. [Tool 基类](#tool-基类)
2. [ToolRegistry](#toolregistry)
3. [内置工具](#内置工具)
4. [自定义工具](#自定义工具)
5. [工具注册](#工具注册)
6. [工具执行](#工具执行)
7. [安全机制](#安全机制)
8. [使用示例](#使用示例)

---

## Tool 基类

`Tool` 是所有工具的抽象基类，定义了工具的标准接口。

### 类定义

```python
from abc import ABC, abstractmethod
from typing import Any, Dict, List
from pydantic import BaseModel

class Tool(ABC):
    """所有工具的抽象基类"""

    @property
    @abstractmethod
    def name(self) -> str:
        """工具名称（用于函数调用）"""
        pass

    @property
    @abstractmethod
    def description(self) -> str:
        """工具功能描述（给 LLM 看）"""
        pass

    @property
    @abstractmethod
    def parameters(self) -> dict[str, Any]:
        """参数的 JSON Schema"""
        pass

    @abstractmethod
    async def execute(self, **kwargs: Any) -> str:
        """执行工具逻辑"""
        pass

    def validate_params(self, params: dict) -> list[str]:
        """验证参数（可选实现）"""
        return []
```

### 核心属性

#### name
工具的唯一标识符，用于 LLM 函数调用。

```python
@property
def name(self) -> str:
    return "read_file"
```

#### description
工具的功能描述，帮助 LLM 理解工具的用途。

```python
@property
def description(self) -> str:
    return "读取文件的全部内容，支持文本文件"
```

#### parameters
参数的 JSON Schema 定义，用于 LLM 参数生成。

```python
@property
def parameters(self) -> dict[str, Any]:
    return {
        "type": "object",
        "properties": {
            "path": {
                "type": "string",
                "description": "文件路径（相对或绝对路径）"
            }
        },
        "required": ["path"]
    }
```

### 核心方法

#### execute()
工具的主要执行方法。

```python
async def execute(self, **kwargs: Any) -> str:
    """执行工具逻辑

    参数:
        kwargs: 工具参数

    返回:
        str: 执行结果（给 LLM 的文本响应）
    """
    # 实现工具逻辑
    result = await self._do_work(**kwargs)
    return f"执行成功：{result}"
```

#### validate_params()
验证工具参数（可选实现）。

```python
def validate_params(self, params: dict) -> list[str]:
    """验证参数

    参数:
        params: 工具参数

    返回:
        list[str]: 错误信息列表，空列表表示验证通过
    """
    errors = []

    if "path" not in params:
        errors.append("缺少必需参数 'path'")

    if not isinstance(params.get("path"), str):
        errors.append("'path' 必须是字符串")

    return errors
```

---

## ToolRegistry

`ToolRegistry` 负责工具的注册、管理和执行。

### 类定义

```python
class ToolRegistry:
    """工具注册和执行管理器"""

    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool) -> None:
        """注册工具"""
        pass

    def get_definitions(self) -> list[dict]:
        """获取所有工具的 OpenAI 格式定义"""
        pass

    async def execute(
        self,
        name: str,
        params: dict[str, Any]
    ) -> str:
        """执行工具"""
        pass

    @property
    def tool_names(self) -> list[str]:
        """获取所有工具名称"""
        pass
```

### 方法详解

#### register()

```python
def register(self, tool: Tool) -> None:
    """注册工具

    参数:
        tool: Tool 实例

    注意:
        - 如果工具名称已存在，会被覆盖
        - 建议在应用启动时注册所有工具
    """
    name = tool.name
    self._tools[name] = tool
    logger.info(f"Registered tool: {name}")
```

#### get_definitions()

```python
def get_definitions(self) -> list[dict]:
    """获取所有工具的 OpenAI 格式定义

    返回:
        list[dict]: 工具定义列表，用于 LLM 函数调用
    """
    definitions = []
    for tool in self._tools.values():
        definitions.append({
            "type": "function",
            "function": {
                "name": tool.name,
                "description": tool.description,
                "parameters": tool.parameters,
            }
        })
    return definitions
```

#### execute()

```python
async def execute(
    self,
    name: str,
    params: dict[str, Any]
) -> str:
    """执行工具

    参数:
        name: 工具名称
        params: 工具参数

    返回:
        str: 工具执行结果

    异常:
        ToolNotFoundError: 工具不存在
        ToolExecutionError: 工具执行失败
    """
    # 1. 查找工具
    tool = self._tools.get(name)
    if not tool:
        available = ", ".join(self.tool_names)
        raise ToolNotFoundError(f"Tool '{name}' not found. Available: {available}")

    # 2. 验证参数
    errors = tool.validate_params(params)
    if errors:
        raise ToolExecutionError(f"Invalid parameters: {'; '.join(errors)}")

    # 3. 执行工具
    try:
        result = await tool.execute(**params)
        logger.info(f"Tool '{name}' executed successfully")
        return result
    except Exception as e:
        logger.error(f"Tool '{name}' failed: {e}")
        raise ToolExecutionError(f"Tool execution failed: {str(e)}")
```

---

## 内置工具

### 1. 文件系统工具

#### ReadFileTool

```python
class ReadFileTool(Tool):
    """读取文件内容"""

    name = "read_file"
    description = "读取文件的全部内容，支持文本文件"

    parameters = {
        "type": "object",
        "properties": {
            "path": {
                "type": "string",
                "description": "文件路径（相对或绝对路径）"
            }
        },
        "required": ["path"]
    }

    def __init__(
        self,
        workspace: Path,
        allowed_dir: Path | None = None
    ):
        self.workspace = workspace
        self.allowed_dir = allowed_dir

    async def execute(self, path: str) -> str:
        """执行文件读取"""
        full_path = self._resolve_path(path)

        if self.allowed_dir and not self._is_allowed(full_path):
            return f"错误：访问被拒绝（路径在允许范围外）：{path}"

        try:
            content = full_path.read_text(encoding="utf-8")
            return content
        except Exception as e:
            return f"读取文件失败：{str(e)}"
```

#### WriteFileTool

```python
class WriteFileTool(Tool):
    """写入文件内容"""

    name = "write_file"
    description = "将内容写入文件（会覆盖现有文件）"

    parameters = {
        "type": "object",
        "properties": {
            "path": {
                "type": "string",
                "description": "文件路径"
            },
            "content": {
                "type": "string",
                "description": "要写入的内容"
            }
        },
        "required": ["path", "content"]
    }

    def __init__(
        self,
        workspace: Path,
        allowed_dir: Path | None = None
    ):
        self.workspace = workspace
        self.allowed_dir = allowed_dir

    async def execute(self, path: str, content: str) -> str:
        """执行文件写入"""
        full_path = self._resolve_path(path)

        # 安全检查
        if self.allowed_dir and not self._is_allowed(full_path):
            return f"错误：访问被拒绝"

        # 创建父目录
        full_path.parent.mkdir(parents=True, exist_ok=True)

        try:
            full_path.write_text(content, encoding="utf-8")
            return f"成功写入 {len(content)} 字符到 {path}"
        except Exception as e:
            return f"写入文件失败：{str(e)}"
```

#### EditFileTool

```python
class EditFileTool(Tool):
    """编辑文件（插入/替换内容）"""

    name = "edit_file"
    description = "编辑文件内容（支持插入、替换、删除）"

    parameters = {
        "type": "object",
        "properties": {
            "path": {
                "type": "string",
                "description": "文件路径"
            },
            "operation": {
                "type": "string",
                "enum": ["insert", "replace", "delete"],
                "description": "编辑操作类型"
            },
            "content": {
                "type": "string",
                "description": "要插入/替换的内容"
            },
            "position": {
                "type": "integer",
                "description": "操作位置（字符索引，默认为文件末尾）"
            }
        },
        "required": ["path", "operation"]
    }
```

### 2. Shell 工具

#### ExecTool

```python
class ExecTool(Tool):
    """安全的 Shell 命令执行"""

    name = "exec"
    description = "执行 Shell 命令（有安全限制）"

    parameters = {
        "type": "object",
        "properties": {
            "command": {
                "type": "string",
                "description": "要执行的命令"
            },
            "timeout": {
                "type": "integer",
                "description": "超时时间（秒）",
                "default": 30
            }
        },
        "required": ["command"]
    }

    # 危险命令黑名单
    DENY_PATTERNS = [
        r"rm -rf /",
        r"rm -rf \*",
        r":\( \)\{ :\|:& \};:",
        r"> /dev/",
        r"mkfs\.",
        r"dd if=/",
    ]

    def __init__(
        self,
        working_dir: Path | None = None,
        allowed_dirs: list[Path] | None = None,
        deny_patterns: list[str] | None = None,
        timeout: int = 30
    ):
        self.working_dir = working_dir or Path.cwd()
        self.allowed_dirs = allowed_dirs or [self.working_dir]
        self.deny_patterns = deny_patterns or self.DENY_PATTERNS
        self.default_timeout = timeout

    async def execute(
        self,
        command: str,
        timeout: int | None = None
    ) -> str:
        """执行命令（带安全检查）"""

        # 1. 安全检查
        for pattern in self.deny_patterns:
            if re.search(pattern, command):
                return f"错误：命令被安全策略阻止：{command}"

        # 2. 执行命令
        try:
            proc = await asyncio.create_subprocess_shell(
                command,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
                cwd=self.working_dir,
                env=self.env,
            )

            stdout, stderr = await asyncio.wait_for(
                proc.communicate(),
                timeout=timeout or self.default_timeout,
            )

            output = stdout.decode() + stderr.decode()
            return output if output else "(命令无输出)"

        except asyncio.TimeoutError:
            return f"错误：命令执行超时（{timeout}秒）"
        except Exception as e:
            return f"错误：命令执行失败：{str(e)}"
```

### 3. Web 工具

#### WebSearchTool

```python
class WebSearchTool(Tool):
    """网络搜索工具"""

    name = "web_search"
    description = "使用搜索引擎搜索网络信息"

    parameters = {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "搜索查询"
            },
            "max_results": {
                "type": "integer",
                "description": "最大结果数",
                "default": 10
            },
            "language": {
                "type": "string",
                "description": "搜索语言",
                "default": "zh"
            }
        },
        "required": ["query"]
    }

    def __init__(
        self,
        search_engine: str = "brave",
        api_key: str | None = None
    ):
        self.search_engine = search_engine
        self.api_key = api_key

    async def execute(
        self,
        query: str,
        max_results: int = 10,
        language: str = "zh"
    ) -> str:
        """执行搜索"""

        if self.search_engine == "brave":
            results = await self._brave_search(query, max_results)
        else:
            results = await self._default_search(query, max_results)

        # 格式化结果
        formatted = [f"搜索结果：{query}\n"]
        for i, result in enumerate(results[:max_results], 1):
            formatted.append(f"{i}. {result['title']}")
            formatted.append(f"   {result['url']}")
            formatted.append(f"   {result['snippet']}")
            formatted.append("")

        return "\n".join(formatted)
```

#### WebFetchTool

```python
class WebFetchTool(Tool):
    """获取网页内容"""

    name = "web_fetch"
    description = "获取网页的文本内容"

    parameters = {
        "type": "object",
        "properties": {
            "url": {
                "type": "string",
                "description": "网页 URL"
            },
            "max_length": {
                "type": "integer",
                "description": "最大内容长度",
                "default": 10000
            }
        },
        "required": ["url"]
    }

    async def execute(
        self,
        url: str,
        max_length: int = 10000
    ) -> str:
        """获取网页内容"""
        try:
            async with httpx.AsyncClient() as client:
                response = await client.get(url)
                response.raise_for_status()

                # 提取文本内容
                from bs4 import BeautifulSoup
                soup = BeautifulSoup(response.text, 'html.parser')

                # 移除脚本和样式
                for script in soup(["script", "style"]):
                    script.decompose()

                text = soup.get_text()
                text = " ".join(text.split())  # 清理空白

                if len(text) > max_length:
                    text = text[:max_length] + "..."

                return text
        except Exception as e:
            return f"获取网页失败：{str(e)}"
```

### 4. 定时任务工具

#### CronTool

```python
class CronTool(Tool):
    """定时任务管理工具"""

    name = "cron"
    description = "管理定时任务（创建、删除、列出）"

    parameters = {
        "type": "object",
        "properties": {
            "action": {
                "type": "string",
                "enum": ["add", "remove", "list"],
                "description": "操作类型"
            },
            "cron_expression": {
                "type": "string",
                "description": "Cron 表达式（用于 add 操作）"
            },
            "task": {
                "type": "string",
                "description": "任务内容（用于 add 操作）"
            },
            "task_id": {
                "type": "string",
                "description": "任务 ID（用于 remove 操作）"
            }
        },
        "required": ["action"]
    }

    async def execute(
        self,
        action: str,
        cron_expression: str | None = None,
        task: str | None = None,
        task_id: str | None = None
    ) -> str:
        """执行定时任务操作"""

        if action == "add":
            if not cron_expression or not task:
                return "错误：添加任务需要 cron_expression 和 task"

            # 添加定时任务
            task_id = str(uuid.uuid4())
            # 实现添加逻辑...

            return f"已添加定时任务 {task_id}: {task}"

        elif action == "remove":
            if not task_id:
                return "错误：删除任务需要 task_id"

            # 删除任务
            # 实现删除逻辑...

            return f"已删除任务 {task_id}"

        elif action == "list":
            # 列出所有任务
            tasks = await self._list_tasks()
            return "定时任务列表：\n" + "\n".join(tasks)

        else:
            return f"错误：不支持的操作 {action}"
```

### 5. 子代理工具

#### SpawnTool

```python
class SpawnTool(Tool):
    """创建子代理任务"""

    name = "spawn"
    description = "创建子代理任务来处理复杂任务"

    parameters = {
        "type": "object",
        "properties": {
            "task": {
                "type": "string",
                "description": "要处理的任务描述"
            },
            "model": {
                "type": "string",
                "description": "使用的模型",
                "default": "claude-3-5-sonnet-20241022"
            },
            "timeout": {
                "type": "integer",
                "description": "超时时间（秒）",
                "default": 300
            }
        },
        "required": ["task"]
    }

    async def execute(
        self,
        task: str,
        model: str = "claude-3-5-sonnet-20241022",
        timeout: int = 300
    ) -> str:
        """创建子代理任务"""

        try:
            # 创建子代理
            sub_agent = await self._create_sub_agent(model)

            # 执行任务
            result = await asyncio.wait_for(
                sub_agent.process_task(task),
                timeout=timeout
            )

            return f"子代理任务完成：\n{result}"

        except asyncio.TimeoutError:
            return f"任务超时（{timeout}秒）"
        except Exception as e:
            return f"子代理任务失败：{str(e)}"
```

---

## 自定义工具

### 创建自定义工具

#### 1. 基础自定义工具

```python
from nanobot.agent.tools.base import Tool

class MyCustomTool(Tool):
    """我的自定义工具"""

    name = "my_tool"
    description = "我的工具描述"

    parameters = {
        "type": "object",
        "properties": {
            "input": {
                "type": "string",
                "description": "输入参数"
            }
        },
        "required": ["input"]
    }

    def __init__(self, config: dict):
        self.config = config

    async def execute(self, input: str) -> str:
        """执行工具逻辑"""
        # 实现你的业务逻辑
        result = await self._process_input(input)
        return f"处理结果：{result}"

    def validate_params(self, params: dict) -> list[str]:
        """参数验证"""
        errors = []

        if "input" not in params:
            errors.append("缺少必需参数 'input'")
        elif not isinstance(params["input"], str):
            errors.append("'input' 必须是字符串")

        return errors

    async def _process_input(self, input_str: str) -> str:
        """处理输入"""
        # 实现具体逻辑
        return input_str.upper()
```

#### 2. 带状态的工具

```python
class StatefulTool(Tool):
    """带状态的工具"""

    def __init__(self):
        self._state: dict = {}

    @property
    def name(self) -> str:
        return "stateful_tool"

    @property
    def description(self) -> str:
        return "带状态管理工具"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "action": {
                    "type": "string",
                    "enum": ["get", "set", "clear"],
                    "description": "操作类型"
                },
                "key": {
                    "type": "string",
                    "description": "键名（get/set 操作）"
                },
                "value": {
                    "type": "any",
                    "description": "值（set 操作）"
                }
            },
            "required": ["action"]
        }

    async def execute(self, action: str, key: str | None = None, value: any = None) -> str:
        """执行状态操作"""

        if action == "get":
            if not key:
                return "错误：get 操作需要 key"

            value = self._state.get(key, "无")
            return f"键 '{key}' 的值：{value}"

        elif action == "set":
            if not key:
                return "错误：set 操作需要 key"

            self._state[key] = value
            return f"已设置键 '{key}' 的值为 {value}"

        elif action == "clear":
            self._state.clear()
            return "已清空所有状态"

        else:
            return f"错误：不支持的 action {action}"
```

#### 3. 异步工具

```python
import aiohttp

class AsyncApiTool(Tool):
    """异步 API 调用工具"""

    name = "async_api"
    description = "异步调用 REST API"

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
                "default": "GET"
            },
            "headers": {
                "type": "object",
                "description": "请求头"
            },
            "data": {
                "type": "object",
                "description": "请求数据"
            }
        },
        "required": ["url"]
    }

    async def execute(
        self,
        url: str,
        method: str = "GET",
        headers: dict | None = None,
        data: dict | None = None
    ) -> str:
        """异步调用 API"""

        try:
            async with aiohttp.ClientSession() as session:
                async with session.request(
                    method,
                    url,
                    headers=headers,
                    json=data
                ) as response:

                    response_data = await response.text()
                    return f"""
状态码: {response.status}
响应头: {dict(response.headers)}
响应体: {response_data}
"""

        except Exception as e:
            return f"API 调用失败：{str(e)}"
```

---

## 工具注册

### 注册单个工具

```python
from nanobot.agent.tools.registry import ToolRegistry

# 创建工具注册表
registry = ToolRegistry()

# 注册工具
registry.register(MyCustomTool())

# 注册内置工具
from nanobot.agent.tools.filesystem import ReadFileTool
registry.register(ReadFileTool(workspace=Path.cwd()))
```

### 批量注册工具

```python
def register_all_tools(registry: ToolRegistry, workspace: Path):
    """批量注册所有工具"""

    # 文件系统工具
    registry.register(ReadFileTool(workspace=workspace))
    registry.register(WriteFileTool(workspace=workspace))
    registry.register(EditFileTool(workspace=workspace))

    # Shell 工具
    registry.register(ExecTool(working_dir=workspace))

    # Web 工具
    registry.register(WebSearchTool())
    registry.register(WebFetchTool())

    # 定时任务工具
    registry.register(CronTool())

    # 子代理工具
    registry.register(SpawnTool())

# 使用批量注册
registry = ToolRegistry()
register_all_tools(registry, Path.cwd())
```

### 条件注册

```python
def register_conditional_tools(registry: ToolRegistry, config: dict):
    """根据配置条件注册工具"""

    # 如果配置了搜索引擎 API，注册搜索工具
    if config.get("brave_api_key"):
        registry.register(WebSearchTool(
            search_engine="brave",
            api_key=config["brave_api_key"]
        ))

    # 如果允许执行命令，注册 exec 工具
    if config.get("exec_enabled", False):
        registry.register(ExecTool(
            working_dir=Path(config.get("working_dir", "/tmp")),
            allowed_dirs=[Path(config.get("working_dir", "/tmp"))]
        ))

    # 其他条件注册...
```

---

## 工具执行

### 基本执行

```python
# 注册工具
registry = ToolRegistry()
registry.register(MyCustomTool())

# 执行工具
result = await registry.execute("my_tool", {"input": "hello"})
print(result)  # "处理结果：HELLO"
```

### 错误处理

```python
try:
    result = await registry.execute("my_tool", {"input": "hello"})
    print(f"执行成功：{result}")
except ToolNotFoundError as e:
    print(f"工具不存在：{e}")
except ToolExecutionError as e:
    print(f"工具执行失败：{e}")
except Exception as e:
    print(f"未知错误：{e}")
```

### 工具链执行

```python
async def execute_tool_chain(registry: ToolRegistry, steps: list[dict]) -> str:
    """执行工具链"""

    results = []

    for step in steps:
        tool_name = step["tool"]
        params = step.get("params", {})

        try:
            result = await registry.execute(tool_name, params)
            results.append(f"{tool_name}: {result}")
        except Exception as e:
            results.append(f"{tool_name}: 失败 - {e}")

    return "\n".join(results)
```

---

## 安全机制

### 1. 参数验证

所有工具都应该实现 `validate_params()` 方法：

```python
def validate_params(self, params: dict) -> list[str]:
    """验证参数"""
    errors = []

    # 检查必需参数
    required = ["path", "content"]
    for param in required:
        if param not in params:
            errors.append(f"缺少必需参数 '{param}'")

    # 检查参数类型
    if "path" in params and not isinstance(params["path"], str):
        errors.append("'path' 必须是字符串")

    return errors
```

### 2. 路径安全

```python
class SecureFileTool(Tool):
    """安全的文件操作工具"""

    def __init__(self, workspace: Path):
        self.workspace = workspace.absolute()

    def _resolve_path(self, path: str) -> Path:
        """解析并验证路径"""
        p = Path(path).absolute()

        # 检查是否在工作空间内
        try:
            p.relative_to(self.workspace)
            return p
        except ValueError:
            raise SecurityError("路径访问被拒绝")
```

### 3. 命令过滤

```python
class SecureExecTool(Tool):
    """安全的命令执行工具"""

    DENY_PATTERNS = [
        r"rm -rf /",
        r"sudo ",
        r"su ",
        r"> /dev/",
        r"mkfs\.",
        r"dd if=",
    ]

    def _check_command(self, command: str) -> bool:
        """检查命令是否安全"""
        for pattern in self.DENY_PATTERNS:
            if re.search(pattern, command):
                return False
        return True
```

---

## 使用示例

### 完整示例

```python
import asyncio
from pathlib import Path
from nanobot.agent.tools.registry import ToolRegistry
from nanobot.agent.tools.filesystem import ReadFileTool, WriteFileTool
from nanobot.agent.tools.shell import ExecTool

async def tool_usage_example():
    """工具使用完整示例"""

    # 1. 创建工具注册表
    registry = ToolRegistry()

    # 2. 注册工具
    workspace = Path.cwd()
    registry.register(ReadFileTool(workspace=workspace))
    registry.register(WriteFileTool(workspace=workspace))
    registry.register(ExecTool(working_dir=workspace))

    # 3. 使用工具
    try:
        # 读取文件
        result = await registry.execute("read_file", {"path": "README.md"})
        print("文件内容：")
        print(result[:500] + "..." if len(result) > 500 else result)

        # 写入文件
        await registry.execute("write_file", {
            "path": "test_output.txt",
            "content": "Hello from tools!"
        })

        # 执行命令
        result = await registry.execute("exec", {
            "command": "echo 'Hello World'"
        })
        print("命令执行结果：")
        print(result)

    except Exception as e:
        print(f"工具使用失败：{e}")

# 运行示例
asyncio.run(tool_usage_example())
```

### 动态工具加载

```python
class DynamicToolLoader:
    """动态工具加载器"""

    def __init__(self):
        self.registry = ToolRegistry()

    async def load_tools_from_directory(self, dir_path: Path):
        """从目录加载工具"""
        for file_path in dir_path.glob("*.py"):
            module_name = file_path.stem
            spec = importlib.util.spec_from_file_location(module_name, file_path)
            module = importlib.util.module_from_spec(spec)

            # 加载模块
            spec.loader.exec_module(module)

            # 查找工具类
            for name in dir(module):
                obj = getattr(module, name)
                if isinstance(obj, type) and issubclass(obj, Tool) and obj != Tool:
                    tool_instance = obj()
                    self.registry.register(tool_instance)
                    print(f"加载工具: {tool_instance.name}")

# 使用动态加载
loader = DynamicToolLoader()
await loader.load_tools_from_directory(Path("tools/"))
```

---

## 总结

### 关键要点

1. **Tool 基类**：所有工具都必须实现 name、description、parameters 和 execute 方法
2. **ToolRegistry**：负责工具的注册和管理，提供统一的执行接口
3. **内置工具**：提供了文件操作、Shell执行、Web交互等功能
4. **自定义工具**：可以通过继承 Tool 基类创建自定义功能
5. **安全机制**：参数验证、路径检查、命令过滤等安全措施

### 最佳实践

1. **参数验证**：始终实现 validate_params 方法
2. **错误处理**：在 execute 方法中妥善处理异常
3. **安全考虑**：对文件路径和命令进行安全检查
4. **文档完善**：提供清晰的描述和参数说明
5. **测试覆盖**：为工具编写完整的测试用例

### 下一步

- 阅读 [Agent API](./API_AGENT.md) 了解如何集成工具
- 查看 [Channel API](./API_CHANNEL.md) 了解渠道工具
- 了解 [Provider API](./API_PROVIDER.md) 了解提供商集成

---

**文档版本**: 1.0.0
**最后更新**: 2026-03-03