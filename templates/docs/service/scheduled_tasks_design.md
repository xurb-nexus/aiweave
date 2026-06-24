# 定时任务与 MQ 消费者设计

> 规范来源：`aiweave/docs-spec/10_scheduled_tasks_spec.md` + `11_mq_consumer_spec.md`

## 1. 任务总览

### 1.1 三种执行模式对比

| 模式 | 触发方式 | 适用场景 | 并行特点 |
|------|---------|---------|---------|
| cobra | `go run main.go {task}` | 间隔 ≥ 30min | 调度平台单进程 |
| cycle | 随 HTTP 启动 | 间隔 < 30min，不重叠 | 多实例并行（需 SPOP / 锁） |
| crontab | 随 HTTP 启动 | 必须定点触发 | 多实例并行（需锁） |

### 1.2 任务清单总表

| 任务名 | 类型 | Service 方法 | 触发频率 | 并行策略 | 文档章节 |
|-------|------|-------------|---------|---------|---------|
| ... | ... | ... | ... | ... | ... |

### 1.3 触发频率与时序图

## 2. cobra 任务

### 2.1 任务命名约定
- kebab-case
- 动词在前

### 2.2 注册代码模板（router/command.go::Commands）

```go
rootCmd.AddCommand(&cobra.Command{
    Use: "{task-name}",
    Run: func(cmd *cobra.Command, args []string) {
        run(engine, command_ctrl.{TaskFunc}, args...)
    },
})
```

### 2.3 单个任务详细设计

### 2.X {task_name}
**目标**：
**触发**：cobra，调度平台 ...
**并行策略**：
**Service 方法**：
**执行流程**：
**Controller 入口**：
**异常处理**：
**性能预期**：
**监控**：

## 3. cycle 任务（内部协程）

### 3.1 cycle vs crontab 选择规则
- 不重叠语义 → cycle
- 必须定点 → crontab

### 3.2 注册代码模板（startCycle）

```go
c := command.InitCycle(engine)
c.AddFunc(5*time.Minute, func(ctx *gin.Context) error {
    return {module}.FlushStateToDb(ctx)
})
c.Start()
```

### 3.X 单个 cycle 任务详细设计

## 4. crontab 任务

### 4.1 注册代码模板（startCrontab）

```go
c := command.InitCrontab(engine)
c.AddFunc("0 0 2 * * *", func(ctx *gin.Context) error {
    return {module-N}.{AggregateMethod-A}(ctx)
})
// InitCrontab 内部已 Start，不要再 Start
```

### 4.X 单个 crontab 任务详细设计

## 5. 异常恢复与容错

### 5.1 任务崩溃恢复
### 5.2 锁未释放兜底
### 5.3 数据不一致修复
### 5.4 监控告警阈值

## 6. Controller 入口与 Service 方法签名

| 任务 | command/{file}.go | service.{Method} | 参数 |
|------|------------------|-----------------|------|
| ... | ... | ... | ... |

## 7. 并行策略详细

### 7.1 SPOP 模式（推荐 flush 类）

```go
for {
    member, err := conn.Do("SPOP", "{name}:dirty")
    if member == nil { break }
    ...
}
```

### 7.2 分布式锁模式

```go
locked, _ := conn.Do("SET", "lock:{name}", "1", "NX", "EX", lockTTL)
if locked == nil { return nil }
defer conn.Do("DEL", "lock:{name}")
```

### 7.3 单进程外部调度

## 8. router/command.go 完整结构

```go
package router

import (...)

func Commands(rootCmd *cobra.Command, engine *gin.Engine) { ... }
func run(engine *gin.Engine, f func(...) error, args ...string) error { ... }
func Tasks(engine *gin.Engine) { startCycle(engine); startCrontab(engine) }
func startCycle(engine *gin.Engine) { ... }
func startCrontab(engine *gin.Engine) { ... }
```

---

## 9. MQ 消费者（如有）

### 9.1 总览

**消费的 topic**：
| Topic | 消费者文件 | 消费 group | 用途 |
|-------|----------|-----------|------|
| ... | ... | ... | ... |

**生产的 topic**：
| Topic | 生产者文件 | 调用方 | 用途 |
|-------|----------|--------|------|

### 9.2 每个 Consumer 的详细设计

#### 9.2.1 消息结构

```go
type {Topic}Msg struct {
    // ...
}
```

| 字段 | 类型 | 必填 | 单位 | 来源 | 说明 |
|------|------|------|------|------|------|

#### 9.2.2 消费流程
- 攒批策略
- 落库流程
- offset 提交语义

#### 9.2.3 失败处理
- 重试策略
- 死信队列
- 告警

#### 9.2.4 性能与并发

### 9.3 Producer 侧

### 9.4 helpers 集成

```go
var {Topic}PubClient *kafka.PubClient
var {Topic}SubClient *kafka.SubClient
```

### 9.5 配置

```yaml
kafkaPub:
  {topic}:
    brokers: [...]
    topic: {topic}
kafkaSub:
  {topic}:
    brokers: [...]
    topic: {topic}
    group: ...
```


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
