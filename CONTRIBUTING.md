# 贡献指南

感谢您对 UniDAG-Store 项目的关注！本文档将帮助您了解如何参与贡献。

## 📋 开发准备

### 环境要求

- Python 3.10+
- Poetry 1.5+
- Git
- Protocol Buffers Compiler (protoc)

### 安装开发环境

```bash
# 克隆仓库
git clone https://github.com/chisuhua/brain-unidag-store.git
cd brain-unidag-store

# 安装依赖
poetry install

# 激活虚拟环境
poetry shell
```

## 🔄 开发流程

### 1. 创建分支

```bash
# 从 develop 分支创建特性分支
git checkout develop
git pull origin develop
git checkout -b feature/your-feature-name
```

分支命名规范：
- `feature/` - 新功能
- `fix/` - Bug 修复
- `docs/` - 文档更新
- `refactor/` - 代码重构
- `test/` - 测试相关

### 2. 编写代码

- 遵循 [PEP 8](https://pep8.org/) 代码风格
- 使用类型注解（type hints）
- 为公共 API 编写 docstring
- 编写单元测试

### 3. 代码格式化

```bash
# 自动格式化
poetry run black .

# 检查代码风格
poetry run flake8 .

# 类型检查
poetry run mypy .
```

### 4. 运行测试

```bash
# 运行所有测试
poetry run pytest

# 运行测试并查看覆盖率
poetry run pytest --cov --cov-report=html

# 运行特定测试
poetry run pytest tests/test_storage.py
```

### 5. 提交代码

Commit 消息格式：`<type>(<scope>): <subject>`

类型（type）：
- `feat` - 新功能
- `fix` - Bug 修复
- `docs` - 文档更新
- `test` - 测试相关
- `refactor` - 代码重构
- `perf` - 性能优化
- `chore` - 构建/工具相关

示例：
```bash
git add .
git commit -m "feat(storage): implement SQLite store"
git push origin feature/your-feature-name
```

### 6. 创建 Pull Request

1. 在 GitHub 上创建 Pull Request
2. 填写 PR 模板，说明变更内容
3. 确保 CI 检查全部通过
4. 等待代码审查

## 📝 代码规范

### Python 代码风格

```python
from typing import Optional, List, Dict, Any

class ExampleClass:
    """
    示例类的简短描述。
    
    详细说明可以写在这里，解释类的用途和使用方法。
    
    Attributes:
        name: 名称属性
        value: 数值属性
    """
    
    def __init__(self, name: str, value: int):
        """
        初始化方法。
        
        Args:
            name: 名称
            value: 数值
        """
        self.name = name
        self.value = value
    
    def process(self, data: List[str]) -> Dict[str, Any]:
        """
        处理数据的方法。
        
        Args:
            data: 输入数据列表
            
        Returns:
            处理结果字典
            
        Raises:
            ValueError: 当输入数据无效时
        """
        if not data:
            raise ValueError("Data cannot be empty")
        
        # 处理逻辑
        return {"processed": True}
```

### 命名规范

- **变量/函数**：`snake_case`（例如：`node_count`、`get_topology`）
- **类**：`PascalCase`（例如：`UnifiedDAG`、`DAGStore`）
- **常量**：`UPPER_SNAKE_CASE`（例如：`MAX_NODES`、`DEFAULT_TIMEOUT`）
- **私有成员**：前缀 `_`（例如：`_internal_method`）

### 文档字符串

使用 Google 风格的 docstring：

```python
def example_function(param1: str, param2: int) -> bool:
    """
    函数的简短描述（一行）。
    
    详细说明可以写在这里，解释函数的用途和行为。
    
    Args:
        param1: 第一个参数的说明
        param2: 第二个参数的说明
        
    Returns:
        返回值的说明
        
    Raises:
        ValueError: 参数无效时抛出
        RuntimeError: 运行时错误
        
    Example:
        >>> example_function("test", 42)
        True
    """
    pass
```

## 🧪 测试指南

### 编写测试

```python
import pytest
from unidag_store import UnifiedDAG

class TestUnifiedDAG:
    """UnifiedDAG 类的测试"""
    
    def test_create_dag(self):
        """测试 DAG 创建"""
        dag = UnifiedDAG(
            dag_id="test-001",
            name="Test DAG"
        )
        assert dag.dag_id == "test-001"
        assert dag.name == "Test DAG"
    
    @pytest.mark.asyncio
    async def test_persist_dag(self, store):
        """测试 DAG 持久化"""
        dag = UnifiedDAG(dag_id="test-002")
        snapshot_id = await store.persist(dag, {}, "user123")
        assert snapshot_id.startswith("dag_v")
```

### 测试覆盖率

- 目标：≥80%
- 重点覆盖：核心逻辑、边界情况、错误处理

## 🔍 代码审查

### 审查检查项

- [ ] 代码遵循项目规范
- [ ] 有适当的单元测试
- [ ] 测试覆盖率达标
- [ ] API 有完整的文档
- [ ] 没有引入安全漏洞
- [ ] 性能没有明显下降
- [ ] 兼容现有代码

### 审查反馈

- 建设性反馈，具体说明问题
- 提供改进建议
- 尊重他人的工作

## 📚 文档贡献

### 文档类型

- **架构文档**：系统设计、技术决策
- **API 文档**：接口说明、使用示例
- **用户手册**：快速入门、教程
- **开发文档**：开发流程、规范

### 文档格式

- 使用 Markdown 格式
- 中英文混排时注意空格
- 代码示例要能运行
- 链接要有效

## 🐛 问题反馈

### 提交 Issue

1. 搜索是否已有相关 issue
2. 使用 issue 模板
3. 提供详细信息：
   - 问题描述
   - 复现步骤
   - 预期行为
   - 实际行为
   - 环境信息

### Issue 标签

- `bug` - Bug 报告
- `enhancement` - 功能请求
- `documentation` - 文档相关
- `question` - 问题咨询
- `help wanted` - 需要帮助

## 🤝 社区准则

- 尊重他人
- 建设性讨论
- 包容多样性
- 遵守行为准则

## 📞 联系方式

- GitHub Issues：https://github.com/chisuhua/brain-unidag-store/issues
- 项目讨论：GitHub Discussions

## 📄 许可证

通过贡献代码，您同意您的贡献将在 Apache License 2.0 下发布。

---

再次感谢您的贡献！🎉
