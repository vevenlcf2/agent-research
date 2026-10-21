# 自建智能体调度平台 · 开源架构调研总报告

**受众**：产品 / 架构 / 安全工程  
**日期**：2026-10-16（Asia/Shanghai）  
**约束**：只讨论调度、上下文、门禁与技术栈；不提供攻击利用步骤或 exploit PoC。  
**竞品**：ARTEX、Strix、AuroraRecon、VulnHunter、Full-Chain-Sentinel-System、DeepAudit  
**框架**：pi、dsh、LangChain、LangGraph  

---

## 1. 调研背景与问题定义

我们希望建设**自有智能体调度平台**，支撑漏洞扫描与挖掘类工作负载。平台核心不是单一扫描器，而是：

1. **调度管理**：任务如何创建、派发、并行/串行、失败恢复  
2. **上下文治理**：长程 Agent 如何不爆窗、如何跨 Agent 共享关键状态  
3. **门限审核**：人在环路、范围强制、工具权限、自治等级  

本报告从开源竞品与主流 harness/编排框架抽取可落地的架构模式，形成选型建议。

---

## 2. 执行摘要（结论先行）

| # | 建议 | 依据 |
|---|---|---|
| 1 | **战役编排用 LangGraph** | Aurora 三类图 + 官方 interrupt/checkpointer；最贴「阶段+审批」 |
| 2 | **会话执行参考 pi（或 dsh）** | 工业 compaction、会话审计、扩展缝；Sentinel 已落地渗透壳 |
| 3 | **长期记忆用双图/世界模型** | ARTEX 资产图∪探索图+冷 digest；Aurora AttackGraph+prior |
| 4 | **门禁四层基线** | scope → 确定性规则 → 不可逆分级 → 自治档位（Aurora）；审批 fail-closed |
| 5 | **质量/审计流水线** | Clouditera VulnHunter：YoungFlow decide 螺旋 + gate/decision + 双向论证 + sandbox_cfg |

推荐逻辑架构见第 7 节。

---

## 3. 竞品架构要点（摘要）

详细章节见 `01-competitors/`；图示见 `diagrams/`。

### 3.1 ARTEX — 双图驱动的平台型调度

- Go + Postgres + Next.js；norma SDK  
- Planner 唯一产意图，Worker 领意图；共享 todolist 保串行链  
- **冷图压缩**：热/冷戳 + minor/major LLM digest，压缩探索图而非只砍聊天记录  
- Guard/Intercept：ask 人工审批；阻断文案标明平台策略  

**产品启示**：长程渗透的「记忆」应结构化（图），聊天压缩是第二层。

### 3.2 Strix — 多 Agent 操作系统

- AgentCoordinator：树、邮箱、唤醒、预算暂停、快照  
- Compaction：安全域结构化摘要，强制枚举发现/凭据/死路  
- Docker 沙箱 + 授权声明 + coverage gate  

**产品启示**：子 Agent 生命周期与消息投递语义（交互 vs 非交互）必须产品化定义。

### 3.3 AuroraRecon — LangGraph 战役蓝本

- linear / feedback / supervisor 三模式可切换  
- Postgres checkpointer + human_approval interrupt  
- scope_guard + safety_gate(YAML) + irreversible_gate + autonomy_level  
- engagement_memory 跨任务先验（凭据仅存在性）  

**产品启示**：门禁应确定性优先、LLM 不可覆盖；自治≠取消不可逆审批。

### 3.4 VulnHunter（Clouditera）— YoungFlow 审计工作台

- **纠偏**：非 capitalone Claude Code 三技能；锚点为 Clouditera/VulnHunter  
- VulnForge `flow.audit.yaml`：**decide** 调度螺旋（onboard→cognize→hunt→verify→可选 POC/EXP→report）  
- 门禁：`gate.yaml` fail-closed + `decision.yaml` 路由 + 双向论证 + `sandbox_cfg` / SandboxPlane  
- 引擎：**YoungFlow**（LangGraph + pi）；平台壳 Hono/React/PG/MinIO  

**产品启示**：可借鉴「YAML flow 资产包 + 文件路由信号 + 沙箱平面代理」；调度内核用 YoungFlow 或自研等价物。

### 3.5 Full-Chain-Sentinel — pi 渗透壳

- 上游 pi harness + Web + Kali 容器 + 审批三档  
- Compaction/会话树完整继承  
- 文档坦陈：AI 审批 fail-open、容器非整体沙箱  

**产品启示**：Harness 很好用，但**安全边界必须产品层补强**并写进威胁模型。

### 3.6 DeepAudit — 审计向多 Agent 产品

- Orchestrator→Recon/Analysis/Verification  
- MemoryCompressor + 熔断限流 checkpoint + SSE  
- 偏 SAST，门禁深度不足进攻场景  

**产品启示**：事件模型与 TaskHandoff 可复用到「审计类」工作负载。

---

## 4. 三维能力深潜

### 4.1 调度管理：三种主流范式

```
A. 意图队列型（ARTEX）
   事件 → Planner → Frontier(intent) → Worker池 → 写回图 → 再唤醒

B. 图状态机型（Aurora / LangGraph）
   StateGraph 节点=阶段；边=路由；interrupt=审批；checkpointer=续跑

C. 会话树型（pi / Sentinel / 部分 Strix 子会话）
   用户/系统 prompt → tool loop → compact → 分支/恢复
```

**平台建议**：控制面任务状态机用 B；单阶段深度推理用 C；资产覆盖推进用 A 的意图/ frontier 概念。

### 4.2 上下文压缩：四条技术路线

| 路线 | 代表 | 优点 | 风险 |
|---|---|---|---|
| Transcript 摘要 | Strix / pi / DeepAudit | 通用、实现清 | 摘要丢洞 |
| 结构化安全摘要模板 | Strix | 强制保留 findings | 依赖摘要模型质量 |
| 图外置 + 冷压缩 | ARTEX | 长程可扩展 | 工程复杂 |
| decide 螺旋 + wiki/todo 落盘 | Clouditera VulnHunter / YoungFlow | 防编排污染、可续跑 | 路由靠 LLM，需 max_loops |

**平台建议**：默认「图/工件外置 + 近端原文 + 安全模板摘要」；禁止静默合并 findings。

> **定位注记（2026-10-21）**：「图外置 + 冷压缩」中的图，首先服务于**资产覆盖可视化、审计回放与跨 Worker 共享效率**；勿把图 Schema 的精巧程度当作能力银弹。依据为 ChainReactor / M09ic《黑板之外-走向更开放的AI渗透架构设计》（2026-10-20，<https://mp.weixin.qq.com/s/bmQmIMtiPANCC8hEwmO0ew>） 所述初步消融（结构化黑板未见提升解题率，未独立复现），详见 [05 §4 行业注记](00-fundamentals/05-multi-agent-collaboration.md#4-blackboard黑板) 与 [CYBER-Cairn](01-competitors/CYBER-Cairn.md)。本注记不改变记忆层选 ARTEX 双图的结论。

### 4.3 门限审核：分层模型

```
L0 授权与承诺（登录/任务创建）
L1 Scope 强制（命令/URL/IP 解析 ∈ authorized_scope）
L2 确定性策略（黑白名单 YAML，LLM 不可改）
L3 不可逆分级（可逆自动 / 不可逆必批）
L4 人在环路 / 守门员模型（fail-closed 优先）
L5 质量门（证伪、覆盖率、独立复核）
L6 运行时沙箱（容器、网络、密钥）
```

Aurora 覆盖 L1–L4 最完整；Clouditera VulnHunter 强在 L5（decide/gate/双向论证）与 L6（SandboxPlane）；Strix/Sentinel 强在 L6 但 L4 策略不同。

---

## 5. 框架层结论

详见 `02-frameworks/pi-dsh-langchain-langgraph.md`。

- **LangGraph**：战役编排与 HITL 首选  
- **pi**：会话循环与 compaction 首选参考/可嵌入  
- **dsh**：插件化能力缝与审批策略插件化；文档已公开  
- **LangChain**：模型与工具适配，不作编排内核  

论文 2608.23953 提醒：主流 harness 仍缺 **external verifiability**——建议我们在路线图中规划「不可篡改审计导出」（签名会话、策略决策日志）。

---

## 6. 对比矩阵指针

完整表格：[`03-comparison-matrix.md`](03-comparison-matrix.md)。

---

## 7. 推荐目标架构（逻辑视图）

```
                    ┌──────────────────────────┐
                    │ 控制面 Web / API / RBAC   │
                    │ 项目·授权·配额·审计查询    │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │ 战役编排 LangGraph        │
                    │ 阶段节点 · 回流 · interrupt│
                    │ autonomy / irreversible   │
                    └────────────┬─────────────┘
                                 │ 派生子会话
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
     ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
     │ 执行 Harness    │ │ 世界模型/双图   │ │ 质量管道        │
     │ pi/dsh/YoungFlow│ │ 资产·探索·攻击图│ │ decide·gate·论证│
     │ compact·tools  │ │ digest·prior   │ │ 独立只读 Agent  │
     └────────┬───────┘ └────────┬───────┘ └────────┬───────┘
              │                  │                  │
              └──────────────────┼──────────────────┘
                                 ▼
                    ┌──────────────────────────┐
                    │ 适配层 LiteLLM/LangChain  │
                    │ MCP · Skills · 沙箱运行时 │
                    └──────────────────────────┘
```

### 7.1 非目标（首期不做）

- 不绑定单一云厂商模型  
- 不在编排层内嵌具体 exploit 知识库为默认自动执行  
- 不采用 AGPL 整仓作为闭源商业内核  

### 7.2 首期里程碑建议

1. LangGraph 线性战役 + Postgres checkpointer + 审批 interrupt  
2. 接入 pi 或等价 session compact + JSONL 审计  
3. Scope + safety YAML + irreversible  
4. 工件落盘与发现 schema；预留 decide/gate/双向论证槽  

---

## 8. 风险与合规

- 所有进攻性工具仅限授权目标；平台需强制 scope 与操作审计  
- License：优先 Apache/MIT 组件；AGPL 仅作思想参考  
- Sentinel 式 fail-open 守门员不可直接用于生产默认档  
- 双用途模型厂商政策（如 Anthropic Cyber Verification）需纳入部署清单  

---

## 9. 附录：交付物清单

| 文件 | 说明 |
|---|---|
| `00-index.md` | 索引与结论 |
| `01-competitors/*.md` | 六竞品全文 |
| `01-competitors/CYBER-Cairn.md` | 闭源信号摘要（ChainReactor CYBER / Cairn；非开源竞品，未核实） |
| `02-frameworks/pi-dsh-langchain-langgraph.md` | 框架专章 |
| `03-comparison-matrix.md` | 对比矩阵 |
| `04-architecture-research.md` | 本总报告 |
| `diagrams/<name>/{architecture,sequence,dataflow}.json` | Archify IR（showcase 几何已校正） |
| `diagrams/<name>/{architecture,sequence,dataflow}.html` | Archify showcase HTML（**18/18 已 render**） |
| `diagrams/RENDER-STATUS.md` | 渲染状态清单 |
| `diagrams/<name>/*-mermaid.md` | Mermaid 备用 |

---

## 10. 证据索引（抽样）

| 主题 | 路径 |
|---|---|
| ARTEX 压缩 | `agent/compaction.go` |
| ARTEX 门禁 | `guard/guard.go` |
| ARTEX 规划 | `agent/planner.go` |
| Strix 压缩 | `strix/llm/compaction.py` |
| Strix 协调 | `strix/core/agents.py`, `strix/tools/agents_graph/tools.py` |
| Aurora 编排 | `backend/agents/orchestrator.py` |
| Aurora 门禁 | `scope_guard.py`, `safety_gate.py`, `irreversible_gate.py` |
| Aurora 记忆 | `engagement_memory.py` |
| VulnHunter（Clouditera） | `flows/vulnforge/README.md` + YoungFlow |
| YoungFlow | `02-frameworks/youngflow.md` |
| Sentinel/pi 压缩 | `packages/coding-agent/docs/compaction.md` |
| DeepAudit 压缩 | `backend/app/services/llm/memory_compressor.py` |
| DeepAudit 图控 | `core/graph_controller.py` |
| dsh 架构 | `deepseek-ai/deepseek-harness/docs/architecture.md` |

---

*本报告由产品调研执行器基于 GitHub API 与公开文档生成，未整仓 clone。Archify showcase HTML 已补齐（18/18），详见 [`diagrams/RENDER-STATUS.md`](diagrams/RENDER-STATUS.md)。*
