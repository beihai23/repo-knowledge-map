# 统一图模型(graph.json schema)

所有适配器的 `extract()` 都产出这个模型,下游渲染只认它。这是"换工具不换产物"的契约。

## 节点 Node

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string | 稳定唯一 id,建议格式 `kind:path:name` |
| `kind` | enum | `Service` / `Subsystem` / `Module` / `Function` / `Endpoint` / `DataStore` |
| `name` | string | 展示名 |
| `path` | string | 代码位置(目录或文件) |
| `cite` | string | `file://相对路径#L起-L止`;`Function`/`Endpoint` 尽量必填 |
| `files` | string[] | 该节点覆盖的顶层文件/目录路径(相对仓库根)。供覆盖审计抽覆盖集(见 `coverage-audit.md`)。`Subsystem`/`Service` 常关联多文件,单 `cite` 不足以表达覆盖范围——`cite` 是单点证据,`files` 是覆盖范围,二者互补。 |
| `meta` | object | 扩展字段(如 Service 的 `protocol`、DataStore 的 `engine`) |

## 边 Edge

| 字段 | 类型 | 说明 |
|---|---|---|
| `from` | string | 起点节点 id |
| `to` | string | 终点节点 id |
| `kind` | enum | `calls` / `depends-on` / `data-flow` / `exposes` / `owns` |
| `evidence` | string | 这条边的来源:`greptile` / `scip` / `tree-sitter` / `grep推断` |
| `cite` | string | 证据位置(可选) |

## kind 语义

节点:
- **Service** — 多仓库模式下,一个可独立部署的服务/仓库。
- **Subsystem** — 单仓库模式下,顶层子系统/领域(由目录结构 + 入口文件启发式判定)。
- **Module** — 包/目录级单元。
- **Function** — 函数/方法级。
- **Endpoint** — 对外 API 端点。
- **DataStore** — 数据表 / 缓存 / MQ topic。

边:
- **calls** — A 调用 B(函数级或服务级)。
- **depends-on** — A 依赖 B(模块/包级,如 import)。
- **data-flow** — A 把数据写给 B(如服务写入数据表)。
- **exposes** — A 对外暴露 B(如 Service 暴露 Endpoint)。
- **owns** — A 拥有/包含 B(如 Service owns Module)。

## 推断标记

`evidence` 含"推断"时,渲染阶段该节点/边必须打 `[推断]`,report 统计推断占比。关键路径(调用链、跨服务依赖边)优先选用非推断证据。
