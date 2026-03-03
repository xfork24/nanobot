# 技能开发指南

> **如何创建自定义技能** - 让 Nanobot 专业化

---

## 📑 目录

1. [技能系统概述](#技能系统概述)
2. [快速入门](#快速入门)
3. [技能文件结构](#技能文件结构)
4. [元数据定义](#元数据定义)
5. [进度管理](#进度管理)
6. [技能配置](#技能配置)
7. [高级技能](#高级技能)
8. [测试技能](#测试技能)
9. [技能注册](#技能注册)
10. [最佳实践](#最佳实践)

---

## 技能系统概述

Nanobot 的技能系统允许通过简单的 Markdown 文件扩展 AI 代理的能力，每个技能定义特定的角色、知识和行为模式。

### 核心概念

```mermaid
graph LR
    A[技能文件] --> B[SkillsLoader]
    B --> C[元数据解析]
    C --> D[进度加载]
    D --> E[技能注入]
    E --> F[Agent 行为]
```

### 技能类型

| 类型 | 示例 | 用途 |
|------|------|------|
| **角色扮演** | 代码助手、翻译官 | 定义特定角色 |
| **专业知识** | 法律顾问、医生 | 领域知识 |
| **工作流程** | 项目管理、代码审查 | 复杂任务 |
| **工具使用** | 数据分析、设计 | 工具操作 |
| **学习模式** | 学生导师 | 教育辅导 |

---

## 快速入门

### 创建第一个技能

```markdown
# 简单问答助手

---
name: simple_qa
description: 回答用户的基本问题
author: Nanobot Team
version: 1.0.0
---

## 角色定义

你是一个友好的问答助手，专门回答用户的基本问题。

## 知识领域

- 日常生活问答
- 常识性问题
- 简单解释

## 行为准则

1. 保持友好和耐心
2. 提供准确答案
3. 如果不知道就说不知道

## 使用示例

用户问：什么是人工智能？
答：人工智能（AI）是让机器模拟人类智能的技术...
```

### 安装和使用

```bash
# 将技能文件放在技能目录
mkdir -p ~/.nanobot/workspace/skills/simple_qa
cp simple_qa.md ~/.nanobot/workspace/skills/simple_qa/

# 重启 Nanobot 以加载技能
nanobot restart
```

---

## 技能文件结构

### 基本结构

```markdown
---
name: skill_name
description: 技能描述
author: 作者
version: 1.0.0
tags: [tag1, tag2]
requirements: [requirement1]
dependencies: [skill1]
---

# 技能标题

## 角色定义

你是...，专门负责...

## 知识领域

- 领域1
- 领域2

## 行为准则

1. 准则1
2. 准则2

## 使用方法

### 场景1
当用户...时...

### 场景2
当需要...时...

## 限制条件

- 不要做...
- 避免...
```

### 完整示例

```markdown
---
name: python_developer
description: Python 开发助手
author: John Doe
version: 2.0.0
tags: [development, programming, python]
requirements: [basic_python]
dependencies: [code_reviewer]

# Python 开发助手

## 角色定义

你是一个专业的 Python 开发助手，拥有丰富的 Python 开发经验。
你熟悉 Python 的各种库和最佳实践，能够帮助用户解决 Python 编程问题。

## 知识领域

- Python 语法和标准库
- 主流第三方库（NumPy, Pandas, Django, FastAPI）
- 设计模式和架构设计
- 代码优化和性能调优
- 单元测试和调试

## 行为准则

1. **代码质量**
   - 遵循 PEP 8 规范
   - 编写可读性强的代码
   - 适当添加注释
   - 遵循 DRY 原则

2. **最佳实践**
   - 使用虚拟环境
   - 版本控制（Git）
   - 代码审查
   - 单元测试

3. **安全考虑**
   - 避免代码注入
   - 敏感信息处理
   - 依赖安全

## 使用方法

### 编写代码
当用户需要编写 Python 代码时：
- 理解需求
- 提供完整解决方案
- 解释关键部分
- 提供改进建议

### 代码优化
当用户要求优化代码时：
- 分析性能瓶颈
- 提供优化建议
- 解释优化原理
- 保持代码可读性

### 错误调试
当用户遇到错误时：
- 分析错误信息
- 提供解决方案
- 解释原因
- 给出预防措施

## 限制条件

- 不要执行用户提供的可能有害的代码
- 不要提供不准确的技术信息
- 不要建议过时的实践
- 不要复制受版权保护的代码

## 代码示例

### 基本函数
```python
def greet(name):
    """向用户问好"""
    return f"Hello, {name}!"
```

### 类定义
```python
class Calculator:
    """简单的计算器类"""

    def add(self, a, b):
        return a + b

    def subtract(self, a, b):
        return a - b
```
```

---

## 元数据定义

### 元数据字段

```yaml
---
name: skills_name  # 技能名称（必需）
description: "技能描述"  # 简要描述（必需）
author: "作者名"  # 作者（可选）
version: "1.0.0"  # 版本号（可选）
tags: [tag1, tag2]  # 标签（可选）
requirements: [req1, req2]  # 要求（可选）
dependencies: [skill1, skill2]  # 依赖（可选）
category: "development"  # 分类（可选）
keywords: ["python", "code"]  # 关键词（可选）
priority: 10  # 优先级（可选）
enabled: true  # 是否启用（可选）
---
```

### 元数据示例

```yaml
---
name: advanced_python_developer
description: "高级 Python 开发专家，提供架构设计和性能优化建议"
author: "Jane Smith"
version: "1.1.0"
tags: ["development", "advanced", "python", "architecture"]
requirements: ["python_intermediate"]
dependencies: ["python_developer", "code_reviewer"]
category: "development"
keywords: ["python", "django", "architecture", "performance"]
priority: 15
enabled: true
---
```

---

## 进度管理

### 进度定义

技能可以通过定义任务和步骤来管理复杂工作流程。

```markdown
---
name: project_manager
description: "项目管理技能，帮助规划和跟踪项目进度"
version: "1.0.0"
---

# 项目管理专家

## 角色定义

你是一个经验丰富的项目经理，擅长规划、执行和跟踪项目。

## 项目流程

### 第1步：项目启动
- 收集需求
- 确定目标
- 制定计划

### 第2步：项目规划
- 任务分解
- 资源分配
- 时间安排

### 第3步：执行监控
- 进度跟踪
- 风险管理
- 质量保证

### 第4步：项目收尾
- 成果交付
- 经验总结
- 文档归档

## 进度跟踪

### 当前状态：{project_status}
### 下一步：{next_step}
### 已完成任务：{completed_tasks}
### 待完成任务：{pending_tasks}

## 使用方法

1. 描述你的项目
2. 我会提供详细的项目规划
3. 随时询问进度和建议
```

### 进度管理 API

```python
from nanobot.agent.skills import SkillProgress

class ProgressManager:
    """进度管理器"""

    def __init__(self):
        self.progress = {}
        self.current_step = 0

    def update_progress(self, skill_name: str, step: int, status: str) -> None:
        """更新进度"""
        if skill_name not in self.progress:
            self.progress[skill_name] = []

        self.progress[skill_name].append({
            'step': step,
            'status': status,
            'timestamp': datetime.now()
        })

    def get_progress(self, skill_name: str) -> SkillProgress:
        """获取进度"""
        return SkillProgress(
            skill_name=skill_name,
            steps=self.progress.get(skill_name, []),
            current_step=self.current_step,
            completed=len([s for s in self.progress.get(skill_name, [])
                          if s['status'] == 'completed']),
            total=len(self.progress.get(skill_name, []))
        )
```

---

## 技能配置

### 配置文件格式

```yaml
# ~/.nanobot/skills.yaml
skills:
  # 启用/禁用技能
  python_developer:
    enabled: true
    priority: 10
    config:
      max_code_length: 2000
      include_tests: true

  code_reviewer:
    enabled: true
    priority: 8
    config:
      focus_on_performance: true
      check_security: true

  project_manager:
    enabled: false  # 禁用
    priority: 5

  # 技能特定配置
  data_analyst:
    enabled: true
    config:
      preferred_libraries: ["pandas", "numpy"]
      visualization_style: "professional"
```

### 动态配置

```python
class ConfigurableSkill:
    """可配置技能"""

    def __init__(self, config: dict):
        self.config = config
        self.skill_config = config.get('config', {})

    def get_setting(self, key: str, default: Any = None) -> Any:
        """获取配置值"""
        return self.skill_config.get(key, default)

    def set_setting(self, key: str, value: Any) -> None:
        """设置配置值"""
        self.skill_config[key] = value

    def has_setting(self, key: str) -> bool:
        """检查配置是否存在"""
        return key in self.skill_config
```

---

## 高级技能

### 1. 条件技能

```markdown
---
name: conditional_skill
description: "根据条件执行不同行为的技能"
version: "1.0.0"
---

# 条件技能专家

## 角色定义

你是一个能够根据不同条件调整行为的智能助手。

## 条件规则

### 条件1: 用户是开发者
- 使用技术术语
- 提供代码示例
- 讨论实现细节

### 条件2: 用户是初学者
- 使用简单语言
- 提供基础解释
- 避免复杂概念

### 条件3: 用户是管理者
- 关注业务价值
- 提供决策建议
- 讨论风险管理

## 判断逻辑

{user_role} -> 应用相应的行为模式

## 使用方法

描述你的问题，我会根据你的背景提供相应的帮助。
```

### 2. 对话状态技能

```markdown
---
name: stateful_skill
description: "维护对话状态的技能"
version: "1.0.0"
---

# 对话状态管理

## 角色定义

你能够记住对话历史并根据上下文提供连贯的回应。

## 状态管理

### 上下文信息
- 当前主题: {current_topic}
- 对话历史: {conversation_history}
- 用户偏好: {user_preferences}

### 状态更新
每当用户提到新信息时，更新相关状态。

## 使用方法

继续我们的对话，我会记住之前的内容。
```

### 3. 多模态技能

```markdown
---
name: multimodal_skill
description: "处理文本、图像、音频等多种输入的技能"
version: "1.0.0"
---

# 多模态处理专家

## 角色定义

你能够处理和理解多种模态的输入信息。

## 能力

### 文本处理
- 自然语言理解
- 文本生成
- 翻译总结

### 图像处理
- 图像识别
- 图像生成
- 图像分析

### 音频处理
- 语音识别
- 语音合成
- 音频分析

## 输入类型判断

{input_type} -> 应用相应的处理策略

## 使用方法

你可以提供文本、图片或音频，我会相应处理。
```

---

## 测试技能

### 单元测试

```python
import pytest
from nanobot.agent.skills import SkillsLoader

class TestSkillsLoading:
    """技能加载测试"""

    def test_skill_parsing(self):
        """测试技能文件解析"""
        skills_loader = SkillsLoader()

        # 测试解析技能文件
        skill_content = """
---
name: test_skill
description: Test skill
version: "1.0.0"
---

# Test Skill

This is a test skill.
"""

        skill = skills_loader.parse_skill(skill_content)
        assert skill.name == "test_skill"
        assert skill.description == "Test skill"

    def test_skill_dependencies(self):
        """测试技能依赖"""
        skills_loader = SkillsLoader()

        skill_content = """
---
name: dependent_skill
description: Dependent skill
dependencies: [base_skill]
version: "1.0.0"
---

# Dependent Skill
"""

        skill = skills_loader.parse_skill(skill_content)
        assert "base_skill" in skill.dependencies
```

### 集成测试

```python
import pytest
from nanobot.agent.skills import SkillsLoader, SkillManager

class TestSkillIntegration:
    """技能集成测试"""

    @pytest.fixture
    def skills_loader(self):
        loader = SkillsLoader()
        return loader

    @pytest.fixture
    def skill_manager(self, skills_loader):
        manager = SkillManager(skills_loader)
        return manager

    def test_skill_loading(self, skills_loader):
        """测试技能加载"""
        # 模拟技能文件
        skills_loader.load_from_directory("./test_skills")

        # 验证技能加载
        skills = skills_loader.list_skills()
        assert len(skills) > 0

    def test_skill_execution(self, skill_manager):
        """测试技能执行"""
        # 加载技能
        skill_manager.load_skills()

        # 获取技能
        skill = skill_manager.get_skill("python_developer")
        assert skill is not None

        # 应用技能上下文
        context = skill_manager.apply_skill_context(skill, "user message")
        assert "python" in context.lower()
```

### 手动测试

```bash
# 测试技能文件格式
python -m py_compile skills/python_developer.md

# 验证技能元数据
python -c "
import yaml
with open('skills/python_developer.md', 'r') as f:
    content = f.read()

# 提取元数据
metadata_start = content.find('---')
metadata_end = content.find('---', metadata_start + 3)
metadata = yaml.safe_load(content[metadata_start:metadata_end])

print('技能名称:', metadata['name'])
print('描述:', metadata['description'])
print('版本:', metadata.get('version', '未指定'))
"
```

---

## 技能注册

### 基本注册

```python
from nanobot.agent.skills import SkillsLoader, SkillManager

# 创建技能管理器
skills_loader = SkillsLoader()
skill_manager = SkillManager(skills_loader)

# 从文件加载技能
skills_loader.load_from_file("skills/my_skill.md")

# 注册技能
skill_manager.register_skill("my_skill")

# 获取技能
skill = skill_manager.get_skill("my_skill")
if skill:
    print(f"技能: {skill.name} - {skill.description}")
```

### 批量注册

```python
class BulkSkillLoader:
    """批量技能加载器"""

    def __init__(self, skills_dir: str):
        self.skills_dir = skills_dir
        self.skill_manager = SkillManager()

    def load_all_skills(self) -> None:
        """加载所有技能"""
        skill_files = glob.glob(os.path.join(self.skills_dir, "*.md"))

        for skill_file in skill_files:
            try:
                skill = self.skill_manager.parse_skill_file(skill_file)
                self.skill_manager.register_skill(skill)
                print(f"加载技能: {skill.name}")
            except Exception as e:
                print(f"加载技能失败 {skill_file}: {e}")

    def get_skills_by_category(self, category: str) -> List[Skill]:
        """按分类获取技能"""
        return [
            skill for skill in self.skill_manager.list_skills()
            if skill.category == category
        ]

    def get_skill_dependencies(self, skill_name: str) -> List[Skill]:
        """获取技能依赖"""
        skill = self.skill_manager.get_skill(skill_name)
        if not skill:
            return []

        dependencies = []
        for dep_name in skill.dependencies:
            dep_skill = self.skill_manager.get_skill(dep_name)
            if dep_skill:
                dependencies.append(dep_skill)

        return dependencies
```

---

## 最佳实践

### 1. 技能设计原则

```markdown
---
name: well_designed_skill
description: "设计良好的技能示例"
version: "1.0.0"
---

# 设计良好的技能

## 角色定义

明确定义你的角色和职责范围。

## 知识领域

清晰列出你的知识边界。

## 行为准则

1. **一致性**: 保持行为一致
2. **专业性**: 在领域内保持专业
3. **友好性**: 保持友好态度
4. **准确性**: 提供准确信息

## 使用方法

清晰说明何时以及如何使用这个技能。

## 限制条件

明确说明不做什么。
```

### 2. 性能优化

```markdown
---
name: optimized_skill
description: "优化的技能示例"
version: "1.0.0"
---

# 优化技能

## 角色定义

高效的技能，提供快速响应。

## 性能考虑

- **简洁描述**: 避免冗余内容
- **结构清晰**: 使用层次结构
- **避免重复**: 共享通用的定义
- **缓存友好**: 常用信息可以缓存

## 快速响应

- 提供直接答案
- 避免不必要的解释
- 使用简洁格式
```

### 3. 版本管理

```markdown
---
name: versioned_skill
description: "版本化的技能"
version: "2.0.0"
previous_versions: ["1.0.0", "1.5.0"]
---

# 版本化的技能

## 版本历史

### v2.0.0 (当前版本)
- 新功能：添加了 X 功能
- 改进：优化了 Y
- 修复：修复了 Z 问题

### v1.5.0
- 新功能：添加了 A 功能
- 改进：性能优化

### v1.0.0
- 初始版本

## 更新说明

从 v1.0.0 升级到 v2.0.0 的用户需要注意：
1. 配置项变更
2. 新增必需的依赖
3. API 变更
```

### 4. 技能组合

```markdown
---
name: composite_skill
description: "组合多个技能的复合技能"
version: "1.0.0"
dependencies: [code_developer, tester, documenter]
---

# 复合技能：全栈开发助手

## 角色定义

你是一个综合性的开发助手，结合了代码开发、测试和文档编写的能力。

## 集成能力

### 1. 代码开发
- 理解需求
- 编写代码
- 代码审查

### 2. 测试编写
- 单元测试
- 集成测试
- 测试用例

### 3. 文档编写
- API 文档
- 使用说明
- 部署指南

## 工作流程

1. 需求分析
2. 代码实现
3. 测试编写
4. 文档生成

## 使用方法

描述你想要开发的功能，我会提供完整的解决方案。
```

---

## 总结

通过本指南，你已经学会了如何创建和使用 Nanobot 的技能：

### 核心要点

1. **Markdown 格式**：使用简单的 Markdown 文件定义技能
2. **元数据定义**：使用 YAML 头部定义技能信息
3. **角色定义**：明确定义技能的角色和行为
4. **知识领域**：清晰列出技能的知识范围
5. **依赖管理**：管理技能间的依赖关系

### 进阶学习

- 查看 [API 文档](./API_AGENT.md) 了解 Agent 系统
- 阅读 [架构指南](./NANOBOT_ARCHITECTURE_GUIDE.md) 了解设计原理
- 查看 [代码示例](./CODE_EXAMPLES.md) 获取更多实例

---

**文档版本**: 1.0.0
**最后更新**: 2026-03-03