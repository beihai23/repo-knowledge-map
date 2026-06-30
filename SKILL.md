---
name: repo-knowledge-map
description: 从一个或多个代码仓库提取分层知识地图(多粒度索引),产出 L0 全局总图 / L1 服务或子系统概览 / L2 模块地图 / L3 实现细节四层,每层带 <cite> 溯源(file://路径#行号)。自适应探测环境里已有的代码图谱工具(Greptile / Sourcegraph / tree-sitter),优先用最强可用层,grep/rg 兜底,换工具不换产物。当用户提到梳理、理解、概览一个代码库或一组微服务,生成知识库/索引/文档,新人 onboarding,多服务依赖梳理,画调用图 / 依赖图 / 时序图 / 流程图 / 状态机 / 架构图,或问"这仓库怎么跑起来 / 各服务什么关系 / 某功能在哪实现"时,使用本 skill。即使用户没明说"知识地图",只要意图是把仓库"看懂、理清结构、做成可下钻的文档",就应触发。
---

# repo-knowledge-map

把代码仓库变成一套**可下钻的分层知识地图**:最粗比例尺给全局视野,最细比例尺精确定位到某行代码,中间靠 `<cite>` 链接串起来。

## 核心原理(理解它,后面步骤都是自然的)

**四层是固定的"比例尺刻度",与范围(单/多仓库)正交。每层映射到的代码实体随范围缩放。**

这是"不同比例尺地图"的准确含义——同一套四层,节点粒度自适应:

| 层 | 多仓库模式的节点 | 单仓库模式的节点 |
|---|---|---|
| L0 全局总图 | 各仓库/服务 | 仓库内顶层子系统/领域 |
| L1 概览 | 每个服务 | 每个顶层子系统 |
| L2 模块地图 | 服务内模块 | 子系统内模块/包 |
| L3 实现细节 | 关键函数 / 表 / 异常 | 同左 |

**四层在两种模式下都全量存在,单仓库不缺 L0。** 不要写 `if 多仓库/单仓库` 两套逻辑——同一套层,只是 L0/L1 的节点单元不同。

为什么分层? 一个新人面对上百个 md 的扁平知识库,不知道从哪看起;面对十几个微服务,不知道谁调用谁。分层地图解决"从哪看"和"怎么下钻"。

## 执行流程

按顺序走。每步都和用户确认范围,不要闷头跑。

### 1. 确认范围
问清楚(AskUserQuestion 或直接问):
- 单仓库还是多仓库?路径是?
- 深度:到 L3 全量,还是只到 L2(概览+模块,省时间)?
- 语言:中文(默认)?
- 是否 `--link-existing`?(目标仓库已有知识库如 repowiki 时,只在 L0/L2 加下钻链接、不重写现有 md)

### 2. 探测工具链,选定 Tier
读 `references/adapters.md`,按优先级探测,选**最高可用层**:

| Tier | 工具 | 何时用 |
|---|---|---|
| 1 | Greptile API / Sourcegraph(SCIP 或自托管) / Cody | 探测到 API key 或本地实例 |
| 2 | tree-sitter + graphviz | `tree-sitter` 在 PATH(跨语言 AST→调用图/依赖图) |
| 3 | grep / ripgrep / tree | 永远兜底 |

**先努力安装,确认装不上才兜底。** 探测到 tree-sitter / graphviz 缺失时,先引导用户装(往往一行 `brew install`,几分钟),从 Tier 3 升 Tier 2 收益巨大——不要图省事直接兜底。只有用户拒绝或环境装不了(内网/无权限)才落 Tier 3。

探测完**打印告知**:当前 Tier、为什么是这个 Tier、怎么升到更高 Tier。绝不静默降级——用户不知道你在用 grep 兜底,就会误以为调用图是准的。

**硬约束:Tier 3 必须零依赖独立可跑。** Greptile(云+付费+需 GitHub 索引)、Sourcegraph(部署重)都是可选增强。内网/私有仓/离线环境只有 Tier 3,skill 也不能失能。

tree-sitter 安装见 `references/tree-sitter-setup.md`。需要时**请用户协助配置**,不要假装它能用。

### 3. 扫描 + 构图
**先判断并发**:默认单 agent 串行。仓库规模大时(聚类后域数 > 约 8,或源码量超单 context 容量——如 taskon-server 的 65 个 service 域、ont.io 多微服务)启用**并发模式**:主 agent 串行打地基(探测 + schema + L0 聚类 + 切片边界),再按业务域 fan-out sub-agent 各产切片,**整合 agent** 做跨域边补全 + 一致性合并。详见 `references/parallel-extraction.md`。

**并发禁区(主 agent 串行,绝不 fan-out)**:L0 全局总图、跨域依赖边、schema 定义、共享支撑层(store/common/offchain)。这些需要全局视野,并发会破坏一致性。

无论串行还是并发,都用选定 Tier 的适配器提取节点和边,归一化成统一图模型,写入 `.knowledge-map/zh/meta/graph.json`(并发时各 agent 写切片片段,整合 agent 归并)。

统一图模型(详见 `references/graph-model.md`):
- 节点 kind:`Service` | `Subsystem` | `Module` | `Function` | `Endpoint` | `DataStore`
- 边 kind:`calls` | `depends-on` | `data-flow` | `exposes` | `owns`
- 每个节点带 `cite`(file://相对路径#L起-L止,单点证据)+ `files`(覆盖的顶层文件/目录,覆盖范围,供覆盖审计抽覆盖集),每条边带 `evidence`(哪个工具/哪段代码得出)

**归一化是这套设计的灵魂**:无论上游是 Greptile 的 call graph JSON、Sourcegraph 的 SCIP、还是 tree-sitter 的 AST,都转成上面这个模型。下游 L0-L3 渲染和 `<cite>` 产出完全不变。换工具只换适配器,产物稳定——用户哪天装了 Sourcegraph,skill 自动跳 Tier 1,产出更准,文档结构不动。

### 4. 分层渲染(双向迭代,不是单向流水线)
**不是一次性 L0→L3。** 先快速产 **L0 骨架**(基于入口/目录/README,每条域划分和依赖边显式标 `[假设]`),再下探 L1/L2/L3 取真实证据,**回写修正 L0**(域划错就重聚类、依赖边被推翻就删、子系统不存在就撤),反复到收敛。cite 是修正的依据。详见 `references/iterative-refinement.md`。

小仓库可单轮(骨架→下探→自检);大仓库**必须迭代收敛**,否则 L0 一定带着没验证的猜测。每层产出模板和必含图见 `references/layer-templates.md`。

要点:
- **每条结论都挂 `<cite>` + 「来源」标注 `file://相对路径#L行号`。** 溯源比结论本身更重要——没有溯源的结论等于猜测。
- **图渲染**:graphviz 装了渲染 svg,没装 fallback 用 mermaid 文本图(零依赖)。L0 必有服务/子系统依赖图;L2 必有模块依赖图 + 核心调用链。
- **L0 节点判定**:多仓库=每个目标仓库一个 Service;单仓库=由顶层目录结构 + 入口文件启发式判定 Subsystem。**若某层子目录过多(如 taskon-server `service/` 下 65 个子域),按业务语义聚类成 5-9 个高层域,不平铺成几十个节点**——L0 是最粗比例尺。无法归类的归"其他"。L0 必须非空。

### 5. 自检,产出 report.md
见下面"质量保证"。把 report 放到 `.knowledge-map/report.md`。

### 6. 交付
把 `INDEX.md` 路径告诉用户。多仓库时 INDEX=L0 跨服务总图;单仓库时 INDEX=L0 顶层子系统总图。

## 输出目录结构

不碰目标仓库源码,产出落到目标仓库根下的 `.knowledge-map/`(`--out` 可改):

```
<repo>/.knowledge-map/
├── INDEX.md            # 入口:多仓库=L0 跨服务总图,单仓库=L0 顶层子系统总图
├── zh/
│   ├── content/        # L1/L2/L3 的 md(按 服务/子系统/模块 组织)
│   └── meta/
│       ├── graph.json  # 归一化图模型(节点+边),机器可读,可被后续工具复用
│       └── index.json  # 倒排索引:按 业务场景 / 技术主题 / 排查路径
├── assets/             # graphviz 渲染的 .svg/.png
└── report.md           # 自检报告
```

`graph.json` 是关键中间产物——它是"换工具不换产物"的保证,也是后续工具可复用的图数据。

## 质量保证(产出的可信度全靠这一节)

每次产出都跑这几项,写进 report.md。**第 0 项是产出前置门禁,退出码非 0 不交付。**

0. **强制覆盖审计(门禁)**:产出前跑 `templates/audit.py.tmpl` 实例化的 `audit.py`——拿**文件系统客观测量**当标尺(`os.listdir`/`wc`/`find`),验 AI 产物有无结构性遗漏。**退出码 0 才可交付**:A 关孤儿清零(无重大遗漏)且 C 关 cite 全过 且 文件级规模 wc 实测真实。非 0 回 `iterative-refinement.md` 收敛闭环。规范见 `references/coverage-audit.md`。
   - **标尺独立性**:A 关标尺**必须**是文件系统,**绝不**是 AI 产物(graph/README/AGENTS.md 当标尺会被污染——实测两份 AI 文档对账一致,却都把 5679 行抄成 ~12k)。
   - **职责分离**:AI 理解代码(生成 graph+文档,**会漏**);命令穷举校验(**不漏**);两者互校。audit 不理解代码,只做 AI 做不到的穷举。
1. **链接自检**(并入 C 关):所有 `file://路径#行号` 必须真实存在。失效的标红、列入 report。溯源失效,整个知识地图就是空中楼阁。
2. **推断标注**:AI 推断(非工具直接产出)的内容打 `[推断]`。report 给推断占比。关键路径(调用链、依赖边)优先用工具直接产出,不要靠推断。
3. **图-码一致性**:Tier 2/3 时,tree-sitter 的边 vs grep 交叉验证,矛盾处记 report。
4. **透明度 + 诚实留白**:report 必含 当前 Tier、各层覆盖范围、**audit 输出**(孤儿/弱覆盖/cite 有效率/规模核对)、推断占比。**结构层(覆盖/cite/规模)标尺=文件系统,可保证零遗漏;语义层(职责/流程/隐式机制)无客观标尺,保证不了**——report 必须诚实列出"搜了哪些策略""哪些已知未深入",不假装全知。

为什么要这么较真? 知识地图的读者(新人、跨团队)会信任它。一个错的调用链比没有调用链更糟——他们会基于错信息做决策。

## 适配器扩展

一等公民适配器(内置):Greptile、Sourcegraph、tree-sitter。其余工具(Cody / Bloop / SCIP-各语言 / Joern)走通用接口——实现 `probe() → bool` 和 `extract(scope) → Graph` 即可,skill 本体不改。详见 `references/adapters.md`。

## 何时不用这个 skill

- 只想找某个具体符号/函数 → 直接 grep / gopls,不用生成整张地图。
- 只想改某个 bug → 用 debugging skill。
- 仓库极小(单文件或几个文件) → 不值得分层,直接读。
