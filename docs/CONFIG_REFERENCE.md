# 配置参考

> **完整配置选项** - Nanobot 配置详解

---

## 📑 目录

1. [配置文件结构](#配置文件结构)
2. [全局配置](#全局配置)
3. [Agent 配置](#agent-配置)
4. [渠道配置](#渠道配置)
5. [提供商配置](#提供商配置)
6. [工具配置](#工具配置)
7. [技能配置](#技能配置)
8. [内存配置](#内存配置)
9. [日志配置](#日志配置)
10. [高级配置](#高级配置)

---

## 配置文件结构

### 主配置文件

```yaml
# ~/.nanobot/config.yaml
# 优先级：命令行参数 > 环境变量 > 配置文件 > 默认值

# 全局设置
global:
  data_dir: "~/.nanobot"
  workspace: "~/.nanobot/workspace"
  log_dir: "~/.nanobot/logs"

# Agent 配置
agents:
  default_model: "claude-3-5-sonnet-20241022"
  default_provider: "auto"
  max_iterations: 10
  memory_window: 50
  consolidate_threshold: 100
  temperature: 0.7
  max_tokens: 4096
  timeout: 30.0

# 渠道配置
channels:
  telegram:
    enabled: true
    token: "${TELEGRAM_BOT_TOKEN}"
    allow_from: ["*"]
    parse_mode: "Markdown"
    max_connections: 10
    timeout: 30

  discord:
    enabled: false
    token: "${DISCORD_BOT_TOKEN}"
    allow_from: []
    max_connections: 5

# 提供商配置
providers:
  default: "anthropic"

  anthropic:
    api_key: "${ANTHROPIC_API_KEY}"
    model: "claude-3-5-sonnet-20241022"
    max_tokens: 4096
    temperature: 0.7
    timeout: 30
    default: true

  openai:
    api_key: "${OPENAI_API_KEY}"
    model: "gpt-4-turbo-preview"
    max_tokens: 4096
    temperature: 0.7
    timeout: 30

# 工具配置
tools:
  exec:
    enabled: true
    working_dir: "${WORK_DIR:-.}"
    timeout: 30
    max_output: 10000
    allowed_commands:
      - "ls"
      - "cat"
      - "echo"
      - "find"

  read_file:
    enabled: true
    allowed_extensions: [".md", ".txt", ".py", ".js", ".json"]
    max_file_size: 10485760  # 10MB

  write_file:
    enabled: true
    backup_enabled: true
    backup_dir: "${BACKUP_DIR:-./backups}"

# 技能配置
skills:
  path: "~/.nanobot/workspace/skills"
  auto_load: true
  enabled_skills: []
  disabled_skills: []
  default_skills:
    - "general_assistant"
    - "code_helper"

# 内存配置
memory:
  path: "~/.nanobot/workspace/memory"
  max_memory_size: 1000000
  consolidate_threshold: 100
  auto_consolidate: true
  consolidate_interval: 3600

# 日志配置
logging:
  level: "INFO"
  format: "{time:YYYY-MM-DD HH:mm:ss} | {level} | {message}"
  file: "~/.nanobot/logs/nanobot.log"
  max_size: "10MB"
  backup_count: 5
  console: true

# 安全配置
security:
  enable_sandbox: true
  allowed_hosts: ["*"]
  blocked_commands: ["rm -rf", "format", "mkfs"]
  max_request_size: 10485760
  rate_limit: 100

# 数据库配置
database:
  url: "${DATABASE_URL:-sqlite:///~/.nanobot/nanobot.db}"
  echo: false
  pool_size: 5
  max_overflow: 10

# Redis 配置
redis:
  url: "${REDIS_URL:-redis://localhost:6379}"
  db: 0
  decode_responses: true
  max_connections: 10
```

### 配置文件位置

| 位置 | 优先级 | 用途 |
|------|--------|------|
| 命令行参数 | 最高 | 运行时覆盖 |
| 环境变量 | 高 | 系统级配置 |
| 用户配置 | 中 | `~/.nanobot/config.yaml` |
| 全局配置 | 低 | `/etc/nanobot/config.yaml` |
| 默认配置 | 最低 | 内置默认值 |

---

## 全局配置

### global 设置

```yaml
global:
  # 数据目录
  data_dir: "~/.nanobot"                    # 主数据目录

  # 工作空间
  workspace: "~/.nanobot/workspace"          # 工作空间目录

  # 日志目录
  log_dir: "~/.nanobot/logs"                # 日志文件目录

  # 临时目录
  temp_dir: "/tmp/nanobot"                  # 临时文件目录

  # 时区设置
  timezone: "Asia/Shanghai"                 # 时区

  # 语言设置
  language: "zh-CN"                         # 默认语言

  # 主题设置
  theme: "dark"                             # UI 主题

  # 离线模式
  offline_mode: false                       # 是否离线模式
```

### 配置变量

```yaml
# 支持的变量
global:
  # 系统变量
  home: "~"                                 # 用户主目录
  user: "${USER}"                          # 当前用户
  hostname: "${HOSTNAME}"                   # 主机名

  # 自定义变量
  project_dir: "${PROJECT_DIR:-.}"          # 项目目录
  backup_dir: "${BACKUP_DIR:-./backups}"    # 备份目录

  # 路径模板
  templates:
    skill: "{workspace}/skills/{name}"      # 技能路径模板
    memory: "{workspace}/memory/{type}"    # 记忆路径模板
    log: "{log_dir}/{name}.log"            # 日志路径模板
```

---

## Agent 配置

### 基础配置

```yaml
agents:
  # 默认模型
  default_model: "claude-3-5-sonnet-20241022"     # 默认使用的模型

  # 提供商设置
  default_provider: "auto"                          # 默认提供商
  provider_priority: ["anthropic", "openai"]        # 提供商优先级

  # 行为控制
  max_iterations: 10                                # 最大工具调用次数
  max_consecutive_tool_calls: 5                    # 最大连续工具调用

  # 记忆设置
  memory_window: 50                                # 对话记忆窗口大小
  consolidate_threshold: 100                        # 触发记忆整合的阈值
  auto_consolidate: true                           # 自动记忆整合

  # 响应设置
  temperature: 0.7                                 # 温度参数
  max_tokens: 4096                                # 最大令牌数
  timeout: 30.0                                   # 超时时间（秒）

  # 安全设置
  enable_sandbox: true                             # 启用沙箱
  safe_mode: false                                # 安全模式
  max_response_length: 10000                       # 最大响应长度

  # 个性化设置
  personality: "friendly"                          # 个性类型
  creativity: 0.7                                 # 创造性程度
  helpfulness: 0.9                               # 有用性程度
```

### 高级配置

```yaml
agents:
  # 自定义系统提示
  system_prompt: |
    你是一个友好的 AI 助手，名为 Nanobot。
    你可以帮助用户完成各种任务。

    特点：
    - 热情友好
    - 知识渊博
    - 耐心细致

    请始终保持专业和礼貌。

  # 工具设置
  tools:
    auto_select: true                             # 自动选择工具
    required_tools: []                            # 必需的工具
    excluded_tools: ["exec"]                      # 排除的工具

  # 渠道设置
  channels:
    active: ["telegram"]                          # 活跃渠道
    default_channel: "telegram"                   # 默认渠道

  # 会话设置
  sessions:
    max_per_user: 10                              # 每个用户最大会话数
    cleanup_interval: 3600                       # 清理间隔（秒）
    expire_after: 86400                          # 会话过期时间（秒）

  # 错误处理
  error_handling:
    max_retries: 3                               # 最大重试次数
    retry_delay: 1.0                             # 重试延迟
    fallback_provider: "openai"                  # 降级提供商

  # 性能优化
  performance:
    batch_size: 10                               # 批处理大小
    parallel_calls: 3                            # 并行调用数
    cache_enabled: true                          # 启用缓存
    cache_ttl: 3600                              # 缓存过期时间
```

---

## 渠道配置

### Telegram 渠道

```yaml
channels:
  telegram:
    enabled: true                                 # 启用渠道
    token: "${TELEGRAM_BOT_TOKEN}"                # Bot Token

    # 权限设置
    allow_from: ["*"]                             # 允许的用户列表
    deny_from: []                                # 禁止的用户列表
    admin_users: ["123456789"]                   # 管理员用户

    # 消息设置
    parse_mode: "Markdown"                       # 解析模式
    disable_web_page_preview: true                # 禁用网页预览

    # 限制设置
    max_connections: 10                           # 最大连接数
    rate_limit: 30                               # 速率限制（消息/分钟）
    message_timeout: 60                          # 消息超时（秒）

    # 功能设置
    commands:
      new: true                                  # /new 命令
      help: true                                 # /help 命令
      stop: true                                 # /stop 命令
      status: true                               # /status 命令

    # 媒体设置
    max_file_size: 52428800                      # 最大文件大小（50MB）
    allowed_file_types: ["photo", "video", "audio"]

    # Webhook 设置
    webhook:
      enabled: false                             # 启用 Webhook
      url: "https://your-domain.com/telegram/webhook"
      secret_token: "${TELEGRAM_WEBHOOK_SECRET}"
```

### Discord 渠道

```yaml
channels:
  discord:
    enabled: false                                # 启用渠道
    token: "${DISCORD_BOT_TOKEN}"                 # Bot Token

    # 权限设置
    allow_from: []                               # 允许的用户/角色
    allow_guilds: ["123456789"]                  # 允许的服务器
    admin_guilds: ["123456789"]                 # 管理服务器

    # 频道设置
    default_channel: "#general"                  # 默认频道
    command_prefix: "!"                          # 命令前缀

    # 连接设置
    max_connections: 5                           # 最大连接数
    timeout: 30                                  # 超时时间
    presence: "Playing Nanobot"                   # 在线状态

    # 消息设置
    typing: true                                 # 显示输入状态
    delete_commands: false                       # 删除命令消息

    # 功能设置
    intents:
      messages: true
      guilds: true
      reactions: true
    privileged_intents: false
```

### WhatsApp 渠道

```yaml
channels:
  whatsapp:
    enabled: false                                # 启用渠道
    api_key: "${WHATSAPP_API_KEY}"               # API Key
    business_phone_number_id: "${BUSINESS_PHONE}" # 商业电话号码

    # 模板设置
    templates:
      welcome: "welcome_template"                 # 欢迎模板
      fallback: "fallback_template"              # 回退模板

    # 消息设置
    message_limit: 1000                          # 消息长度限制
    retry_attempts: 3                            # 重试次数
    retry_delay: 5                               # 重试延迟

    # webhook 设置
    webhook:
      enabled: false
      url: "https://your-domain.com/whatsapp/webhook"
      verify_token: "${WHATSAPP_VERIFY_TOKEN}"
```

### 自定义渠道

```yaml
channels:
  custom_channel:
    enabled: false                               # 启用渠道
    type: "http"                                 # 渠道类型

    # API 设置
    api_url: "https://api.example.com"           # API 地址
    api_key: "${CUSTOM_API_KEY}"                 # API 密钥
    headers:
      Authorization: "Bearer {api_key}"

    # 端点设置
    endpoints:
      message: "/messages"                      # 消息端点
      webhook: "/webhook"                       # Webhook 端点

    # 消息映射
    message_mapping:
      incoming: "incoming_message"              # 接收消息字段
      outgoing: "outgoing_message"              # 发送消息字段

    # 安全设置
    validate_signature: true                     # 验证签名
    signature_header: "X-Signature"             # 签名头

    # 限制设置
    rate_limit: 10                              # 速率限制
    timeout: 30                                 # 超时时间
```

---

## 提供商配置

### Anthropic 配置

```yaml
providers:
  anthropic:
    enabled: true                               # 启用提供商
    api_key: "${ANTHROPIC_API_KEY}"              # API 密钥

    # 模型设置
    model: "claude-3-5-sonnet-20241022"         # 默认模型
    models:                                    # 可用模型
      - "claude-3-5-sonnet-20241022"
      - "claude-3-opus-20240229"
      - "claude-3-haiku-20240307"

    # 请求设置
    max_tokens: 4096                           # 最大令牌数
    temperature: 0.7                          # 温度
    top_p: 1.0                                 # Top P
    top_k: 40                                  # Top K

    # 功能设置
    enable_streaming: true                      # 启用流式响应
    enable_tools: true                         # 启用工具调用
    enable_json_mode: true                     # 启用 JSON 模式

    # 超时设置
    timeout: 30                                # 请求超时
    connect_timeout: 10                        # 连接超时
    read_timeout: 30                           # 读取超时

    # 重试设置
    max_retries: 3                             # 最大重试次数
    retry_delay: 1                             # 重试延迟
```

### OpenAI 配置

```yaml
providers:
  openai:
    enabled: false                              # 启用提供商
    api_key: "${OPENAI_API_KEY}"                # API 密钥
    organization: "${OPENAI_ORG}"               # 组织 ID（可选）

    # 模型设置
    model: "gpt-4-turbo-preview"               # 默认模型
    models:                                    # 可用模型
      - "gpt-4-turbo-preview"
      - "gpt-4"
      - "gpt-3.5-turbo"
      - "gpt-4o"

    # 请求设置
    max_tokens: 4096                           # 最大令牌数
    temperature: 0.7                          # 温度
    top_p: 1.0                                 # Top P
    frequency_penalty: 0                       # 频率惩罚
    presence_penalty: 0                        # 存在惩罚

    # 功能设置
    enable_streaming: true                      # 启用流式响应
    enable_tools: true                         # 启用工具调用
    enable_json_mode: true                     # 启用 JSON 模式
    enable_vision: true                        # 启用视觉功能

    # 超时设置
    timeout: 30                                # 请求超时
    connect_timeout: 10                        # 连接超时
    read_timeout: 30                           # 读取超时

    # 重试设置
    max_retries: 3                             # 最大重试次数
    retry_delay: 1                             # 重试延迟
```

### OpenRouter 配置

```yaml
providers:
  openrouter:
    enabled: false                             # 启用提供商
    api_key: "${OPENROUTER_API_KEY}"            # API 密钥

    # 模型设置
    model: "anthropic/claude-3.5-sonnet"       # 默认模型
    models:                                    # 可用模型
      - "anthropic/claude-3.5-sonnet"
      - "openai/gpt-4-turbo"
      - "google/gemini-pro"

    # 请求设置
    max_tokens: 4096                           # 最大令牌数
    temperature: 0.7                          # 温度
    top_p: 1.0                                 # Top P

    # 功能设置
    enable_streaming: true                     # 启用流式响应
    enable_tools: true                        # 启用工具调用

    # 超时设置
    timeout: 30                                # 请求超时

    # 重试设置
    max_retries: 3                            # 最大重试次数
    retry_delay: 1                            # 重试延迟

    # 特色功能
    features:
      search: true                             # 启用搜索
      code: true                               # 启用代码生成
      reasoning: true                          # 启用推理
```

### 自定义提供商

```yaml
providers:
  custom_provider:
    enabled: false                             # 启用提供商
    provider_type: "custom"                    # 提供商类型

    # API 设置
    api_base: "https://api.example.com/v1"     # API 基础 URL
    api_key: "${CUSTOM_API_KEY}"              # API 密钥

    # 认证设置
    auth_type: "bearer"                       # 认证类型
    auth_header: "Authorization"               # 认证头

    # 端点设置
    endpoints:
      chat: "/chat/completions"                # 聊天端点
      models: "/models"                       # 模型列表端点

    # 请求设置
    headers:                                  # 自定义请求头
      "Content-Type": "application/json"
      "X-Custom-Header": "value"

    # 超时设置
    timeout: 30                               # 请求超时
    connect_timeout: 10                        # 连接超时

    # 重试设置
    max_retries: 3                            # 最大重试次数
    retry_delay: 1                            # 重试延迟
    retry_status_codes: [408, 429, 500, 502, 503, 504]
```

---

## 工具配置

### 基础工具配置

```yaml
tools:
  # 全局工具设置
  global:
    enabled: true                             # 全局启用
    working_dir: "${WORK_DIR:-.}"             # 工作目录
    timeout: 30                               # 超时时间
    max_output: 10000                        # 最大输出大小

    # 安全设置
    allowed_commands:                         # 允许的命令
      - "ls"
      - "cat"
      - "echo"
      - "find"
      - "grep"
    blocked_commands:                        # 禁止的命令
      - "rm -rf"
      - "format"
      - "mkfs"
      - "dd"

    # 资源限制
    memory_limit: "512MB"                     # 内存限制
    cpu_limit: "50%"                          # CPU 限制
    disk_limit: "1GB"                         # 磁盘限制
```

### exec 工具配置

```yaml
tools:
  exec:
    enabled: true                             # 启用工具
    working_dir: "${WORK_DIR:-.}"             # 工作目录
    timeout: 30                               # 超时时间
    max_output: 10000                        # 最大输出

    # 安全设置
    allowed_commands:                        # 允许的命令
      - "ls"
      - "cat"
      - "echo"
      - "find"
      - "grep"
    blocked_commands:                        # 禁止的命令
      - "rm -rf"
      - "format"
      - "mkfs"

    # 环境设置
    environment:                              # 环境变量
      PATH: "/usr/local/bin:/usr/bin:/bin"
      HOME: "/root"

    # 权限设置
    run_as_user: "nobody"                     # 运行用户
    run_as_group: "nogroup"                  # 运行组

    # 日志设置
    log_commands: true                       # 记录命令
    log_output: true                         # 记录输出
    log_dir: "${LOG_DIR:-./logs}"            # 日志目录
```

### 文件工具配置

```yaml
tools:
  # 文件读取工具
  read_file:
    enabled: true                            # 启用工具
    allowed_extensions:                       # 允许的扩展名
      - ".md"
      - ".txt"
      - ".py"
      - ".js"
      - ".json"
      - ".yaml"
      - ".yml"
    max_file_size: 10485760                  # 最大文件大小（10MB）
    encoding: "utf-8"                       # 文件编码
    encoding_errors: "ignore"               # 编码错误处理

    # 安全设置
    allowed_directories:                     # 允许的目录
      - "${WORK_DIR:-.}"
      - "/tmp"
    blocked_directories:                    # 禁止的目录
      - "/etc"
      - "/var"
      - "/usr"

  # 文件写入工具
  write_file:
    enabled: true                            # 启用工具
    backup_enabled: true                     # 启用备份
    backup_dir: "${BACKUP_DIR:-./backups}"  # 备份目录
    max_file_size: 10485760                 # 最大文件大小
    create_dirs: true                        # 自动创建目录

    # 安全设置
    allowed_directories:                     # 允许的目录
      - "${WORK_DIR:-.}"
    restricted_mode: false                    # 限制模式
```

### Web 工具配置

```yaml
tools:
  # 网页搜索工具
  web_search:
    enabled: true                            # 启用工具
    search_engine: "brave"                    # 搜索引擎
    api_key: "${BRAVE_SEARCH_API_KEY}"       # API 密钥
    max_results: 10                          # 最大结果数
    timeout: 10                              # 超时时间

    # 安全设置
    allowed_domains:                         # 允许的域名
      - "wikipedia.org"
      - "stackoverflow.com"
      - "github.com"
    blocked_domains: []                      # 禁止的域名

    # 内容过滤
    safe_search: true                        # 安全搜索
    adult_content: false                     # 过滤成人内容

  # 网页抓取工具
  fetch_url:
    enabled: true                            # 启用工具
    timeout: 30                              # 超时时间
    max_redirects: 5                         # 最大重定向
    user_agent: "Nanobot/1.0"                # 用户代理

    # 请求设置
    headers:                                 # 自定义请求头
      "Accept": "text/html"
      "Accept-Language": "en-US"

    # 响应处理
    max_response_size: 10485760              # 最大响应大小
    follow_redirects: true                   # 跟随重定向
    verify_ssl: true                         # 验证 SSL
```

---

## 技能配置

### 技能管理配置

```yaml
skills:
  # 路径设置
  path: "~/.nanobot/workspace/skills"        # 技能目录
  auto_load: true                           # 自动加载技能
  watch_changes: true                       # 监听文件变化

  # 加载设置
  load_on_startup: true                      # 启动时加载
  reload_delay: 5                           # 重载延迟（秒）
  priority_skills:                          # 优先技能
    - "emergency"
    - "admin"

  # 启用/禁用
  enabled_skills: []                        # 启用的技能列表
  disabled_skills: []                       # 禁用的技能列表

  # 默认技能
  default_skills:                           # 默认技能
    - "general_assistant"
    - "code_helper"
    - "file_manager"

  # 技能分组
  groups:
    development:
      - "code_helper"
      - "debugger"
      - "tester"
    communication:
      - "translator"
      - "summarizer"
      - "writer"

  # 限制设置
  max_skills_per_session: 10                # 每会话最大技能数
  max_skill_depth: 3                        # 最大技能嵌套深度
  timeout_per_skill: 30                     # 技能执行超时
```

### 技能配置示例

```yaml
skills:
  # 代码助手技能
  python_developer:
    enabled: true
    priority: 10
    config:
      max_code_length: 2000
      include_tests: true
      lint_code: true
      format_code: true

  # 项目管理技能
  project_manager:
    enabled: true
    priority: 8
    config:
      methodologies: ["agile", "scrum", "kanban"]
      templates: ["user_story", "task", "bug"]
      track_progress: true

  # 数据分析师技能
  data_analyst:
    enabled: true
    priority: 7
    config:
      preferred_libraries: ["pandas", "numpy", "matplotlib"]
      visualization_style: "professional"
      statistical_tests: ["t-test", "chi2", "anova"]
```

---

## 内存配置

### 记忆系统配置

```yaml
memory:
  # 路径设置
  path: "~/.nanobot/workspace/memory"        # 记忆文件目录
  auto_save: true                           # 自动保存

  # 记忆文件
  memory_file: "MEMORY.md"                  # 长期记忆文件
  history_file: "HISTORY.md"                # 历史记录文件

  # 记忆设置
  max_memory_size: 1000000                  # 最大记忆大小
  max_history_size: 500000                   # 最大历史大小
  consolidate_threshold: 100                 # 整合阈值

  # 整合设置
  auto_consolidate: true                    # 自动整合
  consolidate_interval: 3600                 # 整合间隔（秒）
  consolidation_model: "claude-3-opus"      # 整合模型

  # 记忆类型
  memory_types:                             # 记忆类型
    facts:                                 # 事实记忆
      file: "facts.md"
      max_entries: 1000
    preferences:                           # 偏好记忆
      file: "preferences.md"
      max_entries: 500
    conversations:                         # 对话记忆
      file: "conversations.md"
      max_entries: 100

  # 安全设置
  encrypt_memory: false                     # 加密记忆
  backup_enabled: true                     # 启用备份
  backup_interval: 86400                   # 备份间隔（秒）
```

### 缓存配置

```yaml
memory:
  # 缓存设置
  cache:
    enabled: true                          # 启用缓存
    type: "redis"                          # 缓存类型
    url: "${REDIS_URL:-redis://localhost:6379}"
    ttl: 3600                              # 缓存过期时间
    max_size: 1000                         # 最大缓存大小

  # 查询缓存
  query_cache:
    enabled: true                          # 启用查询缓存
    max_size: 500                          # 最大查询数
    ttl: 1800                              # 查询缓存过期时间

  # 会话缓存
  session_cache:
    enabled: true                          # 启用会话缓存
    max_size: 100                          # 最大会话数
    expire_after: 3600                     # 会话过期时间
```

---

## 日志配置

### 基础日志配置

```yaml
logging:
  # 日志级别
  level: "INFO"                             # 日志级别
  root_level: "INFO"                       # 根级别

  # 日志格式
  format: "{time:YYYY-MM-DD HH:mm:ss} | {level} | {module} | {message}"
  time_format: "YYYY-MM-DD HH:mm:ss"

  # 输出设置
  outputs:                                 # 输出目标
    console:                               # 控制台
      enabled: true
      format: "{level} | {message}"
      colorize: true

    file:                                  # 文件
      enabled: true
      path: "~/.nanobot/logs/nanobot.log"
      rotation:                          # 轮转设置
        max_size: "10MB"
        backup_count: 5
        backup_format: "{time:YYYY-MM-DD}_{name}.log"
      compression: "zip"                 # 压缩格式

  # 文件日志
  files:                                  # 文件日志配置
    main:                                 # 主日志
      level: "INFO"
      path: "~/.nanobot/logs/nanobot.log"
      max_size: "50MB"
      backup_count: 10

    error:                                # 错误日志
      level: "ERROR"
      path: "~/.nanobot/logs/error.log"
      max_size: "10MB"
      backup_count: 5

    access:                               # 访问日志
      level: "INFO"
      path: "~/.nanobot/logs/access.log"
      max_size: "25MB"
      backup_count: 7

  # 过滤器
  filters:                                # 日志过滤器
    error_only:                          # 仅错误
      - "ERROR"
    performance:                          # 性能日志
      - "PERFORMANCE"
      - "TOOL_EXECUTION"
```

### 高级日志配置

```yaml
logging:
  # 结构化日志
  structured: true                        # 启用结构化日志
  json_format: true                       # JSON 格式
  include_trace_id: true                  # 包含跟踪 ID

  # 性能监控
  performance:
    enabled: true                         # 启用性能日志
    slow_query_threshold: 1000            # 慢查询阈值（毫秒）
    memory_threshold: 100                 # 内存使用阈值（MB）

  # 安全日志
  security:
    enabled: true                         # 启用安全日志
    log_auth: true                        # 记录认证
    log_permissions: true                 # 记录权限
    log_sensitive: false                 # 记录敏感信息

  # 日志聚合
  aggregation:
    enabled: false                        # 启用日志聚合
    endpoint: "http://log-aggregator:8080"
    batch_size: 100                       # 批量大小
    flush_interval: 5                    # 刷新间隔（秒）
```

---

## 高级配置

### 数据库配置

```yaml
database:
  # 主数据库
  url: "${DATABASE_URL:-sqlite:///~/.nanobot/nanobot.db}"
  echo: false                            # 显示 SQL

  # 连接池设置
  pool_size: 5                          # 连接池大小
  max_overflow: 10                       # 最大溢出连接
  pool_timeout: 30                       # 连接池超时
  pool_recycle: 3600                     # 连接回收时间

  # 引擎设置
  engine:                                # 数据库引擎
    default: "sqlite"
    fallback: "postgresql"

  # 迁移设置
  migrations:
    path: "./migrations"                 # 迁移文件目录
    table: "alembic_version"             # 迁移表名

  # 缓存设置
  cache:                                # 数据库缓存
    enabled: true                        # 启用缓存
    type: "redis"
    url: "${REDIS_URL:-redis://localhost:6379}"
    ttl: 3600                            # 缓存过期时间
```

### Redis 配置

```yaml
redis:
  # 连接设置
  url: "${REDIS_URL:-redis://localhost:6379}"
  db: 0                                 # 数据库编号
  decode_responses: true                 # 解码响应

  # 连接池设置
  max_connections: 10                    # 最大连接数
  min_connections: 1                     # 最小连接数
  retry_on_timeout: true                 # 超时重试
  socket_timeout: 5                      # Socket 超时

  # 持久化设置
  persistence:                          # 持久化模式
    enabled: true
    save_interval: 600                   # 保存间隔（秒）
    rdb_enabled: true                   # RDB 持久化
  aof_enabled: false                    # AOF 持久化

  # 内存设置
  maxmemory: "2GB"                      # 最大内存
  maxmemory_policy: "allkeys-lru"      # 内存策略

  # 高级设置
  cluster_mode: false                   # 集群模式
  sentinel_mode: false                  # Sentinel 模式
  tls_enabled: false                    # 启用 TLS
```

### 监控配置

```yaml
monitoring:
  # 健康检查
  health:
    enabled: true                       # 启用健康检查
    endpoint: "/health"
    interval: 30                        # 检查间隔（秒）
    timeout: 5                         # 检查超时

  # 指标收集
  metrics:
    enabled: true                       # 启用指标收集
    endpoint: "/metrics"
    port: 9090                         # 指标端口

  # 告警设置
  alerts:
    enabled: false                      # 启用告警
    channels:                          # 告警渠道
      - "email"
      - "slack"

    # 告警规则
    rules:
      - name: "high_cpu"
        condition: "cpu > 80%"
        duration: "5m"
        severity: "warning"

      - name: "memory_usage"
        condition: "memory > 90%"
        duration: "1m"
        severity: "critical"
```

### 安全配置

```yaml
security:
  # 认证设置
  auth:
    enabled: false                      # 启用认证
    type: "jwt"                         # 认证类型
    secret_key: "${JWT_SECRET}"
    token_ttl: 3600                     # Token 过期时间

  # 加密设置
  encryption:
    enabled: false                      # 启用加密
    algorithm: "AES-256-GCM"            # 加密算法
    key: "${ENCRYPTION_KEY}"           # 加密密钥

  # 访问控制
  access_control:
    enabled: true                       # 启用访问控制
    default_role: "user"                # 默认角色
    roles:                              # 角色定义
      admin:
        permissions: ["*"]
      user:
        permissions: ["read", "write"]
      guest:
        permissions: ["read"]

  # 安全中间件
  middleware:
    enabled: true                       # 启用中间件
    rate_limit: true                    # 速率限制
    cors: true                          # CORS 支持
    helmet: true                        # 安全头
    csrf: true                         # CSRF 保护
```

---

## 环境变量参考

### 内置环境变量

| 变量名 | 描述 | 默认值 |
|--------|------|--------|
| `NANOBOT_ENV` | 运行环境 | `production` |
| `NANOBOT_DEBUG` | 调试模式 | `false` |
| `NANOBOT_CONFIG` | 配置文件路径 | `~/.nanobot/config.yaml` |
| `NANOBOT_DATA_DIR` | 数据目录 | `~/.nanobot` |
| `NANOBOT_WORKSPACE` | 工作空间 | `~/.nanobot/workspace` |
| `NANOBOT_LOG_LEVEL` | 日志级别 | `INFO` |

### 提供商环境变量

| 变量名 | 描述 |
|--------|------|
| `ANTHROPIC_API_KEY` | Anthropic API 密钥 |
| `OPENAI_API_KEY` | OpenAI API 密钥 |
| `OPENAI_ORG` | OpenAI 组织 ID |
| `OPENROUTER_API_KEY` | OpenRouter API 密钥 |
| `GOOGLE_AI_API_KEY` | Google AI API 密钥 |

### 渠道环境变量

| 变量名 | 描述 |
|--------|------|
| `TELEGRAM_BOT_TOKEN` | Telegram Bot Token |
| `DISCORD_BOT_TOKEN` | Discord Bot Token |
| `WHATSAPP_API_KEY` | WhatsApp API Key |
| `SLACK_BOT_TOKEN` | Slack Bot Token |
| `MATRIX_ACCESS_TOKEN` | Matrix Access Token |

---

## 总结

本配置参考文档提供了 Nanobot 的完整配置选项：

### 关键要点

1. **模块化配置**：每个组件都有独立的配置段
2. **灵活配置**：支持多种配置源和变量
3. **安全考虑**：内置安全选项和权限控制
4. **性能优化**：提供了性能相关的配置选项
5. **扩展性**：支持自定义配置和扩展

### 配置优先级

1. 命令行参数
2. 环境变量
3. 用户配置文件
4. 全局配置文件
5. 默认配置

### 最佳实践

1. 使用环境变量保护敏感信息
2. 定期备份配置文件
3. 测试配置更改
4. 监控配置效果
5. 文档化自定义配置

---

**文档版本**: 1.0.0
**最后更新**: 2026-03-03