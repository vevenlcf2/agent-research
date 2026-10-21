# CYBER / Cairn（ChainReactor · 链式反应）闭源信号摘要

> **性质**：闭源产品与公开架构讨论的**信号记录**，不是开源竞品研究；本仓无源码可核，以下凡涉及能力处均为「作者公开口径 / 未核实」。  
> **出处**：ChainReactor 公众号，M09ic《黑板之外-走向更开放的AI渗透架构设计》，2026-10-20 发布，<https://mp.weixin.qq.com/s/bmQmIMtiPANCC8hEwmO0ew>  
> **记录日**：2026-10-21（Asia/Shanghai）  
> **范围**：只记架构主张与时间线；**不含**任何攻击步骤、利用细节或进攻性基础设施描述；文中成绩、排名与营销性表述不作为事实写入 [对比矩阵](../03-comparison-matrix.md)。

## A. 定位与谱系

| 项 | 公开口径（摘述） |
|---|---|
| Cairn | 社区作者「淚笑」设计的黑板式渗透 Agent：把任务看成从起点到目标的状态空间搜索，同质 Worker 只通过共享黑板间接协作，不预设流程、不划分角色、不引入知识库 |
| Cairn_Y | 沿用同一建模，把全部搜索状态外化为只追加的 FGS（Fact–Goal–Step）图，运行分 Decide（评估/排序 Step）与 Execute（持工具、写回 Fact）两类活动，图即共享外部记忆 |
| CYBER | ChainReactor 的产品，文中称「脱胎于 Cairn」，强调无人介入的自主执行与超长链路任务 |
| CYBER 2.0 | 文章所述**预计 2026 年 11 月发布**；作者称将正式放弃以黑板作为推理引擎。具体架构未公开，本仓不推测 |

## B. 公开架构主张（用本仓术语转述）

1. **黑板对象**：Cairn 板上只有三类对象——已确认事实（Fact）、待探索方向（Intent）、经验提示（Hint）。
2. **CYBER 的减法**：去掉 Hint（作者认为其本质服务于人类介入）；保留并重构 Fact / Intent。
3. **Intent 三态（人机交互）**：Oracle（待执行，人类可借此控制下一步，作为 Hint 的替代）→ Intent（执行中）→ Probe（已完成）。
4. **冷热图**：面向单任务超过 1B token 的超长流程，用冷热图算法缓解上下文过载与信息误导；算法细节未公开。
5. **黑板降级**：下一版中黑板不再是 Agent 的主要推理接口，保留用于人类查看状态、回放执行过程与必要干预。
6. **作者的方法论质疑**：
   - 对「预置几十个 Skill」路线存疑（认为是把人的经验固化为清单）；作者曾尝试以 State/Intent 形式算子替代，但自认未形成完备体系。
   - 另一套 Cybernetics 架构只靠 Reason ↔ Action 的双向 Feedback 推进，不显式写共享图；作者认为它与 Cairn 描述的是同一类信息流转，差别只在状态是否显式落到共享图上。

## C. 消融实验（作者转述的初步结果）

| 实验 | 设置（公开口径） | 结论（公开口径） |
|---|---|---|
| 去结构约束 | 基于 Cairn，把 JSON/图黑板换成只追加 Markdown | 结构不影响效果；描述结构的提示词反而多耗 token |
| 去固定黑板 | md 组保留 Markdown 黑板；nomd 组不约定固定文件，Agent 写入工作区、后续按需自查 | 解题能力未见明显差异；16 道题合计 md 组总耗时少约 16%（1004 min vs 1196 min） |
| TCH3 复验 | 另一套 goal + 黑板 / 无黑板架构 | 两者成绩接近，作者据此认为「黑板并非有效因素」 |

**证据边界**：样本量小（16 题）、单模型、单基准，作者自述「仍需更多数据验证」；本仓**未独立复现**。作者的解释是：耗时差主要来自无黑板时每次启动的探索阶段；复杂任务中黑板的效率收益会与长链路信息污染相抵。

## D. 与本仓模式对照

| CYBER / Cairn 概念 | 本仓对应 | 说明 |
|---|---|---|
| Fact / Intent / Hint | [ARTEX](ARTEX.md) §B 探索图 fact / intent / hint | 字段级同构；不代表 ARTEX 源自 Cairn |
| Decide / Execute | ARTEX planner / worker；[VulnHunter](vulnhunter.md) decide 螺旋 | 近似「规划—执行」分工 |
| Oracle（人控下一步） | Aurora interrupt、ARTEX mainagent 注入提示、Sentinel 审批档位 | 同属 HITL 入口；本仓不另造三态状态机 |
| 冷热图 | ARTEX 冷节点 digest（[ARTEX](ARTEX.md) §D） | 思路接近；CYBER 算法未公开，未核实 |
| 黑板 = 观测面 | [05 §4 行业注记](../00-fundamentals/05-multi-agent-collaboration.md#4-blackboard黑板) | 见基础章 |
| 无固定黑板 + 工作区自查找 | [06 §3 渐进披露 / 工具结果落盘](../00-fundamentals/06-context-engineering.md#3-治理手段) | 见上下文工程章 |

## E. 对本仓选型的影响

- **不推翻**现有结论：编排 LangGraph、会话 pi / dsh、记忆 ARTEX 双图、门禁 Aurora 四层、质量 VulnHunter 均保持。
- **口径修正**：双图首先定位为**资产真值 + 人机观测 / 审计 / 回放面**，以及跨 Worker 共享的效率手段；不把图 Schema 的精巧程度当作解题能力来源（见 [04 §4.2](../04-architecture-research.md#42-上下文压缩四条技术路线)）。
- **观察项**：CYBER 2.0 公开材料出来后再评估是否调整记忆层权重；已记入 [00-index 公开信息缺口](../00-index.md#公开信息缺口)。

## 一句话结论

CYBER / Cairn 的公开讨论把「黑板」从推理发动机降为仪表盘：对本仓是**口径与证据注记**，不是选型变更信号。
