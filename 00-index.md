# 智能体调度平台 · 开源竞品架构调研索引

**文档版本**：2026-10-16（Asia/Shanghai）  
**目标**：为自建「漏洞扫描 / 挖掘」智能体调度平台提供架构选型材料。  
**范围**：调度管理、上下文压缩、门限审核、技术栈；**不含** exploit / PoC 利用步骤。  
**阅读顺序**：先读 [`00-fundamentals/`](00-fundamentals/README.md)（Agent 入门与 glossary），再读竞品与框架。  
**交付仓**：`vevenlcf2/agent-research`。

---

## 结论先行（选型建议）

1. **编排层选 LangGraph（战役状态机 + interrupt HITL）**，不要用纯 LangChain Agent 硬撑多阶段渗透流水线。AuroraRecon 是最贴近的开源样板。  
2. **会话执行层参考 pi（或 dsh）**：工业级 compaction、会话树审计、扩展缝；Full-Chain-Sentinel 已验证「pi + 渗透壳」路径。  
3. **记忆层抄 ARTEX 双图思想**（资产真值图 ∪ 探索血缘图 + 冷节点 digest），解决长程任务上下文爆炸。  
4. **门禁层以 Aurora 四件套为基线**：runtime scope → YAML safety → irreversible 分级 → autonomy 档位；审批默认 fail-closed（警惕 Sentinel AI 审批 fail-open）。  
5. **质量/审计流水线可借鉴 Clouditera VulnHunter**：YoungFlow decide 螺旋 + gate/decision 文件路由 + 双向论证 + sandbox_cfg；引擎层见 YoungFlow。  

**不建议**：整仓 fork AGPL 项目（ARTEX/DeepAudit）作内部闭源内核；不建议以 Strix Cloud 产品闭环替代自建控制面。

---

## 文档目录

| 路径 | 内容 |
|---|---|
| [00-fundamentals/](00-fundamentals/README.md) | **入门层**：Agent 是什么、完整架构、关键名词 glossary（图文；含 RAG 等） |
| [00-fundamentals/01-what-is-an-agent.md](00-fundamentals/01-what-is-an-agent.md) | 定义、与 chatbot 差异、核心组成、典型循环 |
| [00-fundamentals/02-agent-architecture.md](00-fundamentals/02-agent-architecture.md) | 工程分层走读（编排/会话/RAG/门禁；外部图作附录） |
| [00-fundamentals/glossary/](00-fundamentals/glossary/README.md) | glossary 索引（Planner、Memory、RAG、MCP、编排、门禁等；见目录内对照表） |
| [01-competitors/ARTEX.md](01-competitors/ARTEX.md) | Go 双图 Planner–Worker 平台 |
| [01-competitors/strix.md](01-competitors/strix.md) | 多 Agent 协调器 + 安全压缩 |
| [01-competitors/AuroraRecon.md](01-competitors/AuroraRecon.md) | LangGraph Kill-Chain + 分层门禁 |
| [01-competitors/vulnhunter.md](01-competitors/vulnhunter.md) | Clouditera 工作台 · YoungFlow decide 螺旋 |
| [02-frameworks/youngflow.md](02-frameworks/youngflow.md) | YoungFlow：LangGraph+pi flow runner |
| [01-competitors/Full-Chain-Sentinel-System.md](01-competitors/Full-Chain-Sentinel-System.md) | pi fork 渗透控制台 |
| [01-competitors/DeepAudit.md](01-competitors/DeepAudit.md) | ReAct 多 Agent 代码审计 |
| [01-competitors/CYBER-Cairn.md](01-competitors/CYBER-Cairn.md) | **闭源信号**：ChainReactor CYBER / Cairn 黑板架构演进与消融（公开口径，未核实） |
| [02-frameworks/pi.md](02-frameworks/pi.md) | **pi** 独立使用：是什么 / 场景 / 用法 / 效益 |
| [02-frameworks/dsh.md](02-frameworks/dsh.md) | **dsh** 独立使用：是什么 / 场景 / 用法 / 效益 |
| [02-frameworks/langchain.md](02-frameworks/langchain.md) | **LangChain** 独立使用：是什么 / 场景 / 用法 / 效益 |
| [02-frameworks/langgraph.md](02-frameworks/langgraph.md) | **LangGraph** 独立使用：是什么 / 场景 / 用法 / 效益 |
| [02-frameworks/pi-dsh-langchain-langgraph.md](02-frameworks/pi-dsh-langchain-langgraph.md) | 四框架**对比**专章（非用法主文档） |
| [03-comparison-matrix.md](03-comparison-matrix.md) | 对比矩阵 |
| [04-architecture-research.md](04-architecture-research.md) | **总架构调研文档（可作 README）** |
| [diagrams/](diagrams/) | Archify JSON/HTML + 粗骨架 mermaid + **代码视角** `code-*-mermaid.md`（见 [diagrams/README.md](diagrams/README.md)） |
| [product-snapshots/](product-snapshots/) | **产品运行页快照**（Web/控制台/TUI；非 GitHub 整页；见 [product-snapshots/README.md](product-snapshots/README.md)） |

## 各竞品一句话

| 竞品 | 一句话 |
|---|---|
| ARTEX | 平台型调度+双图记忆最完整，抄思想不整仓 fork（AGPL）。 |
| Strix | 多 Agent 协作与安全摘要标杆，门禁偏沙箱预算。 |
| AuroraRecon | LangGraph+世界模型+分层门禁，自建渗透调度最贴蓝本。 |
| VulnHunter（Clouditera） | YoungFlow decide 螺旋 + gate/sandbox 的自托管挖掘工作台。 |
| YoungFlow | VulnHunter 的 flow 引擎（LangGraph 编排 + pi 执行）。 |
| Full-Chain-Sentinel | 证明 pi harness 可撑渗透控制台；战役编排仍弱。 |
| DeepAudit | 审计向 Orchestrator 树与压缩/熔断工程化样板。 |

## 纠偏记录

- **VulnHunter 锚点**：原误用 capitalone/VulnHunter；2026-10-16 改为 **Clouditera/VulnHunter**。口语「yongflow」= **Clouditera/Youngflow**。

## 公开信息缺口

| 项 | 说明 |
|---|---|
| dsh | 仓库与 architecture.md **公开完整**；渗透预制技能少，需自研插件。论文 2608.23953 可作收敛背景。 |
| Aurora License | README 写 MIT，GitHub `license` 字段空，需法务再核。 |
| ARTEX norma SDK | 核心 agent 能力在外部 SDK，源码阅读需结合公开 Go 封装层。 |
| Archify render | 本交付为 **typed JSON IR + mermaid**；本机未装 Archify CLI，**待 Archify render** 出 HTML/SVG。 |
| CYBER / Cairn（ChainReactor） | 闭源，无源码可核。据[公开文章](https://mp.weixin.qq.com/s/bmQmIMtiPANCC8hEwmO0ew)（2026-10-20），CYBER 2.0 **预计 2026-11** 发布，将放弃以黑板作推理引擎；待公开材料再评，11 月前不据此调整记忆层选型。摘要见 [CYBER-Cairn](01-competitors/CYBER-Cairn.md)。 |

## 证据方法

- 使用 GitHub API：`get_repository` / `get_git_tree` / `get_file_contents`（**未整仓 clone**）  
- 官方文档与 arXiv 公开论文补充框架层  
