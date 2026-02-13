# brain-unidag-store

统一认知DAG高性能存储系统

## 📚 文档

- [架构文档](docs/ARCHITECTURE.md) - 系统架构设计、技术选型、接口契约
- [开发计划](docs/DEVELOPMENT_PLAN.md) - 开发路线图、详细计划、交付物清单

## 🎯 项目概述

UniDAG-Store 是一个统一认知DAG（有向无环图）高性能存储系统，专为 brain-domain-agent 等认知计算平台设计。

### 核心特性

- **统一的DAG存储抽象**：支持多模态节点（代码、思维、策略等）
- **高性能持久化**：基于Protobuf + SQLite/PostgreSQL + Zarr
- **端到端加密**：AES-GCM-256加密保护用户隐私
- **确定性拓扑操作**：保证拓扑排序结果的一致性
- **多端部署**：支持PC、移动端（离线）和云端（在线）

## 🚀 快速开始

### 安装

```bash
pip install unidag-store
```

### 基本使用

```python
from unidag_store import EmbeddedUniDAGStore, UnifiedDAG

# 创建本地存储实例
store = EmbeddedUniDAGStore(db_path="./data.db")

# 持久化 DAG
dag = UnifiedDAG(...)
snapshot_id = await store.persist(dag, meta={}, user_id="user123")

# 检索 DAG
retrieved_dag = await store.retrieve(snapshot_id, user_id="user123")

# 查询拓扑排序
topology = await store.query_topology(snapshot_id, user_id="user123")
```

## 📋 技术栈

- **序列化**：Protocol Buffers 3.20+
- **本地数据库**：SQLite 3.38+
- **云端数据库**：PostgreSQL 14.0+
- **特征存储**：Zarr 2.13+
- **加密**：AES-GCM-256（PyCryptodome）
- **异步框架**：asyncio + aiohttp

## 🏗️ 项目状态

当前版本：v0.1.0（开发中）

预计 v1.0 发布：2026年5月

## 🤝 贡献

欢迎贡献！请查看 [贡献指南](CONTRIBUTING.md) 了解如何参与项目开发。

## 📖 许可证

Apache License 2.0 - 详见 [LICENSE](LICENSE) 文件