# 多 Agent 协作模式（Multi-Agent Collaboration）

> 主文 · 相关：[glossary/multi-agent.md](glossary/multi-agent.md) · [glossary/orchestration.md](glossary/orchestration.md) · [glossary/context-explosion.md](glossary/context-explosion.md) · [06-context-engineering.md](06-context-engineering.md)  
> 竞品样板：[ARTEX](../01-competitors/ARTEX.md)（Planner–Worker）· [Strix](../01-competitors/strix.md)（AgentCoordinator）

---

## 0. 结论先行

| 模式 | 一句话 | 首选场景 |
|---|---|---|
| Pipeline / Sequential | 固定顺序交接产物 | 步骤稳定、需审计的流程 |
| Hierarchical（Supervisor–Worker） | 主管拆解派发、工人执行回报 | 可分解的大任务、需要并行 |
| Peer-to-Peer Negotiation | 对等 Agent 互相质疑/辩论 | 质量优先、有争议的判断 |
| Blackboard | 共享状态板 + 专家按条件认领 | 问题求解路径不确定、多专长松耦合 |
| Router / Dispatcher | 先分类再交给一个专家 | 请求类型多、每类处理相对独立 |

![多 Agent 协作模式总览](assets/ma-overview.svg)

*图 0：五种模式与各自的核心取舍。PNG：[ma-overview.png](assets/ma-overview.png)*

**默认建议**：先单 Agent + 好工具；不够再用 Router 或 Pipeline；确需动态分解与并行才上 Hierarchical；Peer 与 Blackboard 用于特定难题。见 §6。

---

## 1. Pipeline / Sequential（顺序流水线）

![Pipeline 模式](assets/ma-pipeline.svg)

*图 1：A（采集/抽取）→ B（分析）→ C（撰写/校验）依次交接结构化产物；C 校验失败可回退给 B。PNG：[ma-pipeline.png](assets/ma-pipeline.png)*

- **是什么**：多个 Agent 按固定顺序执行，上一个的输出（最好是结构化产物）是下一个的输入。
- **何时用**：步骤和顺序事先确定；每步需要不同提示/工具/模型；需要逐段审计。
- **如何工作**：
  1. 定义每段的输入/输出 Schema（如 `Findings{items[]}` → `Analysis{...}` → `Report{...}`）。
  2. 编排器（代码或图）依次调用；每段只拿上一段产物，不拿其完整对话。
  3. 段间可插确定性校验；失败时回退到指定段重试（有限次）。
  ```python
  art = extract_agent.run(input)          # → Findings
  art = analyze_agent.run(art)            # → Analysis
  for _ in range(2):
      rep = write_agent.run(art)
      if validate(rep): break
      art = analyze_agent.run(art, feedback=rep.errors)
  ```
- **优点**：简单、确定、易测试、成本可预估、天然隔离上下文。
- **缺点**：不灵活；前段错误向后传播；无并行。
- **例子**：「采集→分析→报告」类研究/文档生成流水线；编排上可用 [LangGraph](../02-frameworks/langgraph.md) 的线性图实现。

---

## 2. Hierarchical（Supervisor–Worker 层级）

![层级模式](assets/ma-hierarchical.svg)

*图 2：Supervisor/Planner 拆出意图放入任务队列（frontier），多个 Worker 以 `claimNext` 领取并行执行，结果回报主管，主管汇总。PNG：[ma-hierarchical.png](assets/ma-hierarchical.png)*

- **是什么**：一个主管 Agent 负责拆解、分派、验收；多个 Worker Agent 执行子任务；可多层嵌套。
- **何时用**：任务可分解但分解方式依赖中间结果；子任务可并行；需要统一验收。
- **如何工作**：
  1. 主管把「子 Agent / 派发动作」当工具（`dispatch(task, to=...)`）或写入任务队列。
  2. Worker 在独立上下文运行（常是 ReAct），完成后回报**摘要 + 产物引用**。
  3. 运行时维护任务状态机（pending/running/done/failed）、父子关系、并发上限、级联取消。
  4. 主管根据结果决定继续派发、重派或结束。
- **优点**：动态分解、可并行、职责清晰、子上下文隔离。
- **缺点**：主管是瓶颈和单点；主管理解偏差会放大；协调 token 开销。
- **真实例子**：
  - **ARTEX**：每任务一个 `plannerLoop` + N 个 worker goroutine；图变更唤醒 planner 派意图进 frontier，worker `claimNext` 领取（见 [ARTEX](../01-competitors/ARTEX.md)）。
  - **Strix**：`AgentCoordinator` 维护 statuses / parent_of / mailbox / wake Event，支持 wait 与 cascade stop（见 [Strix](../01-competitors/strix.md)）。

---

## 3. Peer-to-Peer Negotiation（对等协商 / 辩论）

![对等协商](assets/ma-peer.svg)

*图 3：Proposer 提方案 → Critic 质疑 → 修订，轮数有上限；最后可交 Judge 裁决。PNG：[ma-peer.png](assets/ma-peer.png)*

- **是什么**：地位对等的 Agent 通过消息互相提议、质疑、修订，直到达成一致或由裁判决定。
- **何时用**：单次生成易错、需要交叉验证的判断（方案评审、事实核查、风险评估）。
- **如何工作**：
  1. 为各角色设不同 system prompt（提议者 / 批评者，或持不同立场）。
  2. 共享一个精简的「议题 + 当前方案 + 争议点」状态，而非完整互相转发历史。
  3. 设最大轮数与收敛条件（批评者无新问题 / 分数达标）；不收敛则交 Judge 或人。
  ```python
  plan = proposer(topic)
  for r in range(3):
      issues = critic(topic, plan)
      if not issues: break
      plan = proposer(topic, plan, issues)
  verdict = judge(topic, plan, issues) if issues else plan
  ```
- **优点**：提升质量与鲁棒性，暴露盲点。
- **缺点**：token 成本成倍增加；可能空转或「互相附和」；需要明确终止。
- **例子**：代码生成后由评审 Agent 质疑再修订；参见 [reflection](glossary/reflection.md) 中的 Critic 分离。

---

## 4. Blackboard（黑板）

![黑板模式](assets/ma-blackboard.svg)

*图 4：共享黑板存事实/假设/待办与版本号；各专家 Agent 读取满足自身触发条件的内容并写回；控制器选择下一个被激活的专家，直到满足终止条件。PNG：[ma-blackboard.png](assets/ma-blackboard.png)*

- **是什么**：所有 Agent 通过一个共享的结构化工作区通信，而非直接互相发消息；控制器决定谁下一步行动。
- **何时用**：求解路径不确定、需要多种专长机会式贡献；Agent 数量会变化。
- **如何工作**：
  1. 黑板是带 Schema 与版本号的存储（如 `facts[]`、`hypotheses[]`、`todos[]`）。
  2. 每个专家声明触发条件（「有未验证假设时我能工作」）。
  3. 控制器按优先级/规则/LLM 选择专家；写入用乐观锁（版本号）或按条目认领防冲突。
  4. 终止：目标字段满足或预算耗尽。
- **优点**：松耦合、易增减专家、天然支持并发。
- **缺点**：控制策略难设计；需冲突控制；整体行为较难预测与调试。
- **例子**：ARTEX 的共享 todolist / 探索图被 planner 与 worker 共同读写，具有黑板特征（见 [ARTEX](../01-competitors/ARTEX.md)）。

> **行业注记（2026-10-21）：黑板更像仪表盘，未必是发动机**  
> 出处：ChainReactor / M09ic《黑板之外-走向更开放的AI渗透架构设计》（2026-10-20，<https://mp.weixin.qq.com/s/bmQmIMtiPANCC8hEwmO0ew>）；摘要见 [CYBER-Cairn](../01-competitors/CYBER-Cairn.md)。  
> - 作者团队基于 Cairn 做了两组消融：把 JSON/图黑板换成只追加 Markdown，效果无差别，结构描述反而多耗 token；不设固定黑板、让 Agent 直接写工作区并按需查找，解题能力接近，主要差在效率（16 题合计有黑板组总耗时少约 16%）。  
> - 作者据此把黑板定位为**人机观测、任务回放与干预的界面**，其主要收益是效率与人类可读性；长链路下这部分收益可能被信息污染抵消。其产品 CYBER 2.0（文章所述预计 2026 年 11 月发布）将不再以黑板作为推理引擎。  
> - **证据边界**：样本小、单模型、单基准，作者自述需更多数据；本仓**未独立复现**，只作为行业信号。  
> - **对本仓的含义**：采用黑板/共享图时，优先按「观测面 + 共享效率」设计，不把 Schema 精巧度当作能力来源；不改变 [04 总报告](../04-architecture-research.md) 的记忆层选型。

---

## 5. Router / Dispatcher（路由分发）

![路由模式](assets/ma-router.svg)

*图 5：Router（规则/小模型/LLM 分类）把请求分给专家 Agent；置信度低时走兜底（通用 Agent 或人工）。PNG：[ma-router.png](assets/ma-router.png)*

- **是什么**：入口只做一次分类，把请求交给最合适的一个专家 Agent 处理。
- **何时用**：请求类别多、每类所需工具/知识不同、彼此独立。
- **如何工作**：
  1. 路由器用结构化输出返回 `{route, confidence}`（或用 embedding 相似度匹配路由描述）。
  2. 低于阈值走兜底；专家 Agent 只加载本类工具（顺带减小上下文）。
  3. 记录路由决策用于评测与迭代。
  ```python
  d = router_llm(q, response_format={"route": Literal["billing","tech","rag"], "confidence": float})
  agent = EXPERTS[d.route] if d.confidence >= 0.7 else FALLBACK
  return agent.run(q)
  ```
- **优点**：便宜、快、易扩展、专家上下文小。
- **缺点**：错路由代价大；不适合需要跨专家协作的任务。
- **例子**：客服/内部助手按「账单 / 技术 / 知识问答」分流。

---

## 上下文爆炸

> 总论见 [glossary/context-explosion.md](glossary/context-explosion.md)、[06-context-engineering.md](06-context-engineering.md)、[glossary/context-compaction.md](glossary/context-compaction.md)。

多 Agent 的 token 消耗随 Agent 数、轮数与转发方式放大。各模式的膨胀点与缓解：

| 模式 | 上下文如何膨胀 | 适用缓解手段 |
|---|---|---|
| Hierarchical | 主管累积所有 Worker 回报原文；Worker 被塞入主管完整历史 | Worker 只回 `{summary, artifacts}`；主管滚动摘要；Worker 只接收任务描述 + 必要上下文；子上下文隔离 |
| Peer Negotiation | 每轮双方互相转发全文，轮数 × 方案长度线性甚至平方增长 | 只共享「当前方案 + 未解决争议点」；轮数上限；每轮后摘要旧轮 |
| Pipeline | 若把前段完整对话传给后段，越往后越长 | 段间只传结构化产物；Schema 限制字段；大产物落盘传引用 |
| Blackboard | 黑板只增不减，专家每次读全板 | 按触发条件只读相关分区；过期条目归档/digest（类似 ARTEX 冷节点 digest）；版本化 |
| Router | 一般最轻；风险在于兜底 Agent 加载全部工具 Schema | 专家只加载本类工具；路由前不注入长历史；兜底也做工具按需加载 |

---

## 6. 何时多 Agent 过度 vs 值得

| 信号 | 结论 |
|---|---|
| 单 Agent 加好工具与提示就能稳定完成 | **过度**：多 Agent 只增加成本与故障点 |
| 「角色」只是换个提示词、没有不同工具/知识/模型 | **过度**：用一个 Agent 的多步提示即可 |
| 子任务强耦合、需要频繁同步细节 | **过度**：通信开销 > 收益 |
| 单上下文装不下（工具太多、资料太长） | **值得**：按专长拆分并隔离上下文 |
| 子任务可并行且独立 | **值得**：Hierarchical/Blackboard 降低墙钟时间 |
| 需要独立的交叉验证 | **值得**：Peer/Critic |
| 不同环节需要不同权限边界 | **值得**：按权限拆 Agent，便于门禁 |

**经验法则**：先度量单 Agent 的失败模式（超窗？工具选错？质量不稳？），再选能针对性解决它的最小协作模式。

---

## 7. 技术实现原理（通用骨架）

- **消息与任务**：`Task{id, parent_id, assignee, input, status, result_ref}`；每个 Agent 一个 mailbox 队列。
- **调度**：事件驱动（任务完成/状态变化唤醒协调者）；并发上限；超时与级联取消。
- **状态**：共享状态用类型化 Schema + 版本号；完整轨迹进日志/checkpoint，不进上下文。
- **可观测**：每个 Agent 一个 span 子树，便于定位哪一环出错（见 [evaluation-trace](glossary/evaluation-trace.md)）。
- **门禁**：对外动作统一经门禁层，不因多 Agent 而分散（见 [guardrails-hitl](glossary/guardrails-hitl.md)）。
