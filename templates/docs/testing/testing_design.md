# 测试框架设计

> 规范来源：`aiweave/docs-spec/17_testing_design_spec.md`

## 1. 定位

接口级黑盒测试框架。覆盖全部业务接口 + 副作用断言（MySQL / Redis / Kafka）。

```
perf/   ← 基准压测
e2e/    ← 多接口串联场景
cases/  ← 单接口黑盒
framework/ ← 框架自身
```

## 2. 核心设计原则

### 2.1 in-process 启动
`framework.RunTests(m)` 通过 `httptest.NewServer(engine)` 在进程内启动 Gin。

### 2.2 三种业务校验透明注入
| 路由组 | 注入方式 | 中间件 |
|-------|---------|--------|
| Internal | MD5 签名 + Header Token/Timestamp | {sign_mw} |
| Operator | Header X-{User}-Id | {user_mw} |
| Admin | Header X-{Actor}-Id + X-{Actor}-Role | {admin_mw} |

### 2.3 开发者自备依赖 + yaml 直配
`test/config/resource.yaml` 直接配置。**不引入 docker-compose**。

### 2.4 Seed 数据唯一真相源
`test/fixtures/seed/README.md` 是样本数据的权威清单。

### 2.5 断言双通道
- 协议层：AssertInternalStatus / AssertErrNo / AssertJSONSubset
- 副作用层：AssertRedisHashField / AssertMysqlRow / Kafka.Messages()

### 2.6 Fake MQ Producer
`framework.NewFakeKafkaProducer()` 实现 Pub 接口，捕获消息供断言。

## 3. 目录契约

```
test/
├── README.md
├── .env.example
├── config/                       # 测试配置
├── fixtures/
│   ├── schema/                   # DDL
│   ├── seed/                     # 样本 SQL + README
│   └── redis_warmup.go
├── framework/                    # 框架自身
├── cases/
│   ├── probe/
│   ├── internal/
│   ├── operator/
│   └── admin/
├── e2e/
├── perf/
└── scripts/
```

## 4. 框架 API 稳定契约

### 4.1 入口
```go
func RunTests(m *testing.M) int
```

### 4.2 HTTP 调用
```go
func InternalGet(t *testing.T, path, entityId, secret string, query url.Values) *Response
func InternalPost(t *testing.T, path, entityId, secret string, body interface{}) *Response
func OperatorGet(...) *Response
func OperatorPost(...) *Response
func AdminGet(...) *Response
func AdminPost(...) *Response
```

### 4.3 业务校验工具
```go
func GenInternalAuth(entityId, secret string) (token, timespan string)
// ...
```

### 4.4 协议层断言
```go
func AssertInternalStatus / AssertSucc / AssertFail / AssertJSONSubset / AssertJSONField
```

### 4.5 副作用层断言
```go
func AssertMysqlCount / AssertMysqlField / AssertMysqlExists / AssertMysqlAbsent
func AssertRedisKeyExists / AssertRedisHashField / AssertRedisSetMember / AssertRedisTTLBetween
var Kafka *FakeKafkaProducer
```

### 4.6 数据管理
```go
func WarmupRedis(t *testing.T)
func TruncateAll(t *testing.T)
```

## 5. 测试纪律

### 5.1 写代码必须同步写测试
**铁律**：每个 Skill 生成代码后必须同步测试。漏写测试视为未交付。

### 5.2 测试失败即交付失败

### 5.3 用例覆盖标准（4 类）
- 正向 / 参数校验 / 业务错误 / 副作用

### 5.4 覆盖率目标
- controller ≥ 70% / service ≥ 80%

### 5.5 高级用例类别（触发才强制）
- 并发（共享状态 / 锁 / channel，`-race`）/ 性能回归（热路径，P99 比对）/ 故障注入（多数据源 / 跨服务）
- 尾延迟 / 过载回归（接入对冲 / 自适应限制 / load shedding / 重试预算）：对冲只在超 tie-delay 且幂等时触发；过载先丢低优先级 / 已超截止；重试预算封顶不放大；自适应限制随时延回退（决策见 `aiweave/design-spec/07 §7`）

## 6. 与 Stage 的协同

| Stage | 业务代码 | 测试 |
|-------|---------|------|
| 1 | helpers + constants | helpers/_test.go |
| 2 | 最小关键链路 | cases/{audience}/...test.go |
| ... | ... | ... |

## 7. 配套 skill

## 8. 禁用场景

## 9. 运行流程

```bash
cd test
./scripts/setup.sh
go test ./cases/... -v
```

## 10. 与 docs/ 的交叉引用


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
