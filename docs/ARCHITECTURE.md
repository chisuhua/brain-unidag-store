# UniDAG-Store：统一认知DAG高性能存储系统架构文档

> **文档版本**：v2.2  
> **最后更新**：2026年2月13日  
> **文档状态**：技术评审通过  
> **核心原则**：Protobuf仅作元数据容器；保持DAG语义纯粹性；确定性拓扑操作；明确职责边界；端到端加密

---

## 目录

1. [项目概述](#1-项目概述)
2. [核心变更摘要](#2-核心变更摘要)
3. [Protobuf Schema 设计](#3-protobuf-schema-设计)
4. [存储服务接口契约](#4-存储服务接口契约)
5. [部署模式规范](#5-部署模式规范)
6. [安全与隐私设计](#6-安全与隐私设计)
7. [职责边界声明](#7-职责边界声明)
8. [性能模型与KPI](#8-性能模型与kpi)
9. [兼容性与迁移策略](#9-兼容性与迁移策略)
10. [风险与缓解](#10-风险与缓解)

---

## 1. 项目概述

UniDAG-Store 是一个**统一认知DAG高性能存储系统**，专为 brain-domain-agent 等认知计算平台设计，提供：

- **统一的DAG存储抽象**：支持多模态节点（代码、思维、策略等）
- **高性能持久化**：基于Protobuf + SQLite/PostgreSQL + Zarr
- **端到端加密**：AES-GCM-256加密保护用户隐私
- **确定性拓扑操作**：保证拓扑排序结果的一致性
- **多端部署**：支持PC、移动端（离线）和云端（在线）

### 1.1 设计约束

基于 Protobuf 2GB 硬限制、DAGformer 实际需求以及 brain-domain-agent 集成契约，本系统设计遵循以下约束：

- **Protobuf 2GB 限制**：单个 DAG 消息体不超过 2GB（约10万节点）
- **确定性要求**：拓扑排序结果必须在多次调用间保持一致
- **离线优先**：PC/移动端无需网络依赖即可运行
- **职责分离**：存储层不处理业务逻辑，仅负责数据持久化

---

## 2. 核心变更摘要

本版本在 v2.1 基础上，重点修正 **5 项关键缺陷**，确保与 brain-domain-agent 无缝集成：

| 变更类型 | 问题描述 | 修正方案 | 优先级 |
|----------|----------|----------|--------|
| **字段类型修正** | `version` 字段为 string 导致版本比较失效 | 修正为 `int32`，支持 `>`/`<` 比较 | 🔴 高 |
| **快照ID规范** | `snapshot_id` 格式未定义导致跨模块解析失败 | 规范为 `dag_v{version}_{uuid8}` | 🔴 高 |
| **职责边界修正** | 拓扑排序实现归属错误 | 明确算法实现属 UniDAG-Store，brain-domain-agent 仅消费 | 🟠 中 |
| **加密增强** | 特征向量未加密导致合规风险 | 补充 Zarr 块级 AES-GCM-256 加密 | 🟠 中 |
| **离线能力分级** | "无网络依赖"表述模糊导致用户体验风险 | 明确三级离线能力（完全/降级/不可用） | 🟢 低 |

---

## 3. Protobuf Schema 设计

### 3.1 UnifiedDAG 消息定义

```protobuf
message UnifiedDAG {
  string dag_id = 1;                  // 全局唯一ID（UUID）
  string name = 2;                    // DAG名称
  string description = 3;             // 人类可读描述
  string source = 4;                  // 来源
  repeated string tags = 5;           // 标签
  repeated DAGNode nodes = 6;         // 节点列表（按topo_ranks顺序存储）
  repeated DAGEdge edges = 7;         // 边列表
  repeated string root_node_ids = 8;  // 根节点ID列表（按字典序排序）
  google.protobuf.Timestamp created_at = 9;
  map<string, string> meta = 10;      // 扩展元数据
  repeated int32 topo_ranks = 11;     // 节点拓扑序排名
  string domain = 12;                 // 领域标识（用于brain-domain-agent路由）
  int32 version = 13;                 // ✅ 修正：DAG 版本号（初始为1，每次变更+1）
}
```

> **重要说明**：
> - `version` 是 **DAG 内容版本**（业务数据变更追踪），非 Protobuf Schema 版本（存储格式版本）
> - `int32` 类型支持数值比较（`if new_dag.version <= old_dag.version: raise ContractViolation`）
> - 依据：minion.txt §3.2 要求"DAG 版本号用于追踪变更，技能执行必须验证版本递增"

### 3.2 DAGNode 消息定义

```protobuf
message DAGNode {
  string id = 1;
  string type = 2;
  string content = 3;
  // ✅ 格式规范：
  //   • 本地存储: "local://features.zarr/{node_id}"
  //   • 云端存储: "s3://unidag-bucket/features.zarr/{node_id}"
  string feature_ref = 4;
  int32 depth = 5;
  string modality = 6;
  BoundingBox bbox = 7;
  float confidence = 8;
  string reasoning_role = 9;
  CodeNodeAttributes code_attrs = 10;
  ThoughtNodeAttributes thought_attrs = 11;
  StrategyNodeAttributes strategy_attrs = 12;
  map<string, string> custom_attrs = 13;
}
```

### 3.3 DAGEdge 消息定义

```protobuf
message DAGEdge {
  string from_node_id = 1;
  string to_node_id = 2;
  string type = 3;
  float weight = 4;
  // direction 恒为 "forward"，仅用于兼容性（v1.0 组件）
  string direction = 5 [deprecated = true, default = "forward"];
  map<string, string> custom_attrs = 6;
}
```

### 3.4 节点属性定义

```protobuf
message BoundingBox {
  int32 x = 1;
  int32 y = 2;
  int32 width = 3;
  int32 height = 4;
}

message CodeNodeAttributes {
  string language = 1;
  string syntax_tree = 2;
  repeated string dependencies = 3;
}

message ThoughtNodeAttributes {
  string thought_type = 1;
  float confidence = 2;
  repeated string related_concepts = 3;
}

message StrategyNodeAttributes {
  string strategy_type = 1;
  float priority = 2;
  map<string, string> parameters = 3;
}
```

---

## 4. 存储服务接口契约

### 4.1 IDAGStore 接口定义（v1.0）

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, List

class IDAGStore(ABC):
    """UniDAG-Store 存储服务接口契约"""
    
    @abstractmethod
    async def persist(self,
                     dag: UnifiedDAG,
                     meta: Dict[str, Any],
                     user_id: str) -> str:
        """
        持久化 DAG 快照。
        
        Args:
            dag: 要存储的 DAG 对象（dag.version 必须 > 0）
            meta: 附加元数据（如技能上下文）
            user_id: 用户 ID，用于多租户隔离
            
        Returns:
            snapshot_id: 格式为 "dag_v{version}_{uuid8}"
                        示例: "dag_v3_a1b2c3d4"（version=3, uuid 前8位=a1b2c3d4）
                        
        Note:
            version 必须与 dag.version 一致，用于快照链式追踪
            
        Raises:
            StorageQuotaExceeded: 存储配额超限
        """
        pass
    
    @abstractmethod
    async def retrieve(self,
                      snapshot_id: str,
                      user_id: str) -> UnifiedDAG:
        """
        检索 DAG 快照。
        
        Args:
            snapshot_id: 快照唯一标识（格式: "dag_v{version}_{uuid8}"）
            user_id: 用户 ID，用于权限校验
            
        Returns:
            UnifiedDAG 对象
            
        Raises:
            SnapshotNotFoundError: 快照不存在或无权访问
        """
        pass
    
    @abstractmethod
    async def query_topology(self, snapshot_id: str, user_id: str) -> List[str]:
        """
        返回确定性拓扑排序后的节点 ID 列表。
        
        Returns:
            节点ID列表，顺序满足：
            • 多根DAG：根节点按 root_node_ids 字典序优先
            • 同层节点：按ID字典序排序
            • 保证多次调用返回完全一致结果（确定性）
        """
        pass
    
    @abstractmethod
    async def list_snapshots(self,
                            user_id: str,
                            dag_id: Optional[str] = None,
                            limit: int = 100) -> List[Dict[str, Any]]:
        """
        列出用户的快照列表。
        
        Args:
            user_id: 用户 ID
            dag_id: 可选，过滤特定 DAG 的快照
            limit: 返回结果数量限制
            
        Returns:
            快照元数据列表
        """
        pass
    
    @abstractmethod
    async def delete_snapshot(self, snapshot_id: str, user_id: str) -> bool:
        """
        删除 DAG 快照。
        
        Args:
            snapshot_id: 快照 ID
            user_id: 用户 ID
            
        Returns:
            是否成功删除
            
        Raises:
            SnapshotNotFoundError: 快照不存在
            PermissionDeniedError: 无权删除
        """
        pass
```

### 4.2 标准异常定义

```python
class SnapshotNotFoundError(Exception):
    """快照不存在或无权访问"""
    pass

class StorageQuotaExceeded(Exception):
    """存储配额超限"""
    pass

class VersionContractViolation(Exception):
    """技能执行违反版本契约：新DAG版本未递增"""
    pass

class PermissionDeniedError(Exception):
    """权限拒绝"""
    pass
```

### 4.3 与 brain-domain-agent 集成点

| 集成场景 | 调用方 | 接口 | 关键契约 |
|----------|--------|------|----------|
| **技能执行前** | brain-domain-agent | `retrieve(snapshot_id, user_id)` | 验证 `dag.version` 用于不变性检查 |
| **技能执行后** | brain-domain-agent | `persist(new_dag, meta, user_id)` | 要求 `new_dag.version = old_dag.version + 1` |
| **DAGformer推理** | brain-domain-agent | `query_topology(snapshot_id, user_id)` | 依赖确定性拓扑序生成 attention_mask |
| **快照链追踪** | brain-domain-agent | 解析 `snapshot_id` | 从 `dag_v3_a1b2c3d4` 提取 version=3 |

---

## 5. 部署模式规范

### 5.1 三端部署矩阵

| 部署场景 | 存储实现 | 数据位置 | 网络依赖 | 责任方 |
|----------|----------|----------|----------|--------|
| **PC/移动 APP** | `EmbeddedUniDAGStore`<br>(SQLite + Zarr) | 本地设备 | ❌ 无 | UniDAG-Store 项目组 |
| **浏览器** | `CloudUniDAGStore`<br>(PostgreSQL + S3) | 云端服务器 | ✅ 必需 | UniDAG-Store 项目组 |
| **brain-domain-agent** | 通过 `IDAGStore` 接口调用 | 透明 | 透明 | brain-domain-agent 项目组 |

### 5.2 离线能力分级

> 依据：第二大脑 §9.2 "离线优先设计" 要求明确能力边界

| 能力级别 | 领域示例 | 可用性 | 用户提示 |
|----------|----------|--------|----------|
| ✅ **完全离线** | C++/论文领域 | 100% 功能可用 | 无提示 |
| ⚠️ **降级可用** | 社交/购物领域 | 基础功能可用，高级功能禁用 | "高级分析需联网" |
| ❌ **不可用** | 生物/数据科学 | 核心功能不可用 | "此功能需联网使用" |

### 5.3 关键声明

- **浏览器场景**：UniDAG-Store **仅以服务端形式运行**，不提供 WASM 运行时。前端通过 gRPC/HTTP API 访问。
- **本地场景**：UniDAG-Store 以库形式嵌入 APP，数据经端侧加密后存储于设备本地。

---

## 6. 安全与隐私设计

### 6.1 加密方案矩阵

| 数据类型 | 场景 | 加密方案 | 密钥管理 | 合规依据 |
|----------|------|----------|----------|----------|
| **DAG结构** | 本地 | AES-GCM-256 | Passphrase 派生（PBKDF2-HMAC-SHA256） | manger.txt §3.1 |
| **特征向量** | 本地 | ✅ **Zarr块级AES-GCM-256** | 同上 | GDPR Art. 32 |
| **DAG结构** | 云端 | 传输层：TLS 1.3<br>静态：AES-256 (S3 SSE) | 服务端KMS | GDPR Art. 32 |
| **特征向量** | 云端 | 同上 | 同上 | — |

### 6.2 特征向量加密实现

```python
import numpy as np
import zarr
from Crypto.Cipher import AES

def encrypt_zarr_features(zarr_path: str, key: bytes):
    """
    加密 Zarr 存储的特征向量（按块加密）
    依据: manger.txt §3.1 "用户记忆图谱端侧加密"
    
    Args:
        zarr_path: Zarr 存储路径
        key: 256位加密密钥
    """
    zarr_store = zarr.open(zarr_path, mode='r+')
    
    for chunk_key in zarr_store.chunks.keys():
        # 读取原始块
        chunk_data = zarr_store.chunks[chunk_key][:]
        
        # AES-GCM 加密（含认证标签）
        cipher = AES.new(key, AES.MODE_GCM)
        ciphertext, tag = cipher.encrypt_and_digest(chunk_data.tobytes())
        
        # 写回加密数据（保留元数据结构）
        encrypted_payload = cipher.nonce + tag + ciphertext
        zarr_store.chunks[chunk_key] = np.frombuffer(
            encrypted_payload,
            dtype=np.uint8
        ).reshape(chunk_data.shape)
```

> **合规说明**：特征向量可能包含生物特征（人脸/声纹嵌入）或行为模式（点击流），必须与 DAG 结构同等加密，否则违反 GDPR "适当技术措施" 要求。

### 6.3 密钥派生方案

```python
from hashlib import pbkdf2_hmac
import os

def derive_encryption_key(passphrase: str, salt: bytes = None) -> tuple[bytes, bytes]:
    """
    从用户口令派生加密密钥
    
    Args:
        passphrase: 用户口令
        salt: 盐值（可选，如为None则随机生成）
        
    Returns:
        (key, salt): 256位密钥和盐值
    """
    if salt is None:
        salt = os.urandom(32)
    
    key = pbkdf2_hmac(
        'sha256',
        passphrase.encode('utf-8'),
        salt,
        iterations=100000,  # OWASP 推荐
        dklen=32  # 256位
    )
    
    return key, salt
```

---

## 7. 职责边界声明

为防止职责蔓延，明确划分边界：

| 模块 | 职责 | 非职责 |
|------|------|--------|
| **UniDAG-Store** | • DAG 持久化/检索（含特征向量）<br>• 版本快照管理（snapshot_id 生成）<br>• **拓扑排序算法实现**（确定性 Kahn）<br>• 拓扑查询接口（`query_topology()`）<br>• 多租户隔离与权限校验 | • 语义验证（如 C++ 节点完整性）<br>• 执行状态管理（status/failure_reason）<br>• 业务逻辑校验（技能执行合法性） |
| **brain-domain-agent** | • 语义验证（SkillExecutor）<br>• 执行状态管理（WorkflowEngine）<br>• 业务逻辑校验<br>• **使用拓扑序驱动技能调度**<br>• 调用 IDAGStore 接口 | • 存储实现细节（SQLite/S3）<br>• **拓扑排序算法实现**<br>• 加密密钥管理 |

> **关键原则**：  
> 1. 拓扑排序是 **DAG 结构的固有属性**（Wikipedia DAG 定义：可拓扑排序 ⇔ 无环），算法实现必须归属存储层  
> 2. brain-domain-agent 仅**消费**拓扑序（用于 DAGformer attention_mask 或技能调度）  
> 3. 存储层 **不验证业务语义**，仅保证 DAG 结构完整性（无环、字段合法）

---

## 8. 性能模型与KPI

| 操作 | 目标值 | 测试条件 | 依据 |
|------|--------|----------|------|
| **Protobuf序列化** | <100ms | 10万节点 | 避免2GB限制（Protobuf `ToCachedSize` 强制转为 `int`） |
| **拓扑验证** | <1μs/边 | O(1)查表 | `topo_ranks`数组 |
| **特征加载** | <10ms | 单节点768维 | Zarr内存映射+解密 |
| **批量插入** | 30万节点/秒 | DuckDB COPY | 工业基准 |
| **确定性拓扑排序** | <100ms | 100万节点 | 优化Kahn算法（Cormen §22.4） |
| **Zarr块解密** | <2ms/块 | 1000节点特征块 | AES-NI硬件加速 |

---

## 9. 兼容性与迁移策略

### 9.1 Protobuf Schema 向后兼容

| 字段 | v2.0 状态 | v2.2 处理 | 迁移策略 |
|------|-----------|-----------|----------|
| `version` | 不存在 | 新增 `int32` | 缺失时默认值=1 |
| `domain` | 不存在 | 新增 `string` | 缺失时默认值="unknown" |
| `direction` | 存在 | 标记 deprecated | 保留字段但忽略值 |

### 9.2 快照链迁移

```python
def migrate_snapshot_v20_to_v22(snapshot_id_v20: str) -> str:
    """
    将 v2.0 快照ID迁移至 v2.2 格式
    
    Args:
        snapshot_id_v20: v2.0 快照ID（无版本信息）
        
    Returns:
        v2.2 格式的快照ID
        
    Example:
        输入: "abc123" (v2.0 无版本信息)
        输出: "dag_v1_abc123" (version=1 作为初始版本)
    """
    uuid8 = snapshot_id_v20[:8]
    return f"dag_v1_{uuid8}"
```

> **说明**：v2.0 存量数据通过 `version=1` 作为初始版本导入，后续变更按 `+1` 递增，确保快照链连续性。

---

## 10. 风险与缓解

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| **版本契约断裂** | 高 | 致命 | `version` 修正为 int32；集成测试验证比较逻辑 |
| **快照链断裂** | 中 | 高 | 规范 snapshot_id 格式；提供迁移工具 |
| **职责边界混淆** | 中 | 中 | 通过接口契约 + 联合评审强制隔离 |
| **特征向量泄露** | 低 | 高 | 强制 Zarr 块级加密；安全审计 |
| **离线体验降级** | 低 | 中 | 明确三级能力分级；用户提示优化 |

---

## 附录A：参考文献

1. Protobuf 2GB 限制：[Protocol Buffers Documentation](https://developers.google.com/protocol-buffers)
2. Kahn 算法：Cormen, T. H., et al. (2009). "Introduction to Algorithms" §22.4
3. GDPR 第32条：[General Data Protection Regulation](https://gdpr-info.eu/)
4. OWASP 密钥派生：[OWASP Key Management Cheat Sheet](https://cheatsheetseries.owasp.org/)

---

## 附录B：修订历史

| 版本 | 日期 | 修订内容 | 依据 |
|------|------|----------|------|
| v2.0 | 2026-02-11 | 基础架构设计 | — |
| v2.1 | 2026-02-12 | 新增接口契约/部署/安全/职责边界 | brain-domain-agent 反馈 |
| **v2.2** | **2026-02-12** | **关键修正**：<br>• `version` 字段类型修正为 int32（DAG内容版本）<br>• snapshot_id 格式规范<br>• 拓扑排序职责归属修正（算法实现属存储层）<br>• 特征向量加密方案补充（Zarr块级）<br>• 离线能力分级明确 | **minion.txt §3.2**<br>**第二大脑 §6.1/§9.2**<br>**GDPR Art. 32** |

---

**文档批准**：  
- DAG-Centric AI 系统工程组：✓  
- brain-domain-agent 项目组：✓  
- 安全与合规委员会：✓  
- 日期：2026年2月13日
