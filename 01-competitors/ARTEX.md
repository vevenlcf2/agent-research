# ARTEX（RuoJi6/ARTEX）竞品研究

> 仓库：https://github.com/RuoJi6/ARTEX · 语言 Go · License AGPL-3.0 · Stars ≈229（调研日 2026-10-16）  
> 默认分支 `main`。本仓为 Autumn-27/ARTEX 的持续维护开源版本。

## A. 定位与技术栈

| 项 | 内容 |
|---|---|
| 定位 | LLM 多 Agent 驱动的**自主渗透测试平台**（授权场景），带完整 Web 控制台 |
| 后端 | Go 单体（`cmd/artex`），内嵌 Next.js 静态前端；依赖 `norma` SDK（agentcore / tool / permission / harness / memory / transcript） |
| 存储 | PostgreSQL（双图：资产图 + 探索图）；JWT / SSE |
| LLM | Anthropic / OpenAI 兼容；UI 可配；`llmpool` 池化 |
| 前端 | Next.js（仪表盘、任务、资产覆盖图、拦截审批、人在环路对话） |
| 支撑 | MITM 流量录制代理、MCP、skills、ScopeSentry 资产同步 |

## B. Agent 拓扑

**Planner–Worker + 主 Agent（人在环路）**，非 LangGraph 图运行时，而是自研引擎调度：

- `goals`：目标分解与范围提取
- `planner`：唯一意图生成者（事件驱动）
- `worker ×N`：每条意图一个执行会话
- `mainagent`：操作员对话 / 改目标 / 注入提示

核心数据模型是**双图**：

1. **资产图（全局共享）**：root_domain → subdomain → service → endpoint
2. **探索图（每任务独立）**：goal / intent / fact / finding / hint，靠 spawns / derived_from / yields / proves 血缘相连；`exploration_anchors` 把探索节点锚定到资产

> **字段对照（2026-10-21）**：探索图的 fact / intent / hint 与公开讨论中 Cairn 黑板的 Fact / Intent / Hint 对象在字段级同构（goal 亦对应 Cairn_Y FGS 图中的 Goal）。这只是概念对照，不表示 ARTEX 源自 Cairn；可借鉴结论不变。出处与证据边界见 [CYBER-Cairn](CYBER-Cairn.md)。

## C. 调度管理

### 机制概要

- 每任务：`plannerLoop` + N 个 worker goroutine（默认并发在系统设置，常见 3）
- 图变更 debounce 唤醒 planner → 派 0..N 意图进 frontier → worker `claimNext` 领取
- planner 跨唤醒共享 **todolist**（`TodoStore`），支持串行利用链分步派发，避免并行乱序
- 失败/超时：worker state 区分 `done` / `exhausted` / `blocked`；planner 可 `kill_work` / `steer_work`
- 任务级日历调度见 `server/scheduler.go`、`db/schedules.go`

### 证据

`agent/planner.go`（意图唯一生成者 + 跨唤醒 todolist）：

```go
// todos keeps ONE plan-scratchpad per task ... dispatch it step-by-step over rounds
todoMu sync.Mutex
todos  map[int64]*actool.TodoStore
```

`agent/compaction.go`（planner 每轮挂钩压缩维护）：

```go
// OnPlannerRound ... bumps round, maintains cold stamps, then launches background compaction
func (c *Compactor) OnPlannerRound(ctx context.Context, ts *db.ExplorationStore)
```

README 架构节明确：引擎事件驱动闭环；`prove_goal` 驱动任务完结。

## D. 上下文压缩

ARTEX 的压缩分两层，产品价值极高：

### 1）会话级 compaction（norma agentcore）

Planner/worker 跑 `agentcore` 时配置 `Compaction` 窗口；可选实验特性 **noa** 接管压缩。

### 2）探索图冷节点压缩（自研 cold-digest）— 核心差异点

路径：`agent/compaction.go` + `agent/coldgraph.go`

- 每 planner 轮：`maintain` 同步打冷/热戳（不调 LLM）
- 阈值：未覆盖冷节点 ≥ N=20 → **minor**；活跃 digest ≥ M=8 → **major**
- 后台 LLM 把连通冷子图压成 digest body，带 signature/generation；热节点复活则 commit 时剔除
- 冷却 60s / 单次上限 5min，同任务互斥

摘录：

```go
needMinor := uncompressed >= c.n  // N=20
needMajor := activeDigests >= c.m // M=8
```

另有 `sidequestion/CONTEXT_BUDGET.md` 侧问上下文预算设计。

worker 间还可通过 `search_all_worker_traces` / `get_worker_trace` 做**过程级信息交换**，减少重复劳动而不膨胀主上下文。

## E. 门限审核

路径：`guard/guard.go` + `intercept/`

- PreToolUse / PostToolUse hooks；用户可配拦截规则（allow / deny / ask）
- `ask` → 人工审批；超时或拒绝 → block；文案明确「平台管控·非目标防御」，避免 Agent 当 WAF 绕过
- PostToolUse 对 Bash 结果做 blocked/error/ok 归因（Observer/G5），供 planner 换策略
- 人在环路：mainagent 对话、审批记录 UI、漏洞归并人工确认、复测会话

摘录：

```go
case "ask":
    if !g.interceptor.HandleAsk(...) {
        return g.block(..., "人工审批未通过（用户拒绝或审批超时）", "")
    }
```

## F. 可借鉴 / 不适合

**可借鉴**

1. 双图（资产真值 vs 探索血缘）是调度与覆盖可视化的优秀产品骨架  
2. Planner 共享 todolist + 冷图 digest：长程任务上下文治理的工业级方案  
3. 审批文案防「当 WAF 绕过」——安全产品 UX 细节  
4. worker 过程检索：跨子 Agent 共享而不塞满 prompt  

**不适合直接作为我们底座**

1. AGPL-3.0 传染风险高；深度绑定 `norma` 闭源/半闭源 SDK 生态  
2. 偏完整渗透平台，而非「可插拔调度内核」；二次定制成本大  
3. 默认定位进攻面广，合规与授权边界需自建更强策略层  

## 一句话结论

**ARTEX 是六家里「平台型调度 + 双图记忆」最完整的参考实现，选型应抄其双图与冷压缩思想，而非 fork 整仓。**

## 图示

见 `diagrams/ARTEX/`（architecture / sequence / dataflow JSON + mermaid）。
