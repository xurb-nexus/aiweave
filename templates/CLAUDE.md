# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

> 项目启动时复制本骨架到项目根 `CLAUDE.md`，按 `aiweave/docs-spec/03_claude_md_spec.md` 规范填充。

---

## 项目概述

{ProjectName}（{ProjectAlias}），基于 {Framework}（Go {Version}），是一个{独立微服务/库/...}，提供 {核心能力清单}。架构上设计为 {N 种运行模式共一份代码库}，由 {cli framework} 子命令区分入口。

> **当前实现状态**：项目处于"{当前阶段}"阶段。完整实现状态一览见 [`docs/BUILD_STATUS.md`](docs/BUILD_STATUS.md)。当你读到文档里某个 service 方法、接口或定时任务时，**不要假设它已在代码中实现**——先查 BUILD_STATUS，再决定是"调用现有实现"还是"按 skill 生成新实现"。

> **⛔ 暂不实现模块（如有，临时决策）**：**{module} 模块**（涉及 `{path1}`、`{path2}`）当前阶段**一律不进入代码实现**。设计文档完整保留，但 AI 在任何 skill 下都**不得**为这些模块生成代码。完整列表、检查规则、解除条件见 [`docs/BUILD_STATUS.md`](docs/BUILD_STATUS.md) §0。

## 常用命令

### 构建与运行

```bash
# 启动 HTTP 服务（默认）
go run main.go

# 启动指定任务
go run main.go {task-name}

# 构建二进制文件
go build -o {binary-name}

# 查看帮助
go run main.go -h
```

### 测试

```bash
# 运行所有测试
go test ./...

# 运行指定包的测试
go test ./{package}/...

# 运行单个测试文件
go test -v ./{path}/{file}_test.go

# 运行带覆盖率
go test -cover ./...
```

### 代码格式化

```bash
go fmt ./...
```

## 项目架构

### 目录结构

```
project/
├── main.go              # 入口文件，使用 {cli} 管理命令行
├── conf/                # 配置目录
│   ├── app/             # 业务配置（随代码发布）
│   └── mount/           # 环境配置（动态修改）
├── controllers/         # 请求入口
│   ├── command/         # 定时任务入口
│   ├── http/            # HTTP 请求入口
│   └── mq/              # 消息队列消费入口
├── router/              # 路由注册
├── helpers/             # 资源初始化与辅助函数
├── components/          # 公共组件（状态码、常量等）
├── models/              # GORM 数据模型
├── service/             # 内部业务逻辑层
├── api/                 # 外部 API 调用封装（带熔断器）
├── data/                # 数据访问层封装
├── middleware/          # 中间件
├── log/                 # 日志文件目录
├── scripts/             # 脚本文件
├── sql/                 # SQL 脚本
├── docs/                # 技术方案文档（系统蓝图）
└── pkg/                 # 第三方/内部库封装
```

### 分层架构

```
controllers/ → service/ 或 api/ → models/ 或 data/ 或 helpers/
     ↓               ↓                       ↓
  参数校验        业务逻辑              数据访问
```

- **controllers/**：参数校验、调用 service/api、组装返回数据
- **service/**：内部业务逻辑
- **api/**：外部 API 调用，带熔断器保护
- **models/**：GORM 数据模型
- **data/**：复杂数据访问封装
- **helpers/**：资源初始化（MySQL / Redis / MQ / 熔断器等），提供全局客户端实例

> **依赖严格单向、禁止反向 import**：下层（`models`/`data`/`helpers`）不得 import 上层（`service`/`controllers`），同层不得循环依赖。这让 `service` / `data` 成为可无副作用复用的纯单元（决策见 design-spec/06 §3.3）。

### 配置体系

配置文件位于 `conf/` 目录：

- **mount/**：环境相关，可由配置中心动态修改
  - `config.yaml`：基础配置
  - `api.yaml`：外部 API 调用配置
  - `resource.yaml`：资源配置（MySQL / Redis / Kafka / MQ）
- **app/**：业务相关，随代码一起发布
  - `app.yaml`：熔断器配置等

敏感信息使用 `@@key` 格式占位，运行时由配置中心替换。

### 接口返回格式

> 如有多种响应格式（如 Internal vs UserSelf），列出每种。

```json
{ "errNo": 0, "errStr": "", "data": {} }
```

### 错误码规范

定义在 `components/error.go` / `components/service_error.go`，按业务范围划分（详见 `docs/architecture/status_codes.md`）。

### 路由注册

HTTP 路由在 `router/http.go` 注册，路由前缀为 `/{prefix}`。

MQ 消费者在 `router/mq.go` 中注册。

定时任务在 `router/command.go` 通过 cobra 命令注册。

### 支持的基础设施

> 列出本工程实际使用的基础设施。例：

- **MySQL**：N 个库
- **Redis**：N 个实例
- **Kafka**：N 个 topic
- **本地缓存**：gcache 等
- **熔断器**：Google SRE 自适应算法

### 核心设计原则

> 列出 3-5 条核心设计断言。例：

- **Redis-first**：高流量操作先写 Redis，定时任务 flush 到 MySQL
- **MQ 异步落库**：业务事件流水走 Kafka 异步持久化
- **统一熔断器**：Redis、MySQL、外部 API 全部由同一套 CircuitBreaker 接口保护

## 开发注意事项

- 控制器只负责参数校验、调用 service/api、组装返回数据，不写业务逻辑
- URI 命名使用小写字母和连字符，不使用下划线
- 外部 API 调用必须使用熔断器保护
- 新增错误码按规范定义在 `components/error.go`
- service 层处理内部业务逻辑，api 层处理外部服务调用
- 测试文件与被测文件同目录，命名为 `*_test.go`

### 日志规范（强制）

统一使用 Structured API（`tlog.*Logger`），**禁止使用** Sugared API（`tlog.*f`）。详细规范见 `docs/architecture/logging.md`。

```go
// 正确 ✅
tlog.InfoLogger(ctx, "task completed",
    tlog.String("task", taskName),
    tlog.Int("processed", count))

// 错误 ✗ — Sugared API
tlog.Infof(ctx, "task completed: %s, processed=%d", taskName, count)
```

**级别判定**：能自愈 → WARN；需人工介入 → ERROR。
**敏感数据禁止打印**：EntitySecret、Token、密码绝对不输出到日志。
**数据层禁日志**：`service` / `data` 层**不打日志**，靠 `return error` 上传，由 `controller` 在请求边界统一定级（可自愈 WARN / 需人工 ERROR）——同一方法被 HTTP/cron/job 多入口复用，自打会重复打点 + 级别误判（"未找到"在 DB 层是 `ErrRecordNotFound`，在 controller 层才是正常业务）。详见 `docs/architecture/logging.md` 分层日志纪律。

### Hooks 机制（强制 / L0 自动化防御层）

本项目通过 `.claude/settings.json`（团队共享，入 git）配置**两类强制 Hook**，由 Claude Code 在工具事件机械执行，不依赖 AI 自觉：

| Hook 族 | 事件 | 职责 |
|--------|------|------|
| **IO 铁律检查** | 编辑/写 `.go` 后 | 跑 `R-IO-*` grep 锚，命中 N+1 / 串行编排 → 告警（可升级 PreToolUse 阻断） |
| **代码↔文档双向同步** | 编辑 `.go` 后 + 会话 Stop | 按范围判定表提醒同步 md；Stop 校验"文档同步：..."声明存在 |

Hook 脚本在 `scripts/hooks/`，纯标准工具实现。配置规范见 `aiweave/skills-spec/02_settings_local_json_spec.md §4`。Hooks 是 6 层防御的 L0 层，与下方文档同步规则（L1-L5）叠加生效。

## 文档同步规则（强制）

> **本规则是本项目的最高优先级纪律，优先级高于一切其他开发规范。任何不满足文档同步的交付都视为未完成，必须补齐后才能结束。**

### 愿景

`docs/` 中的 md 是整个系统的完整蓝图。目标是：**仅凭 docs/ 中的 md 就能用 AI 从零重建整个工程**。

### 研发流程：设计先行

```
1. 先写/改 md（技术方案文档）
2. 再根据 md 写/改代码
3. 如果实现中发现 md 需要调整，立即同步更新 md
```

### 核心原则：双向同步（绝对强制）

| 变更方向 | 强制要求 |
|---------|---------|
| **md → 代码** | 严格按 md 实现；实现中发现 md 需调整，必须同步更新 md |
| **代码 → md** | 找到关联 md 并必须同步更新；如无对应 md，必须新建 |
| **新增代码** | 必须新建/更新对应 md，并在 INDEX.md 登记 |
| **删除代码** | 必须同步删除/更新对应 md，并更新 INDEX.md |

### 20 条具体执行规则

#### 规则 1：修改代码前 — 查找关联文档
按下方"范围判定表"逐条检查，确认本次修改涉及哪些 md。

#### 规则 2：修改代码后 — 同步更新文档
准确反映代码变更到对应 md。

#### 规则 3：新增代码 — 必须有文档（零容忍）
新增任何 controller / service / 任务 / 表 / Redis Key / 错误码 / 中间件 → 必须有对应 md，并在 INDEX 登记。

#### 规则 4：修改 md 后 — 检查代码一致性
md 与代码冲突时按规则 7 处理。

#### 规则 5：更新 INDEX.md
任何新增 / 删除 / 重命名 md 后，必须同步 INDEX.md。

#### 规则 6：自检清单（每次交付前必须过一遍）
- [ ] 本次新增代码 → 是否有对应 md？
- [ ] 本次修改代码 → 是否更新关联 md？
- [ ] 本次新增/修改 md → 是否更新 INDEX.md？
- [ ] 本次删除 → 是否同步另一侧？

#### 规则 7：文档优先级的例外——签名冲突裁决
- 文档"未来设计"vs 代码不存在 → **以文档为准**
- 文档"既有契约"vs 代码已被引用 → **以代码为准**

#### 规则 8：BUILD_STATUS 先查
生成"已有文档"代码前，先查 BUILD_STATUS，区分 🟢 已实现 / ⬜ 待实现 / 🚫 暂不实现。

#### 规则 9：代码必须同步测试用例（硬约束）
任何 skill 生成代码后，**必须**同步新增/扩展 `test/cases/` 用例。漏写测试视为未交付。

#### 规则 10：并发安全约束（强制）

- 新增任何全局可变状态 → 必须在 `docs/architecture/concurrency_safety.md §2` 共享状态注册表登记
- 新增 `go func()` → 必须说明生命周期终止条件（context/signal/close），并在 §1.1 Goroutine 生命周期图追加节点
- 修改锁策略 → 必须更新 §3 锁粒度决策表；多锁场景同步 §3.2 加锁顺序约束
- 新增 `chan T` → 必须在 §4.1 Channel 清单登记，含 buffer / 满时策略 / 关闭条件
- 生成代码中使用 `map` / `slice` 并发写 → 必须声明保护机制（`sync.Map` 或外层 `RWMutex`）
- 命中 `concurrency_safety.md §6 危险操作清单` → 必须修正或在行内注解 `// aiweave:allow=<rule-id>` 抑制（每文件 ≤ 3 处）

#### 规则 11：事务与一致性约束（强制）

- 涉及多数据源写入的 Service 方法 → 必须在 `docs/service/transaction_design.md §2` 事务边界清单登记
- Service 方法伪码涉及事务 / Saga / 幂等 / 锁 / 不变量 → 必须使用 `[TXN-*]` / `[SAGA-STEP-N]` / `[IDEMPOTENT-CHECK]` / `[LOCK-*]` / `[INVARIANT-CHECK]` 统一标记
- 新增写入类接口未设计幂等 Key → AI 必须主动追问"是否需要幂等保证"
- 跨数据源写入未在 §2 登记 → AI 必须先补登记再生成代码
- 强一致引入大量复杂度时 → 先评估**读写分离消幂等**（读判定 + 写执行拆两接口）/ **弱一致 + 边界兜底 + 监控**是否够用；高频数值变更热路径不加锁，越界在持久化边界 `GREATEST`/`LEAST` 夹断（见 `transaction_design.md §4.4/§5.2`）

#### 规则 12：性能合约约束（强制）

- 在热路径（`performance_contract.md §2` 清单）中生成代码时：
  - 禁止使用 `reflect.*`
  - 禁止 `fmt.Sprintf` / `fmt.Errorf`（除错误 wrap 外）
  - 禁止在循环内分配大对象
  - 禁止同步调用外部 API
  - 禁止单条 MySQL 查询（必须批量或走缓存）
- 新增热路径方法 → 必须在 §2 清单登记 + 方法伪码加 `[HOT-PATH]` 标记
- 新增 channel → 容量必须在 `performance_contract.md §6.1` 登记，与 `concurrency_safety.md §4.1` 双向对齐
- 修改 SLA 数值 → 必须同步 `performance_contract.md §1` 与 `observability.md §5.2` 告警阈值

#### 规则 13：可观测性约束（强制）

- 禁止为业务数据创建 Metric（`{业务实体}_count` / `{核心数值字段}_sum` / `{状态字段}_distribution` 等 → 用日志 + 离线分析）
- 禁止在 `helpers/metrics.go` 之外注册 Metric
- 新增 Metric 前必须评估 cardinality（label 各维度枚举值相乘 ≤ 100 默认上限；突破需在 `ai_dev_guide.md §9` 登记）
- 禁止使用无界值作为 label（`{actor-id}` / IP / `request_id` / 动态路径）
- url label 必须使用路由模板，不可使用实际请求 URL
- 新增告警规则 → 同步 `observability.md §5.2`

#### 规则 14：Schema 演进约束（强制）

- 新增字段必须有默认值
- 字段类型变更必须兼容旧数据（如 INT → BIGINT 可；BIGINT → INT 不可）
- 删除字段必须先在 `database_design.md §8.1` 登记废弃，经过至少一个发版周期后再物理删除（DROP COLUMN）
- 修改字段语义 → 必须同步 `database_design.md §8.2` 字段语义变更记录

#### 规则 15：安全重构约束（强制）

- 不得一次性删除并重写超过 200 行的已有代码
- 重构必须拆分为 ≤ 5 个独立可回滚的 commit
- 每个 commit 必须通过完整测试
- 涉及"行为变更"的重构必须按 `mvp_rebuild_path.md §11.5.4` 灰度切流量（Shadow read → 双写 → 1%/10%/50%/100%）

#### 规则 16：跨服务合约约束（强制）

- 调用下游服务 / 新增 `api/` client → 必须在 `cross_service_contract.md §3` 下游合约登记（含 SLA / 超时 / 重试 / 熔断 / 降级）
- 新增对外路由被外部接入 → 必须在 `cross_service_contract.md §2` 上游合约登记
- 修改对外接口签名 → 必须按 `cross_service_contract.md §5` 评估是否需要升主版本号
- 下游调用必须显式设超时（禁止 0 或缺省）；本服务对下游设的超时必须 < 上游对本服务的超时（防级联）
- 禁止 `_ = downstream.Call()` 静默吞下游错误；每条失败分支必须在 `cross_service_contract.md §4` 故障传播矩阵覆盖

#### 规则 17：IO 聚合约束（强制 / 三条 IO 铁律）

- **铁律零（批量优先）**：对一批 key 的查询必须用批量方法一次完成；数据层缺批量方法时**先补 `{...List}`**（分表按 shard 分组并行）再调用，禁止退化为循环单查
- **铁律一（禁止 N+1）**：禁止在循环 / 递归内对同类资源逐条查询（DB / 缓存 / RPC / 外部 API）→ 必须循环外收集 key + 一次批量读 + `map` 装配
- **铁律二（禁止独立串行编排）**：多个**互不依赖**的 IO 禁止逐个串行 await → 必须聚合器或扇出并行；仅当后查询入参真实依赖前查询结果时串行才合法
- 多次跨实例 / 跨数据源读 → 必须走聚合器（按实例分组批量 + 并行回源 + 单飞合并 + 异步写回）
- 聚合器 / single-flight 的共享结果指针 **只读**，禁止就地改字段（并发写竞争）→ 需改先深拷贝
- 循环内单条写 → 必须批量 / pipeline
- **反向纪律（防过度优化）**：同阶段仅 1 个查询 → 禁止套并行编排器；已被本地缓存命中的读 → 禁止再合并 Pipeline（绕过本地缓存反劣化）
- **批量写极致**：插入/更新混合 → 批量 Upsert（`ON DUPLICATE KEY` / `ON CONFLICT`，一次往返免 select-then-write）；大批量装载（冷启 / 迁移 / 离线导入）→ `LOAD DATA` / `COPY`（见 `io_contract.md §5.7`）
- **读路径覆盖索引**：高频查询优先覆盖索引（index-only scan 免回表）+ ICP（WHERE 非前缀条件在索引层先过滤）；同时不过度索引（见 `database_design.md §7.1/§7.2`）
- 新增"集合访问资源 / 多依赖编排"方法 → 必须在 `io_contract.md §1` 登记往返预算、`§3/§4` 登记原语，伪码加 `[BATCH]` / `[PARALLEL]` 标记
- 拿不准是否构成 N+1 / 伪串行 → 主动报告并建议聚合方案，不擅自串行

#### 规则 18：配置中心与凭据加密约束（强制）

- 配置中心 / 资源连接凭据（账号 / 密码）→ 必须密文存放（对称加密，如 AES-256-GCM），**禁止明文**落盘 / 入库
- 加密密钥 → 必须由应用层 / 环境注入，**禁止硬编码、禁止入 git**；框架库不内置密钥
- 环境相关配置以**配置中心为权威源**，本地文件是启动期落盘副本；按 `config.md §5` 权威源矩阵归位，不混放业务 / 环境 / 凭据
- 新增配置中心拉取项 → 登记 `config.md §7.3` 拉取映射；拉取失败 / 空内容 → 必须 **fail-fast**（禁止静默空/旧配置启动）
- 接入新配置中心 → 实现 `config.md §7.1` Client 接口，网络调用只在实现层
- 出现明文凭据 / 硬编码密钥 → 主动报告，拒绝提交

#### 规则 19：部署与运行时生命周期约束（强制）

- 新增 / 修改进程入口（`main` / 子命令）→ 必须注册 `SIGTERM`/`SIGINT` 并接入优雅关闭；无信号处理 / 收到信号即 `os.Exit` → `R-SHUTDOWN-NO-SIGNAL` / `R-SHUTDOWN-NO-DRAIN`
- 优雅关闭顺序（强制）：readiness 转 false → 停 accept → 排空在途（带超时）→ 停常驻 goroutine / 消费者（先提交位点）→ 关连接池 → 退出；超时强杀兜底并记录未排空项
- 新增常驻 goroutine / 消费者 → 必须同时在 `concurrency_safety.md §1.1` 登记**且**接入关闭排空；只登记不排空 → `R-SHUTDOWN-LEAK-GOROUTINE`
- liveness 探针**禁止**查外部依赖（DB/缓存/下游），依赖检查只放 readiness → `R-PROBE-LIVENESS-DEEP`；对外服务必须暴露 liveness + readiness（慢启动加 startup），缺失 → `R-PROBE-MISSING`
- readiness 置 true 必须晚于"配置就绪（规则 18）+ 资源初始化 + 信号注册"；未就绪即 accept → 拒绝
- 容器镜像运行层**禁止** root → `R-DEPLOY-ROOT`；编排清单**禁止**缺 requests/limits → `R-DEPLOY-NO-LIMITS`；镜像引用**禁止**可变 tag（`latest`），必须内容寻址
- 含不兼容 Schema 变更的发布 → 回滚前必须确认 `database_design.md §8` 软兼容已就位
- 详见 `docs/architecture/deployment.md`

#### 规则 20：缓存设计纪律（强制）

- **本地缓存准入**：放进 L1 进程内缓存的 key 必须同时满足"数据量有界且小 / 低变更 / 可容忍陈旧 / 高频读 / 非强一致 / 低基数判定类"六条准入；高基数 / 高频变更 / 强一致 / 大对象命中排除清单 → 退 L2 Redis（见 `cache_design.md §3.5`）
- **回源保护三类分治**：并发 miss 同一 key → single-flight；同批 key 集体过期（雪崩）→ TTL 随机抖动；单热 key 到期悬崖（击穿）→ XFetch 概率提前重算。三类正交叠加，不可互相替代（见 `cache_design.md §2.10`）
- **省内存编码**：集合类型主动把元素数 / 值长压在紧凑编码阈值内（`listpack` / `intset` / `ziplist`）；只需"计数 / 存在 / 集合运算"而非取回原值时用概率 / 位图结构（HyperLogLog / Bitmap / Roaring）（见 `cache_design.md §2.11/§2.12`）
- **大 key / 热 key 治理 + 分片可扩展**：容量随分片线性扩展的三前提是"散列均匀 / 无大 key / 无热 key"。大 key 拆分桶 / 分块 + `UNLINK` 异步删；热 key 按读热（L1 挡读 / 副本打散）/ 写热（本地聚合 / 分片计数求和）分治；跨分片批量按分片分组并行，扩缩容用一致性哈希 / Cluster slot 迁移、禁裸 `mod N` 重哈希（见 `cache_design.md §9`）
- **读写非对称降级**：缓存故障降级方向由"加速层 vs 数据源"决定——加速层 miss 静默穿透 DB，数据源 miss 快速失败（绝不穿透）；写永远 best-effort 失败静默（见 `cache_design.md §1.4`）

### 危险模式清单

> AI 生成代码前/后必查。命中以下模式 → 停下来确认或修正。
>
> **本清单的 rule-id 索引 + 完整 grep 锚**详见 `docs/architecture/ai_dev_guide.md §10`；本节仅列摘要。
>
> **grep 锚定位为"信号级"非"判定级"**：命中标 🟡 待复核，最终判定权在人工 reviewer。误报通过 `// aiweave:allow=<rule-id>` 行内注解抑制，每个文件不超过 3 处。

| 危险模式 | 风险 | 正确做法 | Rule-id |
|---------|------|---------|---------|
| 锁内做网络 IO（HTTP / DB / Redis） | 锁持有时间不可控 | 先获取数据再加锁 | `R-CONC-LOCK-IO` |
| 锁内打日志 | 同上 + 写盘抖动 | 释放锁后再打日志 | `R-CONC-LOCK-LOG` |
| `map` / `slice` 并发写 | race condition | `sync.Map` 或 `RWMutex` | `R-CONC-MAP-RACE` |
| 关闭已关闭的 channel | panic | 单一所有者 close | `R-CONC-DOUBLE-CLOSE` |
| 向已关闭的 channel 发送 | panic | select + done channel | `R-CONC-SEND-CLOSED` |
| `defer` 在 for 循环内 | 资源延迟释放，可能 OOM | 抽子函数或显式 Close | `R-RESOURCE-DEFER-LOOP` |
| 忽略 `ctx.Done()` 的常驻 goroutine | goroutine 泄漏 | select case `<-ctx.Done()` | `R-CONC-GOROUTINE-LEAK` |
| 写入类接口无幂等 Key | 重试重复写 | SETNX + EXPIRE 或唯一索引 | `R-TXN-NO-IDEMPOTENT` |
| 跨数据源写入未登记事务边界 | 失败补偿无据 | 先补 `transaction_design.md §2` | `R-TXN-CROSS-SOURCE` |
| **【】for 循环内 DB 查询** | N+1 问题 | 批量查询 + map | `R-PERF-LOOP-DB-QUERY` |
| **for 循环内大对象分配** | GC 压力 | 循环前预分配 + 复用 | `R-PERF-LOOP-ALLOC` |
| **热路径用 `reflect.*`** | 反射性能开销 | 显式类型断言 / 代码生成 | `R-PERF-HOT-REFLECT` |
| **热路径用 `fmt.Sprintf`** | 反射 + 内存分配 | `strconv.*` 或 buffer | `R-PERF-HOT-FMT` |
| **全表 `COUNT(*)`** | 慢查询 | 维护计数器 / 限定 WHERE | `R-PERF-FULL-COUNT` |
| **`time.Sleep` 做同步** | 不可靠 + 浪费 CPU | channel / WaitGroup | `R-RESOURCE-SLEEP-SYNC` |
| **Redis 大 Key（> 1MB）** | 网络阻塞 + 慢查询 | 拆分 Hash / 分片 | `R-CACHE-LARGE-KEY` |
| **下游调用未显式设超时** | 上游超时被穿透 | 显式 `WithTimeout({T-ms})` | `R-XSVC-NO-TIMEOUT` |
| **本服务对下游超时 ≥ 上游对本服务超时** | 级联阻塞 | 按 `performance_contract.md §6.2` 协调 | `R-XSVC-TIMEOUT-CASCADE` |
| **`_ = downstream.Call()` 静默吞下游错误** | 故障不可见 | 显式处理 / 上报 | `R-XSVC-SILENT-SWALLOW` |
| **`api/` 新增 client 未登记 cross_service_contract §3** | 下游依赖失控 | 先补 §3 再合并 | `R-XSVC-UNREGISTERED-CLIENT` |
| **代码失败分支未在 transaction_design.md §6 失败路径全景图覆盖** | 漏路径无据可查 | 先补 §6 再合并 | `R-FAIL-PATH-UNDOC` |
| **失败分支无对应测试用例** | 静默回归 | 按 `testing_design.md §5.5` 高级用例 3 类补齐 | `R-FAIL-PATH-NO-TEST` |
| **伪码 `[INVARIANT-CHECK]` 标记与代码实现脱节** | 不变量声明虚化 | 修正标记或修正代码 | `R-INVARIANT-MARK-MISMATCH` |
| **【铁律一】循环内单条查询（DB/缓存/RPC）** | N+1，IO 放大 O(N) | 批量读 + map 装配 | `R-IO-N-PLUS-1` |
| **【铁律二】独立读串行编排（无数据依赖）** | RTT 累加，本可并行 | 聚合器 / 扇出并行 | `R-IO-SERIAL-ORCH` |
| **循环内单条写** | 写放大 | 批量写 / pipeline | `R-IO-LOOP-WRITE` |
| **多次跨实例读未走聚合器** | 往返失控 | 按实例分组批量 | `R-IO-NO-AGGREGATOR` |
| **就地修改 single-flight / Slot 共享指针字段** | 并发写竞争 | 只读；改前深拷贝 | `R-IO-SHARED-MUTATE` |
| **大 key 用 `DEL` 删除** | 阻塞 Redis 单线程 | `UNLINK` 异步惰性删 | `R-CACHE-BIGKEY-DEL` |
| **热 key 砸单分片（无打散）** | 单分片 CPU 饱和，加分片分不走 | 读热 L1/副本打散、写热分片计数（见 cache_design.md §9.2） | `R-CACHE-HOTKEY-SINGLE-SHARD` |
| **高基数 key 塞 L1 本地缓存** | 内存 × N 实例爆 + 实例发散 | 退 L2 Redis（见 cache_design.md §3.5 排除清单） | `R-CACHE-L1-HIGH-CARD` |
| **扩缩容用裸 `mod N` 重哈希** | 全量重哈希、缓存大面积失效 | 一致性哈希（只迁 K/N）/ Cluster slot 迁移 | `R-CACHE-MOD-N-REHASH` |
| **配置中心 / 资源凭据明文落盘或入库** | 凭据泄漏 | 对称加密密文存放 | `R-CONF-PLAINTEXT-CRED` |
| **加密密钥 / 密码硬编码在代码** | 密钥泄漏 | 应用层 / 环境注入 | `R-CONF-HARDCODE-SECRET` |
| **含密钥 / 明文凭据的文件入 git** | 仓库泄密 | `.gitignore` + 环境注入 | `R-CONF-SECRET-COMMIT` |
| **配置中心拉取失败未 fail-fast** | 空/旧配置静默启动 | 拉取失败即终止进程 | `R-CONF-NO-FAILFAST` |
| **高风险开关只靠配置档位（缺 `RUN_ENV` 运行档位）** | 危险开关可能线上误生效 | 配置档位 + `RUN_ENV` 双门禁 | `R-CONF-DANGER-SINGLE-GATE` |
| **进程入口未注册终止信号 / 收到信号即硬退出** | 发布即掐断在途请求 | `signal.Notify` + 优雅关闭 | `R-SHUTDOWN-NO-SIGNAL` |
| **优雅关闭缺在途排空（直接 `os.Exit`）** | 在途请求/消息丢失 | `Shutdown` + drain（带超时） | `R-SHUTDOWN-NO-DRAIN` |
| **常驻 goroutine/消费者未接入关闭排空** | goroutine 泄漏/位点不提交 | 接入 cancel + 先提交位点 | `R-SHUTDOWN-LEAK-GOROUTINE` |
| **liveness 探针做依赖检查（DB/缓存/下游）** | 依赖抖动→误杀重启→放大故障 | 依赖检查只放 readiness | `R-PROBE-LIVENESS-DEEP` |
| **对外服务缺 readiness/liveness 端点** | 流量打到未就绪实例 | 暴露探针 3 件套 | `R-PROBE-MISSING` |
| **容器以 root 运行** | 提权风险 | 运行层声明非 root user | `R-DEPLOY-ROOT` |
| **编排清单缺 requests/limits** | OOMKilled / 调度不可控 | 声明 requests+limits | `R-DEPLOY-NO-LIMITS` |
| **`GOMAXPROCS` 未对齐容器 CPU limit（用默认宿主核数）** | 调度争用 + 节流抖动抬 P99 | automaxprocs 对齐 cgroup 配额 | `R-DEPLOY-GOMAXPROCS` |
| **`GOMEMLIMIT` 未设 / 未对齐 mem limit** | GC 太晚 → OOMKilled | 设为 mem limit 的 ~90% 软上限 | `R-RUNTIME-MEMLIMIT` |
| **重试无预算（占比无上限）** | 重试风暴放大级联故障 | 重试预算 ≤X% + 与熔断联动 | `R-TAIL-RETRY-NOBUDGET` |
| **backoff 无 jitter（重试同步成群）** | 二次冲击波 | 指数退避 + 随机抖动 | `R-TAIL-RETRY-NOJITTER` |
| **对冲 / 重试包裹非幂等调用** | 重复副作用 | 先幂等化；非幂等禁对冲 | `R-TAIL-HEDGE-UNSAFE` |
| **冷启 readiness 早于预热（缓存 / 连接池）** | 流量打到冷实例，头部 P99 飙高 | readiness 晚于预热 | `R-TAIL-COLDSTART` |
| **超高频计数用单点 Mutex / 单 atomic** | 缓存行总线风暴，计数吞吐塌陷 | 分片计数（striped）读时求和 | `R-CONC-COUNTER-CONTENTION` |
| **分片 / 原子计数器未按缓存行对齐** | false-sharing，分片白拆 | padding 到 64B / 独占 cache line | `R-CONC-FALSE-SHARING` |
| **响应直接返回未加密的真实主键 / 内部 ID** | 内部 ID 空间暴露 → 被顺序枚举刷库 | 出口确定性加密（design-spec/05 §9.5） | `R-SEC-ID-PLAINTEXT` |
| **对外 ID 解密失败返差异化错误 / 4xx / 携带失败细节** | 响应差分 → 加密层退化为预言机，可批量探测有效 token | 统一"查无结果"无差分（status_codes §7.5） | `R-SEC-DECRYPT-ORACLE` |
| **本地镜像 `ready` 在事务 COMMIT 前置位 / 跨刷新持长读事务** | 撕裂可见性：读到半截 / 已切本地却读空 | `ready` 仅 COMMIT 后置位 + 短 auto-commit 读（design-spec/05 §9.4） | `R-LOCALMIRROR-STALE-READY` |
| **写库 + 发 MQ 裸双写（先写库再发，中间崩溃丢事件）** | 事件永久丢失，下游不一致 | Outbox / 本地消息表绑同一事务（design-spec/08 §3.3） | `R-MQ-DUAL-WRITE` |
| **at-least-once 消费端无幂等（重复投递重复生效）** | 重复扣费 / 重复插入 | 幂等键去重 / 架构层读写分离（design-spec/08 §3.4） | `R-MQ-NO-IDEMPOTENT` |
| **批量 / 聚合端点无上限（调用方一次拉爆）** | 单请求过载、放大 | 批量硬阈值，超限分页 / 拒绝（design-spec/09 §3.1） | `R-API-BATCH-NOLIMIT` |
| **原地改对外接口字段语义 / 类型 / 删字段** | 打挂存量调用方 | expand-contract 三段 / 升主版本（design-spec/09 §3.3） | `R-API-BREAKING-INPLACE` |
| **单后台协程在多副本下跑成 N 份（重复副作用）** | 重复发通知 / 重复迁移 / 错账 | 单例 vs 每副本判定 + 协调原语（design-spec/10 §3） | `R-SCHED-MULTI-REPLICA` |
| **分布式锁无 TTL（持有者崩溃即死锁）** | 任务永久卡死 | 锁带租约 TTL + 续约（design-spec/10 §3.1/§3.3） | `R-SCHED-LOCK-NO-TTL` |
| **用锁 / leader 无 fencing token（脑裂双主写）** | 过期持有者迟到写覆盖 | 单调 fencing token，下游拒旧 token（design-spec/10 §3.3） | `R-SCHED-NO-FENCING` |
| **全表 `UPDATE`/`DELETE` 一条大事务回填** | 长事务持锁 + 顶爆复制延迟 | keyset 分批小事务 + 限流（design-spec/11 §3.3） | `R-EVO-BACKFILL-BIGTXN` |
| **原地破坏性 DDL（删列 / 缩窄类型）未先登记废弃** | 新旧版本读写崩 | expand-contract，先停用再物理删（design-spec/11 §3.1） | `R-EVO-INPLACE-BREAKING` |
| **加副本不下调每副本连接（连接随副本线性爆）** | 下游连接耗尽 → 全局拒绝 | 连接池代理 / 下调每副本连接（design-spec/12 §3.3） | `R-CAP-CONN-EXPLODE` |

> 上表 08-12 视角的 rule-id 为摘要；各视角完整反模式速查 + grep 锚见对应 `design-spec/0X §6`，本工程的完整 rule-id 索引见 `docs/architecture/ai_dev_guide.md §10`。

### 范围判定表

| 变更的代码路径 / 迹象 | 检查的文档 |
|--------------|----------|
| `main.go`、`helpers/init.go` | `docs/architecture/overview.md` |
| `go.mod`、`go.sum` | `docs/architecture/go_module.md` |
| `helpers/`（资源初始化） | `docs/architecture/infrastructure.md` |
| `conf/` | `docs/architecture/config.md` |
| `router/` | `docs/architecture/routing.md` |
| `middleware/` | `docs/architecture/middleware.md` |
| `components/error.go` / `service_error.go` | `docs/architecture/status_codes.md` |
| `components/constants.go` | `docs/architecture/constants.md` |
| `api/` | `docs/architecture/overview.md`（外部 API 部分） |
| `models/` | `docs/schema/database_design.md` |
| `data/` | `docs/schema/database_design.md §3.0`（数据访问层范式 / shard 并行 / 批量写） |
| `service/{module}/` | `docs/service/{module}_service.md` + `service_design.md` |
| `service/{audit-module}/` | `docs/architecture/audit_log.md` |
| `controllers/http/{audience}/` | `docs/api/{audience}_interfaces.md`（接口粒度 / 批量端点 / 版本兼容决策见 design-spec/09） |
| `controllers/command/`、`router/command.go` | `docs/service/scheduled_tasks_design.md`（单例 vs 每副本 / 协调原语 / fencing 决策见 design-spec/10） |
| `controllers/mq/`、`router/mq.go` | `docs/service/scheduled_tasks_design.md` §3（投递语义 / 双写一致 Outbox / 消费幂等决策见 design-spec/08） |
| `helpers/rediscache/` | `docs/cache/cache_design.md` + `architecture/helpers_api.md` |
| `helpers/circuit_breaker.go` | `docs/circuit_breaker/circuit_breaker_design.md` |
| `helpers/`（工具函数、全局变量） | `docs/architecture/helpers_api.md` |
| `pkg/**` | `docs/architecture/pkg_api.md` |
| 日志相关 | `docs/architecture/logging.md` |
| 实现进度 | `docs/BUILD_STATUS.md` |
| `test/framework/` | `docs/testing/testing_design.md` |
| `test/cases/`、`test/e2e/`、`test/perf/` | `docs/testing/testing_design.md` + 对应 `docs/api/*.md` |
| **【】`sync.Mutex` / `sync.RWMutex` / `sync.Map` / `atomic.*` 新增** | `docs/architecture/concurrency_safety.md` §2 共享状态注册表 |
| **`go func()` / `errgroup.Go` / `chan T` / `make(chan T, N)` 新增** | `docs/architecture/concurrency_safety.md` §1.1 / §4.1 |
| **`db.Transaction()` / `BeginTx` / `tx.Commit` / Saga 步骤标记新增** | `docs/service/transaction_design.md` §2 / §3 / §6；同步 `docs/service/{module}_service.md` §4.N.8 |
| **SETNX / 唯一索引 / 幂等校验类代码新增** | `docs/service/transaction_design.md` §4 幂等性设计 |
| **业务规则校验代码（`if {amount-field} < 0` 等）新增 / 修改** | `docs/service/{module}_service.md` §7 领域不变量 + 伪码 `[INVARIANT-CHECK]` 标记 |
| **约束总清单 / 约束突破登记 / grep 锚 rule-id 索引** | `docs/architecture/ai_dev_guide.md` §8 / §9 / §10 |
| **【】`helpers/metrics.go` 新增 `prometheus.MustRegister` / `NewCounterVec`** | `docs/architecture/observability.md` §2 Metrics 采集范围 + §3 Cardinality 评估 |
| **新增告警规则（YAML / Grafana）** | `docs/architecture/observability.md` §5.2 基础告警规则 |
| **新增热路径方法 / 修改热路径文件** | `docs/architecture/performance_contract.md` §2 热路径清单 + 伪码 `[HOT-PATH]` 标记 |
| **新增 `sync.Pool` / 显式预分配 / 新增 `*_bench_test.go`** | `docs/architecture/performance_contract.md` §3 内存预算 + §7.1 基准测试 |
| **修改 SLA 数值（路由 / 配置）** | `docs/architecture/performance_contract.md` §1 全局性能目标 |
| **GORM struct 字段加 `column:"-"` / DROP COLUMN / 字段类型变更** | `docs/schema/database_design.md` §8 字段演进历史（在线 expand-contract / 限流回填 / 回滚闸门决策见 design-spec/11） |
| **提交说明含 `refactor:` 前缀且改动 ≥ 100 行** | `docs/architecture/mvp_rebuild_path.md` §11 安全重构方法论 + §11.5.4 灰度切流量 |
| **`api/` 目录新增 client / `pkg/` 引入新 RPC 包** | `docs/architecture/cross_service_contract.md` §3 下游合约 |
| **新增 server 端路由被外部调用方接入 / 修改对外接口字段** | `docs/architecture/cross_service_contract.md` §2 上游合约 + §5 接口版本管理 |
| **【铁律】循环内单条查询/写/RPC、独立读串行编排、聚合器/扇出并行/批量读新增** | `docs/architecture/io_contract.md` §2 铁律豁免 + §3 聚合器 + §4 原语 + §1 往返预算 |
| **`helpers/{aggregator}/` 聚合器 / `single-flight` / worker pool 新增** | `docs/architecture/io_contract.md` §3 / §4 / §6（与 concurrency_safety.md §1.1 对齐） |
| **`conf/` 凭据字段 / `Encrypt`·`Decrypt` / 配置中心 `Client`·`Sync` / 拉取映射新增** | `docs/architecture/config.md` §6-§9（配置上云 / 凭据加密，详见 docs-spec/26） |
| **`main.go` / 子命令入口新增 `signal.Notify` / `srv.Shutdown` / `cancel()`** | `docs/architecture/deployment.md` §4 启动就绪 + §5 优雅关闭（与 concurrency_safety.md §1.1 对账） |
| **探针 handler（`/live` / `/ready` / `/startup`）新增 / 修改** | `docs/architecture/deployment.md` §3 探针分层语义 |
| **`{Dockerfile-path}` / 编排清单 / CI-CD 流水线新增 / 修改** | `docs/architecture/deployment.md` §1 产物 + §2 镜像约束 + §7 配额弹性（从 SLA 反推容量 / 余量 / 伸缩信号与防抖 / 资源配比决策见 design-spec/12） |
| **发布策略 / 回滚钩子调整** | `docs/architecture/deployment.md` §6 发布与回滚策略 |
| **`main.go` / 编排清单新增 `GOMAXPROCS` / `GOMEMLIMIT` / automaxprocs** | `docs/architecture/deployment.md` §7 + `docs/architecture/performance_contract.md` §3.4 运行时旋钮 |
| **启动时序新增连接池预热 / MinIdle 预拨 / 缓存预热** | `docs/architecture/deployment.md` §4 启动就绪（readiness 晚于预热） |
| **入口 / 编排新增对冲 / 重试预算 / 自适应并发限制 / load shedding** | `docs/architecture/performance_contract.md` §6.4 自适应过载 + `cross_service_contract.md` §3（决策见 design-spec/07） |
| **新增分片计数 / striped counter / 缓存行对齐 padding** | `docs/architecture/concurrency_safety.md` §2 共享状态注册表（决策见 design-spec/03 §3.5） |
| **新增 `/debug/pprof` 暴露 / 持续 profiling 采集** | `docs/architecture/performance_contract.md` §7.4 生产 profiling + `observability.md` §3.3 |
| **assembler / 出口渲染新增主键·ID 字段 / `keycrypto.Encrypt`·`Decrypt`** | `docs/architecture/status_codes.md` §6.5（解密失败统一查无结果）+ `docs/schema/database_design.md` §6.6 对外 ID 形态（决策见 design-spec/05 §9.5） |
| **`helpers/localsqlcache/` 本地 SQL 镜像 / 镜像表注册 / 刷新协程** | `docs/cache/cache_design.md` §3.4 本地缓存镜像登记（决策见 design-spec/05 §9.4 + 03 §9.4） |

### 输出格式

每次回复末尾必须包含文档同步状态：

- 有变更：「文档同步：更新了 `docs/xxx.md`、新建了 `docs/yyy.md`」
- 无变更：「文档同步：本次修改不涉及文档变更」

### 文档目录结构

```
docs/
├── architecture/        # 架构（总纲、配置、基础设施、路由、中间件、状态码、helpers/pkg API、日志）
├── schema/              # 数据模型（DDL、分库、Redis Key 注册、GORM 规范）
├── cache/               # 缓存设计（Redis-first、Hash 字段映射、Flush、限流）
├── circuit_breaker/     # 熔断器设计
├── service/             # Service 层设计 + 定时任务 + MQ 消费者
├── api/                 # 接口规格（每类调用方一份）
├── testing/             # 测试框架设计
└── INDEX.md             # 文档索引
```


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
