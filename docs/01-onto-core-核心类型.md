# onto-core —— 核心类型模块

> 定位（Cargo.toml:6）："Core types, traits, and error definitions for OntoDB"。
> 源码规模：约 6,387 行，15 个公开模块。它是整个 workspace 的**依赖叶子/底座**，不依赖任何内部 crate。

---

## 1. 职责与边界

onto-core 回答一个问题：**数据如何被表示与编码**，不涉及持久化（LSM）、查询解析或网络。它提供：

- 统一实体身份 `EntityId`（多模态锚点，多租户强隔离）
- 类型化值 `OntoValue`、存储条目 `Entry`、错误 `CoreError`
- 免反序列化的二进制行格式 `BinaryRow`
- 一整套自包含的数据结构与算法：全文 BM25、空间几何与 R*tree、时空四叉树、时间序列、多媒体哈希、血缘追踪

所有算法仅依赖标准库 + serde/base64，**无重型外部依赖**——适合裁剪后在边缘端部署。

---

## 2. 对外核心类型

顶层重导出见 `src/lib.rs:47-50`：

| 类型 | 位置 | 作用 |
|------|------|------|
| `EntityId` | `entity.rs:36-149` | 统一实体身份，`{namespace}::{class}::{pk}` 三段式，三字段强制必填；派生 LSM key / 顶点 ID / 类前缀 |
| `OntoValue` | `value.rs:12-47` | 类型化值枚举（Null/Bool/Int64/Float64/String/Bytes/Array/Object），bincode 编解码，支持从 serde_json 转换 |
| `CoreError` / `Result<T>` | `error.rs:10-56` | 全工作区共用错误类型：Io/Serialization/Corruption/ChecksumMismatch/Inference/OntologyInconsistent/TransactionConflict 等 |
| `Entry` / `EntryKind` / `Key` / `Value` / `SeqNo` / `Bytes` / `Timestamp` | `types.rs:7-59` | 存储引擎条目结构，`Entry::put/delete`、墓碑判定；`SeqNo=u64` 用于 MVCC/WAL 排序 |
| `BinaryRow<'a>` | `binary_row.rs:39-311` | 免 JSON 反序列化的紧凑行访问器，O(1) 字段查找 |

各功能模块内的关键公开类型：

- **binary_fp.rs**：`bytes_to_b64` / `packed_bits_to_f32` / `f32_to_packed_bits` / `json_to_f32_vec`（`:14-75`）——位压缩指纹/Embedding 与 HNSW 对接。
- **fulltext.rs**：`FullTextIndex`、`IndexedDocument`、`PostingList`、`Bm25Config`、`SearchResult`（`:12-68`）。
- **geo.rs**：`Coord`、`Geometry`（Point/LineString/Polygon/MultiPolygon）、DE-9IM 八关系、Haversine、WKB 编解码（`:20-538`）。
- **geohash_index.rs**：`GeohashIndex`（BTreeMap + 反向 HashMap，`:27-34`）。
- **rtree.rs**：`BBox`、`RTree`（`:35-536`）。
- **spatiotemporal.rs**：`STPoint`、`STQuery`、`STIndex`（四叉树 + 时间线，`:32-332`）。
- **sttrl.rs**：`SpatioTemporalEvent`、`RuleType`（10 类）、`Rule`、`RuleEngine`（`:42-500`）——时空规则引擎。
- **time_series.rs**：纳秒级 `Timestamp`/`DataPoint`、`WindowType`、`Aggregation`、`dtw_distance`、`detect_anomalies`（`:23-362`）。
- **multimedia.rs**：`MediaType`、`MediaMetadata`、`MediaRegistry`、`content_hash`（内含自实现 SHA-256，`:11-286`）。
- **lineage.rs**：`LineageSource`、`LineageRecord`、`LineageTracker`、`record_write`/`record_inference`（`:14-208`）。

---

## 3. 核心机制

### 3.1 BinaryRow 二进制行格式（零拷贝过滤）

布局（`binary_row.rs:4-22`）：

```
[num_fields:u16 LE][field_name_lens:u16×n][type_tags:u8×n][name_bytes][value_bytes]
```

- `parse()` 单次遍历预计算每个字段的 `value_size`（`:94-125`），后续访问零解析开销。
- `find_field()` 用 `OnceLock<HashMap>` 惰性建索引（`:163-177`），首次查找才付 O(n) 代价。
- `to_map_projected()` 只转换被 SELECT 的列并保留 `__class__`（`:217-235`），服务于本体层级下推。
- `binary_bytes_to_map()` 是完全跳过中间结构的单遍最快路径（`:313-441`）。
- `binary_value_eq_serde` / `binary_value_ord_serde`（`:613-669`）供 onto-query 做谓词下推比较——**这是"二进制行零拷贝过滤"的技术落点**。

### 3.2 全文检索（BM25 + 中英文混合分词）

- 倒排索引 `term → PostingList{doc_freq}` + 正向索引 `doc_id → IndexedDocument`（`fulltext.rs:57-68`）。
- IDF 用 `ln((N-df+0.5)/(df+0.5)+1)`，BM25 打分见 `:153-156,221-225`。
- `tokenize()`（`:228-266`）：ASCII 词小写化，CJK 单字成 token 并额外生成 bigram，无需外部分词器即可支持中文（测试验证"温度传感器" → 温/度/温度/传感/感器）。

### 3.3 空间与时空索引

- **R*tree**（`rtree.rs`）：`MAX_ENTRIES=25`；`choose_subtree` 按最小面积增量选子树（`:282-305`）；溢出按中心 X 坐标二分分裂（`:374-390`）；`knn_best_first` 用双堆实现 best-first 剪枝（`:425-531`）。
- **GeohashIndex**（`geohash_index.rs`）：BTreeMap 前缀扫描 + 半径/包围盒搜索，半径映射为 geohash 精度（`:118-135,285-333`）。
- **STIndex 时空索引**（`spatiotemporal.rs`）：四叉树（`[SW,SE,NW,NE]`）+ 每节点 `BTreeMap<i64, Vec<STPoint>>` 时间线；查询用 `BTreeMap::range` 做时间剪枝 + 空间过滤（`:261-286`）。

### 3.4 时间序列与 DTW

- `WindowType::{Tumbling,Hopping,Session}`、`Aggregation` 六种聚合（`:229-301`）。
- `dtw_distance` 用 Sakoe-Chiba band 约束动态规划（`:307-333`），带 10 万点上限的溢出/内存保护（`:313-316`）。
- 异常检测是简化的 Grubbs 检验（>2σ，`:337-362`）。

### 3.5 血缘追踪与多媒体

- `LineageTracker::trace_upstream` 用 DFS + visited 防环 + 深度上限（`:93-148`）。
- `content_hash` 使用**内置纯 Rust SHA-256**（`:181-281`），避免为去重引入外部密码学依赖。

---

## 4. 依赖关系

- **被依赖**（谁依赖它）：`onto-storage`（最重用户，engine.rs/LSM/MVCC/索引/向量广泛引用）、`onto-ontology`（仅 CoreError）、`onto-query`、`onto-server`、`onto-graph`、`onto-edge`、`onto-edge-bin`。
- **依赖谁**：无内部依赖。

---

## 5. 设计亮点

1. **多租户强隔离**：`EntityId` 三段式强制 namespace，同一主键在不同租户下生成不同 LSM key（测试 `entity.rs:272-278`）。
2. **零拷贝行式过滤**：`BinaryRow` 不反序列化整行即可取值，配合投影下推到存储层。
3. **算法全自包含**：R*tree / geohash / 四叉树 / BM25 / SHA-256 全部手写，仅标准库 + serde/base64。
4. **中英文混合分词无需词典**。
5. **防御性工程**：DTW 点数上限、geohash 长度上限、位长不符返回 None、血缘环路检测。
6. **许可证内嵌**：`ONTODB_VERSION/COPYRIGHT/LICENSE_URL`（`lib.rs:54-56`）。

---

## 6. 实现现状与边界

- **rtree.rs**：头注释宣称 forced reinsertion 与最小重叠分裂（`:10-13`），实际是简化的中心 X 二分分裂（`:374-390`）；重插入相关常量为 dead code（`:25-29`），父 MBR 更新为 no-op（`:533-535`）。
- **time_series.rs**：`WindowType::{Hopping,Session}` 已定义但仅实现 tumbling（`:240-269`）；DTW 注释称 O(n) 实为带 band 的 O(n·w)。
- **sttrl.rs**：`DirectionMove` 规则落入默认 `_ => NotTriggered`（`:422`）；`UnusualLocation.tolerance_m` 未使用（`:345-348`）。
- **geo.rs**：WKT 解析仅支持 Point/LineString，Polygon/MultiPolygon 明确省略（`:591`）。
- 这些是"设计意图 > 实现完成度"的典型，写上层文档时不应把相应能力描述为完整可用。
