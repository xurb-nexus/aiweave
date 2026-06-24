# 并发安全与运行时约束

> 版本：1.0 | 日期：{YYYY-MM-DD}
>
> 规范来源：[`aiweave/docs-spec/20_concurrency_safety_spec.md`](../../../docs-spec/20_concurrency_safety_spec.md)
>
> 本文档是 **AI 在生成涉及共享状态 / goroutine / channel / 锁 / 资源句柄的代码之前必须读的文档**。

---

## 1. 并发模型总览

### 1.1 Goroutine 生命周期图

```
[main] ──┬── [HTTP server 常驻]
         ├── [{flush-task} 常驻 cycle goroutine]
         ├── [{consumer-N} 常驻 sarama consumer]
         └── 请求 → [handler 请求级] ─→ [errgroup 子 goroutine 请求级]
```

> 落地工程时按实际部署形态填充。**常驻** = 进程级 = 退出条件由 main / signal 触发；**请求级** = 随请求结束。

### 1.2 共享状态清单（概述）

跨 goroutine 共享的全部变量/数据结构按类别列出，详细在 §2：

- 进程级单例（连接池 / 客户端 / 配置）
- 进程级缓存（本地 cache map）
- 跨请求队列（channel / batch）
- 计数器 / 限流器（atomic / RWMutex 保护的计数）

---

## 2. 共享状态注册表（核心）

| 变量/数据结构 | 所在文件 | 访问方 | 保护机制 | 备注 |
|-------------|---------|--------|---------|------|
| `{ns}Cache map` | `helpers/{ns}_cache.go` | HTTP handler + `{flush-task}` | `sync.RWMutex` | 读多写少 |
| `{batch-ch} chan {Event}` | `service/{module}/dispatcher.go` | producer goroutine + consumer pool | 内部 channel buffer={N} | 满时丢弃尾部 |
| `{pool-name} *sql.DB` | `helpers/db.go` | 所有 goroutine | `database/sql` 内部 | MaxOpen={N}、MaxIdle={N} |
| `{hot-counter} [N]分片` | `helpers/{ns}_counter.go` | 所有 goroutine（高频写 + 周期读求和） | 分片计数 + 缓存行对齐（无锁） | 写≫读；分片按缓存行 padding 防 false-sharing |

> **维护规则**：每新增一个跨 goroutine 共享的变量 / channel，本表必须新增一行；删除时同步移除。

---

## 3. 锁策略设计

### 3.1 锁粒度决策表

| 锁名 | 类型 | 保护范围 | 选择原因 |
|------|------|---------|---------|
| `{ns}CacheMu` | `sync.RWMutex` | `{ns}Cache map` 的读写 | 读多写少 |
| `{state-mu}` | `sync.Mutex` | `{state-counter}` 的递增 | 写主导 |

### 3.2 加锁顺序约束

- 如本工程存在多锁同时持有场景，按以下顺序获取，反序释放：
  1. `{lock-A}`
  2. `{lock-B}`
  3. `{lock-C}`
- 如全工程仅有单锁场景：N/A

### 3.3 锁超时策略

| 锁 | 是否带超时 | 超时时长 | 超时后行为 |
|----|-----------|---------|-----------|
| `{lock-A}` | 否（持有时间 < 1ms） | — | — |
| `{lock-B}` | 是 | {N}ms | 返回 `Error{Reason}` + 告警 |

---

## 4. Channel 与 Goroutine 池

### 4.1 Channel 清单

| Channel | 类型 | buffer | 方向 | 满时策略 | 关闭条件 |
|---------|------|--------|------|---------|---------|
| `{batch-ch}` | `chan {Event}` | {N} | producer→consumer pool | 丢弃尾部 + 计数 | main 退出时 close |
| `{shutdown-ch}` | `chan struct{}` | 0 | main→所有 worker | — | main 退出时 close（仅一次）|

### 4.2 Goroutine 池配置

| 池名 | 大小 | 排队策略 | panic recovery |
|------|------|---------|---------------|
| `{worker-pool}` | {N} | 任务 channel buffer={M}，满时丢弃 | `defer recover()` + 错误日志 |

### 4.3 协程池容量登记

> 一张表登记每个池的选定容量与失败语义。**容量怎么定**（IO-bound 的 worker 上限远大于核数；`max(Little 估算, 下游可承载)` 再以内存可控封顶）见 [design-spec/03 §3.2 协程池容量定量方法](../../../design-spec/03_concurrency_design.md)；本处只登记选定值。

| 池名 | 用途 | 容量 `{cap}` | 失败语义 | 阻塞模式 |
|------|------|-------------|---------|---------|
| `{query-pool}` | {高优先级查询并行 fan-out} | `{cap}` | 阻塞排队 | 是 |
| `{orchestrate-pool}` | {跨模块编排 fan-out} | `{cap}` | 阻塞排队 | 是 |
| `{writeback-pool}` | {低优先级 best-effort 回写} | `{cap}` | 池满即丢 | 否 |

**不变量（结构性）**：
- 查询 / 编排类池**必须**阻塞模式（禁非阻塞 / 丢弃）；否则池满静默丢任务 + `WaitGroup` → 死锁。
- best-effort 池要小且"池满即丢"，绝不挤占查询能力（按职责物理隔离 + 容量分级）。
- 容量值同时登记到 `io_contract.md §6` 并与 `performance_contract.md §6.1` 对齐。

### 4.4 内联兜底提交器纪律登记

> 登记本工程"统一提交入口 + 失败则内联执行"的提交器。**为什么禁裸 Submit**（保证 task 永不丢、`wg` 永配平，根除 `wg.Wait()` 永久挂死）见 [design-spec/03 §3.3 协程池工程不变量](../../../design-spec/03_concurrency_design.md)。

| 提交入口 | 所在文件 | 失败兜底 | grep 锚 |
|---------|---------|---------|---------|
| `{Submit-fn}(pool, task)` | `{path}` | Submit 失败 → 当前 goroutine 内联同步执行 `task()` | `R-IO-RAW-SUBMIT` |

**纪律（结构性）**：
- **禁止**裸 `{pool}.Submit(` / 裸 `go func()` 无界扇出；一切池任务提交走唯一封装入口。
- 审计锚 `R-IO-RAW-SUBMIT` 与 [`docs-spec/25`](../../../docs-spec/25_io_aggregation_spec.md) 一致；grep 命中为信号级（🟡 待复核）。

---

## 5. 资源生命周期管理

### 5.1 连接池配置

| 资源 | 配置项 | 数值 | 说明 |
|------|--------|------|------|
| MySQL `{db}` | `MaxOpenConns` | {N} | — |
| MySQL `{db}` | `MaxIdleConns` | {N} | — |
| MySQL `{db}` | `ConnMaxLifetime` | {N}s | 防止单连接长期持有 |
| Redis `{cluster}` | `PoolSize` | {N} | — |
| Redis `{cluster}` | `IdleTimeout` | {N}s | — |
| HTTP Client `{client}` | `Timeout` | {N}s | 强制超时，禁止 0 |

### 5.2 Goroutine 泄漏防护

所有 `go func()` 必须满足以下之一：

- 显式接受 `ctx context.Context`，并在循环或 select 中 `case <-ctx.Done():` 退出
- 由 `errgroup.WithContext` / `sync.WaitGroup` 跟踪
- 短周期任务（< 1s），调用栈可在 `defer recover()` 内自然结束

### 5.3 文件描述符管理

| 句柄类型 | 关闭模式 | 兜底机制 |
|---------|---------|---------|
| `os.File` | `defer f.Close()` | 上游使用 `defer` 模式 |
| `http.Response.Body` | `defer resp.Body.Close()` | 即便 err != nil 也要 close（如 body != nil） |

### 5.4 异步任务 ctx 派生登记

> 登记本工程哪些**异步任务**（缓存回写 / fire-and-forget / 脱离请求的后台写）用**派生 ctx** 脱离请求生命周期，与哪些子任务**透传请求 ctx**。**为什么这么分**（取消/超时传播 → ctx 透传；异步任务派生脱离，否则请求一结束异步写回被 `cancel` 杀掉、写了一半）见 [design-spec/03 §3.1](../../../design-spec/03_concurrency_design.md)；与 [`docs-spec/25 §4`](../../../docs-spec/25_io_aggregation_spec.md) 聚合器异步写回 detach 同源。本处只登记选定项。

| 异步任务 / 子任务 | ctx 类别 | 取消传播 | 派生方式 |
|------------------|---------|---------|---------|
| `{request-subtask}` | 透传请求 ctx | 跟随请求取消 | 直接透传 `ctx` |
| `{cache-writeback}` | 派生 ctx（脱离） | 不被请求取消波及 | `context.WithoutCancel` / 复制新 root + 自带超时 |

**纪律（结构性）**：

- 子任务必须**透传请求 `ctx`**（取消/超时能传播到下游）；
- **异步任务**（缓存回写 / fire-and-forget / 脱离请求的后台写）必须用**派生 ctx** 脱离请求生命周期，否则请求结束即被 `cancel` 杀掉、写了一半。
- 与 §5.2「`ctx.Done()` 退出」正交：一个管"怎么退出"，一个管"别被请求取消误杀"，不可互相替代。
- 派生 ctx 仍须**自带超时**（禁无超时悬挂），并仍服从 §5.2 退出路径。

---

## 6. 危险操作清单（AI 必读）

> 与 `CLAUDE.md` 危险模式清单 + `docs/architecture/ai_dev_guide.md` grep 锚 rule-id 索引联动。

| 危险操作 | 风险 | 正确做法 | Rule-id |
|---------|------|---------|---------|
| 锁内做网络 IO（HTTP / DB / Redis） | 锁持有时间不可控 | 先获取数据再加锁 | `R-CONC-LOCK-IO` |
| 锁内打日志（含文件写盘） | 同上 + 写盘抖动放大锁竞争 | 释放锁后再打日志 | `R-CONC-LOCK-LOG` |
| `map` / `slice` 并发写 | race condition | `sync.Map` 或 `RWMutex` | `R-CONC-MAP-RACE` |
| 关闭已关闭的 channel | panic | 单一所有者 close | `R-CONC-DOUBLE-CLOSE` |
| 向已关闭的 channel 发送 | panic | select + done channel | `R-CONC-SEND-CLOSED` |
| `defer` 在 for 循环内 | 资源延迟释放，可能 OOM | 抽子函数或显式 Close | `R-RESOURCE-DEFER-LOOP` |
| 忽略 `ctx.Done()` 的常驻 goroutine | goroutine 泄漏 | select case `<-ctx.Done()` | `R-CONC-GOROUTINE-LEAK` |
| 超高频计数用单点 Mutex / 单 atomic（缓存行总线风暴） | 高争用下计数吞吐塌陷 | 分片计数（striped，按 P/CPU 散列）读时求和 | `R-CONC-COUNTER-CONTENTION` |
| 分片 / 原子计数器未按缓存行对齐（false-sharing） | 相邻计数器共享 64B 缓存行，分片白拆 | 按缓存行 padding / 独占 cache line | `R-CONC-FALSE-SHARING` |

> grep 命中为**信号级**（🟡 待复核），最终判定权在人工 reviewer。

---

## 7. 并发测试策略

### 7.1 race detector

- 所有包必须能通过 `go test -race ./...`
- CI 强制开启 -race
- 命中 race → 视为严重缺陷，必须修复才能合并

### 7.2 并发压测

- 涉及共享状态的 service 方法应有 `Benchmark{Method}_Parallel`
- 使用 `b.RunParallel` 模拟并发
- 关注：吞吐 / 锁等待 / pprof block profile

### 7.3 goroutine leak 检测

- 引入 `go.uber.org/goleak`
- 关键测试在 `TestMain` 中加 `goleak.VerifyTestMain(m)`
- 常驻 goroutine 需在 `goleak.IgnoreCurrent()` 中预先登记

---

## 8. 维护流程

### 8.1 B1 反向同步规则（强制）

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| 新增 `sync.Mutex` / `sync.RWMutex` / `sync.Map` | §2 新增一行 |
| 新增 `go func()` / `errgroup.Go` | §1.1 追加节点 + §5.2 登记 cancel 路径 |
| 新增异步任务（缓存回写 / fire-and-forget）/ 出现 `context.WithoutCancel` / 复制新 root ctx | §5.4 异步任务 ctx 派生登记新增一行（与 docs-spec/25 §4 detach 同源） |
| 新增 `chan T` / `make(chan T, N)` | §4.1 新增一行 |
| 新增协程池 / 调整池容量或失败语义 | §4.3 协程池容量登记新增 / 更新一行（容量值同步 io_contract §6 / performance_contract §6.1） |
| 新增 / 修改统一提交入口（含内联兜底） | §4.4 内联兜底提交器纪律登记同步（grep 锚 `R-IO-RAW-SUBMIT`） |
| 修改锁字段（新增 / 移除 / 重命名） | §3.1 同步；多锁路径 → §3.2 更新 |
| 新增 `defer xxx.Close()` 资源型 | §5.1 / §5.3 登记 |

### 8.2 与 BUILD_STATUS §11 约束清单状态轨道的关系

每条约束条目（§2 一行 / §3.1 一行）对应 BUILD_STATUS.md §11 的"已设计 / 已启用"计数。新增约束 → BUILD_STATUS §11 对应类目"已设计条目数"+1。


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
