# 中间件设计

> 规范来源：`aiweave/docs-spec/04_architecture_overview_spec.md` §5

## 1. {middleware_1}

### 1.1 职责
（一句话）

### 1.2 注册位置
- 文件：`middleware/{name}.go`
- 路由组：{audience}

### 1.3 输入
- 提取的参数：{query / Header / 何处}
- 依赖的资源：{Redis Key / 配置项}

### 1.4 处理步骤

1. {步骤 1}
2. {步骤 2}
3. ...
N. ctx.Set("{key}", {value})

### 1.5 输出
- ctx.Set 注入字段：{...}
- 错误响应格式：{JSON 结构}

### 1.6 性能 / 缓存
（如有热路径，显式说明）

### 1.7 边界情况
| 情况 | 错误码 |
|------|--------|
| 缺参数 | ... |
| 资源未找到 | ... |
| 时间窗口超时 | ... |

---

## 2. {middleware_2}

（同 §1 结构）

---

## 3. {middleware_3}

（同 §1 结构）


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
