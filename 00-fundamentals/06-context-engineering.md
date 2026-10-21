# 上下文工程（Context Engineering）

> 主文 · 相关：[glossary/context-compaction.md](glossary/context-compaction.md) · [glossary/context-explosion.md](glossary/context-explosion.md) · [glossary/memory.md](glossary/memory.md) · [glossary/rag.md](glossary/rag.md) · [glossary/prompt-system-prompt.md](glossary/prompt-system-prompt.md) · [05-multi-agent-collaboration.md](05-multi-agent-collaboration.md) · [03-large-language-models.md](03-large-language-models.md)（KV Cache、上下文窗口原理）

---

## 0. 结论先行

- **上下文工程 = 每次调用 LLM 前，决定把什么、以什么顺序、多少量放进上下文窗口。** 提示词工程只是其中「system prompt」一块。
- 模型只「看见」上下文里的东西；Agent 的大部分失败（失忆、跑偏、幻觉引用）都可归因到上下文放错、放多或放少。
- 核心手段：预算化、分区、渐进披露、压缩/摘要、截断、工具结果裁剪、prompt cache、子 Agent 隔离。

---

## 1. 上下文窗口与 token 预算

![上下文组成与预算](assets/context-budget.svg)

*图 1：一次调用的上下文分区（自上而下的推荐顺序）：稳定的 System Prompt 与工具 Schema 在前（利于缓存）；随后是记忆召回、检索结果、历史摘要、近端原文、当前输入；末尾必须为模型输出**预留 reserve**。PNG：[context-budget.png](assets/context-budget.png)*

- **窗口**：模型单次可处理的输入+输出 token 上限。
- **预算公式**：`可用输入 = contextWindow − reserve(输出与推理)`；各分区再分配配额（例：系统 5%、工具 10%、记忆 5%、检索 20%、历史摘要 10%、近端原文 剩余）。
- 成本与延迟随输入 token 近似线性增长，所以「装得下」不等于「应该装」。

## 2. 上下文的组成

| 分区 | 内容 | 管理要点 |
|---|---|---|
| System prompt | 角色、边界、输出格式 | 稳定、版本化、放最前以命中缓存（见 [prompt](glossary/prompt-system-prompt.md)） |
| 工具 Schema | 可用工具定义 | 工具多时按意图筛选，描述精炼 |
| 历史 | 对话与工具轨迹 | 摘要 + 近端原文；tool_call 与结果成对 |
| 检索结果 | RAG 片段 | rerank 后少而精，带来源编号（见 [rag](glossary/rag.md)） |
| 记忆 | 用户事实/偏好/经验 | 相关性召回，与工具结果分区（见 [memory](glossary/memory.md)） |

## 3. 治理手段

![上下文治理策略](assets/context-strategies.svg)

*图 2：每轮先估算 token，超过 `window − reserve` 时选择策略（裁剪、摘要、截断、子 Agent 隔离、渐进披露）后重新估算，直到满足预算再调用 LLM。PNG：[context-strategies.png](assets/context-strategies.png)*

| 手段 | 做法 | 代价 |
|---|---|---|
| **渐进披露** | 先给目录/摘要/技能清单，模型需要时再调用工具加载全文 | 多一次调用 |
| **压缩/摘要（compaction）** | 旧轮次→结构化摘要，近端保留原文（见 [context-compaction](glossary/context-compaction.md)） | 有损 |
| **截断/滑动窗口** | 丢最早的非关键轮次 | 可能丢约束 |
| **分支摘要** | 对话树/子任务分支结束时，把该分支压成一段结论再回主干 | 需树状会话结构 |
| **工具结果裁剪** | 大输出只回头尾/统计/摘要，原文落盘给路径 | 需二次读取 |
| **Prompt cache** | 保持前缀稳定，服务端复用已计算的 KV 缓存 | 前缀一变即失效 |
| **子 Agent 隔离** | 探索性子任务在独立上下文跑完，只回传摘要（见 [05](05-multi-agent-collaboration.md#上下文爆炸)） | 协调开销 |
| **检索替代全量** | 文档入库按需检索（见 [vector-database](glossary/vector-database.md)） | 检索可能漏 |

> **交叉注记：工作区自查找 ≈ 无固定黑板**  
> 「工具结果落盘 + 渐进披露」让 Agent 在工作区里按需查找、自行组织上下文——coding Agent 面对大仓库正是如此，并不依赖一块唯一的共享黑板。[ChainReactor 公开文章](https://mp.weixin.qq.com/s/bmQmIMtiPANCC8hEwmO0ew)中「不设固定黑板」的 nomd 组与此同构，初步结果是解题能力接近、主要差在每次启动的探索耗时（未独立复现）。因此工作区自探索可以替代「唯一共享黑板」作为推理接口，黑板更适合做观测面；详见 [05 §4 行业注记](05-multi-agent-collaboration.md#4-blackboard黑板)。

## 4. 常见失败

![上下文常见失败与对策](assets/context-failures.svg)

*图 3：五类失败（污染、丢失、Lost in the Middle、干扰、冲突）及对应对策。PNG：[context-failures.png](assets/context-failures.png)*

- **上下文污染**：错误的工具结果或幻觉被写入历史，后续反复引用 → 观察带来源、反思校验、可回滚 checkpoint。
- **上下文丢失**：压缩/截断丢掉约束或待办 → 结构化摘要模板、「约束区」固定保留。
- **Lost in the middle**：长上下文中间位置的信息被利用得较差（已有研究报告此现象）→ 关键信息放开头或结尾，检索结果少而精。
- **干扰与冲突**：无关工具/文档过多；新旧信息矛盾 → 渐进披露、时间戳、记忆 UPDATE/DELETE。

## 5. 技术实现原理

**上下文组装器（Context Builder）**：

```python
def build(state, window, reserve):
    parts = [
        ("system", system_prompt(state), FIXED),
        ("tools",  select_tools(state.intent), 0.10),
        ("memory", recall(state.query, k=5), 0.05),
        ("rag",    rerank(state.query, search(state.query, 30))[:5], 0.20),
        ("summary", state.summary, 0.10),
        ("recent", pair_safe(state.recent), REST),     # tool_call/result 成对
        ("input",  state.user_input, FIXED),
    ]
    budget = window - reserve
    ctx = allocate(parts, budget)                       # 按配额裁剪各分区
    if tokens(ctx) > budget:
        state = compact(state)                          # 摘要旧轮
        ctx = allocate(rebuild(parts, state), budget)
    return ctx
```

**本仓已有实现参照**（以各文档为准）：

| 系统 | 机制 | 出处 |
|---|---|---|
| **pi** | 触发：`contextTokens > contextWindow − reserveTokens`（默认 reserve 16384）；保留近端 `keepRecentTokens`（默认约 20000）；扩展可拦 `session_before_compact` | [pi.md](../02-frameworks/pi.md) |
| **Strix** | 超窗或 overflow 时旧轮摘要为 `<conversation-checkpoint>`，近端原文保留；安全向结构化摘要模板（Objective / Findings / Credentials / Work State / Dead Ends…），禁止合并去重发现；切分点保证 tool_call/tool_result 成对；`keep_tokens` / `compact_buffer_tokens` / `summary_max_tokens` 可配 | [strix.md](../01-competitors/strix.md) |
| **ARTEX** | 探索图冷节点 digest：未覆盖冷节点 ≥ 20 触发 minor，活跃 digest ≥ 8 触发 major；后台 LLM 把连通冷子图压成 digest，热节点复活时剔除；冷却 60s、单次上限 5min | [ARTEX.md](../01-competitors/ARTEX.md) |

**为什么这样设计**：上下文是有限、昂贵且影响注意力分配的资源；把它当「按预算组装的视图」而非「对话日志」，信息的真源放在外部（文件、向量库、checkpoint、图），每次只投影当前决策需要的部分。

---

下一步：[glossary/context-explosion.md](glossary/context-explosion.md) · [glossary/context-compaction.md](glossary/context-compaction.md)
