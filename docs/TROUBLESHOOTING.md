# 故障排除指南

> **系统诊断和解决方案** - 深入的问题排查方法

---

## 📑 目录

1. [诊断工具](#诊断工具)
2. [常见错误](#常见错误)
3. [系统问题](#系统问题)
4. [网络问题](#网络问题)
5. [性能问题](#性能问题)
6. [数据问题](#数据问题)
7. [恢复操作](#恢复操作)
8. [预防措施](#预防措施)

---

## 诊断工具

### 内置诊断命令

```bash
# 系统健康检查
nanobot doctor

# 检查配置文件
nanobot validate-config

# 测试连接
nanobot test connection

# 测试渠道
nanobot test channel telegram
nanobot test channel discord

# 测试 LLM 提供商
nanobot test provider anthropic
nanobot test provider openai

# 性能测试
nanobot benchmark

# 内存分析
nanobot memory analyze
```

### 日志分析工具

```bash
# 查看最新日志
tail -f ~/.nanobot/logs/nanobot.log

# 按级别过滤
grep "ERROR" ~/.nanobot/logs/nanobot.log
grep "WARNING" ~/.nanobot/logs/nanobot.log
grep "DEBUG" ~/.nanobot/logs/nanobot.log

# 按时间范围
grep "2024-03-03 10:" ~/.nanobot/logs/nanobot.log

# 按组件过滤
grep "telegram" ~/.nanobot/logs/nanobot.log
grep "agent" ~/.nanobot/logs/nanobot.log

# 日志统计
grep -c "ERROR" ~/.nanobot/logs/nanobot.log
```

### 性能分析工具

```bash
# CPU 使用率
top -p $(pgrep nanobot)
htop -p $(pgrep nanobot)

# 内存使用
ps -p $(pgrep nanobot) -o pid,ppid,cmd,%mem,vsz,rss

# 网络连接
netstat -tulpn | grep nanobot
ss -tulpn | grep nanobot

# 磁盘 I/O
iostat -x 1 10
iotop -p $(pgrep nanobot)

# 监控指标
curl http://localhost:9090/metrics
```

### 数据分析工具

```bash
# 数据库大小
sqlite3 ~/.nanobot/nanobot.db "SELECT name FROM sqlite_master WHERE type='table';"

# 记忆文件大小
du -sh ~/.nanobot/workspace/memory/

# 会话数据
ls -la ~/.nanobot/sessions/

# 工作空间使用
du -sh ~/.nanobot/workspace/
```

---

## 常见错误

### 1. 启动失败

**错误**: `Failed to start agent loop`

**诊断步骤**:
```bash
# 检查配置文件
nanobot validate-config

# 查看启动日志
journalctl -u nanobot -n 50

# 检查端口占用
netstat -tulpn | grep 8080

# 检查依赖
pip check
```

**解决方案**:
```yaml
# 修复配置文件权限
chmod 600 ~/.nanobot/config.yaml

# 修复数据目录权限
chmod 755 ~/.nanobot/
chmod 755 ~/.nanobot/workspace/

# 修复 Python 环境
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### 2. API 连接失败

**错误**: `Connection error: Failed to connect to LLM provider`

**诊断步骤**:
```bash
# 测试 API 密钥
curl -H "Authorization: Bearer $ANTHROPIC_API_KEY" \
  https://api.anthropic.com/v1/models

# 测试网络连接
ping api.anthropic.com
curl -I https://api.anthropic.com

# 验证配置
cat ~/.nanobot/config.yaml | grep anthropic
```

**解决方案**:
```bash
# 更新 API 密钥
export ANTHROPIC_API_KEY=sk-ant-xxx

# 使用代理
export HTTPS_PROXY=http://proxy.example.com:8080

# 增加超时时间
echo "timeout: 60" >> ~/.nanobot/config.yaml
```

### 3. 渠道连接问题

**错误**: `Telegram channel not responding`

**诊断步骤**:
```bash
# 测试 Bot Token
curl "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/getMe"

# 检查网络
curl -I https://api.telegram.org

# 查看渠道日志
grep -i "telegram" ~/.nanobot/logs/nanobot.log | tail -20
```

**解决方案**:
```bash
# 更新 Bot Token
export TELEGRAM_BOT_TOKEN=123456:ABC-DEF

# 检查权限
# 在 Telegram 中联系 @BotFather，使用 /mybots 检查 token

# 启用 webhook（可选）
nanobot channel telegram set-webhook
```

### 4. 工具执行失败

**错误**: `Tool execution failed: Permission denied`

**诊断步骤**:
```bash
# 检查工具配置
cat ~/.nanobot/config.yaml | grep tools

# 测试工具权限
which python3
which bash
ls -la /tmp

# 查看工具日志
grep "exec" ~/.nanobot/logs/nanobot.log | tail -10
```

**解决方案**:
```yaml
# 更新工具配置
tools:
  exec:
    allowed_commands:
      - "python3"
      - "bash"
      - "ls"
      - "cat"
    working_dir: "/tmp"
    timeout: 60
```

### 5. 内存不足错误

**错误**: `Memory limit exceeded` / `Out of memory`

**诊断步骤**:
```bash
# 检查内存使用
free -h
cat /proc/meminfo

# 查看进程内存
ps -p $(pgrep nanobot) -o pid,vsz,rss

# 查看内存日志
grep "memory" ~/.nanobot/logs/nanobot.log
```

**解决方案**:
```yaml
# 优化配置
agents:
  memory_window: 20
  max_tokens: 2048
  consolidate_threshold: 50

# 增加系统内存
# 考虑使用 swap
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

---

## 系统问题

### 1. 依赖冲突

**问题**: `ImportError: cannot import name 'X'`

**解决方案**:
```bash
# 创建干净的虚拟环境
python -m venv clean_env
source clean_env/bin/activate
pip install nanobot-ai

# 使用 pip-tools
pip install pip-tools
pip-compile requirements.in > requirements.txt
pip install -r requirements.txt

# 检查冲突
pip check
pipdeptree
```

### 2. 权限问题

**问题**: `Permission denied` 错误

**解决方案**:
```bash
# 修复文件权限
sudo chown -R $USER:$USER ~/.nanobot/
chmod 755 ~/.nanobot/
chmod 644 ~/.nanobot/config.yaml
chmod 600 ~/.nanobot/.env

# 修复目录权限
find ~/.nanobot -type d -exec chmod 755 {} \;
find ~/.nanobot -type f -exec chmod 644 {} \;

# 使用用户安装
pip install --user nanobot-ai
```

### 3. Python 版本问题

**问题**: `requires Python >= 3.11`

**解决方案**:
```bash
# 检查当前 Python 版本
python --version

# 安装 Python 3.11
# Ubuntu/Debian
sudo apt update
sudo apt install python3.11 python3.11-venv

# macOS
brew install python@3.11

# 创建虚拟环境
python3.11 -m venv venv
source venv/bin/activate
pip install nanobot-ai
```

### 4. 系统资源不足

**问题**: 系统响应缓慢或崩溃

**解决方案**:
```bash
# 监控资源
htop
iotop
df -h

# 清理磁盘
sudo apt autoremove
sudo apt clean
docker system prune

# 调整 nanobot 配置
nano ~/.nanobot/config.yaml
```

### 5. 服务管理问题

**问题**: nanobot 服务无法自动启动

**解决方案**:
```bash
# 创建 systemd 服务
sudo nano /etc/systemd/system/nanobot.service

[Unit]
Description=Nanobot AI Agent
After=network.target

[Service]
Type=simple
User=$USER
WorkingDirectory=/home/$USER/nanobot
ExecStart=/home/$USER/nanobot/venv/bin/nanobot gateway
Restart=always
RestartSec=10
Environment=PATH=/home/$USER/nanobot/venv/bin
Environment=NANOBOT_ENV=production

[Install]
WantedBy=multi-user.target

# 启用服务
sudo systemctl daemon-reload
sudo systemctl enable nanobot
sudo systemctl start nanobot
```

---

## 网络问题

### 1. API 连接超时

**问题**: `timeout error` when calling LLM APIs

**解决方案**:
```bash
# 测试网络连通性
ping api.anthropic.com
ping api.openai.com

# 检查代理设置
echo $HTTP_PROXY
echo $HTTPS_PROXY

# 配置代理
export HTTP_PROXY=http://proxy.example.com:8080
export HTTPS_PROXY=http://proxy.example.com:8080

# 增加超时配置
cat >> ~/.nanobot/config.yaml << EOF
providers:
  anthropic:
    timeout: 60
    connect_timeout: 30
EOF
```

### 2. Webhook 配置问题

**问题**: Webhook 不工作

**解决方案**:
```bash
# 测试 webhook
curl -X POST https://your-domain.com/telegram/webhook \
  -H "Content-Type: application/json" \
  -d '{"message": {"text": "test"}}'

# 检查 SSL 证书
openssl s_client -connect your-domain.com:443

# 更新配置
cat >> ~/.nanobot/config.yaml << EOF
channels:
  telegram:
    webhook:
      enabled: true
      url: "https://your-domain.com/telegram/webhook"
      secret_token: "${TELEGRAM_WEBHOOK_SECRET}"
EOF

# 重启服务
nanobot restart
```

### 3. 端口冲突

**问题**: `Address already in use`

**解决方案**:
```bash
# 查找占用端口的进程
lsof -i :8080
netstat -tulpn | grep 8080

# 终止进程
sudo kill -9 $(lsof -t :8080)

# 修改端口
export PORT=8081
cat >> ~/.nanobot/config.yaml << EOF
global:
  port: 8081
EOF
```

### 4. 防火墙配置

**问题**: 外部无法访问

**解决方案**:
```bash
# 检查防火墙状态
sudo ufw status

# 开放端口
sudo ufw allow 8080/tcp

# 检查 iptables
sudo iptables -L -n

# 开放端口
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
```

### 5. DNS 问题

**问题**: 无法解析域名

**解决方案**:
```bash
# 检查 DNS
cat /etc/resolv.conf
nslookup api.anthropic.com

# 修改 DNS
sudo nano /etc/resolv.conf
# nameserver 8.8.8.8
# nameserver 1.1.1.1

# 测试 DNS
dig api.anthropic.com
```

---

## 性能问题

### 1. 响应缓慢

**问题**: Nanobot 响应时间过长

**解决方案**:
```bash
# 性能分析
python -m cProfile -o profile.stats $(which nanobot) agent "test"

# 分析结果
python -m pstats profile.stats

# 优化配置
cat >> ~/.nanobot/config.yaml << EOF
agents:
  timeout: 15
  max_tokens: 2048
  temperature: 0.5

  cache:
    enabled: true
    type: redis
    url: "redis://localhost:6379"
EOF
```

### 2. 内存泄漏

**问题**: 内存使用持续增长

**解决方案**:
```bash
# 监控内存
watch -n 5 'ps -p $(pgrep nanobot) -o pid,rss,vsz'

# 检查日志
grep "memory" ~/.nanobot/logs/nanobot.log

# 重启服务
nanobot restart

# 使用内存限制
cat >> ~/.nanobot/config.yaml << EOF
agents:
  memory_limit: "512MB"
  consolidate_threshold: 50
EOF
```

### 3. 高并发处理

**问题**: 多用户同时访问性能下降

**解决方案**:
```bash
# 增加连接池
cat >> ~/.nanobot/config.yaml << EOF
database:
  pool_size: 20
  max_overflow: 50

channels:
  telegram:
    max_connections: 50
    timeout: 10
EOF

# 使用负载均衡
sudo apt install nginx
```

### 4. 缓存优化

**解决方案**:
```bash
# 启用 Redis 缓存
sudo apt install redis-server
systemctl enable redis-server
systemctl start redis-server

# 配置缓存
cat >> ~/.nanobot/config.yaml << EOF
cache:
  enabled: true
  type: redis
  url: "redis://localhost:6379"
  ttl: 3600
  max_size: 1000
EOF
```

### 5. 日志优化

**解决方案**:
```bash
# 配置日志轮转
sudo logrotate -f /etc/logrotate.d/nanobot

# 使用结构化日志
cat >> ~/.nanobot/config.yaml << EOF
logging:
  structured: true
  json_format: true
  compress_logs: true
  max_log_size: "50MB"
EOF
```

---

## 数据问题

### 1. 数据库损坏

**问题**: `sqlite3 database is locked` 或 `database disk image is malformed`

**解决方案**:
```bash
# 备份数据
cp ~/.nanobot/nanobot.db ~/.nanobot/nanobot.db.backup

# 检查数据库
sqlite3 ~/.nanobot/nanobot.db "PRAGMA integrity_check;"

# 修复数据库
sqlite3 ~/.nanobot/nanobot.db "PRAGMA quick_check;"

# 如果损坏，从备份恢复
cp ~/.nanobot/nanobot.db.backup ~/.nanobot/nanobot.db

# 重新初始化
nanobot init-db
```

### 2. 记忆文件损坏

**问题**: MEMORY.md 或 HISTORY.md 文件损坏

**解决方案**:
```bash
# 备份记忆文件
cp ~/.nanobot/workspace/memory/MEMORY.md ~/.nanobot/workspace/memory/MEMORY.md.backup

# 创建新的记忆文件
echo "# 记忆文件" > ~/.nanobot/workspace/memory/MEMORY.md

# 检查历史文件
cat ~/.nanobot/workspace/memory/HISTORY.md | head -20

# 如果历史文件损坏，清空它
> ~/.nanobot/workspace/memory/HISTORY.md
```

### 3. 配置文件损坏

**问题**: 配置文件语法错误

**解决方案**:
```bash
# 检查 YAML 语法
python -c "
import yaml
try:
    with open('~/.nanobot/config.yaml') as f:
        yaml.safe_load(f)
    print('配置文件语法正确')
except Exception as e:
    print(f'配置文件错误: {e}')
"

# 备份并重新创建配置
cp ~/.nanobot/config.yaml ~/.nanobot/config.yaml.broken
nanobot onboard

# 从备份恢复重要配置
cat ~/.nanobot/config.yaml.broken >> ~/.nanobot/config.yaml
```

### 4. 会话数据丢失

**问题**: 会话历史丢失

**解决方案**:
```bash
# 检查会话目录
ls -la ~/.nanobot/sessions/

# 检查会话文件
sqlite3 ~/.nanobot/nanobot.db "SELECT * FROM sessions;"

# 重建会话
nanobot rebuild-sessions

# 如果问题持续，考虑切换数据库
cat >> ~/.nanobot/config.yaml << EOF
database:
  url: "postgresql://user:pass@localhost:5432/nanobot"
EOF
```

### 5. 文件系统错误

**问题**: 文件读写错误

**解决方案**:
```bash
# 检查文件系统
fsck -t ext4 /dev/sda1

# 修复权限
find ~/.nanobot -type f -exec chmod 644 {} \;
find ~/.nanobot -type d -exec chmod 755 {} \;

# 检查磁盘空间
df -h
du -sh ~/.nanobot/

# 清理临时文件
find ~/.nanobot -name "*.tmp" -delete
```

---

## 恢复操作

### 1. 系统恢复

```bash
# 创建完整备份
tar -czf nanobot-full-backup-$(date +%Y%m%d).tar.gz \
  --exclude='*.log' \
  --exclude='*.tmp' \
  ~/.nanobot/

# 从备份恢复
tar -xzf nanobot-full-backup-20240303.tar.gz -C ~

# 验证恢复
nanobot doctor
```

### 2. 配置恢复

```bash
# 备份配置
cp ~/.nanobot/config.yaml ~/.nanobot/config.yaml.backup

# 恢复到默认配置
nanobot onboard

# 恢复自定义配置
cp ~/.nanobot/config.yaml.backup ~/.nanobot/config.yaml
```

### 3. 数据恢复

```bash
# 恢复数据库
cp ~/.nanobot/nanobot.db.backup ~/.nanobot/nanobot.db

# 恢复记忆文件
cp -r ~/.nanobot/workspace/memory.backup ~/.nanobot/workspace/memory/

# 恢复技能目录
cp -r ~/.nanobot/workspace/skills.backup ~/.nanobot/workspace/skills/
```

### 4. 服务恢复

```bash
# 重启服务
sudo systemctl restart nanobot

# 检查服务状态
sudo systemctl status nanobot

# 查看服务日志
sudo journalctl -u nanobot -f
```

### 5. 紧急恢复

```bash
# 停止所有服务
sudo systemctl stop nanobot

# 备份当前数据
cp -r ~/.nanobot ~/.nanobot.emergency

# 重新安装
pip install --force-reinstall nanobot-ai

# 恢复数据
cp -r ~/.nanobot.emergency/* ~/.nanobot/

# 重新启动
sudo systemctl start nanobot
```

---

## 预防措施

### 1. 定期维护

```bash
# 创建维护脚本
nano ~/.nanobot/scripts/maintenance.sh

#!/bin/bash
# 备份
tar -czf ~/nanobot-backup-$(date +%Y%m%d).tar.gz ~/.nanobot/

# 清理日志
find ~/.nanobot/logs -name "*.log" -mtime +7 -delete

# 清理临时文件
find ~/.nanobot -name "*.tmp" -delete

# 清理数据库
sqlite3 ~/.nanobot/nanobot.db "VACUUM;"

# 检查磁盘空间
df -h | grep nanobot

echo "维护完成"
```

### 2. 监控设置

```bash
# 创建监控脚本
nano ~/.nanobot/scripts/monitor.sh

#!/bin/bash
# 监控进程
if ! pgrep -f "nanobot gateway" > /dev/null; then
    echo "Nanobot not running, restarting..."
    nanobot restart
fi

# 监控内存
MEM_USAGE=$(ps -p $(pgrep nanobot) -o %mem --no-headers)
if (( $(echo "$MEM_USAGE > 80" | bc -l) )); then
    echo "High memory usage: $MEM_USAGE%"
    nanobot restart
fi

# 监控磁盘空间
DISK_USAGE=$(df -h ~/.nanobot | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -gt 80 ]; then
    echo "High disk usage: $DISK_USAGE%"
    # 清理日志
    find ~/.nanobot/logs -name "*.log" -mtime +7 -delete
fi
```

### 3. 自动备份

```bash
# 设置定时任务
crontab -e

# 添加以下内容
# 每日凌晨 2 点备份
0 2 * * * /home/$USER/.nanobot/scripts/maintenance.sh

# 每小时监控
0 * * * * /home/$USER/.nanobot/scripts/monitor.sh
```

### 4. 安全加固

```bash
# 修改文件权限
chmod 600 ~/.nanobot/.env
chmod 700 ~/.nanobot/
chmod 600 ~/.nanobot/config.yaml

# 限制访问
chown -R $USER:$USER ~/.nanobot

# 禁用 root 运行
cat >> ~/.nanobot/scripts/safe-mode.sh << 'EOF'
#!/bin/bash
if [ "$EUID" -eq 0 ]; then
    echo "不要使用 root 运行 Nanobot"
    exit 1
fi
EOF

chmod +x ~/.nanobot/scripts/safe-mode.sh
```

### 5. 版本控制

```bash
# 使用 Git 管理配置
cd ~/.nanobot
git init
git add config.yaml skills/ workspace/identity.md
git commit -m "Initial commit"

# 定期提交
git add .
git commit -m "Daily updates"
```

---

## 紧急联系

### 开发团队支持

- **GitHub Issues**: [github.com/nanobot-ai/nanobot/issues](https://github.com/nanobot-ai/nanobot/issues)
- **紧急邮件**: emergency@nanobot.ai
- **支持电话**: +86 123-4567-8900（仅限紧急情况）

### 社区支持

- **Discord**: [discord.gg/nanobot](https://discord.gg/nanobot)
- **论坛**: [forum.nanobot.ai](https://forum.nanobot.ai)
- **文档**: [docs.nanobot.ai](https://docs.nanobot.ai)

### 故障报告模板

```markdown
## 问题描述
[简要描述问题]

## 环境信息
- OS: [操作系统版本]
- Python: [Python 版本]
- Nanobot: [Nanobot 版本]
- 配置: [相关配置]

## 复现步骤
1. [步骤1]
2. [步骤2]
3. [步骤3]

## 期望行为
[描述期望的结果]

## 实际行为
[描述实际的结果]

## 错误信息
```

[粘贴完整的错误信息]
```

## 日志文件
```

[粘贴相关日志]
```
```

---

**文档版本**: 1.0.0
**最后更新**: 2026-03-03