# 15 - Helpers / pkg API 契约文档规范

> 规定 `docs/architecture/helpers_api.md` 与 `docs/architecture/pkg_api.md` 的内容结构。这两份文档是 **AI 编写业务代码时引用类型/函数签名的唯一真相源**。

---

## 1. 为什么这两份文档极其重要

业务代码**频繁**调用 helpers / pkg 的全局变量与工具函数：

- `helpers.MysqlClientCore`
- `helpers.{Project}CacheClient`
- `helpers.{LocalCache1}`
- `tlog.InfoLogger`
- `base.RenderJsonFail`
- `kafka.PubClient`
- `circuitbreaker.ErrNotAllowed`
- ...

如果这些**签名**没有真相源，AI 在生成业务代码时会**频繁猜错**——参数名、参数类型、返回值数量、错误类型——导致 `go build` 大面积失败。

**结论**：这两份文档是 AI 在生成业务代码前**必读**的契约书。

---

## 2. helpers_api.md vs pkg_api.md 的分工

| 文档 | 内容 |
|------|------|
| `helpers_api.md` | 项目内部 `helpers/` 包的导出内容：工具函数、全局变量、各资源 Client（rediscache.Client、MysqlClient 等） |
| `pkg_api.md` | 项目依赖的内部框架（如 `pkg/golib`、`pkg/gin`）的公开类型与函数签名 |

通用区分：

- **本项目可改的代码** → helpers_api.md
- **依赖的内部框架（不轻易改）** → pkg_api.md

---

## 3. helpers_api.md 顶层结构

```markdown
# 辅助函数与全局变量 API 参考

## 1. 工具函数（helpers/*.go 中的导出函数）
   1.1 IP / 网络
   1.2 加密 / 哈希
   1.3 数学
   1.4 ID 生成（如有）

## 2. 全局变量速查表（强制：所有 helpers 包导出的全局变量）
   一表列出：变量名 / 类型 / 用途 / 实现状态（🟢/🟡/⬜）

## 3. 各资源 Client 详细方法
   3.1 MySQL Client（每个库实例的选择规则）
   3.2 Redis Client（rediscache.Client 完整方法签名）
   3.3 Kafka Client（PubClient / SubClient）
   3.4 GCache Client（BucketCache 完整 API）
   3.5 CircuitBreaker（CircuitBreakerInstance 函数签名）

## 4. 初始化函数
   4.1 PreInit / InitResource / InitResourceForCron 的完整签名与时序

## 5. Render 函数（如有自定义 Render）

## 6. 审计辅助函数（如有）
```

### 3.1 §1 工具函数（必填）

每个工具函数：**完整签名 + 用途 + 调用方**。

```markdown
### 1.2 加密 / 哈希

\`\`\`go
// Md5Hex 计算 MD5 并返回 32 位大写 hex 字符串
//
// 输入：  s string
// 输出：  string（32 位大写 hex）
//
// 调用方：签名校验中间件、签名工具
func Md5Hex(s string) string

// GenerateEntityID 生成 19 位 EntityID："ak_" + 16 位小写 hex
//
// 输入：  无
// 输出：  string
// 性能：  crypto/rand 1 次（约 100ns）
//
// 调用方：service.{module}.Method
func GenerateEntityID() string

// GenerateEntitySecret 生成 32 位 EntitySecret
func GenerateEntitySecret() string

// HashPassword 用 bcrypt 哈希密码（cost=10）
func HashPassword(password string) (string, error)

// CheckPassword 验证密码与 hash 是否匹配
func CheckPassword(password, hash string) bool
\`\`\`
```

### 3.2 §2 全局变量速查表（强制）

```markdown
## 2. 全局变量速查表

| 变量 | 类型 | 用途 | 状态 |
|------|------|------|------|
| `helpers.MysqlClientCore` | `*gorm.DB` | core 库 | 🟢 |
| `helpers.MysqlClientTrade` | `*gorm.DB` | trade 库 | 🟢 |
| `helpers.MysqlClientLog` | `*gorm.DB` | log 库 | 🟢 |
| `helpers.MysqlClientCallLog` | `*gorm.DB` | {example_log} 库（按月分片） | 🟢 |
| `helpers.{Project}CacheClient` | `*rediscache.Client` | Redis 客户端 | 🟢 |
| `helpers.{LocalCache1}` | `*gcache.BucketCache` | 热数据本地缓存 | 🟢 |
| `helpers.{LocalCache3}` | `*gcache.BucketCache` | {受限模块}规则本地缓存 | 🚫 |
| `helpers.{Example}PubClient` | `*kafka.PubClient` | 业务事件流水生产者 | 🟢 |
| `helpers.{Example}SubClient` | `*kafka.SubClient` | 业务事件流水消费者 | ⬜ |
```

每个全局变量必填：**变量名 / 类型 / 用途 / 实现状态**（与 BUILD_STATUS 一致）。

### 3.3 §3 资源 Client 详细方法

#### 3.3.1 MySQL Client 选择规则（必填）

```markdown
### 3.1 MySQL Client 选择规则

按表所属库选择 client：

| 表 | Client |
|----|--------|
| {example_table_user} / {example_table_entity} / {example_table_product} / ... | `helpers.MysqlClientCore` |
| {example_table_order} / {example_table_recharge} / ... | `helpers.MysqlClientTrade` |
| {example_table_stats} / {example_table_event} / {example_table_audit_log} | `helpers.MysqlClientLog` |
| {example_table_log}_YYYYMM | `helpers.MysqlClientCallLog` |

错误的 client 选择会导致："表不存在" 或访问到错误的数据库。
```

#### 3.3.2 Redis Client 完整方法签名（必填）

```markdown
### 3.2 rediscache.Client 完整方法

\`\`\`go
type Client struct {
    // 内部字段省略
}

// String
func (c *Client) Get(ctx *gin.Context, key string) (string, error)
func (c *Client) Set(ctx *gin.Context, key string, value interface{}, expire int64) error
func (c *Client) SetNX(ctx *gin.Context, key string, value interface{}, expire int64) (bool, error)
func (c *Client) Incr(ctx *gin.Context, key string) (int64, error)
func (c *Client) Expire(ctx *gin.Context, key string, expire int64) error
func (c *Client) Del(ctx *gin.Context, keys ...string) error
func (c *Client) Exists(ctx *gin.Context, key string) (bool, error)

// Hash
func (c *Client) HGet(ctx *gin.Context, key, field string) (string, error)
func (c *Client) HSet(ctx *gin.Context, key string, fields ...interface{}) error
func (c *Client) HMSet(ctx *gin.Context, key string, fieldsValues map[string]interface{}) error
func (c *Client) HGetAll(ctx *gin.Context, key string) (map[string]string, error)
func (c *Client) HIncrBy(ctx *gin.Context, key, field string, value int64) (int64, error)
func (c *Client) HDel(ctx *gin.Context, key string, fields ...string) error

// Set
func (c *Client) SAdd(ctx *gin.Context, key string, members ...interface{}) error
func (c *Client) SRem(ctx *gin.Context, key string, members ...interface{}) error
func (c *Client) SMembers(ctx *gin.Context, key string) ([]string, error)
func (c *Client) SIsMember(ctx *gin.Context, key string, member interface{}) (bool, error)
func (c *Client) SPop(ctx *gin.Context, key string) (string, error)
func (c *Client) SPopN(ctx *gin.Context, key string, count int) ([]string, error)
func (c *Client) SCard(ctx *gin.Context, key string) (int64, error)

// Pipeline
func (c *Client) Pipelined(ctx *gin.Context, fn func(p Pipeliner) error) ([]interface{}, error)
\`\`\`

**重要**：所有方法的第一个参数都是 `ctx *gin.Context`。
**错误处理**：熔断时返回 `circuitbreaker.ErrNotAllowed`。
```

签名一字不差。**业务代码直接 copy-paste 调用，不需要查源码**。

---

## 4. pkg_api.md 顶层结构

```markdown
# pkg/ 内部框架 API 契约

## 1. base 包
   1.1 RenderJson* 系列
   1.2 Error 类型
   1.3 Engine

## 2. tlog 包
   2.1 Logger 函数（Sugared / Structured）
   2.2 Field 构造器
   2.3 SetLogger / 日志切割

## 3. redis 包（如有）
   3.1 InitRedis 函数签名
   3.2 PoolConfig

## 4. kafka 包
   4.1 PubClient
   4.2 SubClient
   4.3 InitKafkaPub / InitKafkaSub
   4.4 Pub / Subscribe / GetKafkaMsg

## 5. circuitbreaker 包
   5.1 CircuitBreaker 接口
   5.2 ErrNotAllowed
   5.3 各种 Adapter

## 6. gcache 包
   6.1 BucketCache 类型
   6.2 NewBucketCache
   6.3 Get / Set / Delete / Has

## 7. middleware 包（框架内置中间件）

## 8. env 包

## 9. server 包

## 10. command 包（cobra 封装）

## 11. Gin Context 扩展（如有）
```

### 4.1 单个类型/函数的描述格式

```markdown
### N.M 类型名

\`\`\`go
type {TypeName} struct {
    // 字段（仅导出字段，私有字段不写）
    Field1 string
    Field2 int
}

// 方法 1
func (t *TypeName) Method1(arg string) (string, error)

// 方法 2
func (t *TypeName) Method2(args ...interface{}) error
\`\`\`

**用途**：（一句话）

**典型用法**：

\`\`\`go
// 业务代码 copy-paste 模板
\`\`\`

**注意事项**：（如线程安全、初始化顺序、错误处理）
```

### 4.2 不写的内容

- 内部实现细节（公开 API 不依赖的私有方法）
- 已弃用的接口（要删除）
- 未导出的类型 / 函数

---

## 5. 维护规则（强制）

### 5.1 触发同步的代码变更

| 代码变更 | 必须更新 |
|---------|---------|
| `helpers/` 新增导出函数 | helpers_api.md §1 |
| `helpers/` 新增全局变量 | helpers_api.md §2 |
| 修改任意全局变量类型 | helpers_api.md §2 + 受影响的业务代码 |
| 修改 rediscache.Client 方法签名 | helpers_api.md §3.2 + service 层全部使用点 |
| `pkg/` 新增导出类型 | pkg_api.md 对应章节 |
| 升级 `pkg/` 依赖版本 | pkg_api.md 全量评审 |

### 5.2 文档优先 vs 代码优先

按 `aiweave/PRINCIPLES.md` §5 的签名冲突裁决：

- 既有契约（已被业务代码引用） → **以代码为准**，修正文档
- 未来设计（业务代码尚未引用） → **以文档为准**，按文档生成代码

特别是 helpers_api.md / pkg_api.md，**因业务代码大量引用**，签名冲突时几乎总是**改 md**。

---

## 6. 与其他文档的关系

| 内容 | helpers_api.md / pkg_api.md | infrastructure.md | service/*.md |
|------|-----------------------------|-------------------|--------------|
| 函数签名 | ✅ 完整 | 不写 | 引用 |
| 全局变量声明 | ✅ 完整 | 引用（初始化时序） | 引用 |
| 资源初始化时序 | 简略 | ✅ 完整 | 不写 |
| 业务调用伪码 | 不写 | 不写 | ✅ 完整 |

---

## 7. 不允许的写法

```markdown
❌ "helpers 提供了 MySQL 访问方法"   ← 太空，没有签名
❌ "Redis Client 支持常见操作"        ← 没有方法清单
❌ 函数签名省略错误返回值
❌ 函数签名省略 ctx 参数
❌ 写一个未导出的函数（小写开头）
❌ 写一个 // TODO 的占位函数
```

---

## 8. AI 行为约束

> 所有 Skill（new-service / new-controller / new-middleware 等）的第 1 步必须读 helpers_api.md 和 pkg_api.md。

如果某个签名在 md 中**找不到**，AI 必须：

1. 停止生成业务代码
2. 提醒用户：md 中缺失某签名，可能是 (a) 文档漏写，(b) 业务设计需要新增 helper
3. 由用户裁决：补 md / 补 helper 函数 / 改业务设计避开缺失的签名


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
