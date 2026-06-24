# 00 - 高性能架构方法论总纲

> 规定 `design-spec/` 这一支柱的定位、统一骨架、与 `docs-spec/` 的边界，以及"技术方案生成"如何走完十二大决策视角。
>
> 本支柱回答 `docs-spec/` 不回答的问题：**面对一个需求，怎么"做出"架构决策**——而不是决策定了之后怎么写下来。

---

## 1. 定位：AIWeave 的第三支柱

AIWeave 此前有两根支柱，都是"决策已定之后"的规范：

| 支柱 | 回答 | 体裁 |
| --- | --- | --- |
| `docs-spec/` | 设计怎么**写下来**（章节结构 / 必填精度 / 双向同步） | 表示规范（representation） |
| `skills-spec/` | AI 怎么**执行**（步骤化 / 文档同步 / 测试同步） | 执行规范（execution） |
| **`design-spec/`** 🆕 | 面对需求怎么**做出架构决策** | **设计规范（decision / methodology）** |

`design-spec/` 是**上游**：它在"写 docs/ 之前"介入，把"凭直觉拍架构"变成"按判定树选架构"。它的产物是一份**技术方案（TRD）**，再喂进既有的 A1 forward / B1 sync 流落地为 docs/ 与代码。

> 一句话边界：**`design-spec` 管"怎么选"，`docs-spec` 管"选完怎么写下来"，`skills-spec` 管"按写下来的怎么生成"。**

---

## 2. 第一性目标与设计哲学（四大元方法）

### 2.1 第一性目标的推论

PRINCIPLES §1 的第一性目标是"仅凭 docs/ + .claude/skills/ 重建整个工程"。PRINCIPLES §13.1 已把这一目标推进一步：

> "AI 可重建"不仅要求行为等价，也要求**重建出的代码默认就是 IO 高效的**。

`design-spec/` 把 §13 对 IO 的这条推论**推广到所有架构维度**：重建出来的代码不仅要行为对、写得下来，还要**架构选型本身是对的**。没有 design-spec，docs/ 能忠实记录一个糟糕的设计；有了 design-spec，糟糕的设计在"生成方案"阶段就被判定树拦下。

因此：
- **docs-spec 保证**："写得下来、对得齐、可重建"。
- **design-spec 保证**："设计得对、选型有据、默认即最优"。

### 2.2 IO 是第一性瓶颈（旗舰视角的由来）

绝大多数后端服务的延迟与吞吐天花板，不来自 CPU 或内存算法，而来自**等待 IO**——等库、等缓存、等下游。纳秒级的 CPU 逻辑常被毫秒级的网络往返淹没。由此得出本支柱最被强调的一条命题：

> **把 IO 的核心问题解决了，整个工程的技术天花板就会很高。**

"IO 核心问题"被精确拆成三个可度量维度，是 IO 视角（[`01`](01_io_design.md)）的组织轴：**次数（消灭 N+1）· 串/并行（消灭独立串行）· 往返（消灭远端往返）**。IO 是本支柱的**旗舰范本**——方法论密度最高、且有三条硬铁律兜底（[`docs-spec/25 §2`](../docs-spec/25_io_aggregation_spec.md)），其余十一个视角照它的形组织。

### 2.3 元方法一：读路径 / 写路径分治

读和写是**两种相反的 IO 模式**，必须用相反的手段优化；把它们抹平进统一"数据访问层"是多数性能设计失败的根因。

| 路径 | 目标 | 手段方向 |
| --- | --- | --- |
| **读路径** | 极低延迟 | 消灭往返：本地命中（0 网络）> 单次往返；消重复读、分支裁剪 |
| **写路径** | 极高吞吐 / 削峰 | 先写内存态原子命令 → 脏标记积攒 → 定时增量批量落库；纯追加走 MQ 批量 |

> 这条分治贯穿多个视角：IO（[`01`](01_io_design.md) 读/写两棵决策树）、缓存（[`05`](05_caching_design.md) 读写非对称降级）、事务（[`04`](04_transaction_design.md) 最终一致 + 边界兜底）。

### 2.4 元方法二：借纪律，不借实现（与 §12 同构）

复用别的工程的架构方法论时，**照搬纪律（决策树 / 判定标准 / 反模式），按自己的数据形态重选原语**。同一条"把扇出收敛成一次往返"的纪律，同质大对象选 MGET、异构小字段选 Pipeline——把前者硬搬到后者会直接失效。

> 这条元方法正是 [`PRINCIPLES §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 在架构方法论层的同构表达：**design-spec 主体只写占位符化的纪律，具体原语 / 数值 / 系统名一律进末尾"参考示例"段**。读者带走的应是判定标准，不是某工程的魔法数字。

### 2.5 元方法三：把纪律下沉为机械闸门

人与 AI 都会遗忘纪律。能机械判定的违规（如循环内单条查询、裸提交协程池），必须下沉为**0-token、可复现的独立绊线**，不依赖编写者自觉——尤其当编写者与评审者是同一个（人或模型）时。

> 这条元方法对应 AIWeave 的 **Hooks（L0 自动化防御）**：见 [`PRINCIPLES §14`](../PRINCIPLES.md#14-hooks-机制l0-自动化防御--强制) 与 [`skills-spec/02 §4`](../skills-spec/02_settings_local_json_spec.md)。护栏的判定逻辑必须随被检代码的数据特征演进（如 N+1 检测从"裸循环变量"升级为"污点传播"），否则形同虚设。

---

## 3. 十二大决策视角清单（强制）

每个视角是一个**决策透镜（lens）**：把需求中的某一类架构问题，从"识别形态"一路推到"推荐选型"。每个视角的**决策真相源**在 `design-spec/0X`，**表示真相源**仍在对应 `docs-spec/NN`（避免双源真相，见 §5）。

> **视角分四组（按它治理的"轴"）**：① **同步请求路径**（01-07，请求进来在本进程内怎么快、怎么不长尾）；② **异步面**（08 消息 / 事件——把工作解耦出请求生命周期）；③ **契约面**（09 API——比 IO 还上游，接口粒度直接决定调用方 N+1）；④ **水平与时间轴**（10 多副本协调 / 11 在线演进 / 12 容量伸缩）。01-07 是旗舰范本写透的核心，08-12 把方法论从"单请求"扩到"异步 + 契约 + 多副本 + 时间轴"四条边。

| 视角 | 优化目标 | 决策真相源 | 落到 docs/ | 表示真相源 |
| --- | --- | --- | --- | --- |
| **IO 设计** ⭐ | 每请求 / 每批 IO 往返最小化、并行最大化 | [`01_io_design.md`](01_io_design.md) | `architecture/io_contract.md` | [`docs-spec/25`](../docs-spec/25_io_aggregation_spec.md) |
| **数据建模** | 表结构 / 索引 / 分库 / 演进的正确性与可扩展性 | [`02_data_model_design.md`](02_data_model_design.md) | `schema/database_design.md` | [`docs-spec/05`](../docs-spec/05_schema_design_spec.md) |
| **并发模型** | 共享状态安全、goroutine 生命周期、无数据竞争 | [`03_concurrency_design.md`](03_concurrency_design.md) | `architecture/concurrency_safety.md` | [`docs-spec/20`](../docs-spec/20_concurrency_safety_spec.md) |
| **事务一致性** | 事务边界、补偿、幂等、一致性窗口 | [`04_transaction_design.md`](04_transaction_design.md) | `service/transaction_design.md` | [`docs-spec/21`](../docs-spec/21_distributed_transaction_spec.md) |
| **缓存策略** | 缓存层级、Key 设计、回源、失效与一致性 | [`05_caching_design.md`](05_caching_design.md) | `cache/cache_design.md` | [`docs-spec/06`](../docs-spec/06_cache_design_spec.md) |
| **稳定性** | 熔断 / 降级 / 限流 / 性能合约的选型与参数 | [`06_resilience_design.md`](06_resilience_design.md) | `circuit_breaker/circuit_breaker_design.md`、`architecture/performance_contract.md` | [`docs-spec/12`](../docs-spec/12_circuit_breaker_spec.md)、[`docs-spec/22`](../docs-spec/22_performance_contract_spec.md) |
| **尾延迟与过载** | P99/P999 长尾治理、过载有序降级（对冲 / 自适应限制 / load shedding / 重试预算） | [`07_tail_latency_design.md`](07_tail_latency_design.md) | `architecture/performance_contract.md`、`architecture/cross_service_contract.md` | [`docs-spec/22`](../docs-spec/22_performance_contract_spec.md)、[`docs-spec/24`](../docs-spec/24_cross_service_contract_spec.md)、[`docs-spec/12`](../docs-spec/12_circuit_breaker_spec.md) |
| **异步消息 / 事件驱动** 🆕 | 解耦写/事件、投递语义、顺序分区、双写一致、消费幂等与背压 | [`08_messaging_design.md`](08_messaging_design.md) | `service/scheduled_tasks_design.md §MQ` | [`docs-spec/11`](../docs-spec/11_mq_consumer_spec.md) |
| **API 契约 / 接口粒度** 🆕 | 接口粒度（防客户端 N+1）、批量/聚合端点、分页契约、版本兼容、协议选型 | [`09_api_contract_design.md`](09_api_contract_design.md) | `api/*.md`、`architecture/*_flow.md` | [`docs-spec/07`](../docs-spec/07_api_interfaces_spec.md)、[`docs-spec/08`](../docs-spec/08_api_flow_spec.md) |
| **定时任务 / 分布式协调** 🆕 | 单例 vs 每副本、leader 选举/分布式锁/原子认领、幂等可重入、fencing 防脑裂 | [`10_scheduling_coordination_design.md`](10_scheduling_coordination_design.md) | `service/scheduled_tasks_design.md` | [`docs-spec/10`](../docs-spec/10_scheduled_tasks_spec.md) |
| **演进与在线迁移** 🆕 | 扩展-收缩、在线 DDL、限流回填、双写双读灰度切换、回滚闸门 | [`11_evolution_migration_design.md`](11_evolution_migration_design.md) | 分布落地（`schema/database_design.md §8`、`mvp_rebuild_path.md §11`） | [`docs-spec/05`](../docs-spec/05_schema_design_spec.md)、[`docs-spec/19`](../docs-spec/19_incremental_sync_spec.md) |
| **容量规划与弹性伸缩** 🆕 | 从 SLA 反推容量、余量（ρ≈0.8 膝盖）、资源配比、伸缩信号与防抖 | [`12_capacity_scaling_design.md`](12_capacity_scaling_design.md) | 分布落地（`performance_contract.md`、`deployment.md`） | [`docs-spec/22`](../docs-spec/22_performance_contract_spec.md)、[`docs-spec/27`](../docs-spec/27_deployment_runtime_spec.md) |

> 视角不要求每个需求都全过一遍——由需求形态触发（见各视角 §2 输入信号）。但 IO 视角对"涉及多次数据访问 / 集合处理 / 多依赖编排"的需求**强制过一遍**。**11（演进迁移）/ 12（容量伸缩）无独立 docs-spec，按双源真相分布落地到既有表示真相源**（同 07 模式）。

### 3.1 视角联动（一个需求常同时命中多个，彼此有依赖）

复杂需求很少只落一个视角——它们交织，且**一个视角的选型会约束另一个**。`/design-solution` 第 2 步识别命中清单、第 3 步逐个过、第 4 步在 TRD 里把这些**跨视角依赖**显式标出：

| 起点视角 | 联动到 | 联动点 |
| --- | --- | --- |
| **IO 读路径**（01 §3.1） | 缓存（05）+ 数据建模（02） | 多级缓存 / 非对称降级（05）；缓存 miss 回源走覆盖索引免回表（02 §3.5） |
| **IO 写路径**（01 §3.2） | 事务（04）+ 缓存（05） | 写削峰的最终一致 + 边界兜底（04）；Redis-first / delta flush 落缓存（05） |
| **IO 并行编排**（01 §4/§5） | 并发（03） | 聚合器 / 扇出的协程池容量（Little 定律）、内联兜底、阻塞不变量（03） |
| **缓存分片可扩展**（05 §3.6） | IO（01）+ 数据建模（02） | 跨分片批量按分片分组并行（01 聚合器）；大 key 拆分用 Hash 分桶（02 / 05 存储） |
| **数据建模 shard 并行**（02 §3.3） | IO（01）+ 并发（03） | 三段式批量编排（01）；多 shard 并行的池（03） |
| **稳定性**（06） | 全部数据访问 | 连接池基线 / 熔断包所有 DB·缓存·RPC；数据层禁日志支撑数据访问层可无副作用复用 |
| **尾延迟治理**（07） | IO（01）+ 并发（03）+ 稳定性（06） | 对冲叠加扇出 best-effort（01 §3.4）；自适应并发限制是静态 Little 池（03）的升级；与熔断 / 降级叠加而非替代（06，分界见 07 §8） |
| **异步消息**（08） | 事务（04）+ IO 写路径（01）+ 并发（03）+ 稳定性（06） | Outbox/本地消息表落库（04 §3.1）；写路径削峰→投 MQ（01 §3.2）；消费池容量与排空（03 §3.4 / 27 §5）；DLQ 即降级、消费熔断（06） |
| **API 契约**（09） | IO（01）+ 数据建模（02）+ 尾延迟（07） | 接口粒度直接决定调用方 N+1（01 上游闸门）；分页契约↔keyset 游标（02 §3.6）；批量端点内部走聚合器/对冲（01 §3 / 07 §3.2） |
| **定时/协调**（10） | 并发（03）+ 事务（04）+ 缓存（05）+ 部署（27） | "单后台协程"在多副本下须单例化（03 §3.1）；SPOP 原子认领消幂等（04 §3.2/§9.3）；缓存巡检/预热错峰（05 §3.7/§3.8）；leader 退出接优雅关闭（27 §5） |
| **演进/迁移**（11） | 数据建模（02）+ 事务（04）+ IO（01）+ 缓存（05） | 扩展-收缩↔字段演进（02 §3.8 / docs-spec 05 §8）；限流回填走 keyset 分批 + 大写分批（01 §3.5 / 02 §3.6）；迁移期一致性窗口（04 §3.2）；缓存世代式切换（05 §3.7） |
| **容量/伸缩**（12） | 尾延迟（07）+ 并发（03）+ 稳定性（06）+ 数据建模（02） | 排队律 ρ≈0.8 膝盖留余量（07 §9.6）；池容量 Little 反推（03 §3.2）；连接池基线/配比（06 §9.2）；有状态伸缩↔一致性哈希/分库分表（02 / 05 §3.6） |

> **IO（01）是枢纽，但不再是唯一入口**：数据访问类需求先过 IO 视角牵出缓存 / 数据建模 / 并发 / 事务；**异步类需求先过 08（解耦判定）、对外接口先过 09（粒度闸门）、多副本部署先过 10（单例 vs 每副本）、带流量变更先过 11（演进路径）、定容量先过 12（SLA 反推）**。读写路径分治（§2.3）横跨 IO / 缓存 / 事务三视角；**"破坏性变更必须新老并存一个窗口"则横跨 08（消息 schema）/ 09（API 版本）/ 11（数据演进）三视角，是同一条元方法的三处实例**。

---

## 4. 统一 lens 骨架（强制）

每篇 `design-spec/0X` 必须按以下九节组织。这是本支柱区别于 docs-spec"章节结构"的**决策骨架**——以"识别 → 判定 → 选型 → 权衡 → 落地"为主轴：

| 节 | 标题 | 内容 |
| --- | --- | --- |
| `## 1.` | 设计目标 | 本视角优化什么 / 第一性约束（不可妥协项） |
| `## 2.` | 输入信号识别 | 怎么判断需求落入本视角、属于哪一类形态（可机械识别的迹象） |
| `## 3.` | 决策树 | 形态 → 推荐方案的判定路径（强制 ASCII） |
| `## 4.` | 默认选型与升级路径 | 默认选 X（最简）；触发条件 → 升级到 Y / Z |
| `## 5.` | 权衡矩阵 | 候选方案 × 评估维度的对比表 |
| `## 6.` | 反模式目录 | 高频错误选型 + grep 锚（引用既有 docs-spec 锚，不复制规则） |
| `## 7.` | 产出与落地 | 本视角结论落到 docs/ 哪篇、按哪条 docs-spec 表示 |
| `## 8.` | 与既有规范的边界 | 双源真相规避表（本视角 vs 相关 docs-spec / 其他视角） |
| `## 9.` | 参考示例 | 占位符豁免段（段首 `> ⚠️`，见 PRINCIPLES §12） |

---

## 5. 双源真相规避（强制 / 本支柱的命脉）

`design-spec` 与 `docs-spec` 在同一个维度（如 IO）上都会出现，必须严守边界，否则规则会在两处漂移：

| 内容 | design-spec/0X | docs-spec/NN |
| --- | --- | --- |
| 怎么识别形态、怎么选方案（决策树 / 权衡 / 升级路径） | ✅ 真相源 | 不写 |
| 选定后的硬规则（如三条 IO 铁律、字段精度、锁策略） | **引用，不复制** | ✅ 真相源 |
| 结论落到 docs/ 的章节结构与必填精度 | 不写 | ✅ 真相源 |
| 代码 ↔ docs 的 B1 反向同步 | 引用 | ✅ 真相源 |

**铁律**：凡 design-spec 提到一条已在 docs-spec 定义的硬规则（如"禁止 N+1"），必须写"详见 docs-spec/25 §2"并**只引用、不重述规则正文**。design-spec 只新增 docs-spec 没有的东西：**形态判定树、默认选型、升级路径、权衡矩阵、反模式速查**。

---

## 6. 技术方案（TRD）产物结构

走完被触发的视角后，AI 输出一份**技术方案**，作为 A1 forward / B1 sync 的输入。推荐结构：

```markdown
# {需求名} 技术方案

## 1. 需求与边界（做什么 / 不做什么）
## 2. 触发的决策视角清单（本方案命中了哪几个 lens，各引用 design-spec/0X）
## 3. 各视角决策结论（每个视角：识别的形态 → 选定方案 → 关键权衡 → 升级触发条件）
## 4. 对 docs/ 的影响清单（每条结论落到哪篇 docs/，按哪条 docs-spec 表示）
## 5. 对 BUILD_STATUS 的影响（新增 ⬜ / 触碰 🚫）
## 6. 风险与未决项
```

> 方案本身不替代 docs/——它是"决策记录 + 落地索引"。落地仍走既有 docs-spec 表示规范，由 Skill 生成代码与测试。

---

## 7. 占位符纪律（继承 PRINCIPLES §12）

`design-spec/*` 同属"指导文档"，**主体规范内容必须占位符化**，不得出现具体业务名；具体业务示例只能进末尾"参考示例（仅示意，落地按业务替换）"段，段首带 `> ⚠️`。决策树、判定表、权衡矩阵中的方案名用通用技术词（如"聚合器 / 扇出 / pipeline / JOIN / 批量读"），不用业务名。`doc-sync-check` 的业务名泄漏专项检查同样覆盖本目录。

---

## 8. 与 Skill 的联动

| Skill | 与 design-spec 的关系 |
| --- | --- |
| **`/design-solution`**（A0 设计类 · 上游执行器） | 输入需求 → 按十二视角逐个过决策树 → 产出 §6 的技术方案 TRD |
| `/io-review` / `/concurrency-review` / `/performance-review` | 审计代码时，命中反模式可回链到对应 design-spec/0X §6 的选型建议 |
| `/sync-feature-to-docs`（B1） | 需求迭代触及某视角时，先回看对应 lens 是否需要更新决策结论 |

> `design-spec/` 现有十二视角骨架（01-07 旗舰核心 + 08-12 异步 / 契约 / 多副本 / 时间轴四条边），其唯一执行器为 `/design-solution`（模板见 [`templates/skills/design-solution/SKILL.md`](../templates/skills/design-solution/SKILL.md)，已在 INDEX / OPERATIONS / PRINCIPLES / skills-spec 登记为 A0 设计类 Skill）。后续新增架构维度时，按 §9 维护流程评估是否扩 lens。

---

## 9. 维护流程

| 触发 | 动作 |
| --- | --- |
| 某视角出现新的"形态 → 选型"经验 | 更新对应 lens §3 决策树 / §4 升级路径 |
| 某条选型在实践中被证伪 | §5 权衡矩阵记录，§6 反模式补充 |
| docs-spec 对应篇的硬规则变更 | 检查 lens §6 / §8 的引用是否仍指向正确锚点（只改引用，不复制规则） |
| 新增一个架构维度 | 评估是否新增 lens（沿用 §4 九节骨架）+ 在 §3 清单登记 + INDEX 登记 |


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
