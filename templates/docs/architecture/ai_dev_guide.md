# AI 原生研发手册

> 规范来源：`aiweave/docs-spec/04_architecture_overview_spec.md` §10

## 1. 什么是 AI 原生工程

本工程**不是** "一个能用 AI 辅助的项目"，而是**为 AI 而设计**的项目。

> docs/ 是系统的完整蓝图，.claude/skills/ 是 AI 的操作手册。仅凭这两者，AI 可以从零重建整个工程。

## 2. 核心资产一览

| 资产 | 位置 | 作用 |
|------|------|------|
| CLAUDE.md | 项目根 | AI 总入口 |
| Skills | `.claude/skills/` | AI 操作手册 |
| 架构文档 | `docs/architecture/` | 系统级设计 |
| 接口规格 | `docs/api/` | N 个 API 完整定义 |
| Service 层 | `docs/service/` | 业务逻辑伪代码 |
| 数据模型 | `docs/schema/` | DDL + Redis Key |
| 缓存设计 | `docs/cache/` | 缓存策略 |
| 文档索引 | `docs/INDEX.md` | 文档目录 |
| 实现进度 | `docs/BUILD_STATUS.md` | 状态 |

## 3. Skill 体系（19 个）

详见 `.claude/skills/`。

## 4. 研发流程：设计先行

```
①  写 / 改 md
        ↓
②  AI 按 md 生成代码（Skill）
        ↓
③  go build / go test 验证 + 反向同步
```

## 5. 文档同步纪律

详见 `CLAUDE.md` 的「文档同步规则」章节。

## 6. 典型工作流示例

详见项目根 `aiweave/OPERATIONS.md`（如已并入工程，则放本文档）。

## 7. 对研发人员的建议

### 你需要做的
- 写好设计文档
- 审查 AI 输出
- 维护接口契约
- 定期审计

### 你不需要做的
- 手写 GORM model
- 手写 Controller boilerplate
- 手写路由注册
- 手写定时任务框架
- 维护 INDEX.md

### 黄金法则
> 文档写得越精确，AI 生成的代码越正确。把时间花在设计上，而不是编码上。

---

## 8. 约束总清单

> 本节是工程内所有"强制约束"的入口索引。各类约束的详细规则在对应文档；本节只列**约束名 → 文档锚点 → 当前状态**。
>
> 与 `CLAUDE.md 规则 X 范围判定表`的关系：CLAUDE.md 提供"代码路径 → 文档"的映射，本节提供"约束类型 → 文档锚点"的映射。两者互补，CLAUDE.md 是入口，本节是详情。

### 8.1 并发安全约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 共享状态注册表 | [`concurrency_safety.md §2`](concurrency_safety.md) | 🟢 |
| 锁粒度决策 / 加锁顺序 | [`concurrency_safety.md §3`](concurrency_safety.md) | 🟢 |
| Channel 与 Goroutine 池 | [`concurrency_safety.md §4`](concurrency_safety.md) | 🟢 |
| 资源生命周期管理 | [`concurrency_safety.md §5`](concurrency_safety.md) | 🟢 |
| 危险操作清单（含 rule-id） | [`concurrency_safety.md §6`](concurrency_safety.md) + 本文档 §10 | 🟢 |
| 并发测试策略（race / 压测 / leak） | [`concurrency_safety.md §7`](concurrency_safety.md) | 🟢 |

### 8.2 事务与一致性约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 事务模型选型 | [`transaction_design.md §1`](../service/transaction_design.md) | 🟢 |
| 事务边界清单 | [`transaction_design.md §2`](../service/transaction_design.md) | 🟢 |
| 补偿逻辑 / Saga 步骤 | [`transaction_design.md §3`](../service/transaction_design.md) | 🟢 |
| 幂等性设计 | [`transaction_design.md §4`](../service/transaction_design.md) | 🟢 |
| 最终一致性窗口 | [`transaction_design.md §5`](../service/transaction_design.md) | 🟢 |
| 失败路径全景图 | [`transaction_design.md §6`](../service/transaction_design.md) | 🟢 |
| 读写分离消幂等（读判定 + 写执行拆两接口） | [`transaction_design.md §4.4`](../service/transaction_design.md) | 🟢 |
| 弱一致 + 边界兜底（`GREATEST`/`LEAST` 夹断）| [`transaction_design.md §5.2`](../service/transaction_design.md) | 🟢 |
| delta 增量 flush 一致性不变量 | [`transaction_design.md §5.3`](../service/transaction_design.md) | 🟢 |

### 8.3 领域不变量约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 业务规则（必须始终成立） | `docs/service/{module}_service.md §7.1` | 🟢 |
| 隐式约束（代码中不明显） | `docs/service/{module}_service.md §7.2` | 🟢 |
| 业务状态机 | `docs/service/{module}_service.md §7.3` | 🟢 |

### 8.4 伪码标记语法（统一锚点）

| 标记 | 关联约束类 | 状态 |
|------|----------|------|
| `[TXN-START]` / `[TXN-COMMIT]` / `[TXN-ROLLBACK]` | 事务边界 | 🟢 |
| `[SAGA-STEP-N]` / `[COMPENSATE-N]` | Saga 正向 / 补偿 | 🟢 |
| `[IDEMPOTENT-CHECK: key={...}]` | 幂等校验 | 🟢 |
| `[INVARIANT-CHECK: I-N]` | 领域不变量 | 🟢 |
| `[LOCK-ACQUIRE: {lock-name}]` / `[LOCK-RELEASE]` | 锁 | 🟢 |
| `[HOT-PATH]` | 热路径 | 🟢 |
| `[METRIC-EMIT: name{labels}]` | 指标 | 🟢 |
| `[BATCH]` | IO 聚合：循环外批量读 / 写（消灭 N+1） | 🟢 |
| `[PARALLEL]` | IO 并行：聚合器 / 扇出并行编排（消灭串行） | 🟢 |

详细使用规则见 [`aiweave/docs-spec/09 §10.5`](../../../aiweave/docs-spec/09_service_design_spec.md)。

### 8.5 性能合约约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 全局 SLA（P99 / QPS / 内存 / CPU） | [`performance_contract.md §1`](performance_contract.md) | 🟢 |
| 热路径清单 + 禁止操作 | [`performance_contract.md §2`](performance_contract.md) | 🟢 |
| 内存分配预算 / 对象池 | [`performance_contract.md §3`](performance_contract.md) | 🟢 |
| 数据访问性能约束（Redis / MySQL / 缓存命中率） | [`performance_contract.md §4`](performance_contract.md) | 🟢 |
| 背压与容量保护 / 链路超时协调 / 降级层级 | [`performance_contract.md §6`](performance_contract.md) | 🟢 |
| 尾延迟治理（重试预算 + jitter / 对冲幂等 / 自适应过载 / load shedding） | [`performance_contract.md §6.4`](performance_contract.md) | 🟢 |
| 性能回归测试（基准 + P99 偏差 + allocs/op） | [`performance_contract.md §7`](performance_contract.md) + [`testing_design.md §4.9`](../testing/testing_design.md) | 🟢 |
| 生产 profiling 兜底（`/debug/pprof` 持续采集） | [`performance_contract.md §7.4`](performance_contract.md) + [`observability.md §3.3`](observability.md) | 🟢 |

### 8.6 可观测性约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| Metrics 采集范围（HTTP / 健康 / Runtime） | [`observability.md §2`](observability.md) | 🟢 |
| Cardinality 默认上限（≤ 500） | [`observability.md §3.1`](observability.md) | 🟢 |
| Label 禁止清单（不可登记突破） | [`observability.md §3.2`](observability.md) | 🟢 |
| Metric 命名约定 | [`observability.md §4`](observability.md) | 🟢 |
| 服务级告警规则（P0 / P1 / P2） | [`observability.md §5`](observability.md) | 🟢 |
| Metric 集中注册（`helpers/metrics.go`） | [`observability.md §6.1`](observability.md) | 🟢 |

### 8.7 Schema 演进约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 废弃字段注册表 | `docs/schema/database_design.md §8.1` | 🟢 |
| 字段语义变更记录 | `docs/schema/database_design.md §8.2` | 🟢 |
| Schema 兼容性规则（默认值 / 类型兼容 / 软删除窗口） | `docs/schema/database_design.md §8.3` | 🟢 |

### 8.8 安全重构约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 重构前准备清单（变更边界 / 测试覆盖 / 回滚方案 / 兼容期） | `docs/architecture/mvp_rebuild_path.md §11` | 🟢 |
| 重构 5 步模板 | `docs/architecture/mvp_rebuild_path.md §11` | 🟢 |
| AI 行为约束（≤ 200 行 / ≤ 5 commit / 每 commit 测试通过） | `docs/architecture/mvp_rebuild_path.md §11` | 🟢 |
| 灰度切流量（Shadow read → 双写 → 1%/10%/50%/100%） | `docs/architecture/mvp_rebuild_path.md §11.5.4` | 🟢 |

### 8.9 跨服务合约约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 上下游依赖图 | [`cross_service_contract.md §1`](cross_service_contract.md) | 🟢 |
| 上游合约（谁调我 / SLA / 超时 / 重试） | [`cross_service_contract.md §2`](cross_service_contract.md) | 🟢 |
| 下游合约（我调谁 / 超时 / 熔断 / 降级） | [`cross_service_contract.md §3`](cross_service_contract.md) | 🟢 |
| 故障传播矩阵 | [`cross_service_contract.md §4`](cross_service_contract.md) | 🟢 |
| 接口版本管理 | [`cross_service_contract.md §5`](cross_service_contract.md) | 🟢 |

### 8.10 运行时基线（如启用 runtime_profile.md）

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 流量模式 | [`runtime_profile.md §1`](runtime_profile.md) | ⬜ / 🟢 视工程启用 |
| 数据规模 | [`runtime_profile.md §2`](runtime_profile.md) | ⬜ / 🟢 视工程启用 |
| 资源消耗基线 | [`runtime_profile.md §3`](runtime_profile.md) | ⬜ / 🟢 视工程启用 |
| 关键超时链 | [`runtime_profile.md §4`](runtime_profile.md) | ⬜ / 🟢 视工程启用 |

### 8.11 IO 铁律与聚合约束

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 三条 IO 铁律（批量优先 / 禁止 N+1 / 禁止独立串行编排）+ 豁免登记 | [`io_contract.md §2`](io_contract.md) | 🟢 |
| 聚合器模式（收集→批量读→并行回源→单飞→异步写回） | [`io_contract.md §3`](io_contract.md) | 🟢 |
| 并行编排原语（聚合器 / 扇出 / 流水线） | [`io_contract.md §4`](io_contract.md) | 🟢 |
| 数据访问聚合约束（批量读写 / 预加载 map / JOIN vs 多查询） | [`io_contract.md §5`](io_contract.md) | 🟢 |
| 批量写极致（插入/更新混合 → Upsert；大批量装载 → `LOAD DATA`/`COPY`） | [`io_contract.md §5.7`](io_contract.md) | 🟢 |
| 读路径覆盖索引 + ICP + 不过度索引 | [`database_design.md §7.1`](../schema/database_design.md) / [`§7.2`](../schema/database_design.md) | 🟢 |
| IO 往返预算 + IO 回归测试（计数断言无 N+1） | [`io_contract.md §1`](io_contract.md) + [`§7`](io_contract.md) | 🟢 |

### 8.12 配置中心与凭据加密约束（如启用配置上云）

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 配置上云权威源矩阵 | [`config.md §6`](config.md) | ⬜ / 🟢 视工程启用 |
| 配置中心 Client 抽象 + 拉取落盘编排 | [`config.md §7`](config.md) | ⬜ / 🟢 视工程启用 |
| 凭据对称加密（密钥应用层持有 / 不入库 / 轮换） | [`config.md §8`](config.md) | ⬜ / 🟢 视工程启用 |
| 启动期 fail-fast 加载时序 | [`config.md §9`](config.md) | ⬜ / 🟢 视工程启用 |

### 8.13 部署与运行时生命周期约束（如启用本规范）

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 部署产物蓝图（镜像 / 编排 / 流水线） | [`deployment.md §1`](deployment.md) | ⬜ / 🟢 视工程启用 |
| 容器镜像约束（多阶段 / 非 root / 最小镜像） | [`deployment.md §2`](deployment.md) | ⬜ / 🟢 视工程启用 |
| 健康探针语义（liveness / readiness / startup 分层） | [`deployment.md §3`](deployment.md) | ⬜ / 🟢 视工程启用 |
| 启动就绪（readiness 晚于预热）+ 优雅关闭时序 | [`deployment.md §4`](deployment.md) + [`§5`](deployment.md) | ⬜ / 🟢 视工程启用 |
| 发布回滚策略 + 资源配额弹性 | [`deployment.md §6`](deployment.md) + [`§7`](deployment.md) | ⬜ / 🟢 视工程启用 |
| 运行时旋钮对齐容器配额（`GOMAXPROCS` / automaxprocs / `GOMEMLIMIT`） | [`deployment.md §7`](deployment.md) + [`performance_contract.md §3.4`](performance_contract.md) | ⬜ / 🟢 视工程启用 |

> 运行时基线属于"AI 不直接感知"维度，启用与否由工程负责人决定（详见 INDEX.md §0 采纳进度）。

### 8.14 缓存设计约束（如启用本地缓存 / 多分片）

| 约束类 | 文档锚点 | 状态 |
|--------|---------|------|
| 本地缓存准入条件（六条须同时满足）+ 排除清单 | [`cache_design.md §3.5`](../cache/cache_design.md) | ⬜ / 🟢 视工程启用 |
| 回源保护三类分治（single-flight / TTL 抖动 / XFetch 概率提前重算） | [`cache_design.md §2.10`](../cache/cache_design.md) | ⬜ / 🟢 视工程启用 |
| 紧凑编码阈值（listpack / intset / ziplist）+ 概率/位图结构（HLL / Bitmap / Roaring） | [`cache_design.md §2.11`](../cache/cache_design.md) / [`§2.12`](../cache/cache_design.md) | ⬜ / 🟢 视工程启用 |
| 大 key 治理（拆分 + `UNLINK`）/ 热 key 读写分治 / 多分片可扩展（Hash Tag / 禁 `mod N`） | [`cache_design.md §9`](../cache/cache_design.md) | ⬜ / 🟢 视工程启用 |
| 读写非对称降级（加速层 miss 穿透 / 数据源 miss 快速失败；写 best-effort 静默） | [`cache_design.md §1.4`](../cache/cache_design.md) | ⬜ / 🟢 视工程启用 |

---

## 9. 约束突破登记表

> 部分约束（如 metrics cardinality 默认上限、Metric 总数）是**默认上限**而非硬限制。如确有业务理由必须突破，必须在本表登记。

| 日期 | 突破的约束 | 实际值 | 理由 | 替代方案评估 | 责任人 | 关联 PR |
|------|----------|--------|------|-------------|-------|---------|
| {YYYY-MM-DD} | {约束名（如 metric_cardinality_total_500）} | {实际值} | {业务理由} | {为什么替代方案不可行} | {开发者} | {PR-link} |

**登记纪律**：

- 突破未登记 → 视为隐式违规，审计 Skill 命中将阻断合并
- 同类突破累计 3 次 → 触发约束本身的回头复审（可能需要放宽默认值或引入新约束类）

> 本表预期为空；当 metrics / performance 上限引入后才有突破场景。

---

## 10. grep 锚 rule-id 索引

> 危险模式清单的机械审计入口。每条 rule-id 对应一条 grep / AST 规则，由 `concurrency-review` / `performance-review`等审计 Skill 触发。
>
> **grep 锚定位为"信号级"非"判定级"**：命中标 🟡 待复核，最终判定权在人工 reviewer。误报通过 `// aiweave:allow=<rule-id>` 行内注解抑制。

### 10.1 并发安全类

| Rule-id | 含义 | grep 锚（参考） | 误报排除 | 状态 |
|---------|------|---------------|---------|------|
| `R-CONC-LOCK-IO` | 锁内做网络 IO | `\.Lock\(\)[\s\S]{0,500}?(http\.\|Mysql\|Redis)` | RLock 中读小对象 | 🟢 |
| `R-CONC-LOCK-LOG` | 锁内打日志 | `\.Lock\(\)[\s\S]{0,300}?(tlog\.\|log\.)` | — | 🟢 |
| `R-CONC-MAP-RACE` | map 并发写 | 文件含 `var .* map\[` 且同文件存在 `go func` 且无 `sync\.` | 局部 map 不跨 goroutine | 🟢 |
| `R-CONC-DOUBLE-CLOSE` | 关闭已关闭的 channel | `close\([^)]+\)` 出现多次且无 sync.Once 保护 | — | 🟢 |
| `R-CONC-SEND-CLOSED` | 向已关闭的 channel 发送 | — | 见 §10.1 备注 | 🟢 |
| `R-RESOURCE-DEFER-LOOP` | defer 在 for 循环内 | `for [^{]*\{[\s\S]{0,300}?defer ` | 循环体内显式调用 Close | 🟢 |
| `R-CONC-GOROUTINE-LEAK` | 忽略 ctx.Done() | `go func` 块内无 `ctx\.Done\(\)` 且无 `select` | 短周期任务（< 1s）| 🟢 |
| `R-CONC-COUNTER-CONTENTION` | 超高频计数用单点 Mutex / 单 atomic | 热路径单一 `atomic\.Add` / `Mutex` 计数器被高频写（结合热路径判定） | 已分片计数（striped）读时求和（concurrency_safety.md §2） | 🟡 |
| `R-CONC-FALSE-SHARING` | 分片 / 原子计数器未按缓存行对齐（false-sharing） | — 非 grep 可判（人工 / AST：相邻原子字段无 padding） | 已 padding 到 64B / 独占 cache line | 🟡 |

### 10.2 性能 / 资源类

| Rule-id | 含义 | grep 锚（参考） | 误报排除 | 状态 |
|---------|------|---------------|---------|------|
| `R-PERF-LOOP-DB-QUERY` | for 循环内 DB 查询（N+1） | `for [^{]*\{[\s\S]{0,300}?\.(Find\|First\|Where\|Take)\(` | 显式批量 API（In/Pluck/Scan）/ 循环次数硬编码 ≤ 5 | 🟢 |
| `R-PERF-LOOP-ALLOC` | for 循环内大对象分配 | 循环体含 `make([]T, N)` 且 N > 1024 或 struct 字段数 > 20 | 循环前已 capacity 预分配 / 仅 error 分支 | 🟢 |
| `R-PERF-HOT-REFLECT` | 热路径用反射 | 热路径文件含 `reflect\.` | 文件含 `//go:build !hotpath` | 🟢 |
| `R-PERF-HOT-FMT` | 热路径用 fmt.Sprintf / fmt.Errorf | 热路径文件含 `fmt\.Sprintf\(\|fmt\.Errorf\(` | 错误路径 / 用于 wrap error | 🟢 |
| `R-PERF-FULL-COUNT` | 全表 COUNT(*) | `COUNT\(\*\)\s+FROM` 且无 WHERE / WHERE 列无索引 | 小表（< 1 万行）/ 后台批处理 | 🟢 |
| `R-RESOURCE-SLEEP-SYNC` | time.Sleep 做同步 | `time\.Sleep\(` 出现在 `go func` 或 `for` 内 | 重试退避（指数退避） / 测试代码 | 🟢 |
| `R-CACHE-LARGE-KEY` | Redis 大 Key（> 1MB） | `\.Set\(` 紧随大对象序列化 / SADD 一次 > 1000 元素 | 显式分片 | 🟢 |
| `R-CACHE-BIGKEY-DEL` | 大 key 用 `DEL` 阻塞单线程 | `\.Del\(` 作用于大 value / 海量元素集合 key | 已用 `UNLINK` / 小 key | 🟢 |
| `R-CACHE-HOTKEY-SINGLE-SHARD` | 热 key 流量全砸单分片（加分片分不走） | 选型 / 人工审查：单 key 超高频无打散 | L1 / 读副本 / `{hotkey}:{i}` 打散 / 分片计数（cache_design §9.2） | 🟡 |
| `R-CACHE-L1-HIGH-CARD` | 高基数 / 高频变更 key 塞进 L1 进程内 | L1 写入 key 为 per-entity / 组合键 / 计数类 | 满足 L1 准入（万级 + 低频变更，cache_design §3.5） | 🟡 |
| `R-CACHE-MOD-N-REHASH` | 分片节点选择用裸 `mod N`（扩缩容全量重哈希） | `%\s*len\(.*[Nn]ode` / `%\s*nodeCount` 用于节点选择 | 一致性哈希 / Cluster slot 迁移（cache_design §9.3） | 🟢 |
| `R-LOCALMIRROR-STALE-READY` | 本地镜像 `ready` 在事务 COMMIT 前置位 / 跨刷新持长读事务（撕裂可见性） | `ready.Store(true)` / 置位早于 `Commit()` 返回；或读事务跨多轮刷新不重开 | `ready` 仅 COMMIT 后置位 + 短 auto-commit 读（cache_design §3.4） | 🟡 |

### 10.3 事务 / 幂等类

| Rule-id | 含义 | grep 锚（参考） | 状态 |
|---------|------|---------------|------|
| `R-TXN-NO-IDEMPOTENT` | 写入类接口无幂等 Key | 路由含 `POST` 且 controller 无 SETNX / 唯一索引引用 | 🟢 |
| `R-TXN-CROSS-SOURCE` | 跨数据源写入未在 transaction_design.md §2 登记 | git diff 同时含 MySQL 写 + Redis 写 + 未登记 | 🟢 |

### 10.4 领域不变量类

| Rule-id | 含义 | 检查方式 | 状态 |
|---------|------|---------|------|
| `R-INVARIANT-MARK-MISMATCH` | 伪码 `[INVARIANT-CHECK: I-N]` 标记与代码实现脱节 | 由 `domain-invariant-check` 提取标记 + 静态分析对比 | 🟢 |
| `R-INVARIANT-MISSING-CHECK` | §7.1 业务规则在 "代码位置" 列指明的文件中无校验逻辑 | 由 `domain-invariant-check` 扫描 | 🟢 |
| `R-STATE-ILLEGAL-TRANSITION` | UPDATE 状态字段时未校验当前状态 / 跳转不在 §7.3 状态机 | 由 `domain-invariant-check` 扫描 | 🟢 |

### 10.5 跨服务合约类

| Rule-id | 含义 | grep 锚（参考） | 状态 |
|---------|------|---------------|------|
| `R-XSVC-NO-TIMEOUT` | 下游调用未显式设超时 | `(http\.Client\|grpc\.Dial)` 后无 `Timeout` / `WithTimeout` | 🟢 |
| `R-XSVC-TIMEOUT-CASCADE` | 本服务对下游设的超时 ≥ 上游对本服务的超时 | 由 `failure-path-review` + 22 §6.2 链路超时协调表对比 | 🟢 |
| `R-XSVC-SILENT-SWALLOW` | 下游调用错误被 `_ = ...` 静默吞 | `_ = .*\.Call\(` / `_ = .*\.Get\(` 等 | 🟢 |
| `R-XSVC-UNREGISTERED-CLIENT` | `api/` 新增 client 但未在 cross_service_contract.md §3 登记 | 由 `doc-sync-check` + `failure-path-review` 联合检查 | 🟢 |

### 10.6 失败路径类

| Rule-id | 含义 | 检查方式 | 状态 |
|---------|------|---------|------|
| `R-FAIL-PATH-UNDOC` | 代码中失败分支 (F-N) 未在 transaction_design.md §6 失败路径全景图覆盖 | 由 `failure-path-review` 静态分析 | 🟢 |
| `R-FAIL-PATH-NO-TEST` | 失败分支 (F-N) 无对应测试用例 | 由 `failure-path-review` 扫描 test/cases/ | 🟢 |
| `R-FAIL-PATH-WEAK-ASSERT` | 测试用例仅断言 `err != nil` 未断言 errNo / 副作用 | 由 `failure-path-review` 扫描断言模式 | 🟢 |

### 10.7 IO 铁律类（由 `io-review` 触发）

| Rule-id | 含义 | grep 锚（参考） | 误报排除 | 状态 |
|---------|------|---------------|---------|------|
| `R-IO-N-PLUS-1` | 铁律一：循环内单条查询（N+1） | `for [^{]*\{[\s\S]{0,300}?\.(Find\|First\|Take\|Get\|HGet\|Call)\(` | 批量 API（In/Pluck/Scan/MGET）/ 循环次数硬编码 ≤ 5 / io_contract.md §2.1 登记 | 🟢 |
| `R-IO-SERIAL-ORCH` | 铁律二：独立读串行编排（无数据依赖） | 连续 ≥ 2 个独立读串行展开（结合依赖链分析） | 真实依赖链（后查询入参取自前查询结果）/ io_contract.md §2.2 登记 | 🟢 |
| `R-IO-LOOP-WRITE` | 循环内单条写 | `for [^{]*\{[\s\S]{0,300}?\.(Create\|Save\|Insert\|HSet\|Exec)\(` | 批量写 / pipeline / 循环次数 ≤ 5 | 🟢 |
| `R-IO-NO-AGGREGATOR` | 多次跨实例读未走聚合器 | 同函数 ≥ N 次跨实例 / 跨表读未聚合 | 已用聚合器（Preload/Fetch） | 🟢 |
| `R-IO-LOOP-RPC` | 循环内单条 RPC / 外部调用 | `for [^{]*\{[\s\S]{0,300}?\.(Call\|Invoke\|Do)\(` 命中 api/ | 批量接口 / 并行扇出 | 🟢 |
| `R-IO-SHARED-MUTATE` | 就地修改 single-flight / Slot 共享指针字段 | `\.Value\(\)` 后对返回指针字段赋值 | 已深拷贝 | 🟢 |
| `R-IO-RAW-SUBMIT` | 裸提交协程池（绕过内联兜底统一入口，可致 WaitGroup 挂死） | `\.Submit\(` 未走统一兜底提交器 | 已走"失败内联兜底"统一入口（concurrency_safety.md §4.4） | 🟢 |

### 10.8 配置安全类（由 `doc-sync-check` 触发）

| Rule-id | 含义 | grep 锚（参考） | 误报排除 | 状态 |
|---------|------|---------------|---------|------|
| `R-CONF-PLAINTEXT-CRED` | 配置中心 / 资源凭据明文落盘或入库 | 配置文件中 `password:` / `secret:` 后非密文（非 Base64 / 非 `@@`） | 占位符 `@@key` / 已是密文 | 🟢 |
| `R-CONF-HARDCODE-SECRET` | 密钥 / 密码硬编码在代码 | 源码含字面量 `key := "..."` / `password := "..."`（长度 ≥ 16 的字面量） | 测试 fixture / 从环境读取 | 🟢 |
| `R-CONF-SECRET-COMMIT` | 含密钥 / 明文凭据的文件入 git | git diff 含密钥文件且不在 `.gitignore` | — | 🟢 |
| `R-CONF-NO-FAILFAST` | 配置中心拉取失败未 fail-fast | 拉取调用后 `if err != nil` 分支无 `panic` / `os.Exit` / `Fatal` / `return err`（启动期） | 非启动期 / 有降级设计且已登记 | 🟢 |
| `R-CONF-DANGER-SINGLE-GATE` | 高风险开关只靠配置档位、缺运行档位（`RUN_ENV`）硬门禁 | 危险开关判定无 `RUN_ENV` / 运行环境校验叠加 | 配置档位 + `RUN_ENV` 双门禁（config.md §10） | 🟢 |

### 10.9 部署与运行时生命周期类（由 `doc-sync-check` 触发）

| Rule-id | 含义 | grep 锚（参考） | 误报排除 | 状态 |
|---------|------|---------------|---------|------|
| `R-SHUTDOWN-NO-SIGNAL` | 进程入口未注册终止信号 / 收到信号即硬退出 | 含 `func main` / 子命令入口但无 `signal.Notify` | 库 / 非常驻命令（一次性脚本） | 🟢 |
| `R-SHUTDOWN-NO-DRAIN` | 优雅关闭缺在途排空（直接 `os.Exit`） | `signal.Notify` 后直接 `os.Exit` 无 `Shutdown` / drain | 已调 `srv.Shutdown` / `cancel()` + 等待 | 🟢 |
| `R-SHUTDOWN-LEAK-GOROUTINE` | 常驻 goroutine / 消费者未接入关闭排空 | §1.1 登记的常驻 goroutine 在关闭路径无对应 cancel | concurrency_safety.md §1.1 已对账覆盖 | 🟢 |
| `R-PROBE-LIVENESS-DEEP` | liveness 探针内做依赖检查（DB/缓存/下游） | liveness handler 含 `.Ping(` / `.Get(` / 下游调用 | 仅 readiness handler | 🟢 |
| `R-PROBE-MISSING` | 对外服务缺 readiness / liveness 端点 | 路由表无 `{live-path}` / `{ready-path}` | 纯消费者 / 任务进程（无对外 HTTP） | 🟢 |
| `R-DEPLOY-ROOT` | 容器以 root 运行 | `{Dockerfile-path}` 无 `USER` 非 root 声明 | 已声明非 root user | 🟢 |
| `R-DEPLOY-NO-LIMITS` | 编排清单缺 requests/limits | `{deploy-manifest-path}` 无 `requests` / `limits` | — | 🟢 |
| `R-DEPLOY-GOMAXPROCS` | `GOMAXPROCS` 未对齐容器 CPU limit（用默认宿主核数） | 编排有 cpu limit 而代码无 `automaxprocs` / `maxprocs\.Set` / `GOMAXPROCS` 对齐 | 已用 automaxprocs（deployment.md §7） | 🟡 |
| `R-RUNTIME-MEMLIMIT` | `GOMEMLIMIT` 未设或未对齐 mem limit | 编排有 mem limit 但无 `GOMEMLIMIT` / `debug\.SetMemoryLimit` | 已设软上限（~90% mem limit，deployment.md §7） | 🟡 |

### 10.10 尾延迟 / 运行时调优类（由 `performance-review` 触发）

| Rule-id | 含义 | grep 锚（参考） | 误报排除 | 状态 |
|---------|------|---------------|---------|------|
| `R-TAIL-RETRY-NOBUDGET` | 重试无预算（占比无上限） | retry 循环 / 库调用无全局预算 / 令牌限制 | 已接重试预算（≤X% + 熔断联动，performance_contract.md §6.4） | 🟡 |
| `R-TAIL-RETRY-NOJITTER` | backoff 无 jitter（重试同步成群） | 退避计算（`backoff` / `<<` 指数项）无 `rand` 抖动项 | 含随机抖动 | 🟡 |
| `R-TAIL-HEDGE-UNSAFE` | 对冲 / 重试包裹非幂等调用 | hedge / retry 入口下游为写类方法（`Create` / `Update` / `Delete` / `Incr`） | 调用幂等（幂等 Key / 唯一索引） | 🟡 |
| `R-TAIL-COLDSTART` | 冷启 readiness 早于预热（缓存 / 连接池） | readiness 置 true 早于缓存 / 连接池预热完成（结合 deployment.md §4 时序） | readiness 晚于预热 | 🟡 |

### 10.11 数据对外暴露 / 加密类（由 `doc-sync-check` 触发）

| Rule-id | 含义 | grep 锚（参考） | 误报排除 | 状态 |
|---------|------|---------------|---------|------|
| `R-SEC-ID-PLAINTEXT` | 响应直接返回未加密的真实主键 / 内部 ID | 出口 / assembler 把 `Id` / `KeyNo` / 主键字段直接赋值返回、未经 `Encrypt` | 已确定性加密（`keycrypto.Encrypt`）/ 非 ID 字段 | 🟡 |
| `R-SEC-DECRYPT-ORACLE` | 对外 ID 解密失败返差异化错误 / 4xx / 携带失败细节（响应差分预言机） | 解密失败分支返回专用错误码 / 4xx / "解密失败"文案，而非统一"查无结果" | 统一返回查无结果（status_codes §6.5 / docs-spec/14 §9.5） | 🟡 |

### 10.12 可观测性类（由 `performance-review` / `observability-review` 触发）

| Rule-id | 含义 | grep 锚（参考） | 误报排除 | 状态 |
|---------|------|---------------|---------|------|
| `R-OBS-CONTROL-LOOP-BLIND` | 开了控制环（worker 池 / 熔断器 / 自适应限制 / shed）却无对应饱和度 / 触发态 collector | 新增 `{池构造}` / `{熔断器构造}` / `{限制器构造}` 但 collector 注册处无对应项 | 控制环已注册对应 collector（observability §6.4） | 🟡 |
| `R-OBS-LATENCY-COUNTER-ONLY` | 关键路径延迟只有累计耗时 Counter、无延迟 Histogram（看不见分位） | 热路径延迟埋点用 `Counter` 累加耗时、无 `Histogram` / `Observe` | 已用 Histogram 分位（observability §6.1 / §7.3） | 🟡 |
| `R-OBS-DASH-MISSING` | 暴露了控制环指标却无对应面板格（盲采：采了不看 = 没采） | 选型 / 人工审查：observability §6.4 有指标但 observability_dashboard 无对应面板格 | 面板已补格 | 🟡 |

> 决策真相源见 [`design-spec/13_observability_design.md`](../../../design-spec/13_observability_design.md)（控制环可观测性 / RED·USE·SLO 反推 / 面板）。

### 10.13 规则维护

- 新增 rule-id → 本节新增一行；同步在 `concurrency-review` / `performance-review` / `io-review` Skill 实现 grep 模式
- 误报率高的规则 → 优化 grep 锚或下调严重级别（🔴 → 🟡）
- 长期 0 命中的规则 → 评估是否仍有价值；可降级为"可选规则"


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
