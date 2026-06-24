# 20 - 并发安全与运行时约束规范

> 规定 `docs/architecture/concurrency_safety.md` 的内容结构与维护规则。
>
> 本规范是 AIWeave 的 P0 规范之一，对应痛点 #5 并发安全隐式约束、#7 锁粒度与死锁、#9 资源泄漏。

---

## 1. 定位

`concurrency_safety.md` 是 **AI 在生成涉及共享状态 / goroutine / channel / 锁 / 资源句柄的代码之前必须读的文档**。它回答两个问题：

> 1. 这个工程里所有"跨 goroutine 共享的状态"是什么？每一处的保护机制是什么？
> 2. 哪些操作在哪些上下文里是被显式禁止的？

没有这篇文档：
- AI 容易把 `map` / `slice` 当作单 goroutine 数据结构使用，引入 race
- AI 容易在锁内做网络 IO，引入锁持有时间不可控
- AI 容易 `go func()` 后不绑定生命周期，引入 goroutine 泄漏

---

## 2. 顶层结构（强制）

`concurrency_safety.md` 必须按以下章节顺序组织：

- `## 1.` 并发模型总览（Goroutine 生命周期图 + 共享状态清单）
- `## 2.` 共享状态注册表（核心）
- `## 3.` 锁策略设计（粒度 / 顺序 / 超时）
- `## 4.` Channel 与 Goroutine 池（含 §4.3 协程池容量登记 / §4.4 内联兜底提交器纪律登记）
- `## 5.` 资源生命周期管理（连接池 / context cancel / fd / §5.4 异步任务 ctx 派生登记）
- `## 6.` 危险操作清单（AI 必读）
- `## 7.` 并发测试策略（race / 压测 / leak 检测）
- `## 8.` 维护流程（含 B1 反向同步规则）

> **完整章节骨架见** [`templates/docs/architecture/concurrency_safety.md`](../templates/docs/architecture/concurrency_safety.md)。

---

## 3. §1 并发模型总览

### 3.1 Goroutine 生命周期图

ASCII 图，分类标注**常驻 goroutine** vs **请求级 goroutine**：

```
[main] ──┬── [HTTP server 常驻]
         ├── [{flush-task} 常驻 cycle goroutine]
         ├── [{consumer-N} 常驻 sarama consumer]
         └── 请求 → [handler 请求级] ─→ [errgroup 子 goroutine 请求级]
```

### 3.2 共享状态清单（概述）

一句话列出"跨 goroutine 共享的全部变量/数据结构"的类别（详细在 §2）：

- 进程级单例（连接池 / 客户端 / 配置）
- 进程级缓存（本地 cache map）
- 跨请求队列（channel / batch）
- 计数器 / 限流器（atomic / RWMutex 保护的计数）

---

## 4. §2 共享状态注册表（核心 / 强制）

### 4.1 表格模板

```markdown
| 变量/数据结构 | 所在文件 | 访问方 | 保护机制 | 备注 |
|-------------|---------|--------|---------|------|
| `{ns}Cache map` | `helpers/{ns}_cache.go` | HTTP handler + `{flush-task}` | `sync.RWMutex` | 读多写少 |
| `{batch-ch} chan {Event}` | `service/{module}/dispatcher.go` | producer goroutine + consumer pool | 内部 channel buffer={N} | 满时策略：{丢弃尾部 / 阻塞 / 回压} |
| `{pool-name} *sql.DB` | `helpers/db.go` | 所有 goroutine | `database/sql` 内部 | MaxOpen={N}、MaxIdle={N} |
| `{hot-counter} [N]分片` | `helpers/{ns}_counter.go` | 所有 goroutine（高频写 + 周期读求和） | 分片计数 + 缓存行对齐（无锁） | 写≫读；分片按缓存行 padding 防 false-sharing |
```

### 4.2 必填字段

| 字段 | 说明 |
|------|------|
| 变量/数据结构 | Go 变量名 + 类型（含容量信息，如 `map[{K}]{V}`、`chan {T}`） |
| 所在文件 | 定义文件路径；如分散在多文件，列首要定义点 |
| 访问方 | 谁会并发读 / 谁会并发写；用"goroutine 角色"标识（如 "HTTP handler" / "{flush-task}"）|
| 保护机制 | `sync.Mutex` / `sync.RWMutex` / `sync.Map` / `atomic.*` / 分片计数（striped + 缓存行对齐）/ "内部 channel" / "无需保护（只读）" |
| 备注 | 访问比例（读多写少 / 写多读少）、TTL、容量限制等 |

### 4.3 反例（不应该的写法）

```markdown
| 全局 map | helpers/ | service / task | 加了锁 | — |
```

——含糊到无法定位代码位置、无法判定并发模式。

---

## 5. §3 锁策略设计

### 5.1 锁粒度决策表

```markdown
| 锁名 | 类型 | 保护范围 | 选择原因 |
|------|------|---------|---------|
| `{ns}CacheMu` | `sync.RWMutex` | `{ns}Cache map` 的读写 | 读多写少，RWMutex 提升并发读 |
| `{state-mu}` | `sync.Mutex` | `{state-counter}` 的递增 | 写主导，普通互斥即可 |
```

### 5.2 加锁顺序约束（多锁场景强制）

如工程内存在多把锁需要同时持有的场景，**必须**显式声明加锁顺序，防止死锁：

```markdown
**全局加锁顺序**（按以下顺序获取，反序释放）：
1. `{lock-A}`
2. `{lock-B}`
3. `{lock-C}`

违反该顺序的代码视为缺陷。
```

如全工程仅有单锁场景，本子节写"无多锁场景，N/A"。

### 5.3 锁超时策略

```markdown
| 锁 | 是否带超时 | 超时时长 | 超时后行为 |
|----|-----------|---------|-----------|
| `{lock-A}` | 否（持有时间 < 1ms） | — | — |
| `{lock-B}` | 是 | {N}ms | 返回 `Error{Reason}` + 告警 |
```

---

## 6. §4 Channel 与 Goroutine 池

### 6.1 Channel 清单

```markdown
| Channel | 类型 | buffer | 方向 | 满时策略 | 关闭条件 |
|---------|------|--------|------|---------|---------|
| `{batch-ch}` | `chan {Event}` | {N} | producer→consumer pool | 丢弃尾部 + 计数 | main 退出时 close |
| `{shutdown-ch}` | `chan struct{}` | 0 | main→所有 worker | — | main 退出时 close（仅一次）|
```

### 6.2 Goroutine 池配置

```markdown
| 池名 | 大小 | 排队策略 | panic recovery |
|------|------|---------|---------------|
| `{worker-pool}` | {N} | 任务 channel buffer={M}，满时丢弃 | `defer recover()` + 错误日志 |
```

### 6.3 §4.3 协程池容量登记（如有并行 fan-out）

凡用协程池做并行 fan-out 的工程，必须在 §4.3 登记一张表，记录每个池"选了什么"——用途 / 容量 / 失败语义 / 是否阻塞模式。这是**表示记录槽位**（登记选定值），不是定容方法论。

```markdown
| 池名 | 用途 | 容量 `{cap}` | 失败语义 | 阻塞模式 |
|------|------|-------------|---------|---------|
| `{query-pool}` | {高优先级查询并行 fan-out} | `{cap}` | 阻塞排队 | 是 |
| `{writeback-pool}` | {低优先级 best-effort 回写} | `{cap}` | 池满即丢 | 否 |
```

随表附结构性不变量（占位符化）：查询 / 编排类池**必须**阻塞模式（否则池满静默丢任务 + WaitGroup → 死锁）；best-effort 池小且"池满即丢"，按职责物理隔离 + 容量分级；容量值同步 `io_contract.md §6` 并与 `performance_contract.md §6.1` 对齐。

> **容量怎么定不在此复制**：IO-bound worker 上限远大于核数、`max(Little 估算, 下游可承载)` 再以内存可控封顶的定量方法，真相源在 [`design-spec/03 §3.2`](../design-spec/03_concurrency_design.md)，§4.3 槽位必须反向引用它，本规范只规定"该有此登记表 + 登记选定值"。

### 6.4 §4.4 内联兜底提交器纪律登记（如有协程池）

凡用协程池的工程，必须在 §4.4 登记"统一提交入口 + 失败则内联执行"的提交器，含 grep 锚。这是**表示记录槽位**（登记入口 + 锚），不是方法论。

```markdown
| 提交入口 | 所在文件 | 失败兜底 | grep 锚 |
|---------|---------|---------|---------|
| `{Submit-fn}(pool, task)` | `{path}` | Submit 失败 → 当前 goroutine 内联同步执行 `task()` | `R-IO-RAW-SUBMIT` |
```

随表附结构性纪律（占位符化）：**禁止**裸 `{pool}.Submit(` / 裸 `go func()` 无界扇出；一切池任务提交走唯一封装入口。grep 锚 `R-IO-RAW-SUBMIT` 与 [`docs-spec/25`](25_io_aggregation_spec.md) 一致。

> **为什么禁裸 Submit 不在此复制**：内联兜底"保证 task 永不丢、wg 永配平、根除 wg.Wait() 永久挂死"的推导，真相源在 [`design-spec/03 §3.3`](../design-spec/03_concurrency_design.md)，§4.4 槽位必须反向引用它。

---

## 7. §5 资源生命周期管理

### 7.1 连接池配置

```markdown
| 资源 | 配置项 | 数值 | 说明 |
|------|--------|------|------|
| MySQL `{db}` | `MaxOpenConns` | {N} | — |
| MySQL `{db}` | `MaxIdleConns` | {N} | — |
| MySQL `{db}` | `ConnMaxLifetime` | {N}s | 防止单连接长期持有 |
| Redis `{cluster}` | `PoolSize` | {N} | — |
| Redis `{cluster}` | `IdleTimeout` | {N}s | — |
| HTTP Client `{client}` | `Timeout` | {N}s | 强制超时，禁止 0 |
```

### 7.2 Goroutine 泄漏防护

```markdown
**强制规则**：所有 `go func()` 必须满足以下之一：

- 显式接受 `ctx context.Context`，并在循环或 select 中 `case <-ctx.Done():` 退出
- 由 `errgroup.WithContext` / `sync.WaitGroup` 跟踪
- 短周期任务（< 1s），调用栈可在 `defer recover()` 内自然结束

`go func()` 之后**不允许**只靠"goroutine 自己跑完"而无任何退出信号——这是漏检兜底。
```

### 7.3 文件描述符管理（如有）

```markdown
| 句柄类型 | 关闭模式 | 兜底机制 |
|---------|---------|---------|
| `os.File` | `defer f.Close()` | 上游使用 `defer` 模式 |
| `http.Response.Body` | `defer resp.Body.Close()` | 即便 err != nil 也要 close（如 body != nil） |
```

### 7.4 ctx 透传与异步任务派生（脱离请求生命周期 / 强制）

与 §7.2「goroutine 生命周期终止（`ctx.Done()` 退出）」是**两个正交维度**：§7.2 管"监听退出信号怎么退出"，本节管"异步任务别被请求取消误杀"——一个是监听退出，一个是派生脱离，不可互相替代。

> **决策真相源**：取消/超时传播 → ctx 透传；异步任务用派生 ctx 脱离请求生命周期，决策方法论在 [`design-spec/03 §3.1`](../design-spec/03_concurrency_design.md)（并发原语选型决策树）。本节只规定"该有此登记表 + 登记选定项"。

两类 ctx 用法必须显式区分并登记：

```markdown
| ctx 类别 | 适用任务 | 取消传播 | 派生方式 |
|---------|---------|---------|---------|
| 透传请求 ctx | 取消/超时要传播到下游的请求级子任务 | 跟随请求一起取消 | 直接透传请求 `ctx` |
| 派生 ctx（脱离） | 异步任务（缓存回写 / fire-and-forget / 脱离请求的后台写） | 不被请求取消波及 | `context.WithoutCancel` / 复制出新 root + 自带超时 |
```

**纪律（结构性）**：

- 子任务必须**透传请求 `ctx`**——取消/超时能传播下去；
- 但**异步任务**（缓存回写、fire-and-forget、脱离请求的后台写）必须用**派生 ctx**脱离请求生命周期；否则请求一结束，异步写回即被 `cancel` 杀掉、写了一半。
- 派生 ctx 与请求 ctx 脱钩后**仍必须自带超时**（禁止无超时悬挂），并仍服从 §7.2 的退出路径要求。
- 本纪律与 [`docs-spec/25 §4`](25_io_aggregation_spec.md)「聚合器异步写回需 detach（复制 ctx）脱离请求生命周期」同源一致——IO 聚合的异步写回是本纪律的一个落地场景。

> **登记哪些异步任务用派生 ctx**：每个走 detach 的异步任务（如缓存回写）应在 `concurrency_safety.md §5.4` 的「异步任务 ctx 派生登记」槽位记一行；本规范只规定该登记表的存在。

---

## 8. §6 危险操作清单（AI 必读）

这部分与 `CLAUDE.md 危险模式清单`（详见 [`templates/CLAUDE.md`](../templates/CLAUDE.md)）+ `ai_dev_guide.md` grep 锚 rule-id 索引联动。

```markdown
| 危险操作 | 风险 | 正确做法 |
|---------|------|---------|
| 锁内做网络 IO（HTTP / DB / Redis） | 锁持有时间不可控，引发拥塞 | 先获取数据再加锁，或缩短锁内逻辑 |
| 锁内打日志（含文件写盘） | 同上 + 写盘抖动放大锁竞争 | 释放锁后再打日志 |
| `map` / `slice` 并发写 | race condition | `sync.Map` 或外层 `RWMutex` 保护 |
| 关闭已关闭的 channel | panic | 单一所有者 close，其他 goroutine 仅读 |
| 向已关闭的 channel 发送 | panic | 用 select + done channel 兜底 |
| `defer` 在 for 循环内 | 资源延迟释放至函数返回，可能 OOM | 抽子函数或显式 Close |
| 忽略 `ctx.Done()` 的常驻 goroutine | 退出信号无法传递，goroutine 泄漏 | 必须 select case `<-ctx.Done()` |
| 超高频计数用单点 Mutex / 单 atomic（缓存行总线风暴） | 高争用下计数吞吐塌陷 | 分片计数（striped，按 P/CPU 散列）读时求和 |
| 分片 / 原子计数器未按缓存行对齐（false-sharing） | 相邻计数器共享 64B 缓存行，分片白拆 | 按缓存行 padding / 独占 cache line |
```

> **机械审计**：上述每条规则对应一个 rule-id（如 `R-CONC-LOCK-IO`），grep 锚维护在 `docs/architecture/ai_dev_guide.md`。grep 命中为**信号级**（🟡 待复核），最终判定权在人工 reviewer。

---

## 9. §7 并发测试策略

### 9.1 race detector

```markdown
- 所有包必须能通过 `go test -race ./...`
- CI 强制开启 -race
- 命中 race → 视为严重缺陷，必须修复才能合并
```

### 9.2 并发压测

```markdown
- 涉及共享状态的 service 方法应有基准测试 `Benchmark{Method}_Parallel`
- 使用 `b.RunParallel` 模拟并发
- 关注：吞吐 / 锁等待 / pprof block profile
```

### 9.3 goroutine leak 检测

```markdown
- 引入 `go.uber.org/goleak`
- 关键测试在 `TestMain` 中加 `goleak.VerifyTestMain(m)`
- 常驻 goroutine 需在 `goleak.IgnoreCurrent()` 中预先登记
```

---

## 10. §8 维护流程（含 B1 反向同步规则）

### 10.1 B1 反向同步规则（强制）

> 镜像 [`PRINCIPLES.md §1.5`](../PRINCIPLES.md#15-两大使用模式) 的"建设规则必须伴随维护规则"要求。每条代码迹象对应一条文档同步动作：

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| 新增 `var x sync.Mutex` / `sync.RWMutex` / `sync.Map` | §2 共享状态注册表新增一行（变量 / 文件 / 访问方 / 保护机制） |
| 新增 `go func()` / `errgroup.Go` | §1.1 Goroutine 生命周期图追加节点 + §5.2 Goroutine 泄漏防护登记 cancel 路径 |
| 新增异步任务（缓存回写 / fire-and-forget）/ 出现 `context.WithoutCancel` / 复制新 root ctx | §5.4 异步任务 ctx 派生登记新增一行（透传 vs 派生、派生方式、自带超时）；与 docs-spec/25 §4 detach 同源 |
| 新增 `chan T` / `make(chan T, N)` | §4.1 Channel 清单新增一行（方向 / buffer / 满时策略 / 关闭条件） |
| 新增协程池 / 调整池容量或失败语义 | §4.3 协程池容量登记新增 / 更新一行（容量值同步 io_contract §6 / performance_contract §6.1） |
| 新增 / 修改统一提交入口（含内联兜底） / 出现裸 `Pool.Submit(` | §4.4 内联兜底提交器纪律登记同步（grep 锚 `R-IO-RAW-SUBMIT`） |
| 修改锁字段（新增 / 移除 / 重命名） | §3.1 锁粒度决策表同步；如形成多锁路径 → §3.2 加锁顺序约束更新 |
| 新增 `defer xxx.Close()` 资源型 | §5.1 连接池或 §5.3 文件描述符登记 |
| 引入 `goleak.IgnoreCurrent` 新登记项 | §9.3 常驻 goroutine 白名单同步 |

### 10.2 维护触发

| 触发 | 动作 |
|------|------|
| 新增共享状态 | §2 必须新增一行 + 同步 BUILD_STATUS 约束清单状态轨道 |
| 修改锁策略 | §3 + 同步影响的方法在 service docs 内伪码加 `[LOCK-ACQUIRE/RELEASE]` 标记 |
| 引入新依赖（DB / Cache / MQ） | §5.1 连接池配置追加 |
| `goleak` 报告漏报 | §9.3 调整 IgnoreCurrent 清单 |

---

## 11. 与其他文档的关系

| 内容 | concurrency_safety.md | service docs | helpers_api.md |
|------|----------------------|--------------|----------------|
| 共享状态注册表 | ✅ 真相源 | 引用（在方法 §4.N.4 数据访问明细中标注"读 `{ns}Cache`"） | 不重复，仅在 §1 全局变量表标"详见 concurrency_safety §2" |
| 锁粒度决策 | ✅ 真相源 | 方法伪码用 `[LOCK-ACQUIRE: {lock-name}]` 标记引用 | 不写 |
| 危险操作清单 | ✅ 真相源 | 不重复 | 不重复 |
| 测试策略 | ✅ 并发部分 | 引用 testing_design.md 总体框架 | 不写 |

---

## 12. 与 Skill 的联动

- **`/new-service`** / **`/new-controller`** 生成代码前：读 §2 / §6，若新增的变量属于共享类，必须更新 §2
- **`/concurrency-review`**：扫描新增代码对照 §2 / §6 + grep 锚
- **`/doc-sync-check`**：扫描代码中的 `sync.*` / `chan` / `go func()` 是否在 §2 / §4.1 已登记

---

## 13. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§12）已用占位符表达；落地工程时按业务语义替换占位符，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 13.1 §2 共享状态注册表（示例）

| 变量/数据结构 | 所在文件 | 访问方 | 保护机制 | 备注 |
|-------------|---------|--------|---------|------|
| `userCache map[int64]*User` | `helpers/user_cache.go` | HTTP handler + flush task | `sync.RWMutex` | 读多写少，TTL 5min |
| `orderEventCh chan OrderEvent` | `service/order/dispatcher.go` | producer goroutine + consumer pool | 内部 channel buffer=1024 | 满时丢弃尾部 |
| `coreDB *sql.DB` | `helpers/db.go` | 所有 goroutine | `database/sql` 内部 | MaxOpen=100 |

### 13.2 §3.1 锁粒度决策表（示例）

| 锁名 | 类型 | 保护范围 | 选择原因 |
|------|------|---------|---------|
| `userCacheMu` | `sync.RWMutex` | `userCache map` 的读写 | 读多写少，RWMutex 提升并发读 |
| `balanceMu` | `sync.Mutex` | `balanceCounter` 的递增 | 写主导，普通互斥即可 |

### 13.3 §6 危险操作（示例）

错误代码：

```go
mu.Lock()
defer mu.Unlock()
resp, _ := http.Get(url) // ❌ 锁内做网络 IO
```

正确代码：

```go
resp, _ := http.Get(url)
mu.Lock()
defer mu.Unlock()
cache[key] = resp.Body
```


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
