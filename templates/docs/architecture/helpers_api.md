# 辅助函数与全局变量 API 参考

> 规范来源：`aiweave/docs-spec/15_helpers_pkg_api_spec.md`

## 1. 工具函数

### 1.1 IP / 网络
```go
func IsInternalIP(ip string) bool
```

### 1.2 加密 / 哈希
```go
func Md5Hex(s string) string
func GenerateEntityID() string
func GenerateEntitySecret() string
func HashPassword(password string) (string, error)
func CheckPassword(password, hash string) bool
```

### 1.3 数学
```go
func AbsInt64(x int64) int64
```

### 1.4 ID 生成（如有）

## 2. 全局变量速查表

| 变量 | 类型 | 用途 | 状态 |
|------|------|------|------|
| `helpers.MysqlClientCore` | `*gorm.DB` | core 库 | ⬜ |
| `helpers.{Project}CacheClient` | `*rediscache.Client` | Redis 客户端 | ⬜ |
| `helpers.{LocalCache1}` | `*gcache.BucketCache` | 热数据本地缓存 | ⬜ |
| `helpers.{Example}PubClient` | `*kafka.PubClient` | Kafka Producer | ⬜ |
| `helpers.{Example}SubClient` | `*kafka.SubClient` | Kafka Consumer | ⬜ |

## 3. 各资源 Client 详细方法

### 3.1 MySQL Client 选择规则

| 表 | Client |
|----|--------|
| ... | `helpers.MysqlClientCore` |
| ... | `helpers.MysqlClientTrade` |

### 3.2 rediscache.Client 完整方法

```go
func (c *Client) Get(ctx *gin.Context, key string) (string, error)
func (c *Client) Set(ctx *gin.Context, key string, value interface{}, expire int64) error
// ... 完整方法签名
```

### 3.3 Kafka Client

### 3.4 GCache Client

### 3.5 CircuitBreaker

## 4. 初始化函数

| 函数 | 签名 | 用途 |
|------|------|------|
| `helpers.PreInit` | `PreInit()` | 配置 / 日志 |
| `helpers.InitResource` | `InitResource(engine *gin.Engine)` | HTTP 服务模式 |
| `helpers.InitResourceForCron` | `InitResourceForCron(engine *gin.Engine)` | cobra 任务模式 |

## 5. Render 函数（如有）

## 6. 审计辅助函数（如有）


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
