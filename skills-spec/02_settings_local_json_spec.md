# 02 - settings.local.json 规范

> 规定 `.claude/settings.local.json` 的内容结构与推荐配置。

---

## 1. 定位

`.claude/settings.json`（共享 / 入 git）与 `.claude/settings.local.json`（本机 / 不入 git）是 Claude Code 的配置：

- **permissions** —— 自动放行的工具调用，避免每次都要确认
- **hooks** —— 在某些事件触发时自动执行的脚本（如 PostToolUse / Stop / PreToolUse）
- **environment** —— 环境变量
- **disabled-skills** —— 禁用的 Skill

> ⚠️ `*.local.json` 属于本机/项目设置，**不入版本控制**。需团队共享的设置放 `.claude/settings.json`（去掉 `.local`）。

> **🔒 Hooks 是规范要求（强制）**：本规范把 Hooks 机制定位为 **L0 自动化防御层**（不依赖 AI 自觉，由 Claude Code 机械执行）。§4 定义的**两类强制 Hook**（IO 铁律检查 + 代码↔文档双向同步）必须配置在**团队共享** `settings.json`（入 git）中，使全团队一致生效。原理见 [`PRINCIPLES.md §14`](../PRINCIPLES.md#14-hooks-机制l0-自动化防御--强制)。

---

## 2. 顶层结构

```json
{
  "permissions": { ... },
  "hooks": [ ... ],
  "env": { ... },
  "disabledSkills": [ ... ]
}
```

---

## 3. permissions 配置（推荐）

### 3.1 自动放行 Bash 命令

```json
{
  "permissions": {
    "allow": [
      "Bash(go build *)",
      "Bash(go vet *)",
      "Bash(go fmt *)",
      "Bash(go test *)",
      "Bash(go run *)",
      "Bash(git status)",
      "Bash(git log *)",
      "Bash(git diff *)",
      "Bash(git show *)",
      "Bash(git branch)",
      "Bash(ls *)",
      "Bash(find *)",
      "Bash(grep *)",
      "Bash(rg *)",
      "Bash(cat *)",
      "Bash(head *)",
      "Bash(tail *)",
      "Bash(wc *)",
      "Bash(mkdir *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(git reset --hard *)"
    ]
  }
}
```

### 3.2 自动放行其他工具

```json
{
  "permissions": {
    "allow": [
      "Read(*)",
      "Write(*)",
      "Edit(*)",
      "Glob(*)",
      "Grep(*)"
    ]
  }
}
```

读写文件已是基础操作，建议全部放行。删除/破坏性命令保持需确认。

---

## 4. hooks 机制（强制 / L0 自动化防御层）

> Hooks 由 Claude Code 在工具事件触发时**机械执行**，不依赖 AI 自觉，构成 L1-L5 之前的 **L0 层**。
> 下面**两类 Hook 族是规范要求**，必须放在团队共享 `settings.json`（入 git）；§4.3 的本机便利 Hook 可放 `settings.local.json`。

### 4.0 两类强制 Hook 族总览

| Hook 族 | 事件 | 职责 | 关联规范 |
| --- | --- | --- | --- |
| **A. IO 铁律检查** | PostToolUse（编辑/写 `.go`） | 机械跑 `R-IO-*` grep 锚，命中 N+1 / 串行编排 → 告警；可升级为 PreToolUse 阻断 | [`docs-spec/25`](../docs-spec/25_io_aggregation_spec.md) |
| **B. 代码↔文档双向同步** | PostToolUse（编辑 `.go`）+ Stop | 编辑后按范围判定表提醒同步 md；Stop 时校验"文档同步：..."声明存在 | [`PRINCIPLES.md §2`](../PRINCIPLES.md) + [`19`](../docs-spec/19_incremental_sync_spec.md) |

> 脚本应放工程 `scripts/hooks/` 下，纯标准工具（grep / shell），不引入业务耦合。退出码约定：`0` 放行；非 `0` 在 PreToolUse 下阻断、在 PostToolUse/Stop 下告警。

> 此外，[`design-spec/07 §6.2`](../design-spec/07_tail_latency_design.md)（机械闸门下沉 L0）把三条可机械判定的尾延迟 / 运行时违规也下沉为编辑 `.go` 时的 grep 信号——`R-TAIL-RETRY-NOJITTER`（重试退避无 jitter）、`R-TAIL-HEDGE-UNSAFE`（对冲 / 重试包裹非幂等调用）、`R-DEPLOY-GOMAXPROCS`（部署无 GOMAXPROCS 对齐）。它们与族 A 同样是"编辑后机械跑 grep 锚"的形态，挂在族 A 的 PostToolUse 上作为附加 L0 信号（见 §4.1 末），不另立强制族；"预算是否合理 / 优先级是否分层"等语义类违规走选型审查（软提醒）。
>
> [`design-spec/08-12 §6.2`](../design-spec/08_messaging_design.md) 同理各自下沉若干可机械判定的违规——`R-MQ-DUAL-WRITE`（写库 + 发 MQ 裸双写）、`R-SCHED-LOCK-NO-TTL`（分布式锁无 TTL）、`R-SCHED-MULTI-REPLICA`（后台协程含全局副作用无认领 / 锁）、`R-EVO-BACKFILL-BIGTXN`（全表回填大事务）、`R-API-BATCH-NOLIMIT`（批量端点无上限）等。同挂族 A PostToolUse 作附加 L0 信号（见 §4.1 末）；"投递语义选对没 / 单例判对没 / 是否该走 expand-contract"等语义类违规走选型审查（软提醒）。

### 4.1 Hook 族 A — IO 铁律检查（强制）

PostToolUse 在每次编辑 / 写入 `.go` 后，对改动文件机械跑 IO 铁律 grep 锚（与 [`docs-spec/25 §2`](../docs-spec/25_io_aggregation_spec.md) / `/io-review` 同源）：

```json
{
  "hooks": [
    {
      "name": "io-iron-law-check-on-go-edit",
      "event": "PostToolUse",
      "matcher": { "tool": "Edit|Write", "filePathPattern": "*.go" },
      "command": "scripts/hooks/io_iron_law_check.sh \"$CLAUDE_FILE_PATH\""
    }
  ]
}
```

`scripts/hooks/io_iron_law_check.sh` 行为约定（抽象，不绑定业务）：

```
1. 对传入的 .go 文件 grep R-IO-N-PLUS-1 模式（循环体内单条 .Find/.Get/.HGet/.Call 等，key 来自循环变量）
2. grep R-IO-LOOP-WRITE / R-IO-LOOP-RPC 模式
3. 命中且无行末 // aiweave:allow=R-IO-* 注解、且不在 io_contract.md §2 豁免登记内
   → 打印 🟡 告警（文件:行 + 铁律编号 + "建议走聚合器，详见 io_contract.md §3"）
4. 退出码非 0 仅在升级为 PreToolUse 阻断模式时使用；PostToolUse 模式恒返回 0（仅告警）
```

> 升级阻断：对高纪律团队，可把铁律一（N+1）命中改为 PreToolUse + `exit 1` 直接拦截写入，强制先改批量。

附加 L0 grep 信号（下沉自 [`design-spec/07 §6.2`](../design-spec/07_tail_latency_design.md)，同样跑在编辑 `.go` 的 PostToolUse 上，可与 `io_iron_law_check.sh` 合并或并列一个 `tail_runtime_check.sh`）：

```
5. grep R-TAIL-RETRY-NOJITTER 模式（重试 backoff 拼装缺随机项：sleep = base × 2^attempt 无 + rand/jitter）
6. grep R-TAIL-HEDGE-UNSAFE 模式（写类方法名 —— Create/Update/Delete/Incr 等 —— 进对冲 / 重试入口，污点非幂等）
7. grep R-DEPLOY-GOMAXPROCS 模式（main / 编排清单缺 automaxprocs / GOMAXPROCS 对齐 cgroup CPU 配额）
8. 命中且无行末 // aiweave:allow=R-TAIL-* / =R-DEPLOY-GOMAXPROCS 注解 → 打印 🟡 告警（文件:行 + rule-id + "决策见 design-spec/07 §6"）；其中 R-TAIL-HEDGE-UNSAFE（非幂等上对冲）属硬违规，高纪律团队可升级为 PreToolUse + exit 1 拦截
```

附加 L0 grep 信号（下沉自 [`design-spec/08·10·11·09 §6.2`](../design-spec/08_messaging_design.md)，同挂编辑 `.go` 的 PostToolUse，可并入 `tail_runtime_check.sh` 或并列 `lens_gate_check.sh`）：

```
9.  grep R-MQ-DUAL-WRITE 模式（同函数体内 DB 写 + mq.Send/Produce 相邻、且不在 outbox 事务内 → 污点）
10. grep R-SCHED-LOCK-NO-TTL 模式（分布式锁获取 SETNX / Lock 缺 TTL / EX / 续约参数）
11. grep R-EVO-BACKFILL-BIGTXN 模式（UPDATE/DELETE 无 WHERE 限定 / 无分批游标 → 全表大事务）
12. grep R-API-BATCH-NOLIMIT 模式（批量入参 slice 直接进查询，缺长度上限校验）
13. 命中且无行末 // aiweave:allow=R-MQ-* / =R-SCHED-* / =R-EVO-* / =R-API-* 注解 → 打印 🟡 告警（文件:行 + rule-id + "决策见 design-spec/08·10·11·09 §6"）；其中 R-MQ-DUAL-WRITE（裸双写丢事件）、R-SCHED-NO-FENCING（脑裂双主写）属硬违规，高纪律团队可升级为 PreToolUse + exit 1 拦截
```

### 4.2 Hook 族 B — 代码↔文档双向同步（强制）

#### 4.2.1 PostToolUse：编辑 `.go` 后按范围判定表提醒

```json
{
  "hooks": [
    {
      "name": "remind-doc-sync-on-go-edit",
      "event": "PostToolUse",
      "matcher": { "tool": "Edit|Write", "filePathPattern": "*.go" },
      "command": "scripts/hooks/scope_hint.sh \"$CLAUDE_FILE_PATH\""
    }
  ]
}
```

`scripts/hooks/scope_hint.sh`：按 CLAUDE.md 范围判定表，把改动文件路径映射到应同步的 md，打印提醒（如改 `service/` → 提醒同步 `{module}_service.md`）。

#### 4.2.2 Stop：会话结束校验"文档同步声明"

```json
{
  "hooks": [
    {
      "name": "enforce-doc-sync-statement",
      "event": "Stop",
      "command": "scripts/hooks/check_doc_sync_statement.sh"
    }
  ]
}
```

`scripts/hooks/check_doc_sync_statement.sh`：检查本回合是否改过 `.go`/`docs/`；若改过但 AI 最近一条回复末尾无「文档同步：...」声明 → 打印 🔴 告警（提醒补声明再交付）。

### 4.3 本机便利 Hook（可选，放 settings.local.json）

防止误删 docs/：

```json
{
  "hooks": [
    {
      "name": "prevent-docs-deletion",
      "event": "PreToolUse",
      "matcher": { "tool": "Bash", "commandPattern": "rm.*docs/" },
      "command": "exit 1",
      "blockMessage": "禁止用 rm 直接删除 docs/ 下文件，请用 Edit 工具或显式确认"
    }
  ]
}
```

> 凭据安全可选 Hook：PreToolUse 拦截把含明文密码 / 密钥的配置文件 `git add` 进库（对应 [`docs-spec/26 §7.4`](../docs-spec/26_config_center_spec.md) `R-CONF-SECRET-COMMIT`），命令模式匹配 `git add` + 凭据文件名。

---

## 5. env 配置

```json
{
  "env": {
    "GO111MODULE": "on",
    "GOPROXY": "https://goproxy.cn,direct",
    "TLOG_OUTPUT_PATH": "log",
    "PROJECT_ENV": "dev"
  }
}
```

---

## 6. disabledSkills 配置

如果某些 Skill 在当前项目中不适用（如项目未启用 Kafka），可禁用：

```json
{
  "disabledSkills": [
    "new-mq-consumer"
  ]
}
```

被禁用的 Skill 在 AI 调用时立即报错，避免误用。

---

## 7. 完整示例

```json
{
  "permissions": {
    "allow": [
      "Bash(go build *)",
      "Bash(go vet *)",
      "Bash(go fmt *)",
      "Bash(go test *)",
      "Bash(git status)",
      "Bash(git log *)",
      "Bash(git diff *)",
      "Bash(ls *)",
      "Bash(find *)",
      "Bash(grep *)",
      "Bash(rg *)",
      "Bash(mkdir *)",
      "Read(*)",
      "Write(*)",
      "Edit(*)",
      "Glob(*)",
      "Grep(*)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(git reset --hard *)"
    ]
  },
  "env": {
    "GO111MODULE": "on",
    "GOPROXY": "https://goproxy.cn,direct"
  },
  "disabledSkills": []
}
```

---

## 8. 与 settings.json（共享） vs settings.local.json（本机）的区别

| 文件 | 用途 | 入 git |
|------|------|--------|
| `.claude/settings.json` | 团队共享配置：**§4.1 / §4.2 两类强制 Hook 族**（IO 铁律 + 双向同步）+ 通用 permissions | ✅ |
| `.claude/settings.local.json` | 本机偏好（个人 env / §4.3 调试 hooks） | ❌（应在 .gitignore） |

> **强制**：§4 的两类 Hook 族（A IO 铁律 / B 双向同步）属于团队级强制规则，**必须放共享 `settings.json` 并入 git**，确保全员一致生效——这是它们作为 L0 防御层的前提。§4.3 本机便利 Hook 放 `settings.local.json`。

---

## 9. 维护

| 触发 | 动作 |
|------|------|
| 新增项目级工具 | permissions.allow 追加 |
| 出现误操作风险 | permissions.deny 追加 + hooks 拦截 |
| 新增团队级强制规则 | 移到 `.claude/settings.json`（共享） |

---

## 10. 模板

可直接复制到新项目的骨架：

`aiweave/templates/.claude/settings.local.json`（如有需要可生成）。


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
