# 基础设施初始化

> 规范来源：`aiweave/docs-spec/04_architecture_overview_spec.md` §3

## 1. 资源清单

| 资源 | 实例数 | 用途 |
|------|--------|------|
| MySQL | {N} | {库划分} |
| Redis | {N} | {用途} |
| Kafka topic | {N} | {用途} |
| 本地缓存 | {N} | {用途} |

## 2. 初始化时序

```
helpers.PreInit()
   └── 配置加载、日志初始化

helpers.InitResource()
   ├── InitCircuitBreaker()    # 必须最先
   ├── InitRedis()
   ├── InitMysql()
   ├── InitGPool()
   ├── InitGCache()
   └── InitKafkaProducer()
```

## 3. 各资源初始化函数

| 函数 | 文件 | 职责 |
|------|------|------|
| `InitCircuitBreaker` | `helpers/circuit_breaker.go` | 注册 CB registry |
| `InitRedis` | `helpers/redis.go` | rediscache.Client 实例 |
| `InitMysql` | `helpers/mysql.go` | GORM Plugin 注入 CB |
| ... | ... | ... |

## 4. 全局客户端

详见 [`helpers_api.md`](helpers_api.md) §2。

## 6. 连接池配置基线登记

> 决策/基线见 [design-spec/06_resilience_design.md §9.2](../../../design-spec/06_resilience_design.md)。本节只登记本工程"选了什么"，不复制方法论。

每个资源（DB / 缓存 / 外部 API）一行，登记连接池与超时取值：

| 资源 | `{maxIdle}` | `{maxOpen}` | `{conn-timeout-ms}` | `{read-timeout-ms}` | `{write-timeout-ms}` | `{conn-max-lifetime}` | 重试 |
|------|------|------|------|------|------|------|------|
| `{db-resource}` | `{maxIdle}` | `{maxOpen}` | `{conn-timeout-ms}` | `{read-timeout-ms}` | `{write-timeout-ms}` | `{conn-max-lifetime}` | `{retry}` |
| `{cache-resource}` | `{maxIdle}` | `{maxOpen}` | `{conn-timeout-ms}` | `{read-timeout-ms}` | `{write-timeout-ms}` | `{conn-max-lifetime}` | `{retry}` |
| `{external-api}` | `{maxIdle}` | `{maxOpen}` | `{conn-timeout-ms}` | `{read-timeout-ms}` | `{write-timeout-ms}` | `{conn-max-lifetime}` | `{retry}` |

> **约束提示**：写超时 ≥ 读超时（压缩 / 大包写入更慢）；外部调用用短超时 + 重试，避免被慢依赖拖垮。

## 7. 嵌入式库资源登记（如启用本地镜像）

> 决策/基线见 [design-spec/03_concurrency_design.md §9.4](../../../design-spec/03_concurrency_design.md)。本节只登记本工程"选了什么"，不复制方法论。

启用进程内嵌入式库（如本地镜像）时，按"读 CPU-bound / 写串行"双池登记：

| 池 | `{maxOpen}` | 取值方法 | 连接策略 |
|----|------|---------|---------|
| 读池 | `{read-maxOpen}` | `min(max(2×NumCPU, 16), 64)` | 长存保温（连接复用保住私有 page cache） |
| 写池 | `1` | `MaxOpenConns=1` | Go 池层钳成串行，根除 `database is locked` |

> **约束提示**：读是进程内 CPU-bound 访问（µs 级），按核数定容而非套用网络池量级；写并发钳为 1，单后台协程顺序刷新全部表。

## 8. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意：本文主体已用占位符表达；本节具体数值为某真实工程经验基线，落地工程按业务 / 压测结果替换。服从 [`PRINCIPLES.md §12`](../../../PRINCIPLES.md#12-占位符规则强制--双轨结构) 参考示例段豁免。

§6 连接池基线（经验值，非金科玉律）：

| 资源 | maxIdle / maxOpen | conn-timeout | read-timeout | write-timeout | conn-max-lifetime | retry |
|------|------|------|------|------|------|------|
| MySQL（每库） | 100 / 100 | 1500ms | 3s | 3s | 3600s | — |
| Redis | 100 / 100 | 300ms | 300ms | 600ms | 10m | — |
| 外部 API | — / 100 | 300ms | 300ms | — | 300s | 1 |

§7 嵌入式库（本地 SQLite）：读池 `MaxOpenConns = min(max(2×NumCPU,16),64)` 长存保温；写池 `MaxOpenConns=1`。


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
