# 并发抽取模式(parallel extraction)

默认主流程是**单 agent 串行**。当仓库规模超过单 agent 的 context 容量,或域数多到串行太慢时,启用并发模式。

## 核心原则

一个 agent 一个**独立问题域**,context 隔离,由主 agent 精确构造每个 agent 的输入(不继承会话历史——你构造什么,它就看到什么)。这是并发不崩盘的前提。

## 何时启用(前提全部满足)

1. 切片**正交独立**:各业务域分析互不依赖。
2. 规模够大:聚类后域数 > 约 8,或源码量超单 context 容量。
3. 归一化契约已定:统一图模型 schema + cite 格式 + 模板已固定(见 `graph-model.md` / `layer-templates.md`)。
4. **L0 总图与切片边界已由主 agent 串行确定**——这是并发的前提,不是并发的一部分。

典型触发:taskon-server(65 个 service 域)、ont.io(十几个微服务)。

## 绝不并发的部分(串行保护区,主 agent 自己做)

这些需要全局视野或定义共享契约,并发会破坏一致性:
- **L0 全局总图**:依赖图、业务域聚类、技术栈——要看全部域。
- **统一图模型 schema**:并发前必须先定死。
- **跨域依赖边**:task 域调用 reward 域,需同时看两边。
- **共享支撑层**(store / common / offchain / utils):被所有域依赖,单点负责,不让每个域 agent 各画一遍。

## 切片策略

- **按业务域切片**(L0 聚类后的高层域),不按原始子目录(taskon-server 65 个 service 子目录平铺 = 整合爆炸)。
- 粒度:agent 数 ≈ 聚类域数(5-9)。太细整合难,太粗没并发。
- **共享层单点**:派一个专门 agent 负责 store/common/offchain,其余域 agent 只 cite 它。
- 各 agent 输出落到**互不冲突的路径**(`zh/content/<域>/`),**不直接写同一个 graph.json**。

## 每个 agent 的 prompt 必须包含(主 agent 构造,自包含)

1. **切片范围**:只负责哪个域、哪些路径、不碰什么。
2. **全局 L0 总图**:让它知道自己在哪、和谁接壤(避免发明不一致的边界)。
3. **统一术语表 + 跨域接口契约**:该域对外暴露/依赖的接口。
4. **工具链 Tier + cite 规范 + 模板**:严格按 `graph-model.md` / `layer-templates.md` 产出。
5. **明确输出**:该域的 graph 片段 + L1/L2(+L3)md,严格 schema。
6. **约束**:只产切片内部边,**不要猜跨域边**(留给整合 agent);只写自己的输出路径。

## 整合流程(并发结束后,整合 agent 做,必须全局视野)

1. **归并 graph**:各域片段合并成全局 graph.json。节点 id 全局唯一(加域前缀,如 `task:service/task:GetTaskTemplateById`),避免冲突。
2. **跨域边后置串行补**:基于全局 graph 抽跨域 calls/depends-on(并发的盲区,必须补)。
3. **一致性校验**:术语统一、cite 格式、抽象层级对齐。
4. **冲突去重**:共享模块若被某域 agent 误画,归一到支撑层单点产物。
5. **运行覆盖审计(串行,门禁)**:归并完、产 report 前跑 `audit.py`(见 `coverage-audit.md`)——拿文件系统标尺验全量覆盖、cite、规模。**整合 agent 串行跑一次,绝不并发**(audit 读全局 graph + 全部文档,并发既无意义又竞争)。退出码 0 才进下一步;非 0 回 `iterative-refinement.md` 收敛。
6. **产出 report.md**:整合的冲突、去重、跨域边补全情况 + audit 结论。

## 质量保证(承认 sub-agent 单点质量略低于主 agent,用机制拉回)

- **schema 是安全网**:严格 schema 校验,格式不对打回。
- **cite 自检脚本化(= 覆盖审计 C 关)**:确定性脚本验证所有 file:// 链接,不靠 agent 自证——已统一进 `audit.py`(见 `coverage-audit.md` §6)。
- **audit 由整合 agent 串行跑一次**:归并后、report 前,不并发;退出码 0 才进 report(见上方整合流程第 5 步)。
- **spot check**:整合 agent 抽查各域核心调用链(agent 可能有系统性错误,如漏 cite、乱推断)。
- **关键部分回退串行**:L0、跨域边、核心调用链若整合时发现质量不够,串行重做,不将就。

## 反模式

- ❌ 65 个 service 子目录各派一个 agent → 整合爆炸、一致性崩。
- ❌ 让并发 agent 直接写同一个 graph.json → 互相覆盖。
- ❌ 让并发 agent 猜跨域边 → 必然不准。
- ❌ 没定 schema 就并发 → 无法归并。
- ❌ 小仓库为快而并发 → 启动+整合开销 > 收益。
- ❌ 把 L0 总图也并发 → 每个 agent 看不全,总图碎裂。
