# repo-knowledge-map

> 一个 Claude Code skill:从代码仓库提取**分层知识地图**(多粒度索引)——让新人 5 分钟看懂全局,也能下钻到某行代码。

## 这是什么

`repo-knowledge-map` 是一个 [Claude Code](https://claude.com/claude-code) skill。给它一个或多个代码仓库,它产出一套**四层可下钻的知识地图**:

| 层 | 内容 |
|---|---|
| **L0 全局总图** | 子系统/服务依赖图 + 架构图 + 部署图 + 业务域划分 |
| **L1 概览** | 每个服务/子系统的边界、对外接口、上下游、存储 |
| **L2 模块地图** | 模块依赖图 + 核心业务流程图 / 时序图 / 状态机 |
| **L3 实现细节** | 关键函数(嵌真实代码)、数据表、异常 / 性能 |

每条结论都挂 `<cite>` 溯源(`file://路径#行号`),行号由 tree-sitter AST 校验,**不是 AI 凭直觉编的**。

## 核心特性

- **分层探测 + 归一化**:自适应探测环境里已有的工具(Greptile / Sourcegraph / tree-sitter),优先用最强层,grep/rg 兜底。换工具不换产物。
- **四层节点粒度自适应**:四层正交于范围——单仓库的 L0 是顶层子系统,多仓库的 L0 是服务。两种模式都全量四层。
- **双向迭代收敛**:不是单向流水线。骨架假设 → 下探取证 → 回写修正 L0 → 收敛。深入细节发现的反证会纠正总图。
- **大仓库并发**:按业务域 fan-out sub-agent;L0 总图 / 跨域边 / schema 串行(需全局视野)。
- **业务流程图为核心**:架构图 / 部署图 / 协作图 / 时序图 / 流程图 / 状态机全套,不只是模块依赖图。
- **真实溯源**:每条 cite 精确到行,经 AST 校验。AI 推断的内容显式标 `[推断]`。

## 安装

```bash
git clone https://github.com/beihai23/repo-knowledge-map ~/.claude/skills/repo-knowledge-map
```

clone 后 Claude Code 自动识别(新会话生效)。

**可选升级(显著提升质量)**:装 tree-sitter + graphviz,skill 从 Tier 3(grep 兜底)升到 Tier 2(语法级调用图 + 精确 cite + svg 图):
```bash
brew install tree-sitter-cli graphviz
tree-sitter init-config
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-go ~/github/tree-sitter-go
```
其他语言 grammar 按目标仓库技术栈装,详见 `references/tree-sitter-setup.md`。

> ⚠️ brew 坑:`brew install tree-sitter` 装的是**库**不是 CLI,要 `tree-sitter-cli`。

## 使用

在 Claude Code 里说类似的话即可触发:

- "梳理一下这个仓库 / 帮我理解这个代码库"
- "画一下各服务的调用图 / 依赖图 / 时序图"
- "新人 onboarding,做个知识库"
- "这几个微服务之间什么关系?"

产出落到目标仓库的 `.knowledge-map/` 目录,入口是 `INDEX.md`。

## 文件结构

```
SKILL.md                         # 主流程
references/
├── layer-templates.md           # 图表矩阵 + 各层模板
├── adapters.md                  # 工具分层探测 + 适配器接口
├── graph-model.md               # 统一图模型 schema
├── tree-sitter-setup.md         # Tier 2 安装(实测真流程)
├── parallel-extraction.md       # 并发模式
└── iterative-refinement.md      # 双向迭代收敛
```

## 设计理念

比 AI 工具批量生成的文档(如 qoder repowiki)强在:**丰富的业务图 + 真实溯源 + 迭代修正**。批量文档图好看但行为图常虚构——实测发现某 repowiki 的任务状态机(`待处理→进行中→已完成→已结算`)在代码中根本不存在。本 skill 的图都挂在经 AST 校验的真实行号上,且通过迭代收敛纠正骨架期的错误假设。

## License

MIT
