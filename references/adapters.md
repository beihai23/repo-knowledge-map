# 工具适配器

按优先级探测,选**最高可用层**。绝不静默降级——选完要打印告知用户。

## 探测与选用顺序

### Tier 1 — 语义/AI 级(最准,有就用)

**Greptile**
- `probe()`:环境变量 `GREPTILE_API_KEY` 存在,且目标仓库已在 Greptile 索引(或可触发索引)。
- `extract()`:调 Greptile API v2 的 call-graph / class-graph 端点,把返回的调用关系转成统一图模型的 `calls`/`depends-on` 边。
- 限制:云、付费、需仓库可被 GitHub 访问、首次需索引。

**Sourcegraph**
- `probe()`:本地 Sourcegraph 实例可达(`http://localhost:7080` 或自配地址),**或** 仓库内已有 SCIP 索引文件(`*.scip`,由 scip-go / scip-typescript 等生成)。
- `extract()`:Sourcegraph API 查精确引用/调用关系;或直接解析 `.scip` 文件抽符号关系。
- 限制:自托管部署重;SCIP 需各语言 indexer。

**Cody**(可选增强):环境可用时作 AI 问答补充,不作为图数据主源。

### Tier 2 — 语法级(自建,跨语言)

**tree-sitter + graphviz**
- `probe()`:命令行 `tree-sitter` 在 PATH。
- `extract()`:用 tree-sitter 解析各语言 AST,抽函数定义/调用、import、类/接口,构建调用图与依赖图。跨语言(Go / TS / JS / Python / Solidity / Rust / Move 等)。
- 限制:语法级,不能解析跨文件语义(如接口的具体实现归谁);需装各语言 grammar(见 `tree-sitter-setup.md`)。

### Tier 3 — 兜底(永远可用,零依赖)

**grep / ripgrep + tree**
- `probe()`:永远 true。
- `extract()`:用 `tree` 取目录结构,用 `rg` 搜函数定义(`func `、`def `、`function `)、调用、import,推断节点和边。
- 限制:粗粒度,准确度靠推断,**必须打 `[推断]` 标记**,report 里如实反映。

## 归一化(关键)

无论哪个 Tier,`extract()` 的返回都**必须是统一图模型**(见 `graph-model.md`):节点 `{id, kind, name, cite}` + 边 `{from, to, kind, evidence}`。下游渲染只认这个模型,不认具体工具的原始格式。

这样带来的好处要记住:换工具只换适配器,产物结构和 `<cite>` 不变。用户环境升级(装了 Sourcegraph),skill 自动跳 Tier 1,产出更准,文档结构不动。

## 通用适配器接口(扩展用)

要接入新工具(Cody / Bloop / SCIP-各语言 / Joern 等),实现两个方法即可,skill 本体不改:

```
probe()        → bool        // 这个环境里该工具是否可用
extract(scope) → Graph       // 扫描 scope,返回统一图模型
```

实现完,skill 的探测链自动纳入它,按 Tier 优先级选用。
