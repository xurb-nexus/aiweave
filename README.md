# AIWeave — Go 后端工程「AI 原生 + AI 可重建 + 高性能架构」规范

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![docs-spec](https://img.shields.io/badge/docs--spec-28-informational.svg)](docs-spec/)
[![design-spec](https://img.shields.io/badge/design--spec-13-informational.svg)](design-spec/)
[![skills](https://img.shields.io/badge/skills-19-informational.svg)](templates/skills/)
[![Author](https://img.shields.io/badge/author-XuRuibo-orange.svg)](#10-作者与联系方式)

> **作者**：XuRuibo &lt;hustxurb@163.com&gt;
> 适用：基于 Gin / GORM / cobra / sarama 的 Go 后端微服务
> 目标读者：项目架构师、AI Agent（Claude Code 等）

**AIWeave**（AI 编织）—— 名字呼应规范的两面：

- **编织**：代码 ↔ 文档持续双向同步，永不断裂
- **AI**：所有编织动作都由 AI 按规范执行，不依赖人工自觉

> *docs/ 是系统的完整蓝图，.claude/skills/ 是 AI 的操作手册。仅凭这两者，AI 可以从零重建整个工程。*

AIWeave 把这一断言落地为：**高性能架构方法论 + 目录结构规范 + 文档内容规范 + Skill 步骤化规范 + 工作流 + 自检清单 + 可复制骨架**，让任何符合规范的 Go 后端工程都能享受"AI 原生 + AI 可重建 + 高性能架构"三重红利。

> **三支柱**——AIWeave 不只"事后织文档"，也**参与架构决策**：
>
> - **`design-spec/`** 高性能架构方法论 —— 面对需求**怎么做出**架构决策（技术方案 / TRD 生成参考）
> - **`docs-spec/`** 文档表示规范 —— 决策**怎么写下来**（精准到可重建代码）
> - **`skills-spec/`** Skill 执行规范 —— AI **怎么按文档生成**代码
>
> 三者正交：**design 管"怎么选" · docs 管"怎么写" · skills 管"怎么做"**。

---

## 1. 这套规范要解决什么问题

传统后端工程的代码与文档关系是：**人写代码 → 人写文档（通常滞后或缺失）**。

这种范式下：
- 重构发生时，文档无法跟上代码漂移；
- 新成员（无论人或 AI）阅读代码无法快速建立"为什么这样设计"的心智模型；
- 想把同一个系统在新技术栈/新平台上重建时，必须重新做一遍设计——文档没有保留可执行级别的精度。

**AI 原生 + AI 可重建** 范式倒转了这个关系：

```
用 design-spec 十三视角做架构决策  ──→  产出技术方案(TRD)
        │
        ▼
人按 TRD 写设计文档（docs/）  ──→  AI 按文档生成代码（按 .claude/skills/）  ──→  代码变更反向同步文档
       ↑                                                                         │
       └──────────────────────  双向一致性约束  ───────────────────────────────────┘
```

核心断言：

> **`docs/` 是系统的完整蓝图，`.claude/skills/` 是 AI 的操作手册。仅凭这两者，AI 可以从零重建整个工程。**

为达成这个断言，文档必须满足两条硬约束：

1. **代码与文档无比保持一致**——任何方向的变更都强制同步另一侧；
2. **文档必须精准到可重建代码**——伪代码到函数签名级，DDL 到字段类型级，缓存设计到 Hash 字段映射级，路由表到中间件链级。

本规范就是把这两条约束落地成 **可机械检查的目录结构、文件内容规则、Skill 步骤化流程**。

---

## 1.5 两大使用模式 / 四大场景

本规范覆盖工程从启动到长期演进的全生命周期。使用方式分两大模式：

### 模式 A：建设模式（一次性）

把这套规范作为**蓝本**，从零建设或大重构时一次性铺就 docs/ + .claude/skills/。包含两个具体场景：

| 场景 | 触发 | 输入 | AI 任务 |
|------|------|------|---------|
| **A0 — design**（技术方案设计 / 上游） | 需求待定架构 | 业务需求 + design-spec | 过十三视角（`/design-solution`）产出技术方案(TRD) → 喂给 A1 / B1 |
| **A1 — forward**（从零开发） | 新工程启动 | 业务需求（或 A0 的 TRD）+ 本规范 | 按规范填充 docs/ → 按 Stage 顺序生成代码 |
| **A2 — rebuild**（从 docs 重建代码） | 代码丢失/换技术栈 | docs/ + 本规范 | 按 Stage 顺序调用 Skill 重建代码 |

### 模式 B：增量同步模式（持续）

工程上线后进入持续演进期。每次需求交付时，**把"需求技术文档 + 代码改动 + 本规范"一起喂给 AI**，AI 按本规范的"范围判定表"扫描所有改动，**反向同步**全链路文档（INDEX、BUILD_STATUS、各架构 md、API 规格、Service 设计、缓存设计、错误码、常量等）。

| 场景 | 触发 | 输入 | AI 任务 |
|------|------|------|---------|
| **B1 — sync-feature**（需求迭代后同步） | 每次需求合并前 / 发版前 | 需求技术文档 + git diff + 本规范 | 按 [`docs-spec/19_incremental_sync_spec.md`](docs-spec/19_incremental_sync_spec.md) 流程更新所有受影响 md |
| **B2 — refactor**（演进重构） | 跨模块重构 | 旧代码 + 新设计 docs + 本规范 | 按 mvp_rebuild_path 切片替换 |

### 哪个场景属于哪个模式

```
工程生命周期：
  启动 ──── 建设模式（A1 forward）
    │
    ▼
  上线 ──── 进入增量同步模式（B1 sync-feature） ─── 长期运行
    │
    ▼
  代码丢失/换栈 ──── 建设模式（A2 rebuild）
    │
    ▼
  大重构 ──── 建设模式（B2 refactor，复用 mvp_rebuild_path 心智）
```

模式 A 是**一次性事件**，模式 B 是**常态**。一个工程在生命周期中，**模式 B 的次数远多于模式 A**——所以本规范对模式 B 的支持与模式 A 同等重要，详见 [`docs-spec/19_incremental_sync_spec.md`](docs-spec/19_incremental_sync_spec.md)。

---

## 2. 适用与不适用

### 适用

- Go 1.20+ 后端微服务
- HTTP 服务（Gin 或同生态框架）+ 任意子集：MySQL / PostgreSQL、Redis、Kafka / RocketMQ、定时任务、CLI 命令行
- 任意业务复杂度（从单一 CRUD 到多模块大型系统）
- 与 Claude Code、Cursor、Continue 等 AI Coding Agent 协作

### 不适用

- 一次性脚手架代码（用完即弃，写文档投入产出比低）
- 无后端组件的纯前端工程
- 业务高度动态、设计每周变化的早期 PoC（设计未稳定时频繁改 md 反而是负担——稳定后再启用本规范）

---

## 2.5 第三支柱：design-spec 高性能架构方法论

`design-spec/` 回答 docs-spec 不回答的问题——**面对需求，怎么"做出"架构决策**，而不是决策定了之后怎么写下来。它是"技术方案(TRD)生成"的参考，按 **十三大决策视角（lens）** 组织（01-07 同步请求路径核心 + 08-12 异步 / 契约 / 多副本 / 时间轴四条边 + 13 横切观测面），并内置一套从生产实践沉淀的**高性能基线**：

| 视角 | 决策什么（含高性能极致点） |
|------|--------------------------|
| **IO** ⭐ | 三维度（次数 / 串并行 / 往返）+ 读写路径分治 + 三条铁律 + 聚合器五阶段 + 批量 Upsert / 装载 + **计算下推 / 数据本地性 / 大写分批** |
| **数据建模** | 数据访问层范式 / 分库分表 / shard 分组并行 / **覆盖索引 + ICP** / 组装层 map 入参 / **存储引擎匹配（LSM·B-tree·列存 RUM 权衡）/ OLTP-OLAP 分离** |
| **并发** | 同步原语选型 / **协程池 Little 定律定容** / 阻塞不变量 / 内联兜底提交器 |
| **事务** | 一致性强度判定 / **弱一致 + 边界兜底** / 读写分离消幂等 |
| **缓存** | 本地缓存准入 / 读写非对称降级 / 回源保护三类（含 XFetch）/ 紧凑编码 + 概率位图 / **大 key·热 key·分片可扩展** |
| **稳定性** | **SRE 概率丢弃熔断** / 连接池基线 / 数据层禁日志 / 危险开关双门禁 |
| **尾延迟与过载** | **对冲 / 备份请求** / 静态池→**自适应并发限制** / **优先级·截止感知 load shedding** / 重试预算 / 冷启预热（压 P99·P999 长尾） |
| **异步消息 / 事件驱动** 🆕 | **投递语义（at-most/at-least/exactly-once）** / 顺序分区 / **双写一致（Outbox·CDC）** / 消费幂等 / 消费扩展与背压 / 扇出（工作队列 vs 发布订阅） |
| **API 契约 / 接口粒度** 🆕 | **接口粒度（chatty vs chunky，决定调用方 N+1）** / 批量·聚合端点 / 分页契约 / **版本兼容 expand-contract** / 字段投影 / 幂等键 / 协议选型 |
| **定时任务 / 分布式协调** 🆕 | **单例 vs 每副本判定** / 原子认领·分布式锁·leader 选举·外部调度 / 幂等可重入 / **fencing 防脑裂** |
| **演进与在线迁移** 🆕 | **扩展-收缩（expand-contract）** / 在线 DDL / **限流回填** / 双写双读 + 影子比对灰度 / 回滚闸门（零停机） |
| **容量规划与弹性伸缩** 🆕 | **从 SLA 反推容量** / 余量（ρ≈0.8 膝盖）/ 资源配比一致性 / 伸缩信号与防抖 / 有状态 vs 无状态伸缩 |
| **可观测性 / 服务可视化** 🆕 | **控制环可观测性（开了控制环就看得见饱和 / 触发）** / 请求面 RED · 资源面 USE · 控制面三面 / **SLO 反推指标+告警+面板** / collector 形态选型 / 服务标准面板四行 |

每个视角 = `识别形态 → 决策树 → 默认选型 + 升级 → 权衡 → 反模式 → 落到哪篇 docs`。**决策方法论在 design-spec，硬规则真相源仍在 docs-spec，只引用不复制**（双源真相规避）。

> **借纪律不借实现**：design-spec 只给占位符化的纪律与原语选型（如"同质大对象 → MGET、异构小字段 → Pipeline"），不抄任何具体工程的魔法数字——这正是 §12 占位符纪律在架构方法论层的同构。

**执行器**：`/design-solution {需求}` —— 过十三视角产出 TRD，再进 A1 forward / B1 sync 落地为 docs/ 与代码。

---

## 3. 规范目录全景

```
aiweave/
├── README.md                # 本文
├── INDEX.md                 # 索引（所有规范文档的目录）
├── PRINCIPLES.md            # 核心原则（黄金法则、双向同步纪律、签名冲突裁决）
│
├── docs-spec/               # docs/ 目录的规范（编号 00-27，覆盖完整设计视角）
│   ├── 00_directory_layout.md      # 目录布局
│   ├── 01_index_md_spec.md         # INDEX.md 写法
│   ├── 02_build_status_md_spec.md  # 实现进度 + 约束清单状态 + 运行时基线
│   ├── 03_claude_md_spec.md        # CLAUDE.md（项目入口 + 同步纪律）
│   ├── 04_architecture_overview_spec.md     # 架构总纲及衍生文档
│   ├── 05_schema_design_spec.md             # 数据库 DDL + 分库 + Redis Key + 字段演进
│   ├── 06_cache_design_spec.md              # KV 缓存设计 + 上下游
│   ├── 07_api_interfaces_spec.md            # API 接口详细设计与实现
│   ├── 08_api_flow_spec.md                  # API 逻辑描述与流程串联
│   ├── 09_service_design_spec.md            # Service 层方法签名 + 伪代码 + 领域不变量
│   ├── 10_scheduled_tasks_spec.md           # 定时任务（cobra/cycle/crontab）
│   ├── 11_mq_consumer_spec.md               # MQ 消费者
│   ├── 12_circuit_breaker_spec.md           # 熔断 / 降级 / 限流（L1）
│   ├── 13_logging_spec.md                   # 结构化日志规范
│   ├── 14_status_codes_spec.md              # 错误码 / 状态码体系
│   ├── 15_helpers_pkg_api_spec.md           # 工具函数 + 框架 API 契约
│   ├── 16_constants_spec.md                 # 业务常量唯一真相源
│   ├── 17_testing_design_spec.md            # 测试框架与纪律（含并发 / 性能回归 / 故障注入）
│   ├── 18_mvp_rebuild_path_spec.md          # 分阶段构建路径 + 安全重构方法论 + 灰度切流量
│   ├── 19_incremental_sync_spec.md          # 增量需求同步规范（B1 场景）
│   ├── 20_concurrency_safety_spec.md        # 并发安全与运行时约束
│   ├── 21_distributed_transaction_spec.md   # 分布式事务与补偿设计
│   ├── 22_performance_contract_spec.md      # 性能合约与热路径
│   ├── 23_observability_spec.md             # 可观测性（Metrics 限服务级）
│   ├── 24_cross_service_contract_spec.md    # 跨服务合约
│   ├── 25_io_aggregation_spec.md            # IO 极致与聚合并行（三条 IO 铁律：批量优先 / 禁止 N+1 / 禁止串行编排）
│   ├── 26_config_center_spec.md             # 配置中心与凭据加密（配置上云 + 密钥不入库 + fail-fast）
│   └── 27_deployment_runtime_spec.md        # 部署与运行时生命周期（部署产物 + 探针分层 + 优雅启停 + 发布回滚）
│
├── design-spec/             # 🆕 高性能架构方法论（第三支柱 · 技术方案生成参考 · 编号 00-13）
│   ├── 00_design_overview.md       # 设计方法论总纲（十三视角清单 + 统一 lens 九节骨架 + 双源真相边界 + TRD 产物）
│   ├── 01_io_design.md      ⭐     # IO 设计决策（三维度 + 读写路径分治 + 三铁律 + 聚合器 + 批量写极致；引用 docs-spec/25）
│   ├── 02_data_model_design.md     # 数据建模决策（数据访问层范式 / 分库分表+shard并行 / 覆盖索引+ICP / 组装层 map 入参）
│   ├── 03_concurrency_design.md    # 并发模型决策（原语选型 / 协程池 Little 定律定容 / 阻塞不变量 / 内联兜底）
│   ├── 04_transaction_design.md    # 事务一致性决策（一致性强度判定 / 弱一致+边界兜底 / 读写分离消幂等）
│   ├── 05_caching_design.md        # 缓存策略决策（本地缓存准入 / 回源保护三类 / 紧凑编码+概率位图 / 大key·热key·分片可扩展）
│   ├── 06_resilience_design.md     # 稳定性决策（SRE 概率丢弃熔断 / 连接池基线 / 数据层禁日志 / 危险开关双门禁）
│   ├── 07_tail_latency_design.md   # 尾延迟与过载决策（对冲/备份请求 / 自适应并发限制 / 优先级·截止 load shedding / 重试预算 / 冷启预热）
│   ├── 08_messaging_design.md  🆕  # 异步消息/事件驱动决策（投递语义 / 顺序分区 / 双写一致 Outbox·CDC / 消费幂等 / 背压 / 扇出）
│   ├── 09_api_contract_design.md 🆕 # API 契约/接口粒度决策（粒度防客户端 N+1 / 批量聚合端点 / 分页契约 / 版本兼容 expand-contract / 协议选型）
│   ├── 10_scheduling_coordination_design.md 🆕 # 定时任务/分布式协调决策（单例 vs 每副本 / 原子认领·锁·leader·外部调度 / 幂等可重入 / fencing 防脑裂）
│   ├── 11_evolution_migration_design.md 🆕 # 演进与在线迁移决策（扩展-收缩 / 在线 DDL / 限流回填 / 双写双读影子比对灰度 / 回滚闸门）
│   ├── 12_capacity_scaling_design.md 🆕 # 容量规划与弹性伸缩决策（从 SLA 反推容量 / ρ≈0.8 余量 / 资源配比 / 伸缩信号与防抖 / 有状态伸缩）
│   └── 13_observability_design.md  🆕 # 可观测性 / 服务可视化决策（控制环可观测性 / RED·USE·控制面三面 / SLO 反推面板 / collector 形态选型）
│
├── skills-spec/             # .claude/skills/ 的规范
│   ├── 00_skill_overview.md         # Skill 体系总览（频率、命名、调用顺序）
│   ├── 01_skill_authoring_guide.md  # 编写指南（frontmatter / 步骤化 / 文档同步 / 测试同步）
│   ├── 02_settings_local_json_spec.md  # settings 规范（permissions + Hooks 强制机制：IO 铁律 + 双向同步）
│
├── OPERATIONS.md            # 操作手册（8 类工作流 + 3 类自检清单 + 增量同步专属 + 6 层防御×W 工作流对照表，含 L0 Hooks）
│
└── templates/               # 可直接复制到新项目的骨架
    ├── CLAUDE.md
    ├── docs/                # 完整 docs/ 骨架（每篇 md 一个空白模板）
    │   ├── INDEX.md
    │   ├── BUILD_STATUS.md  # 含约束清单状态轨道 + 运行时基线区域
    │   ├── architecture/    # 含 concurrency_safety / performance_contract /
    │   │                    # observability / cross_service_contract /
    │   │                    # runtime_profile / ai_dev_guide 等约束类文档
    │   ├── api/
    │   ├── service/         # 含 transaction_design.md
    │   ├── schema/
    │   ├── cache/
    │   ├── circuit_breaker/
    │   └── testing/
    └── skills/              # 19 个 .claude/skills/{name}/SKILL.md 骨架
                             # 5 类分组：设计 1 / 创建 9 / 维护 2 / 审计 6 / 终极 1
```

---

## 4. 如何使用本规范

### 4.1 建设模式（A1：从零启动新项目）

#### 阶段 0：技术方案设计（可选 / 复杂需求强烈推荐）

用 design-spec 十三视角对需求做架构决策，产出 TRD（简单 CRUD 可跳过，直接进阶段 A）：

```
/design-solution {需求}  →  过 IO / 数据建模 / 并发 / 事务 / 缓存 / 稳定性 / 尾延迟
                            / 异步消息 / API 契约 / 定时协调 / 演进迁移 / 容量伸缩 / 可观测性 十三视角  →  技术方案(TRD)
```

TRD 的"对 docs/ 的影响清单"直接告诉你阶段 B 要填哪些 md、按哪条 docs-spec 写。

#### 阶段 A：复制骨架（30 分钟）

```bash
# 1. 复制 docs/ 骨架到新项目根目录
cp -r aiweave/templates/docs/ <new-project>/docs/

# 2. 复制 .claude/skills/ 骨架
mkdir -p <new-project>/.claude
cp -r aiweave/templates/skills/ <new-project>/.claude/skills/

# 3. 复制 CLAUDE.md
cp aiweave/templates/CLAUDE.md <new-project>/CLAUDE.md
```

#### 阶段 B：填充设计（按 `docs-spec/` 顺序）

按以下顺序逐篇填充 `docs/` 下的空白模板。每篇填完都形成一个独立可评审的产物：

1. `docs/INDEX.md` — 列好计划写的所有 md（不必全部写完）
2. `docs/BUILD_STATUS.md` — 标 ⬜ 占位
3. `docs/architecture/overview.md` — 架构总纲（先写完）
4. `docs/schema/database_design.md` — 全部表 DDL（设计先行）
5. `docs/cache/cache_design.md` — Redis Key 全清单 + Hash 字段映射
6. `docs/api/*.md` — 每个调用方的接口规格
7. `docs/service/service_design.md` — Service 层伪代码
8. 其余 md 按需填充

每篇 md 的写法严格遵循 `aiweave/docs-spec/` 对应规范。

#### 阶段 C：分阶段实现（按 `mvp_rebuild_path.md`）

按 `docs/architecture/mvp_rebuild_path.md` 把整工程拆成若干个独立可交付的 Stage。每个 Stage 内：

```
读 BUILD_STATUS  →  调对应 Skill 生成代码  →  go build / go test 验证
       ↑                                                  │
       └─────  更新 BUILD_STATUS + 反向同步 md  ───────────┘
```

### 4.2 建设模式（A2：从 docs 重建代码）

适用场景：代码丢失、换技术栈、大型回滚。

```
前置：docs/ 已完整，.claude/skills/ 已就绪
   ▼
按 docs/architecture/mvp_rebuild_path.md 的 Stage 顺序，对每个 Stage：
   1. 读 BUILD_STATUS 标记 🟡（进行中）
   2. 用 /rebuild-from-docs {stage_module} 生成代码
   3. go build / go test 验证
   4. 测试用例同步
   5. BUILD_STATUS 标记 🟢
   6. 用户确认后进入下一 Stage
```

⚠️ 不要直接 `/rebuild-from-docs all`，按 Stage 推进。

### 4.3 增量同步模式（B1：需求迭代后）

**这是工程的常态使用方式**。每次需求合并前：

```
你（开发者）：把以下三类输入交给 AI——
   ① 本规范（aiweave/）
   ② 新需求的技术设计文档
   ③ 代码改动（git diff / PR / 改动文件清单）

AI（按 docs-spec/19_incremental_sync_spec.md 执行）：
   1. 读输入 + 本工程 INDEX/BUILD_STATUS/CLAUDE
   2. 检查 🚫 暂不实现模块拒绝规则
   3. 按"范围判定表"对改动文件分类
   4. 对每篇受影响的 md，按 docs-spec/ 对应规范更新
   5. 交叉一致性检查（INDEX / BUILD_STATUS / 互引用）
   6. 输出"全链路文档变更清单 + 自检结果 + 文档同步声明"
```

详细流程见 [`docs-spec/19_incremental_sync_spec.md`](docs-spec/19_incremental_sync_spec.md)。
对应 Skill：[`templates/skills/sync-feature-to-docs/SKILL.md`](templates/skills/sync-feature-to-docs/SKILL.md)。

### 4.4 日常维护

- **新增/改代码** → 按 `OPERATIONS.md` 第一部分对应工作流
- **每次需求合并前** → 调 `/sync-feature-to-docs`（增量同步）
- **每次发版前** → 跑 `/doc-sync-check all`（兜底审计）
- **每次交付前** → 跑 `OPERATIONS.md` 第二部分自检清单

---

## 5. 黄金法则

详见 `PRINCIPLES.md`。一句话总览：

> **文档写得越精确，AI 生成的代码越正确。把时间花在设计上，而不是编码上。**

更进一步的硬纪律：

| 法则 | 含义 |
|------|------|
| 双向同步 | 任何方向的变更都同步另一侧。无例外。 |
| 文档优先 | 默认以文档为真相源。 |
| 签名冲突裁决 | 但当文档与既有可编译代码冲突时，以代码为真相源（详见 `PRINCIPLES.md` §5）。 |
| BUILD_STATUS 先查 | 生成任何"已有文档"的代码前，先查 BUILD_STATUS，区分🟢已实现 vs ⬜待实现 vs 🚫暂不实现。 |
| 代码必同步测试 | 业务代码 + 测试 + 文档同步是一次完整交付，缺测试视为未交付。 |
| 暂不实现机制 | 设计完整保留、代码不生成的模块，必须在 BUILD_STATUS 显式登记，并在 Skill 黑名单中拒绝。 |

---

## 6. 关于"参考实现"

AIWeave 中的每条规则都从真实工程实践沉淀。在某些规范文档里，会出现 `本工程` 这样的占位词——它指代任何一个完整落地了 AIWeave 的 Go 后端工程，**不绑定任何特定项目**。

如果你在落地 AIWeave 时希望对照一个真实工程作参考，可以：

- 选一个已按 AIWeave 规范完整建设的内部项目（含完整 `docs/` + `.claude/skills/`）作为蓝本
- 或从你自己的项目逐步建设，每完成一篇 docs 都按 `docs-spec/` 中对应规范自检

AIWeave 自身不耦合任何具体业务领域；规则中出现的 `{module-N}` / `{example_table_*}` / `{ns}:info` 等占位符，都需要落地到具体工程时由你按业务语义替换。

---

## 7. 反馈与演进

本规范不是一次性产物。当某条规则在实践中被证明：

- 太严苛（导致频繁绕过） → 放宽或拆分
- 太宽松（产生不一致） → 收紧或加自动化检查
- 未覆盖（出现新场景） → 增补章节

规范的演进记录在 git 提交历史 / Release 中，每次重要变更在提交信息注明触发的实践事件。

---

## 8. License

本项目采用 **Apache License 2.0** 开源协议——详见 [LICENSE](LICENSE) 文件。

```
Copyright 2026 XuRuibo

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

完整归属声明见 [NOTICE](NOTICE) 文件。

---

## 9. 反馈渠道

欢迎以以下任一形式参与：

- 🐛 [Bug 报告](.github/ISSUE_TEMPLATE/bug_report.md) — 规范笔误 / 链接失效 / 章节错乱
- 📝 [规范条款澄清](.github/ISSUE_TEMPLATE/spec_clarification.md) — 某条规则歧义、矛盾或落地难题
- 💡 [功能请求 / 新规范提议](.github/ISSUE_TEMPLATE/feature_request.md) — 提议新增 docs-spec / Skill / 工作流
- 🔧 提交 Pull Request（使用本仓库 [PR 模板](.github/PULL_REQUEST_TEMPLATE.md)）
- 🌟 Star 本项目 / 在工程中落地后反馈实践案例

---

## 10. 作者与联系方式

**XuRuibo** — AIWeave 框架作者

- 📧 邮箱：[hustxurb@163.com](mailto:hustxurb@163.com)
- 💼 项目仓库：本仓库

如有规范层面的疑问、落地实施经验分享、或希望深度协作，欢迎提 Issue 或邮件联系。

> AIWeave 的所有规则都从真实后端工程实践沉淀而来；社区分享的实践案例会反哺规范的演进（详见 §7）。


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
