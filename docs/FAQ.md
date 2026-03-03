# 常见问题解答

> **Nanobot FAQ** - 解答常见问题和疑问

---

## 📑 目录

1. [安装问题](#安装问题)
2. [配置问题](#配置问题)
3. [功能问题](#功能问题)
4. [工具问题](#工具问题)
5. [渠道问题](#渠道问题)
6. [性能问题](#性能问题)
7. [安全问题](#安全问题)
8. [开发问题](#开发问题)

---

## 安装问题

### Q: 如何检查系统是否符合安装要求？

```bash
# 检查 Python 版本
python --version
# 需要 Python 3.11 或更高版本

# 检查内存
free -h
# 建议 8GB 或更多

# 检查磁盘空间
df -h
# 建议 10GB 可用空间

# 检查网络连接
ping -c 3 github.com
```

### Q: pip 安装失败怎么办？

```bash
# 1. 更新 pip
pip install --upgrade pip

# 2. 清理缓存
pip cache purge

# 3. 使用国内镜像
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple nanobot-ai

# 4. 使用 --user 选项
pip install --user nanobot-ai

# 5. 创建虚拟环境
python -m venv venv
source venv/bin/activate
pip install nanobot-ai
```

### Q: Windows 系统安装失败？

```bash
# 1. 以管理员身份运行 PowerShell
# 2. 安装 Visual Studio Build Tools
winget install Microsoft.VisualStudio.2022.BuildTools

# 3. 安装 Python 3.11
winget install Python.Python.3.11

# 4. 重新安装
pip install nanobot-ai

# 5. 如果仍有问题，使用 conda
conda install -c conda-forge nanobot-ai
```

### Q: 如何验证安装是否成功？

```bash
# 1. 检查命令是否存在
which nanobot

# 2. 检查版本
nanobot --version

# 3. 运行诊断
nanobot doctor

# 4. 测试基本功能
nanobot agent "你好"
```

### Q: 如何更新到最新版本？

```bash
# 1. 备份当前配置
cp -r ~/.nanobot ~/.nanobot.backup

# 2. 更新 nanobot
pip install --upgrade nanobot-ai

# 3. 运行升级脚本
nanobot upgrade

# 4. 重启服务
nanobot gateway
```

---

## 配置问题

### Q: 配置文件放在哪里？

```bash
# 默认位置
~/.nanobot/config.yaml

# 自定义位置
export NANOBOT_CONFIG=/path/to/config.yaml
```

### Q: 如何配置多个 LLM 提供商？

```yaml
# ~/.nanobot/config.yaml
providers:
  default: auto

  anthropic:
    api_key: ${ANTHROPIC_API_KEY}
    model: claude-3-5-sonnet-20241022
    default: true

  openai:
    api_key: ${OPENAI_API_KEY}
    model: gpt-4-turbo-preview

  openrouter:
    api_key: ${OPENROUTER_API_KEY}
    model: anthropic/claude-3.5-sonnet
```

### Q: 如何限制访问权限？

```yaml
# ~/.nanobot/config.yaml
channels:
  telegram:
    allow_from:
      - "123456789"  # 允许的用户 ID
      - "987654321"
    # 不写 allow_from 默认允许所有人

  discord:
    allow_guilds:
      - "123456789"  # 允许的服务器 ID
    admin_users:
      - "123456789"  # 管理员用户 ID
```

### Q: 如何配置环境变量？

```bash
# 方法 1: 直接在终端
export ANTHROPIC_API_KEY=sk-ant-xxx
export TELEGRAM_BOT_TOKEN=123456:ABC-DEF

# 方法 2: 使用 .env 文件
echo "ANTHROPIC_API_KEY=sk-ant-xxx" >> ~/.nanobot/.env
echo "TELEGRAM_BOT_TOKEN=123456:ABC-DEF" >> ~/.nanobot/.env

# 方法 3: 使用 fish shell
set -x ANTHROPIC_API_KEY sk-ant-xxx
set -x TELEGRAM_BOT_TOKEN 123456:ABC-DEF
```

### Q: 如何配置工作空间？

```yaml
# ~/.nanobot/config.yaml
global:
  workspace: "/path/to/workspace"

# 或者使用环境变量
export NANOBOT_WORKSPACE="/custom/workspace"
```

---

## 功能问题

### Q: 如何创建新会话？

```bash
# 方法 1: 在聊天中输入
/new

# 方法 2: 使用命令行
nanobot agent --new-session "你的消息"

# 方法 3: 在程序中调用
from nanobot.bus.events import InboundMessage
await bus.publish_inbound(InboundMessage(
    channel="telegram",
    chat_id="123456789",
    content="/new"
))
```

### Q: 如何查看系统状态？

```bash
# 方法 1: 使用命令
nanobot status

# 方法 2: 访问 Web API
curl http://localhost:8080/health

# 方法 3: 在聊天中输入
/status
```

### Q: 如何自定义系统提示？

```markdown
# ~/.nanobot/workspace/IDENTITY.md

# 你是谁

你是一个专业的 Python 开发助手，具有丰富的开发经验。

## 你的专长

- Python 编程
- 代码审查
- 性能优化
- 最佳实践

## 你的风格

- 简洁明了
- 提供代码示例
- 解释清晰
```

### Q: 如何添加自定义技能？

```markdown
# ~/.nanobot/workspace/skills/my_skill/SKILL.md

---
name: my_skill
description: 我的自定义技能
---

# 技能标题

你是...，专门负责...

## 使用方法

当用户需要...时...
```

### Q: 如何查看历史对话？

```bash
# 查看 MEMORY.md（长期记忆）
cat ~/.nanobot/workspace/memory/MEMORY.md

# 查看 HISTORY.md（对话历史）
cat ~/.nanobot/workspace/memory/HISTORY.md

# 使用命令行工具
nanobot history --limit 10
```

---

## 工具问题

### Q: 如何禁用特定工具？

```yaml
# ~/.nanobot/config.yaml
tools:
  exec:
    enabled: false   # 禁用 exec 工具
  read_file:
    enabled: false   # 禁用 read_file 工具

  # 或者全局禁用
  global:
    enabled: false   # 禁用所有工具
```

### Q: 如何限制工具执行？

```yaml
# ~/.nanobot/config.yaml
tools:
  exec:
    allowed_commands:    # 允许的命令
      - "ls"
      - "cat"
      - "echo"
      - "find"
    blocked_commands:   # 禁止的命令
      - "rm"
      - "mv"
      - "cp"
    working_dir: "/tmp"  # 限制工作目录
```

### Q: 如何添加自定义工具？

```python
# 创建自定义工具
from nanobot.tools.base import Tool

class MyTool(Tool):
    def __init__(self):
        super().__init__(
            name="my_tool",
            description="我的自定义工具",
            parameters={
                "input": {
                    "type": "string",
                    "description": "输入文本",
                    "required": true
                }
            }
        )

    async def execute(self, **kwargs):
        input_text = kwargs["input"]
        return f"处理结果: {input_text}"

# 注册工具
from nanobot.tools.registry import ToolRegistry
tools = ToolRegistry()
tools.register(MyTool())
```

### Q: 工具调用失败怎么办？

```bash
# 1. 查看详细日志
nanobot gateway --log-level DEBUG

# 2. 检查工具配置
cat ~/.nanobot/config.yaml | grep tools

# 3. 测试工具
nanobot test tool exec "ls -la"

# 4. 查看错误日志
tail -f ~/.nanobot/logs/nanobot.log | grep ERROR
```

### Q: 如何添加新的 LLM 工具？

```python
# 创建自定义 LLM 提供商
from nanobot.providers.base import LLMProvider, LLMResponse

class CustomLLMProvider(LLMProvider):
    def __init__(self, config):
        super().__init__("custom", config)
        self.api_key = config["api_key"]

    async def chat(self, messages, **kwargs):
        # 调用自定义 API
        response = await self._call_api(messages)
        return LLMResponse(
            content=response["content"],
            tool_calls=response.get("tool_calls", []),
            usage=response.get("usage", {})
        )

    async def _call_api(self, messages):
        # 实现 API 调用
        pass

# 注册提供商
from nanobot.providers.registry import ProviderRegistry
registry = ProviderRegistry()
registry.register(CustomLLMProvider({
    "api_key": "your-api-key"
}))
```

---

## 渠道问题

### Q: 如何添加新的渠道？

```python
from nanobot.channels.base import BaseChannel

class CustomChannel(BaseChannel):
    def __init__(self, config):
        super().__init__(
            name="custom",
            description="自定义渠道",
            config=config
        )

    async def send(self, message):
        # 实现消息发送
        pass

    async def receive(self):
        # 实现消息接收
        pass

# 注册渠道
from nanobot.channels.manager import ChannelManager
manager = ChannelManager()
manager.register(CustomChannel(config))
```

### Q: Telegram 渠道不响应怎么办？

```bash
# 1. 检查 Token
echo $TELEGRAM_BOT_TOKEN

# 2. 测试 Bot Token
curl "https://api.telegram.org/bot<YOUR_TOKEN>/getMe"

# 3. 检查网络
curl -I https://api.telegram.org

# 4. 查看 Telegram 相关日志
tail -f ~/.nanobot/logs/nanobot.log | grep telegram

# 5. 检查配置
cat ~/.nanobot/config.yaml | grep telegram
```

### Q: 如何设置渠道 webhook？

```yaml
# ~/.nanobot/config.yaml
channels:
  telegram:
    webhook:
      enabled: true
      url: "https://your-domain.com/telegram/webhook"
      secret_token: "${TELEGRAM_WEBHOOK_SECRET}"

# 启用 webhook
nanobot channel telegram set-webhook
```

### Q: 如何处理渠道消息格式不匹配？

```python
# 创建消息转换器
class MessageConverter:
    @staticmethod
    def convert_to_inbound(platform_message):
        # 转换平台特定的消息格式
        return InboundMessage(
            channel="custom",
            sender_id=platform_message["user_id"],
            chat_id=platform_message["chat_id"],
            content=platform_message["text"],
            timestamp=datetime.fromtimestamp(platform_message["timestamp"])
        )

    @staticmethod
    def convert_from_outbound(outbound_message):
        # 转换为平台特定的格式
        return {
            "chat_id": outbound_message.chat_id,
            "text": outbound_message.content
        }
```

---

## 性能问题

### Q: 如何优化内存使用？

```yaml
# ~/.nanobot/config.yaml
agents:
  memory_window: 20        # 减少记忆窗口
  consolidate_threshold: 50 # 降低整合阈值

# 定期清理
nanobot cleanup
```

### Q: 如何提高响应速度？

```yaml
# ~/.nanobot/config.yaml
agents:
  timeout: 15             # 降低超时时间
  max_tokens: 2048       # 减少最大令牌数

# 启用缓存
cache:
  enabled: true
  type: redis
  url: "redis://localhost:6379"
```

### Q: 如何处理高并发？

```yaml
# ~/.nanobot/config.yaml
channels:
  telegram:
    max_connections: 50    # 增加最大连接数
    timeout: 10           # 降低超时时间

database:
  pool_size: 20          # 增加连接池大小
  max_overflow: 50       # 增加溢出连接数
```

### Q: 监控性能指标？

```bash
# 查看 CPU 和内存使用
top -p $(pgrep nanobot)

# 监控网络连接
netstat -an | grep nanobot

# 查看 nanobot 统计
nanobot stats

# 使用 Prometheus 导出指标
curl http://localhost:9090/metrics
```

### Q: 如何配置负载均衡？

```yaml
# 使用多个实例
services:
  - name: nanobot-1
    replicas: 2
    ports:
      - "8080:8080"

  - name: nanobot-2
    replicas: 2
    ports:
      - "8081:8080"

# 使用 Nginx 负载均衡
upstream nanobot {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}

server {
    location / {
        proxy_pass http://nanobot;
    }
}
```

---

## 安全问题

### Q: 如何保护 API 密钥？

```bash
# 使用环境变量
export ANTHROPIC_API_KEY=sk-ant-xxx

# 使用 .env 文件
echo "ANTHROPIC_API_KEY=sk-ant-xxx" >> ~/.nanobot/.env

# 使用密钥管理工具
# 1. 安装 keyring
pip install keyring

# 2. 保存密钥
keyring set nanobot anthropic_api_key

# 3. 在配置中引用
providers:
  anthropic:
    api_key: ${keyring://nanobot/anthropic_api_key}
```

### Q: 如何启用 SSL/TLS？

```yaml
# ~/.nanobot/config.yaml
security:
  ssl:
    enabled: true
    cert_file: "/path/to/cert.pem"
    key_file: "/path/to/key.pem"
    ca_file: "/path/to/ca.pem"

# 使用反向代理
server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Q: 如何限制 API 访问？

```yaml
# ~/.nanobot/config.yaml
security:
  api_keys:
    enabled: true
    keys:
      - "sk-test-1"
      - "sk-test-2"
    rate_limit:
      enabled: true
      requests_per_minute: 100
      requests_per_hour: 1000

# 使用 IP 白名单
ip_whitelist:
  - "192.168.1.0/24"
  - "10.0.0.0/8"
```

### Q: 如何配置审计日志？

```yaml
# ~/.nanobot/config.yaml
logging:
  audit:
    enabled: true
    path: "~/.nanobot/logs/audit.log"
    log_actions:
      - "auth"
      - "tool_usage"
      - "message_processing"
      - "configuration_change"

    log_format: "{time} | {user} | {action} | {details}"
```

---

## 开发问题

### Q: 如何运行开发环境？

```bash
# 1. 克隆仓库
git clone https://github.com/nanobot-ai/nanobot.git
cd nanobot

# 2. 创建虚拟环境
python -m venv venv
source venv/bin/activate

# 3. 安装开发依赖
pip install -e ".[dev]"

# 4. 安装 pre-commit
pre-commit install

# 5. 运行开发服务器
nanobot dev
```

### Q: 如何编写测试？

```python
# test_example.py
import pytest
from nanobot.agent.loop import AgentLoop

@pytest.mark.asyncio
async def test_agent_response():
    # 创建测试 agent
    agent = AgentLoop(...)

    # 测试消息
    message = InboundMessage(
        channel="test",
        sender_id="user123",
        chat_id="test",
        content="你好"
    )

    # 执行测试
    response = await agent._process_message(message)

    # 断言结果
    assert response is not None
    assert "你好" in response.content
```

### Q: 如何贡献代码？

```bash
# 1. Fork 仓库
git clone https://github.com/YOUR_USERNAME/nanobot.git
cd nanobot

# 2. 创建分支
git checkout -b feature/your-feature

# 3. 提交更改
git add .
git commit -m "Add your feature"

# 4. 推送到 fork
git push origin feature/your-feature

# 5. 创建 Pull Request
gh pr create --title "Your Feature" --body "Description"
```

### Q: 如何使用调试工具？

```python
# 启用调试模式
import logging
logging.basicConfig(level=logging.DEBUG)

# 创建调试 agent
debug_agent = AgentLoop(
    ...,
    debug=True
)

# 使用调试工具
from nanobot.utils.debug import debug_message
await debug_agent._process_message(debug_message)
```

### Q: 如何处理代码风格？

```bash
# 格式化代码
ruff format .

# 检查代码
ruff check .

# 运行类型检查
mypy .

# 运行所有检查
pre-commit run --all-files
```

---

## 故障排除流程

### 1. 基础检查

```bash
# 检查运行状态
systemctl status nanobot
ps aux | grep nanobot

# 检查端口占用
netstat -tulpn | grep 8080

# 检查资源使用
top -p $(pgrep nanobot)
df -h
```

### 2. 日志分析

```bash
# 查看最新日志
tail -f ~/.nanobot/logs/nanobot.log

# 搜索错误
grep ERROR ~/.nanobot/logs/nanobot.log

# 搜索特定组件
grep -i "telegram" ~/.nanobot/logs/nanobot.log

# 查看调试信息
grep DEBUG ~/.nanobot/logs/nanobot.log | tail -20
```

### 3. 配置验证

```bash
# 验证配置文件
nanobot validate-config

# 检查配置语法
python -c "
import yaml
with open('config.yaml') as f:
    yaml.safe_load(f)
print('配置文件语法正确')
"

# 导出当前配置
nanobot config export > current_config.yaml
```

### 4. 网络测试

```bash
# 测试 API 连接
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "test"}'

# 测试 LLM 连接
nanobot test connection

# 测试渠道连接
nanobot test channel telegram
```

### 5. 数据备份

```bash
# 备份数据
tar -czf nanobot-backup-$(date +%Y%m%d).tar.gz \
  ~/.nanobot/

# 恢复数据
tar -xzf nanobot-backup-20240303.tar.gz

# 验证恢复
nanobot verify-data
```

---

## 支持资源

### 官方资源

- **文档中心**: [docs.nanobot.ai](https://docs.nanobot.ai)
- **GitHub 仓库**: [github.com/nanobot-ai/nanobot](https://github.com/nanobot-ai/nanobot)
- **问题跟踪**: [GitHub Issues](https://github.com/nanobot-ai/nanobot/issues)
- **讨论区**: [GitHub Discussions](https://github.com/nanobot-ai/nanobot/discussions)

### 社区资源

- **Discord 服务器**: [discord.gg/nanobot](https://discord.gg/nanobot)
- **QQ 群**: 123456789
- **微信群**: 请添加助手 Nanobot 获取邀请

### 学习资源

- **快速入门**: [QUICK_START.md](./QUICK_START.md)
- **架构指南**: [NANOBOT_ARCHITECTURE_GUIDE.md](./NANOBOT_ARCHITECTURE_GUIDE.md)
- **代码示例**: [CODE_EXAMPLES.md](./CODE_EXAMPLES.md)
- **API 文档**: [API_AGENT.md](./API_AGENT.md)

---

## 反馈与建议

### 如何报告问题

```bash
# 使用命令行工具
nanobot bug-report

# 或直接创建 GitHub Issue
gh issue create --title "问题描述" --body "详细描述"
```

### 如何提出建议

1. 在 [GitHub Discussions](https://github.com/nanobot-ai/nanobot/discussions) 提出
2. 加入 Discord 讨论组
3. 发送邮件到 dev@nanobot.ai

### 感谢

感谢您使用 Nanobot！您的反馈帮助我们改进产品。

---

**文档版本**: 1.0.0
**最后更新**: 2026-03-03