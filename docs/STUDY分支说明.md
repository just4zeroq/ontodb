# study 分支 —— 学习专用分支

> 本分支从 `master` 切出，专门用于**阅读和学习 OntoDB 源码**。
> 与主分支的区别仅在于：① 新增了架构文档；② 在核心源码中添加了中文学习注释。
> **所有源码改动都是纯注释，不改动任何一行代码逻辑。**

---

## 本分支包含什么

### 1. 架构文档（docs/ 目录，共 12 篇）

| 文档 | 内容 |
|------|------|
| [00-整体架构.md](00-整体架构.md) | 系统定位、6 层分层、模块依赖、统一锚点、关键数据流、部署 |
| [01-onto-core-核心类型.md](01-onto-core-核心类型.md) | EntityId、BinaryRow、空间/全文/时序算法 |
| [02-onto-storage-存储引擎.md](02-onto-storage-存储引擎.md) | LSM、WAL、MVCC、索引、HNSW、TSM |
| [03-onto-ontology-本体推理.md](03-onto-ontology-本体推理.md) | 本体模型、OWL 2 RL 推理、三元组库 |
| [04-onto-query-查询引擎.md](04-onto-query-查询引擎.md) | Parser、优化器、执行器、OntoQL/SPARQL |
| [05-onto-graph-图引擎.md](05-onto-graph-图引擎.md) | 属性图、遍历、图算法、GNN |
| [06-onto-plugin-插件系统.md](06-onto-plugin-插件系统.md) | 插件 Trait、Hook、注册表 |
| [07-onto-server-服务层.md](07-onto-server-服务层.md) | 四协议、认证、限流、审计、语义 API |
| [08-onto-cli-命令行.md](08-onto-cli-命令行.md) | 命令行客户端 |
| [09-onto-edge-边缘计算.md](09-onto-edge-边缘计算.md) | 边缘 SDK、ESP32、三级部署 |
| [10-ontodb-rules-规则服务.md](10-ontodb-rules-规则服务.md) | 独立规则服务、DSL |
| [11-已知边界与技术债.md](11-已知边界与技术债.md) | 全仓「声明 vs 实现」差异汇总 |

### 2. 源码学习注释

在以下 8 个代表文件的模块头添加了「学习导读」中文注释块（以 `--- 学习导读（study 分支添加）---` 标记，便于识别/检索）：

| 模块 | 文件 | 导读主题 |
|------|------|----------|
| core | `crates/onto-core/src/entity.rs` | 统一语义锚点 EntityId |
| core | `crates/onto-core/src/binary_row.rs` | 二进制行零拷贝格式 |
| storage | `crates/onto-storage/src/lsm/wal.rs` | WAL 格式与恢复 |
| storage | `crates/onto-storage/src/engine.rs` | commit_txn 三阶段锁提交 |
| ontology | `crates/onto-ontology/src/rules.rs` | 7 条 OWL 推理规则 |
| ontology | `crates/onto-ontology/src/reasoner.rs` | 不动点推理主流程 |
| query | `crates/onto-query/src/optimizer/planner.rs` | 查询计划与代价优化 |
| graph | `crates/onto-graph/src/traversal.rs` | 图遍历三路分流 |
| server | `crates/onto-server/src/main.rs` | 服务启动与四协议组装 |

> 检索所有学习注释：`grep -rn "学习导读" crates/`

---

## 推荐学习路线

1. **先读架构总览**：`docs/00-整体架构.md` —— 建立全局认知（分层、依赖、数据流）。
2. **自底向上读模块**：core → storage → ontology/graph → query → server，配合对应的模块文档。
3. **对照源码导读**：每读一个模块，打开上表中对应文件顶部的「学习导读」，带着目标读代码。
4. **注意实现边界**：`docs/11-已知边界与技术债.md` 列出了"文档宣称 vs 实际实现"的差异，避免把规划能力误当已实现。

---

## 与主分支同步

study 分支用于学习，如需吸收 master 的新改动：

```bash
git checkout study
git merge master      # 或 git rebase master
```

注意：若 master 改动了上述带注释的文件，合并时可能产生冲突（因为我们在文件头加了注释块），手动保留注释即可。

---

## 说明

- 本分支的文档基于 OntoDB v0.6.2 源码编写，关键结论附「文件:行号」佐证。
- 学习注释仅为辅助理解而添加，不改变程序行为；`cargo check` 已验证编译通过。
