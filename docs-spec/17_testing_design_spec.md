# 17 - 测试设计文档规范

> 规定 `docs/testing/testing_design.md` 的内容结构。

---

## 1. 定位

测试文档要让 AI 能回答：

1. 测试框架的定位（接口级黑盒 / Service 单元 / e2e 场景）
2. 测试启动方式（in-process / 独立进程）
3. 业务校验如何透明注入（不让业务测试用例手写业务校验细节）
4. seed 数据从哪里来、如何重置
5. 断言 API（协议层 + 副作用层）
6. 测试纪律（与业务代码同步交付）

---

## 2. 顶层结构

```markdown
# 测试框架设计

## 1. 定位
   1.1 测试目标（接口级 / 单元 / e2e / perf）
   1.2 测试类型层级图

## 2. 核心设计原则
   2.1 in-process 启动
   2.2 三种业务校验透明注入
   2.3 开发者自备依赖 + yaml 直配
   2.4 Seed 数据唯一真相源
   2.5 断言双通道
   2.6 Fake MQ Producer

## 3. 目录契约（强制：test/ 目录布局）

## 4. 框架 API 稳定契约（强制：所有可被测试用例引用的 API）
   4.1 入口
   4.2 HTTP 调用
   4.3 业务校验工具
   4.4 协议层断言
   4.5 副作用层断言（MySQL / Redis / Kafka）
   4.6 数据管理（warmup / truncate）

## 5. 测试纪律（硬约束）
   5.1 写代码必须同步写测试
   5.2 测试失败即交付失败
   5.3 用例覆盖标准（4 类）
   5.4 覆盖率目标

## 6. 与 Stage 协同
   每个 Stage 的业务代码 + 测试用例交付清单

## 7. 配套 Skill

## 8. 禁用场景（不写测试的例外）

## 9. 运行流程

## 10. 与 docs/ 的交叉引用
```

---

## 3. §1.2 测试类型层级图

```markdown
\`\`\`
perf/                ← 基准压测（go test -bench，P99 指标验证）
  │
e2e/                 ← 多接口串联场景（{Method-A}→{Method-B}→{Method-C}→{Method-D}→{Method-E}）
  │
cases/               ← 单接口黑盒用例（正向 + 参数校验 + 业务错误 + 副作用）
  │
framework/           ← 框架自身单元测试
\`\`\`
```

---

## 4. §2 核心设计原则（强制 6 条）

### 4.1 in-process 启动

```markdown
### 2.1 in-process 启动

每个测试包的 `TestMain` 调用 `framework.RunTests(m)`，该函数通过 `httptest.NewServer(engine)` 在进程内启动 Gin。**不独立启进程**——所有测试与服务端共享 `helpers.{Project}CacheClient` / `MysqlClient*` 等资源，便于副作用断言。
```

### 4.2 三种业务校验透明注入

```markdown
### 2.2 三种业务校验的透明注入

| 路由组 | 注入方式 | 对应中间件 |
|-------|---------|-----------|
| Internal `/{prefix}/internal/*` | 生成 MD5 签名，自动写入 query `key` + Header `Token`/`Timestamp` | `{sign_mw}` |
| Operator `/{prefix}/operator/*` | 自动写入 Header `X-{User}-Id` | `{user_mw}` |
| Admin `/{prefix}/admin/*` | 自动写入 Header `X-{Actor}-Id` + `X-{Actor}-Role` | `{admin_mw}` |

业务测试用例**只关心业务参数**，不手写业务校验细节。
```

### 4.3 开发者自备依赖 + yaml 直配

```markdown
### 2.3 开发者自备依赖 + yaml 直配

开发者自建 MySQL / Redis，连接地址和密码**直接写在 `test/config/resource.yaml` 里**。**不引入 docker-compose 或 testcontainers**，保持简单。
```

### 4.4 Seed 数据唯一真相源

```markdown
### 2.4 Seed 数据唯一真相源

`test/fixtures/seed/README.md` 是样本数据的"权威清单"——{若干样本数据：{若干样本数据}。任何修改 seed SQL 都必须同步这份 md。
```

### 4.5 断言双通道

```markdown
### 2.5 断言双通道

- **协议层**：`AssertInternalStatus / AssertErrNo / AssertJSONSubset`
- **副作用层**：`AssertRedisHashField / AssertMysqlRow / Kafka.Messages()`

高质量用例**必须**同时覆盖两者，只断言 HTTP 响应是不够的。
```

### 4.6 Fake MQ Producer

```markdown
### 2.6 Fake MQ Producer

`framework.NewFakeKafkaProducer()` 实现与 `kafka.PubClient` 对齐的 `Pub / CloseProducer` 接口，捕获所有消息供断言。业务代码通过**接口类型**引用 Kafka Producer，测试就能无缝替换为 fake。
```

---

## 5. §3 目录契约（强制）

```markdown
## 3. 目录契约

\`\`\`
test/
├── README.md
├── .env.example
├── config/                       # 测试配置
├── fixtures/
│   ├── schema/                   # 与生产 1:1 的 DDL
│   ├── seed/                     # 样本 SQL + README 数据全景
│   └── redis_warmup.go
├── framework/                    # 测试框架自身（11 个文件）
├── cases/
│   ├── probe/
│   ├── internal/
│   ├── operator/
│   └── admin/
├── e2e/                          # 场景测试
├── perf/                         # 性能基准
└── scripts/                      # setup / migrate / seed / reset / run
\`\`\`

目录命名严格对应路由组。
```

---

## 6. §4 框架 API 稳定契约（强制）

```markdown
## 4. 框架 API 稳定契约

以下 API 是测试用例**可以依赖**的：

### 4.1 入口

\`\`\`go
func framework.RunTests(m *testing.M) int
\`\`\`

### 4.2 HTTP 调用

\`\`\`go
// Internal（自动签名）
func InternalGet(t *testing.T, path, entityId, secret string, query url.Values) *Response
func InternalPost(t *testing.T, path, entityId, secret string, body interface{}) *Response

// Operator
func OperatorGet(t *testing.T, path, actorID string, query url.Values) *Response
func OperatorPost(t *testing.T, path, actorID string, body interface{}) *Response

// Admin
func AdminGet(t *testing.T, path, actorID, role string, query url.Values) *Response
func AdminPost(t *testing.T, path, actorID, role string, body interface{}) *Response
\`\`\`

### 4.3 业务校验工具

\`\`\`go
func GenInternalAuth(entityId, secret string) (token, timespan string)
func GenInternalAuthAt(entityId, secret string, unixSec int64) (token, timespan string)
func GenInternalAuthExpired(entityId, secret string) (token, timespan string)
func EntitySecretOf(entityId string) string
\`\`\`

### 4.4 协议层断言

\`\`\`go
func AssertInternalStatus(t *testing.T, r *Response, expectedStatus int)
func AssertInternalSucc(t *testing.T, r *Response)
func AssertErrNo(t *testing.T, r *Response, expectedErrNo int)
func AssertSucc(t *testing.T, r *Response)
func AssertFail(t *testing.T, r *Response, expectedErrNo int)
func AssertJSONSubset(t *testing.T, actual json.RawMessage, expected interface{})
func AssertJSONField(t *testing.T, actual json.RawMessage, fieldPath string, expected interface{})
\`\`\`

### 4.5 副作用层断言

\`\`\`go
// MySQL
func AssertMysqlCount(t *testing.T, lib, table, where string, expected int64, args ...interface{})
func AssertMysqlField(t *testing.T, lib, table, field, where string, expected interface{}, args ...interface{})
func AssertMysqlExists(t *testing.T, lib, table, where string, args ...interface{})
func AssertMysqlAbsent(t *testing.T, lib, table, where string, args ...interface{})

// Redis
func AssertRedisKeyExists(t *testing.T, key string)
func AssertRedisKeyAbsent(t *testing.T, key string)
func AssertRedisStringEq(t *testing.T, key, expected string)
func AssertRedisHashField(t *testing.T, key, field, expected string)
func AssertRedisSetMember(t *testing.T, key, member string)
func AssertRedisTTLBetween(t *testing.T, key string, minSec, maxSec int64)
func ClearRedisKey(t *testing.T, key string)
func FlushRedis(t *testing.T)

// Kafka
var framework.Kafka *FakeKafkaProducer
func (f *FakeKafkaProducer) Messages() []FakeKafkaMessage
func (f *FakeKafkaProducer) MessagesByTopic(topic string) []FakeKafkaMessage
func (f *FakeKafkaProducer) Reset()
\`\`\`

### 4.6 数据管理

\`\`\`go
func WarmupRedis(t *testing.T)
func TruncateAll(t *testing.T)
\`\`\`

### 4.7 故障注入

\`\`\`go
// 资源失败注入
func InjectRedisFailure(t *testing.T, op string, errMode string) // op="GET"/"SET"，errMode="timeout"/"refused"
func InjectMysqlFailure(t *testing.T, lib, op string, errMode string)
func InjectKafkaFailure(t *testing.T, topic string, errMode string)
func ClearAllFailures(t *testing.T) // 测试结束自动调用

// 熔断器状态注入
func ForceCircuitOpen(t *testing.T, resource string)
func ResetCircuit(t *testing.T, resource string)

// 时钟控制（用于 Saga 超时 / 一致性窗口测试）
func FreezeClock(t *testing.T, at time.Time)
func AdvanceClock(t *testing.T, d time.Duration)
\`\`\`

故障注入 API 用于验证 [`21_distributed_transaction.md`](21_distributed_transaction_spec.md) §6 失败路径全景图覆盖的所有分支。

### 4.8 并发测试支持

\`\`\`go
// goroutine 泄漏检测
// 在 framework.RunTests 内部统一注入 goleak.VerifyTestMain；
// 常驻 goroutine 通过 framework.RegisterLongLivedGoroutine(name) 预登记

// 并发 helper
func RunParallel(t *testing.T, n int, fn func(workerID int))
\`\`\`

### 4.9 性能回归

\`\`\`go
// 基准测试入口（与标准 b.RunParallel 协作）
func BenchmarkSetup(b *testing.B)  // 预热 + 重置指标
func RecordP99(b *testing.B, op string, durations []time.Duration) // 输出 P99
\`\`\`

基准测试目标值的真相源在 [`22_performance_contract.md §7`](22_performance_contract_spec.md)；本节仅承载 framework API 签名。
```

API 列表**必须完整**——AI 写测试用例时只能从这里取签名。

---

## 7. §5 测试纪律（强制）

```markdown
## 5. 测试纪律（硬约束）

### 5.1 写代码必须同步写测试

任何 Skill（new-service / new-controller / new-middleware / new-router / new-scheduled-task / new-mq-consumer）在生成业务代码后，**必须**同步新增/扩展 `test/cases/` 下对应目录的测试用例。

交付视角定义：
- ✅ **有交付**：代码 + 测试用例 + 文档同步 → 三件事都做完
- ❌ **未交付**：只写代码、测试留 TODO → 返工

### 5.2 测试失败即交付失败

- 代码生成后必须跑 `go test ./test/cases/{scope}/...` 通过才算完
- CI 里跑 `go test ./test/cases/... ./test/e2e/...`，有红即阻断合并
- 下游 skill 依赖上游时（controller 依赖 service），**先跑上游测试通过再生成下游**

### 5.3 用例覆盖标准（4 类）

每个新接口至少 **4 类**用例：

| 类别 | 例 |
|------|-----|
| 正向成功 | 合法参数 → 期望返回 |
| 参数校验 | 缺字段 / 格式错误 / 超范围 → 对应错误码 |
| 业务错误 | {状态枚举值} / {权限不足} / {资源不足类错误} → 对应错误码 |
| 副作用 | Redis / MySQL / Kafka 是否正确写入 |

### 5.4 覆盖率目标

- 单接口 controller 层：line coverage ≥ 70%
- service 层：核心方法 ≥ 80%
- CI 不硬卡覆盖率，但 PR 时自动评论提示

### 5.5 高级用例类别

在 §5.3 基础 4 类（正向 / 参数校验 / 业务错误 / 副作用）之上，部分场景必须额外补 3 类：

| 类别 | 触发场景 | 必含内容 | 关联规范 |
|------|---------|---------|---------|
| **并发用例** | 方法涉及共享状态 / 锁 / channel | `b.RunParallel` 或 `RunParallel(t, N, fn)` + `go test -race` 必须通过 | [`docs-spec/20`](20_concurrency_safety_spec.md) |
| **性能回归** | 方法在 `performance_contract.md §2` 热路径清单 | `Benchmark{Method}_Parallel` + P99 / allocs/op 比对基线 | [`docs-spec/22 §7`](22_performance_contract_spec.md) |
| **故障注入** | 方法涉及多数据源写入 / 跨服务调用 | 对每条 `transaction_design.md §6 失败路径全景图`分支至少 1 个用例（用 §4.7 注入 API）| [`docs-spec/21 §6`](21_distributed_transaction_spec.md) |
| **尾延迟 / 过载回归** | 接口接入对冲 / 自适应并发限制 / load shedding / 重试预算 | 对冲只在超 tie-delay 且幂等时触发；load shedding 过载时先丢低优先级 / 已超截止；重试预算封顶（占比上限）不放大；自适应并发限制随时延回退（用 §4.7 注入 + §4.9 基准 API）| [`design-spec/07 §7`](../design-spec/07_tail_latency_design.md) |

**纪律**：上述类别"必须"等同于 §5.1 / §5.2 的纪律——漏写视为未交付。但只在触发条件命中时强制（简单 CRUD 不必凑齐 7 类）。
```

---

## 8. §8 禁用场景（不写测试的例外）

```markdown
## 8. 禁用场景

以下情况不强制写测试：

- 内部纯数据结构定义（如 `service/{module-1}/types.go` 中只有 struct）
- 代码生成 / 非业务代码（如 seed SQL 本身、framework 内部辅助）
- 设计文档中明确标注为 🚫 不实现的模块
```

---

## 9. §6 与 Stage 协同（必填）

```markdown
## 6. 与 Stage 的协同

| Stage | 业务代码交付 | 测试用例交付（同步） |
|-------|------------|---------------------|
| 1 工具&常量 | components/constants.go、helpers/*.go | helpers/*_test.go + framework/sign_test.go |
| 2 最小业务校验 | middleware/{sign_mw}、service/{auth-module}、Internal/{module}/{action} | cases/internal/{module}_{action}_test.go |
| 3 {核心实体管理类} | service/{module-1}、service/{module-2}、operator/* | cases/operator/{module}_*_test.go + {module}_*_test.go |
| 4 业务闭环 | service/{module-N}、Internal/{核心闭环 module}/{action}、{example-topic}_consumer | cases/internal/{module}_*_test.go + e2e/scenario_{module}_{flush-action}_test.go |
| 5 后台管理 | middleware/{admin_mw}、service/{module-4}、admin/* | cases/admin/*_test.go |
```

---

## 10. 与其他文档的关系

| 内容 | testing_design.md | api/*.md | service/*.md | mvp_rebuild_path.md |
|------|-------------------|----------|---------------|---------------------|
| 框架 API 签名 | ✅ 完整 | 不写 | 不写 | 不写 |
| 用例覆盖标准 | ✅ 完整 | 不写 | 不写 | 引用 |
| Stage 测试交付清单 | 简略 | 不写 | 不写 | ✅ 详细 |

---

## 11. 维护

| 触发 | 动作 |
|------|------|
| 新增框架 API（assert 函数等） | §4 对应章节 + 同步 framework 实现 |
| 修改 seed 数据 | seed/README.md + 受影响的 cases/*_test.go |
| 调整覆盖率目标 | §5.4 + CI 配置 |
| 新增 Stage | §6 协同表 |


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
