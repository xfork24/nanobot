# 安装指南

> **详细安装说明** - 从零开始部署 Nanobot

---

## 📑 目录

1. [系统要求](#系统要求)
2. [快速安装](#快速安装)
3. [源码安装](#源码安装)
4. [Docker 部署](#docker-部署)
5. [云部署](#云部署)
6. [环境配置](#环境配置)
7. [验证安装](#验证安装)
8. [故障排除](#故障排除)
9. [升级指南](#升级指南)

---

## 系统要求

### 硬件要求

| 组件 | 最低配置 | 推荐配置 |
|------|----------|----------|
| **CPU** | 2 核 | 4 核或更多 |
| **内存** | 4GB | 8GB 或更多 |
| **存储** | 10GB | 50GB SSD |
| **网络** | 稳定连接 | 高速宽带 |

### 软件要求

- **Python**: 3.11 或更高版本
- **操作系统**: Linux、macOS、Windows
- **包管理器**: pip、conda（可选）
- **Git**: 用于源码安装

### Python 依赖

```bash
# 检查 Python 版本
python --version
# 需要 Python 3.11.0 或更高

# 检查 pip
pip --version

# 升级 pip
pip install --upgrade pip
```

---

## 快速安装

### 方法 1：pip 安装（推荐）

```bash
# 1. 安装 nanobot
pip install nanobot-ai

# 2. 验证安装
nanobot --version

# 3. 初始化配置
nanobot onboard

# 4. 启动服务
nanobot gateway
```

### 方法 2：从源码安装

```bash
# 1. 克隆仓库
git clone https://github.com/nanobot-ai/nanobot.git
cd nanobot

# 2. 安装依赖
pip install -e .

# 3. 验证安装
nanobot --version

# 4. 初始化配置
nanobot onboard

# 5. 启动服务
nanobot gateway
```

### 方法 3：虚拟环境安装

```bash
# 1. 创建虚拟环境
python -m venv nanobot-env
source nanobot-env/bin/activate  # Linux/macOS
# 或 nanobot-env\Scripts\activate  # Windows

# 2. 升级 pip
pip install --upgrade pip

# 3. 安装 nanobot
pip install nanobot-ai

# 4. 初始化配置
nanobot onboard

# 5. 启动服务
nanobot gateway
```

---

## 源码安装

### 详细步骤

```bash
# 1. 克隆仓库
git clone https://github.com/nanobot-ai/nanobot.git
cd nanobot

# 2. 创建虚拟环境（推荐）
python -m venv venv
source venv/bin/activate

# 3. 安装开发依赖
pip install --upgrade pip
pip install -e ".[dev]"

# 4. 运行测试
python -m pytest

# 5. 初始化配置
nanobot onboard

# 6. 启动开发模式
nanobot dev
```

### 开发环境设置

```bash
# 安装开发工具
pip install pytest ruff hatch pre-commit

# 设置 pre-commit 钩子
pre-commit install

# 运行代码格式化
ruff format .
ruff check .

# 运行类型检查
mypy .
```

### 依赖说明

```txt
# requirements.txt
# 核心依赖
pydantic>=2.0.0
typer>=0.9.0
loguru>=0.7.0
httpx>=0.24.0
python-dotenv>=1.0.0

# 异步支持
aiohttp>=3.8.0
asyncio-mqtt>=0.13.0

# LLM 集成
litellm>=1.0.0

# 数据处理
pandas>=2.0.0
numpy>=1.24.0

# 可选依赖
[dev]
pytest>=7.0.0
ruff>=0.1.0
mypy>=1.0.0
black>=23.0.0
```

---

## Docker 部署

### 基础 Docker 部署

```bash
# 1. 拉取镜像
docker pull nanobotai/nanobot:latest

# 2. 运行容器
docker run -d \
  --name nanobot \
  -v ~/.nanobot:/app/data \
  -p 8080:8080 \
  -e ANTHROPIC_API_KEY=your-api-key \
  nanobotai/nanobot:latest
```

### Docker Compose 部署

```yaml
# docker-compose.yml
version: '3.8'

services:
  nanobot:
    image: nanobotai/nanobot:latest
    container_name: nanobot
    restart: unless-stopped
    volumes:
      - ~/.nanobot:/app/data
      - ./config.yaml:/app/config.yaml
    ports:
      - "8080:8080"
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
    depends_on:
      - redis
    networks:
      - nanobot-network

  redis:
    image: redis:7-alpine
    container_name: nanobot-redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    networks:
      - nanobot-network

volumes:
  redis_data:

networks:
  nanobot-network:
    driver: bridge
```

启动服务：

```bash
# 启动所有服务
docker-compose up -d

# 查看日志
docker-compose logs -f nanobot

# 停止服务
docker-compose down
```

### Dockerfile 自定义

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    git \
    && rm -rf /var/lib/apt/lists/*

# 复制依赖文件
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制源码
COPY . .

# 创建数据目录
RUN mkdir -p /app/data

# 设置环境变量
ENV PYTHONPATH=/app
ENV PATH=/app:$PATH

# 暴露端口
EXPOSE 8080

# 启动命令
CMD ["nanobot", "gateway"]
```

---

## 云部署

### AWS 部署

#### 1. 使用 AWS EC2

```bash
# 创建 EC2 实例
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.medium \
  --key-name my-key-pair \
  --security-group-ids sg-1234567890abcdef

# 连接到实例
ssh -i my-key-pair.pem ec2-user@your-public-ip

# 在实例上安装 Nanobot
sudo yum update -y
sudo yum install -y python3 python3-pip
pip3 install nanobot-ai
```

#### 2. 使用 AWS ECS

```yaml
# task-definition.json
{
  "family": "nanobot-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "nanobot",
      "image": "nanobotai/nanobot:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ANTHROPIC_API_KEY",
          "value": "your-api-key"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/nanobot",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### Google Cloud 部署

#### 1. 使用 Google Cloud Run

```bash
# 部署到 Cloud Run
gcloud run deploy nanobot \
  --image gcr.io/your-project/nanobot:latest \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars ANTHROPIC_API_KEY=your-api-key
```

#### 2. 使用 Google Compute Engine

```bash
# 创建实例
gcloud compute instances create nanobot-instance \
  --project=your-project \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-11 \
  --image-project=debian-cloud

# SSH 到实例
gcloud compute ssh nanobot-instance

# 安装 Nanobot
sudo apt update
sudo apt install -y python3 python3-pip
pip3 install nanobot-ai
```

### Azure 部署

#### 1. 使用 Azure Container Instances

```bash
# 部署到 ACI
az container create \
  --resource-group nanobot-rg \
  --name nanobot \
  --image nanobotai/nanobot:latest \
  --ports 8080 \
  --environment-variables ANTHROPIC_API_KEY=your-api-key
```

#### 2. 使用 Azure App Service

```bash
# 创建 Web App
az webapp create \
  --resource-group nanobot-rg \
  --name nanobot-app \
  --plan nanobot-plan \
  --runtime "PYTHON:3.11"

# 部署代码
az webapp deployment source config-zip \
  --resource-group nanobot-rg \
  --name nanobot-app \
  --src nanobot.zip
```

---

## 环境配置

### 配置文件设置

```bash
# 创建配置目录
mkdir -p ~/.nanobot

# 生成配置文件
nanobot onboard

# 配置文件结构
~/.nanobot/
├── config.yaml          # 主配置文件
├── .env                 # 环境变量
├── workspace/           # 工作空间
│   ├── skills/          # 技能目录
│   ├── memory/          # 记忆文件
│   └── IDENTITY.md     # 系统提示
└── logs/               # 日志目录
```

### 环境变量配置

```bash
# .env 文件
# LLM 提供商
ANTHROPIC_API_KEY=sk-ant-xxx
OPENAI_API_KEY=sk-xxx
OPENROUTER_API_KEY=sk-or-xxx

# 渠道
TELEGRAM_BOT_TOKEN=123456:ABC-DEF
DISCORD_BOT_TOKEN=MTIzNDU2...
WHATSAPP_API_KEY=xxx

# 数据库（可选）
DATABASE_URL=postgresql://user:pass@localhost:5432/nanobot

# Redis（可选）
REDIS_URL=redis://localhost:6379

# 日志
LOG_LEVEL=INFO
LOG_FILE=~/.nanobot/logs/nanobot.log
```

### 多环境配置

```bash
# 开发环境
export NANOBOT_ENV=development
export ANTHROPIC_API_KEY=dev-key

# 生产环境
export NANOBOT_ENV=production
export ANTHROPIC_API_KEY=prod-key

# 测试环境
export NANOBOT_ENV=test
export ANTHROPIC_API_KEY=test-key
```

---

## 验证安装

### 基础验证

```bash
# 1. 检查版本
nanobot --version

# 2. 检查安装路径
which nanobot

# 3. 检查依赖
pip check
```

### 功能测试

```bash
# 1. 测试配置
nanobot doctor

# 2. 测试连接
nanobot test connection

# 3. 测试模型
nanobot test model

# 4. 测试渠道
nanobot test channel telegram
```

### 完整测试

```bash
# 1. 启动服务
nanobot gateway

# 2. 测试命令行
nanobot agent "你好"

# 3. 测试 API
curl http://localhost:8080/health

# 4. 查看日志
tail -f ~/.nanobot/logs/nanobot.log
```

---

## 故障排除

### 常见问题

#### 1. 安装失败

**问题**：`pip install nanobot-ai` 失败

**解决方案**：
```bash
# 更新 pip
pip install --upgrade pip

# 清理缓存
pip cache purge

# 使用国内镜像
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple nanobot-ai
```

#### 2. Python 版本不兼容

**问题**：`requires Python >= 3.11`

**解决方案**：
```bash
# 升级 Python
# macOS
brew install python@3.11

# Ubuntu
sudo apt update
sudo apt install python3.11 python3.11-venv

# Windows
# 从 python.org 下载 Python 3.11
```

#### 3. 权限问题

**问题**：权限被拒绝

**解决方案**：
```bash
# 使用虚拟环境
python -m venv venv
source venv/bin/activate

# 或使用用户安装
pip install --user nanobot-ai

# 或使用 sudo（不推荐）
sudo pip install nanobot-ai
```

#### 4. 依赖冲突

**问题**：依赖包版本冲突

**解决方案**：
```bash
# 创建新的虚拟环境
python -m venv new-env
source new-env/bin/activate
pip install nanobot-ai

# 使用 pip-tools
pip install pip-tools
pip-compile requirements.in
```

### 调试模式

```bash
# 启用调试模式
NANOBOT_DEBUG=1 nanobot gateway

# 详细日志
nanobot gateway --log-level DEBUG

# 开发模式
nanobot dev --verbose
```

### 日志分析

```bash
# 查看日志
tail -f ~/.nanobot/logs/nanobot.log

# 搜索错误
grep ERROR ~/.nanobot/logs/nanobot.log

# 查看特定组件日志
grep -i "telegram" ~/.nanobot/logs/nanobot.log
```

---

## 升级指南

### 版本升级

```bash
# 1. 备份当前配置
cp -r ~/.nanobot ~/.nanobot.backup

# 2. 备份数据
cp -r ~/.nanobot/workspace ~/.nanobot.workspace.backup

# 3. 停止服务
nanobot stop

# 4. 升级 nanobot
pip install --upgrade nanobot-ai

# 5. 运行升级脚本
nanobot upgrade

# 6. 重启服务
nanobot gateway
```

### 配置迁移

```yaml
# config.yaml 迁移
# 旧版本格式
# old_config:
#   setting1: value1

# 新版本格式
agents:
  default_model: claude-3-5-sonnet-20241022
  memory_window: 50

# 使用迁移工具
nanobot migrate --from-version 0.1.3
```

### 数据迁移

```bash
# 1. 备份数据
tar -czf nanobot-data-backup.tar.gz ~/.nanobot/

# 2. 恢复数据
tar -xzf nanobot-data-backup.tar.gz

# 3. 验证数据
nanobot verify-data
```

---

## 总结

通过本指南，你已经学会了如何安装和部署 Nanobot：

### 关键要点

1. **选择安装方式**：pip、源码或 Docker
2. **环境配置**：设置配置文件和环境变量
3. **验证安装**：运行测试确保一切正常
4. **故障排除**：解决常见安装问题
5. **升级维护**：保持系统最新状态

### 下一步

- 阅读 [快速入门](./QUICK_START.md) 开始使用
- 查看 [配置参考](./CONFIG_REFERENCE.md) 了解详细配置
- 探索 [架构指南](./NANOBOT_ARCHITECTURE_GUIDE.md) 了解系统设计

---

**文档版本**: 1.0.0
**最后更新**: 2026-03-03