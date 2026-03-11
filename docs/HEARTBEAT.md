# Heartbeat 心跳机制

> **智能任务检查** - 基于 LLM 判断的周期性任务触发

---

## 概述

Heartbeat 是 Nanobot 的智能任务检查机制，通过周期性唤醒 agent 来检查是否有待处理的任务。与 Cron 的固定时间调度不同，Heartbeat 利用 LLM 来判断是否需要执行任务，提供了更高的灵活性。

---

## 工作原理

Heartbeat 采用两阶段执行模式：

### 阶段1：决策阶段

1. 读取工作目录下的 `HEARTBEAT.md` 文件
2. 将文件内容发送给 LLM
3. LLM 通过调用 `heartbeat` 工具返回决策：
   - `action: "skip"` - 没有需要处理的任务
   - `action: "run"` - 有待处理的任务，同时返回任务描述

### 阶段2：执行阶段

仅当阶段1 返回 `action: "run"` 时触发：
- 调用 `on_execute` 回调执行任务
- 如果配置了 `on_notify`，将结果发送到指定渠道

---

## 配置选项

Heartbeat 在配置文件中通过以下选项管理：

```yaml
heartbeat:
  enabled: true          # 是否启用心跳功能
  interval_s: 1800       # 检查间隔（秒），默认30分钟
  model: "claude-3-5-sonnet-20241022"  # 使用的模型
```

### 配置参数说明

| 参数 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `enabled` | bool | `true` | 是否启用心跳功能 |
| `interval_s` | int | `1800` | 心跳检查间隔（秒） |
| `model` | string | 主模型 | 用于决策的 LLM 模型 |

---

## HEARTBEAT.md 文件格式

`HEARTBEAT.md` 文件放在 workspace 目录下，用于定义待检查的任务：

```markdown
# 待处理任务

## 需要检查的事项

1. 检查 GitHub 仓库是否有新的 issue
2. 检查是否有未回复的邮件
3. 检查今日待办事项

## 检查频率

- GitHub 检查：每 30 分钟
- 邮件检查：每 15 分钟

## 任务列表

- issue-check: 检查 nanobot-ai/nanobot 的新 issue
- email-check: 检查 Gmail 未读邮件
- todo-check: 读取 TODO.md 获取今日任务
```

### 文件结构建议

- 使用清晰的标题层次
- 列出具体的检查项和检查条件
- 可以包含多个需要检查的任务

---

## 使用场景

### 场景1：监控外部事件

定期检查外部系统状态并采取行动：

```markdown
# 监控任务

检查以下外部事件：

1. 检查 GitHub 仓库 nanobot-ai/nanobot 是否有新的 issue
2. 检查指定 Discord 频道是否有未读消息
3. 检查 cron 任务是否全部执行成功

如果有任何需要处理的事件，返回 run 并描述具体任务。
```

### 场景2：定时报告

定期生成并发送报告：

```markdown
# 定时报告

1. 每天下午6点生成代码统计报告
2. 每周五生成周报

如果到了报告时间，返回 run 并说明需要生成什么报告。
```

### 场景3：智能提醒

根据上下文决定是否提醒：

```markdown
# 智能提醒

检查以下条件：

1. 用户是否长时间没有活动（超过2小时）
2. 是否有即将到期的任务
3. 是否有需要人工确认的操作

根据检查结果决定是否需要提醒用户。
```

---

## 与 Cron 的区别

| 特性 | Heartbeat | Cron |
|------|-----------|------|
| **触发机制** | LLM 决策 | 固定时间 |
| **灵活性** | 高 | 低 |
| **适用场景** | 条件性任务 | 确定性任务 |
| **配置方式** | HEARTBEAT.md + LLM | 时间表达式 |
| **资源消耗** | 较高（需要 LLM 调用） | 低（纯时间计算） |

### 选择建议

- **使用 Heartbeat**：
  - 任务是否执行取决于外部状态
  - 需要根据上下文做出复杂判断
  - 执行频率不固定

- **使用 Cron**：
  - 任务执行时间是确定性的
  - 简单的定时提醒
  - 需要精确控制执行时间

---

## 实现细节

### 核心类

```python
class HeartbeatService:
    def __init__(
        self,
        workspace: Path,
        provider: LLMProvider,
        model: str,
        on_execute: Callable[[str], Coroutine[str]] | None = None,
        on_notify: Callable[[str], Coroutine[None]] | None = None,
        interval_s: int = 30 * 60,
        enabled: bool = True,
    ):
        ...
```

### 心跳决策工具

LLM 使用 `heartbeat` 工具进行决策：

```json
{
  "type": "function",
  "function": {
    "name": "heartbeat",
    "description": "Report heartbeat decision after reviewing tasks.",
    "parameters": {
      "type": "object",
      "properties": {
        "action": {
          "type": "string",
          "enum": ["skip", "run"],
          "description": "skip = nothing to do, run = has active tasks"
        },
        "tasks": {
          "type": "string",
          "description": "Natural-language summary of active tasks (required for run)"
        }
      },
      "required": ["action"]
    }
  }
}
```

---

## 注意事项

1. **LLM 成本**：每次心跳检查都会调用 LLM，请根据需求设置合理的间隔
2. **模型选择**：建议使用较轻量的模型进行决策，以降低成本
3. **文件存在性**：如果 HEARTBEAT.md 不存在或为空，心跳服务会跳过检查
4. **错误处理**：执行失败时会记录错误日志，但不会阻止后续心跳周期
5. **手动触发**：可以使用 `trigger_now()` 方法手动触发一次心跳