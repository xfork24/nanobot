# 最佳实践指南

> **生产环境最佳实践** - 确保 Nanobot 稳定运行

---

## 📑 目录

1. [架构设计](#架构设计)
2. [配置管理](#配置管理)
3. [安全实践](#安全实践)
4. [性能优化](#性能优化)
5. [监控运维](#监控运维)
6. [代码质量](#代码质量)
7. [团队协作](#团队协作)
8. [故障恢复](#故障恢复)

---

## 架构设计

### 1. 微服务架构

```
# 推荐架构
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Gateway       │    │   Agent Pool    │    │   Storage       │
│   (Load Balancer)│    │   (Horizontal)  │    │   (Redis/S3)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Channels      │    │   APIs          │    │   Monitoring    │
│   (Telegram)     │    │   (REST/GraphQL)│    │   (Prometheus)  │
│   (Discord)      │    │                 │    │   (Grafana)     │
│   (Custom)       │    │                 │    │   (Alertmanager)│
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

**最佳实践**:
- 使用负载均衡器分发请求
- Agent 池支持水平扩展
- 独立的存储层
- 完善的监控体系

### 2. 容器化部署

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Load Balancer
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - nanobot-1
      - nanobot-2

  # Nanobot Services
  nanobot-1:
    image: nanobotai/nanobot:latest
    environment:
      - NANOBOT_ENV=production
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    volumes:
      - nanobot_data_1:/app/data
      - ./config:/app/config
    depends_on:
      - redis
      - postgres

  nanobot-2:
    image: nanobotai/nanobot:latest
    environment:
      - NANOBOT_ENV=production
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    volumes:
      - nanobot_data_2:/app/data
      - ./config:/app/config
    depends_on:
      - redis
      - postgres

  # Storage
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: nanobot
      POSTGRES_USER: nanobot
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  nanobot_data_1:
  nanobot_data_2:
  redis_data:
  postgres_data:
```

### 3. 多环境管理

```bash
# environments/
├── development/
│   ├── docker-compose.yml
│   ├── config.yaml
│   └── .env
├── staging/
│   ├── docker-compose.yml
│   ├── config.yaml
│   └── .env
└── production/
    ├── docker-compose.yml
    ├── config.yaml
    └── .env

# Makefile
.PHONY: deploy-dev deploy-staging deploy-prod

deploy-dev:
    cp environments/development/docker-compose.yml .
    cp environments/development/.env .
    docker-compose -f docker-compose.yml up -d

deploy-staging:
    cp environments/staging/docker-compose.yml .
    cp environments/staging/.env .
    docker-compose -f docker-compose.yml up -d

deploy-prod:
    cp environments/production/docker-compose.yml .
    cp environments/production/.env .
    docker-compose -f docker-compose.yml up -d
```

### 4. 数据分层设计

```yaml
# 数据架构配置
data_architecture:
  # 会话层
  sessions:
    strategy: "database"
    cleanup: "daily"
    retention: "30d"

  # 记忆层
  memory:
    strategy: "hybrid"
    short_term: "redis"
    long_term: "postgres"

    # 记忆类型分类
    types:
      facts:
        storage: "postgres"
        index: true
        ttl: "365d"
      preferences:
        storage: "redis"
        ttl: "30d"
      conversations:
        storage: "postgres"
        compression: true

  # 缓存层
  cache:
    level1: "redis"    # 快速缓存
    level2: "memory"    # 进程缓存
    strategy: "write-through"
```

---

## 配置管理

### 1. 配置分层

```yaml
# 环境特定配置
# config/environments/production.yaml
version: "1.0.0"
environment: "production"

# 继承基础配置
extends: "base.yaml"

# 生产特定配置
agents:
  max_iterations: 5
  memory_window: 30
  timeout: 20

database:
  pool_size: 20
  max_overflow: 50

logging:
  level: "WARNING"
  structured: true
```

### 2. 配置验证

```python
# config/validator.py
import yaml
from typing import Dict, Any

class ConfigValidator:
    def __init__(self):
        self.required_fields = {
            'agents': ['default_model', 'default_provider'],
            'channels': ['enabled'],
            'providers': ['enabled']
        }

    def validate(self, config: Dict[str, Any]) -> bool:
        """验证配置文件"""
        try:
            # 验证必需字段
            for section, fields in self.required_fields.items():
                if section not in config:
                    raise ValueError(f"Missing section: {section}")

                for field in fields:
                    if field not in config[section]:
                        raise ValueError(f"Missing field: {section}.{field}")

            # 验证配置类型
            self._validate_types(config)

            return True

        except Exception as e:
            print(f"Config validation failed: {e}")
            return False

    def _validate_types(self, config: Dict[str, Any]) -> None:
        """验证数据类型"""
        # 验证数值范围
        if 'agents' in config:
            agents = config['agents']
            if agents.get('max_iterations', 0) > 20:
                raise ValueError("max_iterations must be <= 20")
            if agents.get('memory_window', 0) > 100:
                raise ValueError("memory_window must be <= 100")
```

### 3. 敏感信息管理

```bash
# 使用 HashiCorp Vault
vault write secret/data/nanobot/config \
    ANTHROPIC_API_KEY=sk-ant-xxx \
    OPENAI_API_KEY=sk-xxx \
    TELEGRAM_BOT_TOKEN=123456:ABC-DEF

# 在配置中引用
providers:
  anthropic:
    api_key: "{{env 'ANTHROPIC_API_KEY'}}"
```

### 4. 配置同步

```python
# config/sync.py
import json
import requests
from pathlib import Path

class ConfigSync:
    def __init__(self, sync_url: str):
        self.sync_url = sync_url
        self.local_config_path = Path("~/.nanobot/config.yaml")

    def sync(self) -> bool:
        """同步配置"""
        try:
            # 获取远程配置
            response = requests.get(f"{self.sync_url}/config")
            remote_config = response.json()

            # 备份本地配置
            self._backup_local_config()

            # 合并配置
            merged_config = self._merge_configs(remote_config)

            # 保存合并后的配置
            with open(self.local_config_path, 'w') as f:
                json.dump(merged_config, f, indent=2)

            return True

        except Exception as e:
            print(f"Config sync failed: {e}")
            return False

    def _backup_local_config(self) -> None:
        """备份本地配置"""
        import shutil
        from datetime import datetime

        backup_path = Path(
            f"~/.nanobot/config_backup_{datetime.now().strftime('%Y%m%d_%H%M%S')}.yaml"
        )
        shutil.copy2(self.local_config_path, backup_path)
```

---

## 安全实践

### 1. 网络安全

```yaml
# 安全配置示例
security:
  # 网络层
  network:
    # 启用 HTTPS
    ssl:
      enabled: true
      cert_file: "/etc/ssl/certs/nanobot.crt"
      key_file: "/etc/ssl/private/nanobot.key"

    # 限制访问
    firewalls:
      - name: "main"
        rules:
          - ip: "10.0.0.0/8"  # 内网允许
          - ip: "172.16.0.0/12"
          - ip: "192.168.0.0/16"
          - deny: "0.0.0.0/0"  # 其他拒绝

    # 端口安全
    ports:
      http:
        allowed: false
      https:
        allowed: true
        rate_limit: 100

  # API 安全
  api:
    # API 密钥认证
    authentication:
      type: "bearer_token"
      header: "X-API-Key"
      validation:
        issuer: "nanobot"
        audience: "api"
        algorithm: "HS256"

    # 速率限制
    rate_limits:
      - endpoint: "*"
        limit: 1000
        period: "1h"
      - endpoint: "/chat"
        limit: 100
        period: "1m"
```

### 2. 数据加密

```python
# encryption/manager.py
from cryptography.fernet import Fernet
from typing import Dict, Any

class EncryptionManager:
    def __init__(self, key: bytes):
        self.fernet = Fernet(key)
        self.key = key

    def encrypt(self, data: str) -> str:
        """加密数据"""
        return self.fernet.encrypt(data.encode()).decode()

    def decrypt(self, encrypted_data: str) -> str:
        """解密数据"""
        return self.fernet.decrypt(encrypted_data.encode()).decode()

    def encrypt_config(self, config: Dict[str, Any]) -> Dict[str, Any]:
        """加密敏感配置"""
        sensitive_fields = ['api_key', 'token', 'password']

        encrypted_config = config.copy()

        for field in sensitive_fields:
            if field in encrypted_config:
                encrypted_config[field] = self.encrypt(
                    str(encrypted_config[field])
                )

        return encrypted_config

    def generate_key(self) -> bytes:
        """生成加密密钥"""
        return Fernet.generate_key()
```

### 3. 权限控制

```python
# rbac/permissions.py
from enum import Enum
from typing import List, Dict, Any

class Role(Enum):
    ADMIN = "admin"
    USER = "user"
    GUEST = "guest"
    SYSTEM = "system"

class Permission(Enum):
    READ_CONFIG = "read:config"
    WRITE_CONFIG = "write:config"
    MANAGE_USERS = "manage:users"
    EXECUTE_CODE = "execute:code"
    VIEW_LOGS = "view:logs"
    SYSTEM_ADMIN = "system:admin"

class RBACManager:
    def __init__(self):
        self.role_permissions = {
            Role.ADMIN: [p.value for p in Permission],
            Role.USER: [
                Permission.READ_CONFIG.value,
                Permission.EXECUTE_CODE.value
            ],
            Role.GUEST: [
                Permission.READ_CONFIG.value
            ],
            Role.SYSTEM: [
                Permission.SYSTEM_ADMIN.value
            ]
        }

    def check_permission(self, user: Dict[str, Any], permission: str) -> bool:
        """检查用户是否有权限"""
        user_role = Role(user.get('role', 'guest'))
        return permission in self.role_permissions.get(user_role, [])
```

### 4. 审计日志

```python
# audit/logger.py
import json
from datetime import datetime
from typing import Dict, Any
from pathlib import Path

class AuditLogger:
    def __init__(self, log_dir: str):
        self.log_dir = Path(log_dir)
        self.log_dir.mkdir(parents=True, exist_ok=True)

        # 按日期分割日志文件
        self.current_date = datetime.now().strftime("%Y-%m-%d")
        self.log_file = self.log_dir / f"audit_{self.current_date}.log"

    def log(self, action: str, user: str, resource: str,
            details: Dict[str, Any] = None) -> None:
        """记录审计日志"""
        log_entry = {
            "timestamp": datetime.now().isoformat(),
            "action": action,
            "user": user,
            "resource": resource,
            "details": details or {},
            "ip": self._get_client_ip(),
            "user_agent": self._get_user_agent()
        }

        with open(self.log_file, 'a', encoding='utf-8') as f:
            f.write(json.dumps(log_entry, ensure_ascii=False) + '\n')

    def _get_client_ip(self) -> str:
        """获取客户端 IP"""
        # 实现获取客户端 IP 的逻辑
        return "127.0.0.1"

    def _get_user_agent(self) -> str:
        """获取用户代理"""
        # 实现获取用户代理的逻辑
        return "Nanobot/1.0"

    def audit_actions(self):
        """审计的动作列表"""
        return [
            "login", "logout", "config_change", "user_create",
            "user_update", "user_delete", "tool_execute",
            "file_access", "api_call", "system_restart"
        ]
```

---

## 性能优化

### 1. 缓存策略

```python
# cache/strategies.py
from abc import ABC, abstractmethod
from typing import Any, Optional
import time
import json

class CacheStrategy(ABC):
    @abstractmethod
    def get(self, key: str) -> Optional[Any]:
        pass

    @abstractmethod
    def set(self, key: str, value: Any, ttl: int = 3600) -> None:
        pass

    @abstractmethod
    def delete(self, key: str) -> None:
        pass

class RedisCache(CacheStrategy):
    def __init__(self, url: str):
        self.redis = redis.from_url(url)

    def get(self, key: str) -> Optional[Any]:
        try:
            value = self.redis.get(key)
            if value:
                return json.loads(value)
            return None
        except Exception:
            return None

    def set(self, key: str, value: Any, ttl: int = 3600) -> None:
        try:
            self.redis.setex(
                key,
                ttl,
                json.dumps(value, ensure_ascii=False)
            )
        except Exception as e:
            print(f"Cache set failed: {e}")

class MemoryCache(CacheStrategy):
    def __init__(self, max_size: int = 1000):
        self.cache = {}
        self.max_size = max_size
        self.expiry = {}

    def get(self, key: str) -> Optional[Any]:
        # 检查过期
        if key in self.expiry and time.time() > self.expiry[key]:
            self.delete(key)
            return None

        return self.cache.get(key)

    def set(self, key: str, value: Any, ttl: int = 3600) -> None:
        # LRU 缓存
        if len(self.cache) >= self.max_size:
            # 删除最旧的键
            oldest_key = next(iter(self.cache))
            del self.cache[oldest_key]
            if oldest_key in self.expiry:
                del self.expiry[oldest_key]

        self.cache[key] = value
        self.expiry[key] = time.time() + ttl

class MultiLevelCache:
    def __init__(self, primary: CacheStrategy, secondary: CacheStrategy):
        self.primary = primary
        self.secondary = secondary

    def get(self, key: str) -> Optional[Any]:
        # 先查 L1
        value = self.primary.get(key)
        if value is not None:
            return value

        # 再查 L2
        value = self.secondary.get(key)
        if value is not None:
            # 回填 L1
            self.primary.set(key, value)
        return value

    def set(self, key: str, value: Any, ttl: int = 3600) -> None:
        # 同时设置 L1 和 L2
        self.primary.set(key, value, ttl)
        self.secondary.set(key, value, ttl)
```

### 2. 数据库优化

```sql
-- 数据库索引优化
CREATE INDEX idx_sessions_chat_id ON sessions(chat_id);
CREATE INDEX idx_sessions_created_at ON sessions(created_at);
CREATE INDEX idx_messages_chat_id ON messages(chat_id);
CREATE INDEX idx_messages_timestamp ON messages(timestamp);

-- 分区表
CREATE TABLE messages (
    id BIGSERIAL PRIMARY KEY,
    chat_id BIGINT NOT NULL,
    content TEXT,
    timestamp TIMESTAMP NOT NULL,
    -- 其他字段
) PARTITION BY RANGE (timestamp);

-- 创建分区
CREATE TABLE messages_2024_q1 PARTITION OF messages
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

-- 查询优化
EXPLAIN ANALYZE
SELECT * FROM messages
WHERE chat_id = 123456789
ORDER BY timestamp DESC
LIMIT 50;
```

### 3. 并发控制

```python
# concurrency/manager.py
import asyncio
from typing import Dict, List, Callable
from dataclasses import dataclass
from enum import Enum

class TaskPriority(Enum):
    LOW = 1
    NORMAL = 2
    HIGH = 3
    CRITICAL = 4

@dataclass
class QueuedTask:
    priority: TaskPriority
    task: Callable
    args: tuple
    kwargs: dict
    future: asyncio.Future

class ConcurrencyManager:
    def __init__(self, max_workers: int = 10):
        self.max_workers = max_workers
        self.workers = []
        self.task_queue = asyncio.Queue()
        self.current_tasks: Dict[str, asyncio.Task] = {}
        self.semaphore = asyncio.Semaphore(max_workers)

    async def submit_task(self,
                         task: Callable,
                         priority: TaskPriority = TaskPriority.NORMAL,
                         task_id: str = None,
                         *args, **kwargs) -> asyncio.Future:
        """提交任务"""
        if task_id is None:
            task_id = f"{id(task)}_{asyncio.get_event_loop().time()}"

        future = asyncio.get_event_loop().create_future()
        queued_task = QueuedTask(priority, task, args, kwargs, future)

        # 优先级队列排序
        await self._insert_priority_task(queued_task)

        # 启动工作者
        if len(self.workers) < self.max_workers:
            self._start_worker()

        return future

    async def _insert_priority_task(self, task: QueuedTask) -> None:
        """插入优先级任务"""
        # 将任务插入队列按优先级排序
        items = list(self.task_queue._queue)
        items.append(task)
        items.sort(key=lambda x: x.priority.value, reverse=True)
        self.task_queue._queue = items

    def _start_worker(self) -> None:
        """启动工作线程"""
        worker = asyncio.create_task(self._worker())
        self.workers.append(worker)

    async def _worker(self) -> None:
        """工作者循环"""
        while True:
            try:
                task = await self.task_queue.get()
                async with self.semaphore:
                    try:
                        result = await task.task(*task.args, **task.kwargs)
                        task.future.set_result(result)
                    except Exception as e:
                        task.future.set_exception(e)
                    finally:
                        self.task_queue.task_done()
            except asyncio.CancelledError:
                break
```

### 4. 资源限制

```python
# resource/limiter.py
import asyncio
import time
from typing import Dict, Optional
from dataclasses import dataclass

@dataclass
class ResourceUsage:
    cpu_percent: float
    memory_percent: float
    disk_percent: float
    network_in: int
    network_out: int

class ResourceLimiter:
    def __init__(self):
        self.limits = {
            'cpu_percent': 80.0,
            'memory_percent': 90.0,
            'disk_percent': 95.0,
            'network_in_mb': 100,
            'network_out_mb': 100
        }
        self.usage_history = []
        self.alert_callback = None

    def set_limits(self, limits: Dict[str, float]) -> None:
        """设置资源限制"""
        self.limits.update(limits)

    async def check_usage(self) -> bool:
        """检查资源使用情况"""
        usage = await self._get_resource_usage()
        self.usage_history.append(usage)

        # 保留最近 100 条记录
        if len(self.usage_history) > 100:
            self.usage_history.pop(0)

        # 检查是否超出限制
        alerts = []
        if usage.cpu_percent > self.limits['cpu_percent']:
            alerts.append(f"CPU 使用率过高: {usage.cpu_percent}%")
        if usage.memory_percent > self.limits['memory_percent']:
            alerts.append(f"内存使用率过高: {usage.memory_percent}%")
        if usage.disk_percent > self.limits['disk_percent']:
            alerts.append(f"磁盘使用率过高: {usage.disk_percent}%")

        if alerts and self.alert_callback:
            await self.alert_callback(alerts)

        return len(alerts) == 0

    async def _get_resource_usage(self) -> ResourceUsage:
        """获取资源使用情况"""
        import psutil

        cpu_percent = psutil.cpu_percent()
        memory = psutil.virtual_memory()
        disk = psutil.disk_usage('/')
        network = psutil.net_io_counters()

        return ResourceUsage(
            cpu_percent=cpu_percent,
            memory_percent=memory.percent,
            disk_percent=disk.percent,
            network_in=network.bytes_recv,
            network_out=network.bytes_sent
        )

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5,
                 recovery_timeout: int = 60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'closed'  # closed, open, half_open

    async def call(self, func: Callable, *args, **kwargs):
        """调用函数，如果断路器打开则抛出异常"""
        if self.state == 'open':
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = 'half_open'
            else:
                raise Exception("Circuit breaker is open")

        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e

    def _on_success(self) -> None:
        """成功回调"""
        self.failure_count = 0
        self.state = 'closed'

    def _on_failure(self) -> None:
        """失败回调"""
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.failure_threshold:
            self.state = 'open'
```

---

## 监控运维

### 1. 指标收集

```python
# monitoring/metrics.py
from prometheus_client import Counter, Gauge, Histogram, generate_latest
from prometheus_client.core import CollectorRegistry
import time

class MetricsCollector:
    def __init__(self):
        self.registry = CollectorRegistry()

        # 计数器
        self.requests_total = Counter(
            'nanobot_requests_total',
            'Total number of requests',
            ['method', 'endpoint', 'status'],
            registry=self.registry
        )

        # 直方图
        self.request_duration = Histogram(
            'nanobot_request_duration_seconds',
            'Request duration in seconds',
            ['method', 'endpoint'],
            registry=self.registry
        )

        # 仪表盘
        self.active_sessions = Gauge(
            'nanobot_active_sessions',
            'Number of active sessions',
            registry=self.registry
        )

        self.memory_usage = Gauge(
            'nanobot_memory_usage_bytes',
            'Memory usage in bytes',
            registry=self.registry
        )

    def record_request(self, method: str, endpoint: str,
                      status: int, duration: float) -> None:
        """记录请求指标"""
        self.requests_total.labels(method, endpoint, str(status)).inc()
        self.request_duration.labels(method, endpoint).observe(duration)

    def update_active_sessions(self, count: int) -> None:
        """更新活跃会话数"""
        self.active_sessions.set(count)

    def update_memory_usage(self, usage: float) -> None:
        """更新内存使用"""
        self.memory_usage.set(usage)
```

### 2. 告警规则

```yaml
# monitoring/alerts.yaml
groups:
  - name: nanobot.alerts
    rules:
      # 高错误率告警
      - alert: HighErrorRate
        expr: rate(nanobot_requests_total{status=~"5.."}[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "高错误率在 {{ $labels.endpoint }}"
          description: "错误率: {{ $value }}"

      # 响应时间过长
      - alert: SlowResponse
        expr: histogram_quantile(0.95,
              rate(nanobot_request_duration_seconds_bucket[5m])) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "响应时间过长: {{ $labels.endpoint }}"
          description: "95分位响应时间: {{ $value }}s"

      # 内存使用过高
      - alert: HighMemoryUsage
        expr: nanobot_memory_usage_bytes / 1024 / 1024 > 512
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "内存使用过高"
          description: "内存使用: {{ $value }}MB"

      # 活跃会话数过多
      - alert: TooManySessions
        expr: nanobot_active_sessions > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "活跃会话数过多"
          description: "当前会话数: {{ $value }}"
```

### 3. 日志聚合

```python
# logging/aggregator.py
import logging
import json
from typing import Dict, Any
from datetime import datetime

class StructuredLogger:
    def __init__(self, service_name: str):
        self.service_name = service_name
        self.logger = logging.getLogger(service_name)
        self.logger.setLevel(logging.INFO)

        # 创建 JSON 格式化器
        json_formatter = JSONFormatter()
        handler = logging.StreamHandler()
        handler.setFormatter(json_formatter)
        self.logger.addHandler(handler)

    def info(self, message: str, **kwargs) -> None:
        """记录信息日志"""
        self._log(logging.INFO, message, **kwargs)

    def error(self, message: str, **kwargs) -> None:
        """记录错误日志"""
        self._log(logging.ERROR, message, **kwargs)

    def _log(self, level: int, message: str, **kwargs) -> None:
        """记录日志"""
        log_data = {
            'timestamp': datetime.now().isoformat(),
            'service': self.service_name,
            'message': message,
            **kwargs
        }

        extra = {'extra_fields': json.dumps(log_data)}
        self.logger.log(level, message, **extra)

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            'timestamp': datetime.fromtimestamp(record.created).isoformat(),
            'level': record.levelname,
            'service': 'nanobot',
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno
        }

        if hasattr(record, 'extra_fields'):
            extra_fields = json.loads(record.extra_fields)
            log_entry.update(extra_fields)

        return json.dumps(log_entry, ensure_ascii=False)
```

### 4. 健康检查

```python
# health/checker.py
from abc import ABC, abstractmethod
from typing import Dict, Any, List
import asyncio

class HealthCheck(ABC):
    @abstractmethod
    async def check(self) -> Dict[str, Any]:
        pass

class DatabaseHealthCheck(HealthCheck):
    def __init__(self, db_url: str):
        self.db_url = db_url

    async def check(self) -> Dict[str, Any]:
        try:
            # 模拟数据库检查
            await asyncio.sleep(0.1)  # 模拟查询延迟
            return {
                'status': 'healthy',
                'response_time': 0.1,
                'database': 'connected'
            }
        except Exception as e:
            return {
                'status': 'unhealthy',
                'error': str(e)
            }

class MemoryHealthCheck(HealthCheck):
    async def check(self) -> Dict[str, Any]:
        import psutil

        memory = psutil.virtual_memory()
        if memory.percent > 90:
            return {
                'status': 'warning',
                'memory_percent': memory.percent
            }
        return {
            'status': 'healthy',
            'memory_percent': memory.percent
        }

class HealthChecker:
    def __init__(self):
        self.checks: List[HealthCheck] = []

    def add_check(self, check: HealthCheck) -> None:
        """添加健康检查"""
        self.checks.append(check)

    async def run_checks(self) -> Dict[str, Any]:
        """运行所有健康检查"""
        results = {}

        # 并行运行检查
        tasks = [check.check() for check in self.checks]
        results_list = await asyncio.gather(*tasks)

        # 合并结果
        for check, result in zip(self.checks, results_list):
            results[check.__class__.__name__] = result

        # 确定整体状态
        overall_status = 'healthy'
        for result in results_list:
            if result.get('status') == 'unhealthy':
                overall_status = 'unhealthy'
                break
            elif result.get('status') == 'warning':
                overall_status = 'warning'

        return {
            'status': overall_status,
            'checks': results,
            'timestamp': datetime.now().isoformat()
        }
```

---

## 代码质量

### 1. 代码规范

```python
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black
        language_version: python3

  - repo: https://github.com/charliermarsh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.0.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]

  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort
```

### 2. 单元测试

```python
# tests/test_agent.py
import pytest
from unittest.mock import Mock, AsyncMock
from nanobot.agent.loop import AgentLoop

@pytest.mark.asyncio
async def test_agent_response():
    """测试 Agent 响应"""
    # 创建模拟的依赖项
    mock_bus = Mock()
    mock_provider = Mock()
    mock_context = Mock()
    mock_tools = Mock()
    mock_sessions = Mock()
    mock_memory = Mock()
    mock_skills = Mock()

    # 配置模拟
    mock_provider.chat.return_value = AsyncMock()
    mock_provider.chat.return_value.content = "测试响应"

    # 创建 Agent
    agent = AgentLoop(
        bus=mock_bus,
        provider=mock_provider,
        model="test-model",
        context=mock_context,
        tools=mock_tools,
        sessions=mock_sessions,
        memory=mock_memory,
        skills=mock_skills
    )

    # 测试消息处理
    response = await agent._process_message({
        'channel': 'test',
        'sender_id': 'user123',
        'chat_id': 'test',
        'content': '你好'
    })

    # 验证结果
    assert response is not None
    assert "测试响应" in response.content

@pytest.mark.asyncio
async def test_agent_error_handling():
    """测试错误处理"""
    mock_provider = Mock()
    mock_provider.chat.side_effect = Exception("API Error")

    agent = AgentLoop(...)

    with pytest.raises(Exception):
        await agent._process_message({
            'channel': 'test',
            'sender_id': 'user123',
            'chat_id': 'test',
            'content': '你好'
        })
```

### 3. 性能测试

```python
# tests/test_performance.py
import pytest
import asyncio
from nanobot.agent.loop import AgentLoop

@pytest.mark.asyncio
async def test_concurrent_requests():
    """测试并发请求处理"""
    # 创建 Agent
    agent = create_test_agent()

    # 模拟 100 个并发请求
    tasks = []
    for i in range(100):
        task = asyncio.create_task(
            agent._process_message({
                'channel': 'test',
                'sender_id': f'user{i}',
                'chat_id': 'test',
                'content': f'消息 {i}'
            })
        )
        tasks.append(task)

    # 等待所有任务完成
    results = await asyncio.gather(*tasks)

    # 验证结果
    assert len(results) == 100
    assert all(result is not None for result in results)

@pytest.mark.asyncio
async def test_memory_usage():
    """测试内存使用"""
    import tracemalloc

    # 开始内存跟踪
    tracemalloc.start()

    # 创建 Agent 并处理大量消息
    agent = create_test_agent()

    for i in range(1000):
        await agent._process_message({
            'channel': 'test',
            'sender_id': 'user123',
            'chat_id': 'test',
            'content': f'消息 {i}'
        })

    # 检查内存使用
    snapshot = tracemalloc.take_snapshot()
    top_stats = snapshot.statistics('lineno')

    # 找出内存使用最多的行
    total = sum(stat.size for stat in top_stats)
    print(f"Total memory usage: {total / 1024:.2f} KB")

    tracemalloc.stop()
```

### 4. API 测试

```python
# tests/test_api.py
import pytest
from fastapi.testclient import TestClient
from nanobot.api.app import app

client = TestClient(app)

def test_health_endpoint():
    """测试健康检查端点"""
    response = client.get("/health")
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "healthy"

@pytest.mark.asyncio
async def test_chat_endpoint():
    """测试聊天端点"""
    response = client.post("/chat", json={
        "message": "你好",
        "session_id": "test-session"
    })
    assert response.status_code == 200
    data = response.json()
    assert "response" in data
```

---

## 团队协作

### 1. Git 工作流

```bash
# .git/hooks/pre-commit
#!/bin/bash
# 运行测试和代码检查
npm test
npm run lint

# 检查配置
if ! nanobot validate-config; then
    echo "配置验证失败"
    exit 1
fi
```

### 2. 代码审查清单

```markdown
## 代码审查清单

### 功能性
- [ ] 代码实现了所有需求
- [ ] 有足够的错误处理
- [ ] 边界条件已处理
- [ ] 测试覆盖完整

### 性能
- [ ] 算法复杂度合理
- [ ] 资源使用效率高
- [ ] 缓存策略正确
- [ ] 并发处理安全

### 安全性
- [ ] 输入验证充分
- [ ] 敏感信息已加密
- [ ] 权限检查正确
- [ ] 没有注入风险

### 可维护性
- [ ] 代码结构清晰
- [ ] 注释完整准确
- [ ] 命名规范合理
- [ ] 遵循项目规范

### 文档
- [ ] API 文档更新
- [ ] 配置说明完整
- [ ] 部署文档更新
- [ ] 变更日志记录
```

### 3. 发布流程

```bash
# release.sh
#!/bin/bash
# 检查代码质量
if ! npm run lint; then
    echo "代码检查失败"
    exit 1
fi

# 运行测试
if ! npm test; then
    echo "测试失败"
    exit 1
fi

# 构建项目
npm run build

# 更新版本
npm version patch

# 创建标签
git push origin main --tags

# 部署到生产环境
ansible-playbook deploy-prod.yml
```

### 4. 知识管理

```markdown
# 知识库结构

## 项目文档
- 架构设计
- API 文档
- 部署指南
- 故障排除

## 开发指南
- 代码规范
- 测试规范
- 发布流程
- 最佳实践

## 运维文档
- 监控指标
- 告警规则
- 应急响应
- 备份策略

## 历史记录
- 变更日志
- 重大事故
- 经验总结
```

---

## 故障恢复

### 1. 故障响应

```python
# incident/response.py
from enum import Enum
from typing import Dict, List
from datetime import datetime

class IncidentSeverity(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CRITICAL = 4

class Incident:
    def __init__(self,
                 title: str,
                 severity: IncidentSeverity,
                 description: str):
        self.id = f"INC-{datetime.now().strftime('%Y%m%d')}-{int(datetime.now().timestamp())}"
        self.title = title
        self.severity = severity
        self.description = description
        self.status = "open"
        self.created_at = datetime.now()
        self.updated_at = datetime.now()
        self.resolution_time = None
        self.affected_users = []
        self.impact = "unknown"
        self.root_cause = None
        self.resolution_steps = []

    def add_impact(self, impact: str) -> None:
        """添加影响范围"""
        self.impact = impact

    def add_root_cause(self, cause: str) -> None:
        """添加根本原因"""
        self.root_cause = cause

    def add_resolution_step(self, step: str) -> None:
        """添加解决步骤"""
        self.resolution_steps.append(step)

    def resolve(self) -> None:
        """解决事件"""
        self.status = "resolved"
        self.resolution_time = datetime.now()
        self.updated_at = datetime.now()

class IncidentManager:
    def __init__(self):
        self.incidents: Dict[str, Incident] = {}
        self.notification_channels = []

    def create_incident(self, title: str, severity: IncidentSeverity,
                       description: str) -> Incident:
        """创建事件"""
        incident = Incident(title, severity, description)
        self.incidents[incident.id] = incident

        # 发送通知
        self._send_notification(
            f"新事件创建: {incident.title}",
            f"ID: {incident.id}\n严重性: {severity.name}\n{description}"
        )

        return incident

    def update_incident(self, incident_id: str, **kwargs) -> None:
        """更新事件"""
        if incident_id in self.incidents:
            incident = self.incidents[incident_id]
            for key, value in kwargs.items():
                if hasattr(incident, key):
                    setattr(incident, key, value)
            incident.updated_at = datetime.now()

    def resolve_incident(self, incident_id: str, resolution: str) -> None:
        """解决事件"""
        if incident_id in self.incidents:
            incident = self.incidents[incident_id]
            incident.resolve()

            # 发送通知
            self._send_notification(
                f"事件已解决: {incident.title}",
                f"ID: {incident.id}\n解决方案: {resolution}"
            )

    def _send_notification(self, title: str, message: str) -> None:
        """发送通知"""
        for channel in self.notification_channels:
            channel.send(title, message)
```

### 2. 灾难恢复

```bash
# disaster_recovery/backup.sh
#!/bin/bash
# 灾难恢复备份脚本

BACKUP_DIR="/backups/nanobot"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# 创建备份
mkdir -p $BACKUP_DIR

# 备份数据库
pg_dump nanobot > $BACKUP_DIR/db_$DATE.sql

# 备份配置
cp -r ~/.nanobot/config $BACKUP_DIR/config_$DATE/

# 备份技能
cp -r ~/.nanobot/workspace/skills $BACKUP_DIR/skills_$DATE/

# 备份记忆
cp -r ~/.nanobot/workspace/memory $BACKUP_DIR/memory_$DATE/

# 清理旧备份
find $BACKUP_DIR -name "*.sql" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "config_*" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "skills_*" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "memory_*" -mtime +$RETENTION_DAYS -delete

# 加密备份
tar -czf - $BACKUP_DIR | openssl enc -aes-256-cbc -salt -pbkdf2 \
  -out $BACKUP_DIR/backup_$DATE.tar.gz.enc

# 删除未加密的备份
rm -rf $BACKUP_DIR/db_$DATE.sql
rm -rf $BACKUP_DIR/config_$DATE/
rm -rf $BACKUP_DIR/skills_$DATE/
rm -rf $BACKUP_DIR/memory_$DATE/
```

### 3. 回滚策略

```yaml
# rollback/strategy.yaml
strategies:
  # 滚回部署
  - name: "deployment_rollback"
    description: "快速回滚到上一个稳定版本"
    steps:
      - name: "停止当前服务"
        command: "docker-compose down"
      - name: "恢复镜像"
        command: "docker pull nanobotai/nanobot:previous-version"
      - name: "启动服务"
        command: "docker-compose up -d"
      - name: "验证服务"
        command: "curl -f http://localhost:8080/health"

  # 数据库回滚
  - name: "database_rollback"
    description: "回滚数据库到指定时间点"
    steps:
      - name: "备份数据库"
        command: "pg_dump nanobot > backup_before_rollback.sql"
      - name: "执行回滚"
        command: "pg_restore -d nanobot backup_20240303.sql"
      - name: "验证数据"
        command: "nanobot verify-data"

  # 配置回滚
  - name: "config_rollback"
    description: "回滚配置文件"
    steps:
      - name: "备份配置"
        command: "cp ~/.nanobot/config.yaml ~/.nanobot/config.yaml.rollback"
      - name: "恢复配置"
        command: "cp ~/.nanobot/config.yaml.backup ~/.nanobot/config.yaml"
      - name: "重启服务"
        command: "nanobot restart"
```

---

## 总结

通过本最佳实践指南，我们涵盖了 Nanobot 在生产环境中的各个方面：

### 关键要点

1. **架构设计**: 微服务、容器化、多环境管理
2. **配置管理**: 分层配置、验证、敏感信息保护
3. **安全实践**: 网络安全、数据加密、权限控制、审计日志
4. **性能优化**: 缓存策略、数据库优化、并发控制、资源限制
5. **监控运维**: 指标收集、告警规则、日志聚合、健康检查
6. **代码质量**: 代码规范、单元测试、性能测试、API 测试
7. **团队协作**: Git 工作流、代码审查、发布流程、知识管理
8. **故障恢复**: 事件响应、灾难恢复、回滚策略

### 持续改进

- 定期 review 和更新最佳实践
- 收集团队反馈和经验教训
- 关注行业新趋势和技术
- 保持文档的更新和维护

---

**文档版本**: 1.0.0
**最后更新**: 2026-03-03