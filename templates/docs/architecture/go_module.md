# Go 模块与依赖规范

> 规范来源：`aiweave/docs-spec/04_architecture_overview_spec.md` §9

## 1. 模块路径

```
module {project_module_name}

go 1.{version}
```

## 2. 直接依赖清单

| 依赖 | 版本 | 用途 |
|------|------|------|
| github.com/gin-gonic/gin | vX.Y.Z | HTTP 框架 |
| gorm.io/gorm | vX.Y.Z | ORM |
| github.com/IBM/sarama | vX.Y.Z | Kafka 客户端 |
| github.com/spf13/cobra | vX.Y.Z | 命令行 |
| go.uber.org/zap | vX.Y.Z | 日志 |
| github.com/gomodule/redigo | vX.Y.Z | Redis |
| ... | ... | ... |

## 3. 不可替换依赖

以下依赖一旦升级会破坏框架契约：
- `pkg/golib/v2`（内部）
- `pkg/gin`（内部）

## 4. go.mod 生成顺序（重建工程时）

1. `go mod init {module}`
2. 添加直接依赖
3. `go mod tidy`


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
