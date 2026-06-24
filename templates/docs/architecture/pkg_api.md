# pkg/ 内部框架 API 契约

> 规范来源：`aiweave/docs-spec/15_helpers_pkg_api_spec.md`
>
> AI 生成任何业务代码前**必读**。本文是签名一字不差的真相源。

## 1. base 包

### 1.1 RenderJson* 系列

```go
func RenderJsonSucc(ctx *gin.Context, data interface{})
func RenderJsonFail(ctx *gin.Context, err error)
func RenderJsonAbort(ctx *gin.Context, err error)
```

### 1.2 Error 类型

```go
type Error struct {
    ErrNo  int
    ErrMsg string
}

func (e Error) Error() string
func (e Error) Sprintf(args ...interface{}) Error
func (e Error) Wrap(err error) Error
```

### 1.3 Engine

## 2. tlog 包

### 2.1 Logger 函数（Structured 强制）

```go
func InfoLogger(ctx *gin.Context, msg string, fields ...Field)
func WarnLogger(ctx *gin.Context, msg string, fields ...Field)
func ErrorLogger(ctx *gin.Context, msg string, fields ...Field)
```

### 2.2 Field 构造器

```go
func String(key, val string) Field
func Int(key string, val int) Field
func Int64(key string, val int64) Field
func Float(key string, val float64) Field
func Bool(key string, val bool) Field
func Time(key string, val time.Time) Field
func Duration(key string, val time.Duration) Field
func Error(err error) Field
func Any(key string, val interface{}) Field
```

## 3. redis 包

### 3.1 InitRedis 函数签名
### 3.2 PoolConfig

## 4. kafka 包

### 4.1 PubClient
### 4.2 SubClient
### 4.3 InitKafkaPub / InitKafkaSub
### 4.4 Pub / Subscribe / GetKafkaMsg

## 5. circuitbreaker 包

### 5.1 CircuitBreaker 接口
### 5.2 ErrNotAllowed
### 5.3 各种 Adapter

## 6. gcache 包

### 6.1 BucketCache
### 6.2 NewBucketCache / Get / Set / Delete / Has

## 7. middleware 包（框架内置）

## 8. env 包

## 9. server 包

## 10. command 包

```go
func InitCycle(engine *gin.Engine) *Cycle
func (c *Cycle) AddFunc(interval time.Duration, f func(*gin.Context) error)
func (c *Cycle) Start()

func InitCrontab(engine *gin.Engine) *Crontab
func (c *Crontab) AddFunc(spec string, f func(*gin.Context) error)
```

## 11. Gin Context 扩展

```go
ctx.Query(key) string
ctx.GetHeader(key) string
ctx.GetString(key) string
ctx.Set(key string, value interface{})
ctx.ShouldBindJSON(obj interface{}) error
ctx.ShouldBindQuery(obj interface{}) error
ctx.JSON(code int, obj interface{})
ctx.AbortWithStatusJSON(code int, obj interface{})
ctx.Next()
ctx.ClientIP() string
```


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
