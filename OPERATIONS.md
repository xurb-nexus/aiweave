# AIWeave 操作手册（OPERATIONS）

> 把"工作流（怎么做）"和"自检清单（做完检查什么）"合并到一份操作手册。
>
> **第一部分：8 类标准研发工作流** —— 从设计到交付的步骤
> **第二部分：3 类自检清单** —— 每次交付前/代码变更后/文档变更后过一遍
> **第三部分：增量同步专属清单** —— B1 场景的额外检查项

---

# 第一部分：标准研发工作流

工作流分两组：

- **建设组**（W1 / W2 / W3 / W4 / W5 / W6）：从零或对现有结构作大段改动
- **演进组**（W7）：增量同步常态
- **兜底**（W8）：定期审计

## W1：设计先行（基础工作流，所有其他工作流的内核）

```
⓪  用 design-spec 十二视角做架构决策 → 技术方案(TRD)（复杂需求强烈推荐，执行器 /design-solution）
        ↓
①  写 / 改 md（按 TRD 落地，技术方案文档）
        ↓
②  AI 按 md 生成代码（调用 Skill）
        ↓
③  go build / go test 验证 + 反向同步 md
```

**纪律**：

- ⓪ 技术方案设计（design-spec / `/design-solution`）对复杂需求强烈推荐；简单 CRUD 可跳过，直接进 ①
- 即使用户跳过 ⓪ / ① 直接让 AI 改代码，③ 仍然是强制的
- ② 失败时不能硬猜伪码 → 回到 ① 修 md → 再 ② → ③
- 每次交付的回复末尾必须有"文档同步：..."声明

---

## W2：新增 API 接口

适用：新增一个 HTTP API（无论是 Internal 内部接口还是 UserSelf/Admin 后台接口）。

```
Step 1：在 docs/api/{audience}_interfaces.md 中补充接口规格
        - 请求方法 + 路径
        - 校验规则表（字段 / 位置 / 类型 / binding）
        - Go 请求 / 响应 struct
        - 错误码映射
        - Controller 伪代码

Step 2：在 docs/service/service_design.md 和 {module}_service.md 补 Service 方法签名 + 步骤伪码
                - 伪码使用统一标记语法（[TXN-*] / [SAGA-STEP-N] / [IDEMPOTENT-CHECK] / [LOCK-*] / [INVARIANT-CHECK]）
        - 涉及多数据源写入 → 同步 docs/service/transaction_design.md §2 事务边界清单
        - 涉及业务规则校验 → 同步 {module}_service.md §7 领域不变量
        - 涉及共享状态 → 同步 docs/architecture/concurrency_safety.md §2 / §3

Step 3：调用 /new-service {module}/{Method}
        - 第 0 步：BUILD_STATUS 检查
        - 第 1 步：读 service_design.md
        - 第 3 步：生成 service/{module}/{module}.go
        - 第 4 步：文档同步
        - 第 5 步：测试同步（test/cases/{audience}/...）
        - 第 6 步：go build / go test 验证

Step 4：调用 /new-controller {audience}/{module}/{action}
        - 自动根据 audience 选择响应模式（Internal 用 renderInternalResponse，其他用 base.Render*）
        - 同步 test/cases/{audience}/{module}_{action}_test.go

Step 5：调用 /new-router {audience}
        - 在 router/http_{audience}.go 追加路由注册行
        - 同步 routing.md §5 路由表

Step 6：调用 /update-index（如有新 md 文件）

Step 7：自检（见第二部分 §1 交付前清单）

文档同步声明：
"文档同步：更新了 docs/api/{audience}_interfaces.md、docs/service/{module}_service.md、docs/architecture/routing.md、docs/INDEX.md、docs/BUILD_STATUS.md"
```

---

## W3：新增数据表

适用：业务需要新增一张 MySQL 表（且可能伴随新 Redis Key）。

```
Step 1：在 docs/schema/database_design.md 补充
        - §1 表清单新增一行（含数据量级）
        - §2 分库设计：确认归入哪个库
        - §3 完整 DDL（字段 / 类型 / 索引 / 字符集 / 注释）
        - §6 GORM 规范
        - §7 索引设计（每个索引的目的）

Step 2（如有 Redis 缓存）：在 docs/cache/cache_design.md §2 补充
        - Redis Key 模式表
        - Hash 字段映射子节
        - 与 MySQL 的同步方式

Step 2.5：如新增表对应"跨数据源写入"场景
        - 同步 docs/service/transaction_design.md §2 事务边界清单
        - 同步 §4 幂等接口清单（如该表写入接口需要幂等）

Step 3：调用 /new-model {table_name}
        - 生成 models/db_{库}/{table}.go

Step 4：影响传播
        - service 层是否需要新增方法？→ W2 新增 API 流程
        - 是否需要新增缓存预热逻辑？→ 同步 cache_warmup 任务伪码
        - 是否需要新增 flush 任务？→ W4 新增定时任务流程

Step 5：自检（见第二部分 §1 交付前清单）

文档同步：updated database_design.md、cache_design.md（如有 Redis）、INDEX.md、BUILD_STATUS.md
```

---

## W4：新增定时任务

适用：新增一个 cobra / cycle / crontab 任务。

```
Step 0：检查 BUILD_STATUS.md §0 — 命中 🚫 任务 → 拒绝

Step 1：在 docs/service/scheduled_tasks_design.md 补充
        - §1.2 任务清单总表
        - §2/§3/§4 单个任务详细设计（Service 方法 / 触发频率 / 并行策略）
        - §6 Controller 入口
        - §7 并行策略详解

Step 2：在 docs/service/{module}_service.md 补 Service 方法签名 + 步骤伪码
                - 定时任务常常涉及"常驻 goroutine + ctx 退出" → 必须在
          docs/architecture/concurrency_safety.md §1.1 Goroutine 生命周期图追加节点
          + §5.2 登记 cancel 路径
        - 并行策略（SPOP / 分布式锁）的选择必须在 §7 隐式约束中说明原因

Step 3：调用 /new-service {module}/{TaskMethod}（如 service 方法不存在）

Step 4：调用 /new-scheduled-task {task-name}
        - 自动追加 cobra/cycle/crontab 注册行
        - 生成 controllers/command/{task}.go

Step 5：测试同步
        - service/{module}/{task}_test.go（推荐）
        - test/e2e/scenario_{task}_test.go（可选）

Step 6：自检

文档同步：scheduled_tasks_design.md、{module}_service.md、INDEX.md、BUILD_STATUS.md
```

---

## W5：新增 MQ topic

适用：新增 Kafka topic 的 Producer / Consumer 之一或两者。

```
Step 1：在 docs/service/scheduled_tasks_design.md §3 补充
        - §3.1 总览（topic / 文件 / 消费 group / 用途）
        - §3.2.1 消息结构（Go struct + 字段表）
        - §3.2.2 消费流程（攒批 / 落库 / Redis 联动 / offset）
        - §3.2.3 失败处理（重试 / 死信）
        - §3.5 配置（resource.yaml）

Step 2：（如 Producer 侧）在对应 service 文档补充 Producer 调用伪码

Step 3：（如 Consumer 侧）调用 /new-mq-consumer {topic}
        - 第 0 步：检查 helpers/kafka_consumer.go 是否存在；不存在先生成
        - 生成 controllers/mq/{topic}_consumer.go
        - 注册 router/mq.go
        - 在 cobra Commands 中追加 {topic}-consumer 子命令
                - MQ 消费是典型最终一致性场景 → 同步 docs/service/transaction_design.md §2
          （列为"MQ 投递"类一致性级别 + 重试/死信失败策略）
        - 消费 goroutine 必须在 concurrency_safety.md §1.1 登记
        - 消息处理伪码使用 [IDEMPOTENT-CHECK] 标记（同一 message 重投必须幂等）

Step 4：测试同步
        - cases/internal/{module}_{action}_test.go 中用 framework.Kafka 断言
        - controllers/mq/{topic}_consumer_test.go 直接调 processBatch 单测
        - test/e2e/scenario_{topic}_e2e_test.go（可选，依赖真 Kafka，默认 Skip）

Step 5：自检

文档同步：scheduled_tasks_design.md §3、helpers_api.md（如新增 SubClient）、routing.md §7、INDEX.md、BUILD_STATUS.md
```

---

## W6：重构 / 重写

适用：技术栈升级、模块拆分、大段重构。

```
Step 1：更新 docs/ 中相关设计文档
        - 如果架构调整 → architecture/overview.md
        - 如果业务流程调整 → architecture/{xxx}_flow.md
        - 如果接口契约调整 → api/*.md
        - 如果数据模型调整 → schema/database_design.md

Step 2：根据改动规模选择路径
        - 单模块 → /rebuild-from-docs {module}（如 service/{auth-module}）
        - 多模块 → 按 docs/architecture/mvp_rebuild_path.md 分阶段推进
        - 整工程 → 不要 /rebuild-from-docs all，强制按 mvp_rebuild_path Stage 顺序

Step 3：每完成一层立即验证
        - go build ./{layer}/...
        - go vet ./{layer}/...
        - 测试用例同步并跑全绿

Step 4：发现伪码漏洞时
        - 不硬写
        - 修 md → 用户评审 → 再继续生成
        - 命中签名冲突裁决（PRINCIPLES.md §5）→ 显式报告由用户决定

Step 4.5：重构涉及共享状态 / 事务 / 不变量时
        - 重构前先读 concurrency_safety.md §2 / §6 + transaction_design.md §2
        - 重构后必须双向同步：代码 ↔ 上述 docs
        - 重构破坏不变量（删除校验代码）→ 拒绝合并，先讨论是否同步删除不变量

Step 5：BUILD_STATUS 全程更新

Step 6：完成后 /doc-sync-check all 兜底

文档同步：按改动范围统计
```

---

## W7：增量需求同步（B1，演进组核心）

适用：**每次需求迭代结束、PR 合并前**——把代码改动反向同步到全链路 docs/。

```
你（开发者）：把以下三类输入交给 AI——
   ① 本规范 aiweave/（PRINCIPLES + docs-spec/ + skills-spec/）
   ② 新需求的技术设计文档（业务背景 + 接口变更 + 数据模型 + 缓存变更 + ...）
   ③ 代码改动（git diff 或 PR 或改动文件清单）

AI（按 docs-spec/19 SOP）：

Step 0：拒绝规则与输入完整性检查
        - 命中 🚫 模块 → 立即拒绝
        - 输入不全 → 提问

Step 1：读输入
        - 读规范 + 工程当前 docs/INDEX/BUILD_STATUS/CLAUDE
        - 读需求文档 + git diff

Step 2：差异分类
        - 按 CLAUDE.md 范围判定表对改动文件分类

Step 3：每篇 md 的更新动作
        - 按各 docs-spec/ 规范执行

Step 4：交叉一致性检查
        - INDEX 登记 / BUILD_STATUS 状态 / Redis Key 双向 / 常量双向 / 路由双向 / 测试覆盖
        - 详见第三部分 增量同步专属清单

Step 5：检测签名冲突（PRINCIPLES.md §5）
        - 显式报告由用户裁决

Step 6：输出
        - 文档变更清单 + 每条 diff 摘要 + 自检结果 + 文档同步声明

工具：调用 /sync-feature-to-docs

最佳实践：
- 每次需求合并前 → /sync-feature-to-docs
- 每次发版前 → /doc-sync-check all（兜底审计）
```

---

## W8：定期审计（兜底机制）

适用：版本发布前、季度健康检查。

```
Step 1：跑 /doc-sync-check all
        - 输出 7 个维度的差异报告
        - 按工程启用的约束类规范，并行跑审计 Skill：
          concurrency-review / performance-review / io-review /
          domain-invariant-check / failure-path-review

Step 2：分类处理
        - 🔴 严重（代码有但文档完全缺失）→ 立即补 md
        - 🟡 警告（不一致）→ 按签名冲突裁决（PRINCIPLES.md §5）
        - 🟢 提示（INDEX 未登记）→ 调 /update-index

Step 3：跑 /update-index 同步索引

Step 4：复查测试覆盖
        - 每个 handler 都有对应 cases/ 测试？
        - 缺失 → 调 /new-test 补齐

Step 5：发布前必须零差异
```

---

## 工作流之间的关系

```
A0 设计（design-spec 十二视角 / /design-solution → TRD）
   │  复杂需求先做架构决策，产出 TRD（决策记录 + 落地索引）；简单 CRUD 可跳过
   ▼
W1（设计先行）
   ├── 内核：所有其他工作流的基础
   │
   ├── W2 新增 API ┐
   ├── W3 新增表    │ — 建设组（A1 forward / 单点变更）
   ├── W4 新增任务  │
   ├── W5 新增 MQ  ┘
   │
   ├── W6 重构 / 重写 — 建设组（A2 rebuild / B2 refactor）
   │
   └── W7 增量同步 — 演进组（B1 sync-feature，常态）

W8 定期审计 — 跨组（兜底机制）
```

每个工作流的最后一步都会引用第二部分的自检清单。

---

# 第二部分：自检清单

> 3 类清单，用于不同时机的交付验收。本部分是 `aiweave/PRINCIPLES.md` §9 的实操化版本。

## §1 交付前清单（delivery_checklist）

> 每次交付（PR 提交 / 发版 / 阶段完成）前，过一遍下面所有项。任何一项不过 → 不交付。

### 1.1 文档同步

- [ ] 本次新增了代码文件？→ 是否有对应 md？
- [ ] 本次修改了已有代码？→ 是否更新了关联 md？
- [ ] 本次新增/修改了 md？→ 是否更新了 INDEX.md？
- [ ] 本次删除了代码或 md？→ 是否同步清理了另一侧？
- [ ] 本次是否更新了 BUILD_STATUS.md 的状态列？

### 1.2 测试同步

- [ ] 本次新增的代码是否同步了测试用例？（4 类底线：正向 / 参数校验 / 业务错误 / 副作用）
- [ ] `go test ./test/cases/{scope}/...` 全绿？
- [ ] 测试覆盖率（line coverage）是否符合工程标准（service ≥ 80%、controller ≥ 70%）？

### 1.3 编译与静态检查

- [ ] `go build ./...` 通过？
- [ ] `go vet ./...` 通过？
- [ ] `go fmt ./...` 通过（无 diff）？

### 1.4 拒绝规则

- [ ] 本次修改是否触碰了 `BUILD_STATUS.md` §0 的 🚫 暂不实现模块？（应当无）

### 1.5 CLAUDE.md 输出格式

- [ ] 回复末尾有"文档同步：..."声明？
- [ ] 内容准确反映本次变更？

### 1.6 安全与脱敏

- [ ] 是否有敏感数据（password/secret/token/手机号）写入日志？
- [ ] 错误返回是否暴露内部细节给外部用户？

### 1.7 尾延迟与运行时（敏感接口才需）

- [ ] 尾延迟 / 过载敏感接口 → 是否评估对冲 / 重试预算 / load shedding？（决策见 [`design-spec/07`](design-spec/07_tail_latency_design.md)；对冲 / 重试只在幂等时启用，重试预算封顶）
- [ ] 容器是否对齐 `GOMAXPROCS`（automaxprocs）/ `GOMEMLIMIT`（mem limit ~90% 软上限）？

---

## §2 代码变更清单（code_change_checklist）

> 写完代码、提交前，按以下清单自查。

### 2.1 范围判定（按 CLAUDE.md 范围判定表）

逐一确认每类改动文件对应的 docs/ 是否同步：

- [ ] 改了 `models/{db}/{table}.go` → 同步 `docs/schema/database_design.md`
- [ ] 改了 `service/{module}/*.go` → 同步 `docs/service/{module}_service.md` + `service_design.md`
- [ ] 改了 `controllers/http/{audience}/*.go` → 同步 `docs/api/{audience}_interfaces.md`
- [ ] 改了 `controllers/command/*.go` → 同步 `docs/service/scheduled_tasks_design.md`
- [ ] 改了 `controllers/mq/*.go` → 同步 `docs/service/scheduled_tasks_design.md` §3
- [ ] 改了 `middleware/*.go` → 同步 `docs/architecture/middleware.md`
- [ ] 改了 `router/http*.go` → 同步 `docs/architecture/routing.md`
- [ ] 改了 `helpers/*.go`（导出） → 同步 `docs/architecture/helpers_api.md`
- [ ] 涉及新 Redis Key / Hash 字段 → 同步 `docs/cache/cache_design.md` + `docs/schema/database_design.md` §4
- [ ] 涉及新错误码 → 同步 `docs/architecture/status_codes.md` + `components/error.go`
- [ ] 涉及新常量 → 同步 `docs/architecture/constants.md` + `components/constants.go`
- [ ] 涉及配置 → 同步 `docs/architecture/config.md` + `conf/`
- [ ] 涉及 `pkg/**` → 同步 `docs/architecture/pkg_api.md`

### 2.2 编码规范

- [ ] 错误类型选择正确？（{特定方法集，详见 docs-spec/09 §5}用 ServiceError，其他用 base.Error）
- [ ] MySQL Client 选择正确？（按表所属库）
- [ ] 双写模式遵循？（先 MySQL → 后 Redis；MySQL 失败回错；Redis 失败 WARN）
- [ ] 日志 Structured API（不用 Sugared）？
- [ ] 魔法数字都用了 constants？
- [ ] Controller 不写业务逻辑？

### 2.3 测试

- [ ] 4 类用例覆盖？
- [ ] 副作用断言（MySQL/Redis/Kafka）？
- [ ] 测试全绿？

---

## §3 文档变更清单（doc_change_checklist）

> 改了 md 后，按以下清单自查。

### 3.1 单 md 内部一致性

- [ ] 顶部版本号 + 日期 已更新？
- [ ] 章节编号连续？
- [ ] 表格格式统一？
- [ ] 引用其他 md 的链接路径正确？

### 3.2 与代码一致性

- [ ] 如果改了 service 方法签名 → 代码已更新？
- [ ] 如果改了 DDL → models/ 中的 GORM struct 已更新？
- [ ] 如果改了 Redis Key 模式 → service 代码中的 Key 引用已更新？
- [ ] 如果改了错误码 → components/ 中的常量定义已更新？

### 3.3 与其他 md 的一致性

- [ ] 改了 cache_design §4.6 字段映射 → database_design §4 同步？
- [ ] 改了 constants §10 Go 文件 → §X 表格同步？
- [ ] 改了 routing §5 路由表 → 对应 api/*.md 接口已存在？
- [ ] 改了 service/{module}_service.md 方法签名 → service_design.md §6 接口映射同步？

### 3.4 索引一致性

- [ ] 新增 md → INDEX.md 已登记？
- [ ] 删除 md → INDEX.md 已移除引用？
- [ ] 重命名 md → INDEX.md 链接已更新？

### 3.5 BUILD_STATUS 一致性

- [ ] 设计完整定稿但代码未实现 → BUILD_STATUS 标 ⬜？
- [ ] 设计 + 代码都完成 → BUILD_STATUS 标 🟢？
- [ ] 设计已定稿但当前阶段不实现 → BUILD_STATUS §0 登记 + 标 🚫？

---

# 第三部分：增量同步专属清单（B1 场景额外检查）

> 用 `/sync-feature-to-docs` 时，AI 必须在输出中显式回答以下：

- [ ] 全链路文档变更清单（新增 / 修改 / 无变更分类）
- [ ] 每条变更的 diff 摘要
- [ ] 22 项交叉一致性检查通过：

  **基础 7 项：**
  - [ ] 1. INDEX 登记
  - [ ] 2. BUILD_STATUS 状态更新（代码模块状态轨道 §3-§9）
  - [ ] 3. Redis Key 在 cache + schema 双向
  - [ ] 4. 常量在 §X 表 + §10 Go 双向
  - [ ] 5. 接口在 api + routing 双向
  - [ ] 6. service 方法在 §3 清单 + §6 接口映射双向
  - [ ] 7. helpers 函数在 §1 + §2 双向

  **约束类 5 项：**
  - [ ] 8. 新增共享状态（sync.* / chan）在 concurrency_safety.md §2 / §4.1 登记，
         BUILD_STATUS §11 约束清单状态轨道对应类目 +1
  - [ ] 9. 新增事务边界（BeginTx / Saga 步骤）在 transaction_design.md §2 +
         service docs §4.N.8 双向登记
  - [ ] 10. 新增业务规则校验代码在 {module}_service.md §7.1 业务规则表登记，
         伪码 §4.N.3 添加 [INVARIANT-CHECK: I-N] 标记
  - [ ] 11. 涉及事务 / 锁 / 不变量 / 幂等的方法伪码使用统一标记语法
         （[TXN-*] / [SAGA-STEP-N] / [IDEMPOTENT-CHECK] / [INVARIANT-CHECK] / [LOCK-*]）
  - [ ] 12. 命中危险模式清单（CLAUDE.md / concurrency_safety.md §6）的代码
         有修正或有 `// aiweave:allow=<rule-id>` 行内注解抑制

  **性能 / 可观测性 / Schema 演进 / 重构 6 项：**
  - [ ] 13. 新增 Metric 在 observability.md §2 登记；label 未命中 §3.2 禁止清单；
         cardinality ≤ §3.1 默认上限（超出在 ai_dev_guide.md §9 登记）
  - [ ] 14. 新增热路径方法在 performance_contract.md §2 清单登记；
         伪码加 [HOT-PATH] 标记；未触犯 §2.2 禁止操作清单
  - [ ] 15. channel 容量在 performance_contract.md §6.1 与 concurrency_safety.md §4.1
         双向对齐
  - [ ] 16. 修改字段类型 / 删除字段在 database_design.md §8 字段演进历史登记；
         物理删除前经过至少一个发版周期软兼容期
  - [ ] 17. refactor 类提交符合 mvp_rebuild_path.md §11 安全重构约束
         （≤ 200 行 / ≤ 5 commit / 行为变更走灰度切流量）
  - [ ] 18. 伪码使用 [HOT-PATH] / [METRIC-EMIT] 标记的，对应代码实现真的执行了
         该动作（避免标记与实现脱节）

  **IO 铁律 / 配置安全 3 项：**
  - [ ] 19. 无循环内单条查询/写/RPC（铁律一 N+1）；无独立读串行编排（铁律二）；
         命中 → 改聚合器/扇出，或在 io_contract.md §2.1/§2.2 登记豁免（真实依赖链）
  - [ ] 20. 新增"集合访问 / 多依赖编排"方法在 io_contract.md §1 往返预算 + §3/§4 原语
         清单登记；聚合器 / single-flight 共享指针未被就地修改（只读）
  - [ ] 21. 配置中心 / 资源凭据为密文（非明文）；密钥未硬编码 / 未入 git；新增拉取项在
         config.md §7.3 登记且失败 fail-fast（命中 R-CONF-* → 拒绝合并）

  **部署 / 运行时生命周期 1 项：**
  - [ ] 22. 进程入口注册终止信号且优雅关闭排空在途（readiness 转 false → 停 accept → drain
         带超时 → 停常驻 goroutine/消费者并提交位点 → 关连接池）；常驻 goroutine 与
         concurrency_safety.md §1.1 清单对账无泄漏；liveness 探针未做依赖检查；镜像非 root +
         编排清单含 requests/limits（命中 R-SHUTDOWN-* / R-PROBE-* / R-DEPLOY-* → 拒绝合并；
         详见 deployment.md §3/§5）

- [ ] 测试用例覆盖检查（每个新接口 4 类用例 + 涉及并发的需 `go test -race` 通过 +
      涉及热路径的需 `go test -bench` 比对基线 + 涉及多数据源写入的需故障注入覆盖
      transaction_design.md §6 失败路径全景图分支 + 涉及"集合访问"的需 IO 计数断言
      证明无 N+1（io_contract.md §7.1））
- [ ] 签名冲突报告（如有，按 PRINCIPLES §5 处理）
- [ ] 文档同步声明

---

# 附录：使用建议

| 场景 | 用哪份清单 |
|------|-----------|
| 单接口提交前 | §2 code_change_checklist |
| 文档大改后 | §3 doc_change_checklist |
| PR 合并前 | §2 + §1 delivery |
| 发版前 | §1 delivery + 跑 /doc-sync-check all |
| 增量同步（B1） | §2 + 第三部分 增量同步专属 |

# 附录：当某项不过怎么办

- 文档不同步 → 立即补 md，再交付
- 测试不通过 → 修代码 / 修测试，**不要为了通过测试而改弱断言**
- 编译错误 → 修代码（如签名冲突按 PRINCIPLES §5 处理）
- 触碰 🚫 模块 → 停止，向用户说明并退回需求

**核心原则**：清单不过 → 视为未交付。**不要妥协**。

---

# 附录：6 层防御 × W 工作流对照表

> 防御层级与工作流的二维映射。每次交付声明里应当显式标注通过的层级，例如：
>
> > 文档同步：更新了 `docs/...` ；防御层级：L0 hooks ✅ / L1 ✅ / L2 ✅ / L3 ✅ / L4 io-review+concurrency-review ✅ / L5 race+bench ✅。

| 层 | 实施位置 | 触发时机 | 关联 W 工作流 |
| --- | --- | --- | --- |
| **L0 Hooks 自动化** | `.claude/settings.json`（共享）两类强制 Hook：IO 铁律检查 + 代码↔文档双向同步 | 编辑/写 `.go` 后 + 会话 Stop（机械执行，不依赖自觉） | 所有 W 工作流共用（事前/事中自动拦截） |
| **L1 文档约束** | `docs/` 各规范（含 `ai_dev_guide.md §8` 约束总清单） | 设计阶段 | W1 设计先行（必经） |
| **L2 Skill 检查步骤** | 每个 `SKILL.md` 第 0 步前置检查 + 中间步骤 | 调用 `/new-*` / `/sync-feature-to-docs` 时 | W2-W5 每个 new-* / W7 增量同步 |
| **L3 自检清单** | OPERATIONS §1.1-§1.6 交付前清单 | 每次交付前 | W2-W7 末步 |
| **L4 审计 Skill** | `doc-sync-check` / `concurrency-review` / `performance-review` / `io-review` / `domain-invariant-check` / `failure-path-review` | 增量同步 + 发版前 | W7 增量同步 / W8 定期审计 |
| **L5 测试验证** | `go test -race` / `go test -bench` / IO 计数断言 / `framework.InjectXxxFailure` 故障注入 | 每次 PR + CI | 所有 W 工作流共用 |

**任一层不过 = 未交付**。每次交付声明里应当显式标注通过的层级。L0 是不依赖 AI 自觉的自动兜底，与 L1-L5 叠加（原理见 `PRINCIPLES.md §14`）。

## 6 层防御与"不可绕过"的关系

| PRINCIPLES.md §2.2 不可绕过项 | 主要由哪一层防御覆盖 |
|----------------------------|---------------------|
| 代码改了，没改文档 | L4 doc-sync-check + L3 §1.1 |
| 文档改了，没改代码 | L4 doc-sync-check |
| 新增 md 没登记到 INDEX | L2 update-index + L4 doc-sync-check 维度 3 |
| 业务代码生成了但测试没同步 | L2 new-test + L5 |
| 删除代码后 md 仍残留对该代码的引用 | L4 doc-sync-check |
| 新增共享状态未登记 §2 | L4 concurrency-review + L3 §1.1 |
| 新增热路径未登记 §2 | L4 performance-review + L3 §1.1 |
| 不变量声明 / 实现脱节 | L4 domain-invariant-check |
| 失败路径文档 / 测试缺失 | L4 failure-path-review + L5 |
| 循环内单条查询（N+1）/ 独立串行编排 | L0 hooks(IO 铁律) + L4 io-review + L5 IO 计数断言 |
| 配置凭据明文 / 密钥硬编码入库 | L0 hooks(凭据可选) + L4 doc-sync-check(R-CONF-*) |
| 优雅关闭缺排空 / 探针误配 / 常驻 goroutine 泄漏 | L4 doc-sync-check(R-SHUTDOWN-*/R-PROBE-*/R-DEPLOY-*) + L4 concurrency-review(§1.1 goroutine 对账) |
| 重试无 jitter / 对冲包裹非幂等 / 部署无 GOMAXPROCS 对齐 | L0 hooks(R-TAIL-RETRY-NOJITTER/R-TAIL-HEDGE-UNSAFE/R-DEPLOY-GOMAXPROCS) + L3 §1.7 + L4 performance-review |
| 漏"文档同步：..."声明 | L0 hooks(双向同步 Stop) + L3 §1.5 |

防御层级的设计哲学：**L0 是自动拦截（不依赖自觉，机械执行），L1-L2 是事前预防（让 AI 不犯错），L3 是事中自检（一次 PR 内的自查），L4 是事后审计（PR 合并前的兜底），L5 是运行时证明（CI 阻断错误代码）**。任一层失效，下一层兜底。

> **A0 在所有防御层之上游**：6 层防御保的是"代码正确实现了 docs/ 约束"；design-spec / A0 保的是"架构决策本身正确"——先有对的设计(TRD)，再由 L0-L5 保证代码忠实落地。决策错了，实现再忠实也白搭（A0 见上方"工作流之间的关系" + [`design-spec/00`](design-spec/00_design_overview.md)）。


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
