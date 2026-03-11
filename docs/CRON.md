# Cron 定时任务系统

> **定时任务管理** - 安排提醒和周期性任务

---

## 概述

Nanobot 内置了功能强大的定时任务系统，支持三种调度模式：
- **at** - 一次性任务，在指定时间执行
- **every** - 固定间隔任务，每隔一段时间执行
- **cron** - Cron 表达式任务，支持复杂的时间规则

---

## 工作原理

### 三种调度模式

#### 1. at 模式 (一次性任务)

在指定时间执行一次任务，执行完成后自动禁用或删除。

```python
# 添加一次性任务
cron(action="add", message="会议提醒", at="2024-03-15T14:00:00")
```

#### 2. every 模式 (固定间隔)

每隔指定秒数执行一次任务。

```python
# 每20分钟提醒一次
cron(action="add", message="时间到了！", every_seconds=1200)

# 每小时执行任务
cron(action="add", message="检查GitHub stars", every_seconds=3600)
```

#### 3. cron 模式 (Cron 表达式)

使用标准 cron 表达式，支持复杂的时间调度。

```python
# 每天早上8点执行
cron(action="add", message="早间站会", cron_expr="0 8 * * *")

# 工作日下午5点执行
cron(action="add", message="下班提醒", cron_expr="0 17 * * 1-5")

# 指定时区（美西时间每天9点）
cron(action="add", message="美西早会", cron_expr="0 9 * * *", tz="America/Los_Angeles")
```

---

## 数据结构

### CronJob

```python
@dataclass
class CronJob:
    id: str              # 任务唯一标识 (8位UUID)
    name: str            # 任务名称
    enabled: bool        # 是否启用
    schedule: CronSchedule  # 调度配置
    payload: CronPayload    # 任务内容
    state: CronJobState     # 运行时状态
    created_at_ms: int      # 创建时间戳
    updated_at_ms: int      # 更新时间戳
    delete_after_run: bool  # 执行后是否删除
```

### CronSchedule

```python
@dataclass
class CronSchedule:
    kind: Literal["at", "every", "cron"]  # 调度类型
    at_ms: int | None = None               # at 模式的毫秒时间戳
    every_ms: int | None = None            # every 模式的间隔毫秒数
    expr: str | None = None                # cron 模式的表达式
    tz: str | None = None                  # 时区 (仅 cron 模式)
```

### CronPayload

```python
@dataclass
class CronPayload:
    kind: Literal["system_event", "agent_turn"] = "agent_turn"
    message: str = ""                       # 任务消息内容
    deliver: bool = False                   # 是否发送响应到渠道
    channel: str | None = None              # 目标渠道 (如 "whatsapp")
    to: str | None = None                   # 目标地址 (如电话号码)
```

---

## 任务管理

### 添加任务

通过 cron 技能添加任务：

```bash
# 固定间隔提醒
cron(action="add", message="Time to take a break!", every_seconds=1200)

# 动态任务（agent 每次执行并返回结果）
cron(action="add", message="Check GitHub stars and report", every_seconds=600)

# 一次性任务
cron(action="add", message="Meeting reminder", at="2024-03-15T14:00:00")

# 时区感知的 cron
cron(action="add", message="Morning standup", cron_expr="0 9 * * 1-5", tz="America/New_York")
```

### 列出任务

```bash
cron(action="list")
```

### 删除任务

```bash
cron(action="remove", job_id="abc12345")
```

### 启用/禁用任务

```python
# 禁用
cron(action="disable", job_id="abc12345")

# 启用
cron(action="enable", job_id="abc12345")
```

---

## 使用示例

### 固定间隔提醒

每 20 分钟提醒用户休息：

```
cron(action="add", message="Time to take a break! Remember to stretch.", every_seconds=1200)
```

### 一次性任务

在指定时间提醒：

```
cron(action="add", message="会议将在5分钟后开始", at="2024-03-20T14:55:00")
```

### Cron 表达式定时任务

| 用户需求 | 参数 |
|---------|------|
| 每天早上8点 | `cron_expr: "0 8 * * *"` |
| 每天下午6点 | `cron_expr: "0 18 * * *"` |
| 工作日每天9点 | `cron_expr: "0 9 * * 1-5"` |
| 每周一早上9点 | `cron_expr: "0 9 * * 1"` |
| 每半小时 | `cron_expr: "*/30 * * * *"` |

### 时区支持

```python
# 纽约时间每天早上9点
cron(action="add", message="Morning standup", cron_expr="0 9 * * *", tz="America/New_York")

# 东京时间每天晚上8点
cron(action="add", message=" Tokyo evening sync", cron_expr="0 20 * * *", tz="Asia/Tokyo")

# 伦敦时间每周一到周五下午5点
cron(action="add", message="London EOD", cron_expr="0 17 * * 1-5", tz="Europe/London")
```

---

## 配置文件格式

任务配置存储在 `jobs.json` 文件中：

```json
{
  "version": 1,
  "jobs": [
    {
      "id": "abc12345",
      "name": "break-reminder",
      "enabled": true,
      "schedule": {
        "kind": "every",
        "everyMs": 1200000
      },
      "payload": {
        "kind": "agent_turn",
        "message": "Time to take a break!",
        "deliver": false
      },
      "state": {
        "nextRunAtMs": 1710422400000,
        "lastRunAtMs": 1710421200000,
        "lastStatus": "ok"
      },
      "createdAtMs": 1710418800000,
      "updatedAtMs": 1710422400000,
      "deleteAfterRun": false
    }
  ]
}
```

### 字段说明

| 字段 | 类型 | 描述 |
|------|------|------|
| `version` | int | 配置版本号 |
| `jobs` | array | 任务列表 |
| `id` | string | 任务唯一标识 |
| `name` | string | 任务名称 |
| `enabled` | bool | 是否启用 |
| `schedule.kind` | string | 调度类型 (at/every/cron) |
| `schedule.atMs` | int | at 模式的时间戳 (毫秒) |
| `schedule.everyMs` | int | every 模式的间隔 (毫秒) |
| `schedule.expr` | string | cron 表达式 |
| `schedule.tz` | string | 时区名称 |
| `payload.message` | string | 任务消息 |
| `payload.deliver` | bool | 是否发送响应到渠道 |
| `payload.channel` | string | 目标渠道 |
| `payload.to` | string | 目标地址 |
| `state.nextRunAtMs` | int | 下次执行时间戳 |
| `state.lastRunAtMs` | int | 上次执行时间戳 |
| `state.lastStatus` | string | 上次执行状态 (ok/error/skipped) |
| `state.lastError` | string | 上次错误信息 |
| `deleteAfterRun` | bool | 执行后是否删除 (at 模式有效) |

---

## 与 Heartbeat 的区别

| 特性 | Cron | Heartbeat |
|------|------|-----------|
| 触发方式 | 固定时间 | LLM 决策 |
| 适用场景 | 确定性任务 | 条件性任务 |
| 灵活性 | 低 (固定调度) | 高 (动态判断) |
| 配置方式 | 声明式 (时间表达式) | 交互式 (HEARTBEAT.md) |

- **Cron** 适合时间确定的任务（如每日提醒、定期检查）
- **Heartbeat** 适合需要根据状态决定是否执行的任务（如检查外部事件）

---

## 注意事项

1. **时区配置**：只有 cron 模式支持时区配置，使用 IANA 时区名称
2. **一次性任务**：at 模式任务执行后默认禁用，可设置 `delete_after_run` 自动删除
3. **外部修改**：`jobs.json` 修改后会自动重新加载
4. **执行权限**：任务需要正确配置 `deliver`、`channel`、`to` 才能发送响应