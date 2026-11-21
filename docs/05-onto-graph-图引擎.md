# onto-graph —— 图引擎模块

> 定位（Cargo.toml:6）："Graph data model and traversal engine for OntoDB"。
> 源码规模：约 9,611 行（store.rs 单文件 3,089 行），13 个公开模块。依赖 onto-core + onto-storage。

---

## 1. 职责与边界

onto-graph 是**纯图计算引擎层**：属性图数据模型、CRUD、遍历、模式匹配、图推理、图算法、GNN、分布式模拟、流式计算、压缩、可视化。它与 onto-ontology 是两套独立的图/本体能力（包括两套 `Reasoner`）。

依赖 `onto-storage`（EngineSet/LsmEngine）做持久化，不依赖 query/server。

---

## 2. 图数据模型（`model.rs`）

属性图（Property Graph）模型：

- `PropValue`（`:11-18`）：Null/Bool/Int/Float/String/List；手写 PartialOrd 仅同类型可比。
- `PropertyMap = HashMap<String, PropValue>`。
- `Vertex { id, labels: Vec<String>, properties }`（`:59-66`）：**多标签**。
- `Edge { id, from, to, label: String, properties }`（`:85-96`）：**单标签、有向**。
- 顶点/边 ID 均为 String，复用 `EntityId` 的 `{namespace}::{class}::{pk}` 格式（`store.rs:1525-1536`），与关系实体共享标识空间。

---

## 3. GraphStore 存储架构（`store.rs`，核心）

### 3.1 字段与锁顺序契约（架构重点）

`GraphStore`（`:114-158`）以 `parking_lot::RwLock` 保护多个字段：

| 类别 | 字段 |
|------|------|
| 主数据 | `vertices`、`edges`（single source of truth） |
| 索引 | `label_index`、`edge_label_index`、`edge_prop_index` |
| 整数索引 | `id_to_idx`、`idx_to_id`、`adj_out`/`adj_in`（邻接表存 `(邻居索引, 边ID)`） |
| 缓存 | `cache_state`、`loading_locks`、`pending_changes`、`partial_loaded` |

**锁顺序契约**（`:100-113`）：显式规定 1→2→3→4→5→6→7，防死锁；`get_or_create_idx` 一次性持有 4 把锁保证整数索引"全有或全无"（`:464-487`）；`add_edge` 持 vertices 读锁覆盖整个内存更新阶段修复 TOCTOU（`:829-833`）。**这是全项目最值得称道的并发设计**。

### 3.2 双层存储：内存 + LSM 持久化

- 持久化键：`__graph_v__{vertex_id}` / `__graph_e__{edge_id}`（`:8-10`）。
- 命名空间路由：顶点按 ID 首段选引擎（`:495-502`），边按 `edge.from` 路由（`:531,566-573`）。
- `load_from_engine()`（`:264-456`）：**两遍流式恢复**（先顶点后边，页 10,000）+ 双预算约束（100 万顶点 / 2GB）+ 无锁本地建索引后一次性替换，避免加载期长期持锁。

### 3.3 关系缓存子系统（独立于图 CRUD 的第二套机制）

把 LSM 关系表按需载入图缓存：

- 状态机 `CacheStatus`：`NotLoaded → Loading → Loaded/Failed`，写操作置 Invalidated（`:23-36,1684-1704`）。
- 三道保护：加载互斥锁（`:1795-1801`）、60s 超时 + 10s 失效防抖、内存上限校验（`:1919-1926`）。
- **全量加载** `load_relation_from_lsm`（`:1812-1882`）与**按顶点懒加载** `ensure_vertex_edges`（`:1999-2097`，优先走 LSM source/target 二级索引）——后者已被 onto-query 真实调用（`executor.rs:12159,12447,13232`）。
- LRU 淘汰 + 命中率统计（`:1761-1792,2154-2197`）。

### 3.4 整数索引快速遍历原语

`bfs_fast`、`bfs_parallel`（Rayon + AtomicU64 位图去重，`:1306-1372`）、`bfs_fast_with_parents` + `reconstruct_path`（`:1383-1455`）——上层遍历/算法/推理统一复用的内层。

---

## 4. 遍历、模式匹配与推理

- **TraversalEngine**（`traversal.rs:90-733`）：`Direction::{Out,In,Both}`；`traverse_bfs` 三路分流（有标签且有索引→索引路径；有标签无索引→过滤路径；无标签→整数 BFS），DFS 是唯一同时填充 edges 和 paths 的遍历；递归深度上限 1000。
- **子图同构**（`pattern.rs:46-231`）：回溯匹配，按标签取候选 + 已匹配顶点的边约束校验；无选择性排序启发式，最坏指数级。
- **图推理**（`reasoning.rs`）：独立三元组规则引擎，7 条 RDFS/OWL 规则（subClassOf 传递、instanceOf、symmetric、transitive、inverseOf、domain、range，`:138-317`），前向链结果**写回 GraphStore** + 反向链证明；带溯源的 `InferenceResult`。服务端 `/api/graph/reasoning/*` 使用此推理器（`http.rs:3928,3976`）。

---

## 5. 图算法与其他模块

- **analytics.rs**（1,253 行）：connected_components、degree_distribution、**pagerank**（悬空节点质量重分配 + L1 收敛，`:177-263`，被 onto-query 调用）、average_clustering_coefficient、betweenness_centrality（Brandes O(VE)）、subgraph_extraction、**dijkstra**（`OrdF64` NaN 安全包装 + 惰性删除最小堆，`:534-648`）。
- **gnn.rs**（551 行）：纯 Rust 前向推理，`GcnLayer` 严格实现 `H' = ReLU(ÂHW+b)`（Â=D^-1/2(A+I)D^-1/2，`:163-197`），`.npy` 导出；无训练（loss 恒空），稠密 O(N²) 邻接。
- **distributed.rs**（513 行）：一致性哈希（150 虚拟节点/分区）+ ghost 顶点 + 分布式 BFS + rebalance——**纯进程内模拟，无 RPC/副本**。
- **streaming.rs + streaming_compute.rs**：GraphStream 有界队列事件管道 + StreamingComputeEngine 增量算法（并查集、简化 PageRank、窗口）——两者未连接。
- **compressed.rs**：CsrGraph（CSR + 标签字典编码）——与 GraphStore 无转换接口，独立工具。
- **visualization.rs**：to_dot / to_d3_json / to_cytoscape_json / to_mermaid，服务端四端点已接通（`:61-441`）。

---

## 6. 依赖关系

- **依赖**：onto-core（EntityId、BinaryRow、CoreError）、onto-storage（EngineSet、LsmEngine）、外部 rayon。
- **被依赖**：onto-query（图查询执行器 + 图算法）、onto-server（启动时构造全局 GraphStore，约 20 个 `/api/graph/*` 端点）。

---

## 7. 设计亮点

1. **显式锁顺序契约 + 原子多锁更新**——并发安全的教科书式实践。
2. **整数索引遍历内层** + Rayon 并行 BFS，字符串仅在边界转换。
3. **双预算流式恢复** + 无锁建索引，面向大图冷启动保护。
4. **关系缓存状态机 + 三级防御 + 全量/懒加载双模式**，已与查询执行器集成。
5. **EntityId 统一命名空间**实现多租户图持久化。
6. **遍历分层优化**（标签索引路径 O(matching)）。
7. **推理带溯源**且写回图，可解释端点直接消费。
8. **Dijkstra 的 NaN 安全包装**、PageRank 悬空节点处理。
9. **GNN 纯 Rust 前向推理 + NumPy 导出**，不引入 ML 运行时。

---

## 8. 实现现状与边界（较多，需重点关注）

1. **enterprise feature 是 vestigial**：所有"企业"能力（betweenness/dijkstra/DistributedGraph/GCN/Reasoner 等）**无条件编译**，`enterprise = []` 为空。
2. `record_edge_change/apply_pending_changes` 增量缓存 API 未接入 CRUD 主路径，仅示例使用。
3. `StreamingPageRank` 是自承 "doesn't track incoming edges" 的启发式（`:256`），非真正增量 PageRank。
4. 三种窗口（Tumbling/Sliding/Session）在 trim_buffer 用同一 duration，slide/gap 未实现。
5. 分布式图是进程内模拟：ghost 占位、rebalance 只复制不删除且与哈希路由不一致；server `/api/graph/distributed/status` 返回静态 `"mode":"standalone"`。
6. GNN：EmbeddingConfig/node2vec 无实现，无训练，稠密 O(N²)。
7. 子图同构无选择性排序、无属性/否定约束。
8. CsrGraph 与 GraphStore 无转换接口，无外部使用。
9. 优化后的 BFS 路径刻意留空 edges/paths；只有 DFS 完整填充。
10. 推理器 `inferred` 缓存从未写入，depth 恒 0，priority 未参与排序。
11. 服务端 `/api/graph/streaming/event` 解析事件后丢弃，`/stats` 为静态 stub。
12. 删除顶点后 `idx_to_id` 留空串墓碑、邻接槽不压缩，靠 `get_id()==None` 跳过。
13. **领域耦合**：`relation_table_name()` 硬编码 `sembio` 命名空间（`store.rs:2331-2339`），`normalize_relation_label` 取点分最后一段——写通用文档时应披露此生物医学领域假设。
