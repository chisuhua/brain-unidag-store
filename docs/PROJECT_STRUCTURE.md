# 项目结构

```
brain-unidag-store/
├── docs/                          # 文档目录
│   ├── ARCHITECTURE.md           # 架构文档
│   └── DEVELOPMENT_PLAN.md       # 开发计划
│
├── src/                          # 源代码目录（待创建）
│   └── unidag_store/
│       ├── __init__.py
│       ├── proto/                # Protobuf 定义
│       │   ├── dag_schema_v2_2.proto
│       │   └── service.proto
│       ├── interface.py          # IDAGStore 接口定义
│       ├── storage/              # 存储实现
│       │   ├── __init__.py
│       │   ├── embedded.py       # EmbeddedUniDAGStore（SQLite）
│       │   ├── cloud.py          # CloudUniDAGStore（PostgreSQL + S3）
│       │   ├── serialization.py  # 序列化/反序列化
│       │   └── zarr_features.py  # Zarr 特征存储
│       ├── crypto/               # 加密模块
│       │   ├── __init__.py
│       │   ├── key_derivation.py # 密钥派生
│       │   ├── dag_encryption.py # DAG 加密
│       │   └── zarr_encryption.py# Zarr 块级加密
│       ├── topology/             # 拓扑算法
│       │   ├── __init__.py
│       │   └── kahn_sort.py      # Kahn 拓扑排序
│       ├── exceptions.py         # 异常定义
│       └── models.py             # 数据模型
│
├── tests/                        # 测试目录（待创建）
│   ├── __init__.py
│   ├── test_storage.py           # 存储测试
│   ├── test_crypto.py            # 加密测试
│   ├── test_topology.py          # 拓扑算法测试
│   ├── test_features.py          # 特征存储测试
│   └── integration/              # 集成测试
│       └── test_brain_agent.py   # brain-domain-agent 集成
│
├── benchmarks/                   # 性能基准测试（待创建）
│   ├── bench_serialization.py
│   ├── bench_topology.py
│   └── bench_storage.py
│
├── examples/                     # 示例代码（待创建）
│   ├── basic_usage.py
│   ├── encryption_example.py
│   └── cloud_storage_example.py
│
├── scripts/                      # 工具脚本（待创建）
│   ├── setup_dev.sh             # 开发环境设置
│   └── migrate_v20_to_v22.py    # 版本迁移工具
│
├── .github/                      # GitHub 配置
│   └── workflows/
│       ├── ci.yml               # 持续集成
│       └── release.yml          # 发布流程
│
├── ARCHITECTURE.md              # 架构文档（符号链接到 docs/）
├── CONTRIBUTING.md              # 贡献指南
├── LICENSE                      # Apache 2.0 许可证
├── README.md                    # 项目说明
├── pyproject.toml              # Poetry 配置（待创建）
└── .gitignore                  # Git 忽略文件（待创建）
```

## 目录说明

### 核心目录

- **`src/unidag_store/`** - 主要源代码
  - `proto/` - Protocol Buffers 定义文件
  - `storage/` - 存储引擎实现
  - `crypto/` - 加密相关功能
  - `topology/` - DAG 拓扑算法

- **`tests/`** - 测试代码
  - 单元测试
  - 集成测试
  - 性能测试

- **`docs/`** - 项目文档
  - 架构设计
  - 开发计划
  - API 文档

### 配置文件

- **`pyproject.toml`** - Python 项目配置（Poetry）
- **`.github/workflows/`** - CI/CD 配置
- **`.gitignore`** - Git 忽略规则

### 其他

- **`examples/`** - 使用示例
- **`benchmarks/`** - 性能基准
- **`scripts/`** - 辅助脚本

## 后续步骤

根据[开发计划](docs/DEVELOPMENT_PLAN.md)，将按照以下顺序创建：

1. **第1周**：创建项目结构、Protobuf Schema
2. **第2周**：实现加密模块
3. **第3-5周**：实现存储引擎
4. **第6-7周**：实现拓扑算法
5. **第8-9周**：集成测试
6. **第10-12周**：性能优化、云端实现、发布

详见[开发计划文档](docs/DEVELOPMENT_PLAN.md)。
