# Handoff: arbetslag — 逐概念核对，第 1 个已定，第 2 个待开始

日期：2026-10-21（第二轮 session）
仓库：`/Users/longfangsong/Projects/arbetslag`，git HEAD `95c58c8 temp`（4 个临时 commit）
**全程用中文交流。**

---

## 0. 进度

**第 1 个领域概念「模板（原称：定义）」核对完毕并落盘。第 2 个「工作流」尚未开始。**

用户上一句是"把达成一致的部分落到文档"，文档已写完，随后要求出这份 handoff。**下一步：回到逐概念核对，从第 2 个概念开始**（问题清单见第 8 节）。

---

## 1. 这个项目是什么

**arbetslag 是一个"独立运行的 Agent workflow"的运行平台。图不是内部接线，而是产品本体：它的形状本身是一份可变、可持久的状态，agent、工具与外部渠道都是图上的参与者。**

- 部署目标：**Node 与 Workers（edge）都要**。
- 仓库形态：monorepo，框架包 + 薄应用。
- Telegram bot **不是主要方向**，只是用来校验"期望用法"的透镜。
- 现有代码（`packages/arbetslag` 约 1700 行）**只当参考，全部重来**。用户明确要求过。

---

## 2. 已落盘的文档（**不要重复它们，直接读/改**）

| 文件 | 内容 |
|---|---|
| `CONTEXT.md` | 术语表。模板、实例化、工作流、形状、节点、邮箱、边、端口、出口节点、消息、投递、Effect、触发器、静止、Run、冻结。含退役词清单 |
| `docs/adr/0001-boundary-is-triggers.md` | 内核只有 `run`；投递=一次 CAS 写+run；触发器在模型外 |

**分工铁律**：`CONTEXT.md` 是纯术语表，不是 spec，不许塞实现细节；实现决定只进 ADR；未定的东西两边都不写。

---

## 3. 架构决定（现在有效的全部，按主题）

### 3.1 本轮新定（细节在 ADR，这里只记核心）

- **边界只有触发器**（ADR-0001）。内核只提供 `run(workflow)`。投递 = 一次带版本校验的写（消息进目标节点的邮箱）+ 一次 `run`。图外的一切（webhook、轮询、cron、命令行、另一个工作流）都是触发器，是 store 与 `run` 的客户端。**没有 `dispatch` 动词，没有入口节点，没有 `Event` 信封。**
- **模板只读一次**（ADR-0004）。

### 3.2 旧定、仍有意义，但仍需和用户核对（**这些不在任何文档里，是本次核对后续概念的底稿**）

**节点与转移**

```ts
type Transition<S> = {
  state: S;
  sends?: Outgoing[];        // 图内投递
  effects?: EffectRequest[]; // 出图请求
  spawn?: NodeSpec[];        // 形状：新增
  connect?: Edge[];          // 形状：接线
  retire?: NodeId[];         // 形状：回收
  awaiting?: Await[];        // 我在等谁，内核据此在对象消失时叫醒我
};

type Message      = { from?: NodeId; type: string; body: JsonValue; correlation?: string };
type Outgoing     = { port?: string; to?: NodeId; type: string; body: JsonValue; correlation?: string };
type EffectRequest= { kind: string; input: JsonValue; correlation?: string };
```

- 节点是**纯的、同步的状态机**：不许 await、不许 fetch、不许依赖外部结果。
- `from` 由内核盖章，外部投递的消息没有 `from`（图里没人可指）。
- `correlation` 内核透传，并发发 N 个 effect 时靠它配对。
- 路由两种寻址并存：`{port}`（静态、图解析、可校验）与 `{to: NodeId}`（动态，用于 spawn 出的子节点）。

**Run 与并行**

- **Run = 跑到静止**。阻塞不是线程阻塞，而是"没有可执行的转移"——所以等子 agent 不需要任何 await 原语，只是邮箱空着。
- 认下的代价：单工作流吞吐上限 = 单核（扩展单位是工作流份数）；`step` 必须快；每次 Run 有**步数预算**防活锁；`step` 耗时指标不是可选项。
- 内核不知道进程、不知道时间。

**Effect**

- **执行器是注入的协作者**；内核只产生"待办 effect"。常驻宿主在本地并发跑；无状态宿主交给请求 / `waitUntil` / DO alarm，结果回来再推进。
- **effect 失败是正常结果**，内核不自动重试；`step` 抛异常 = 代码 bug，立即判死该节点、不重试。
- performer 内部可以任意 async（fetch→解析→重试）：内核只保证 effect 级的重试语义。这是纯 step 设计的逃生门。
- 幂等：`effectId` 即幂等键；崩溃重启后用同一 id 重发（at-least-once，绕不过去）。
- **流式 = 订阅式 effect**：`@effect.chunk / @effect.end / @effect.error`。chunk 引发的转移**易失、不落盘**；事实以 `@effect.end` 为准。崩溃后整段重发（LLM 流无法中途续，物理限制），`generation` 变化 → 文本重置而非拼接。
- 逃生门：`compute.run` effect，把重活挪出 step。

**等待与失败**

- **`awaiting` 是一等字段**：节点声明"我在等谁"，内核维护等待注册表，在对象被 retire / 失败 / 图被改写时投递 `@await.broken`。
- **承认天花板：内核无法自动区分"正常静止"和"死锁"**，两者形状相同。只能靠 awaiting 显式化 + 超时 effect + 可观测。这是结构性代价，别让后来人以为能修。
- retire 的连带规则：到达的消息丢弃；在飞 effect 能取消就取消、不能取消则结果废弃。
- 三类失败分开：step 抛异常（bug，判死）／effect 返回失败（正常业务，节点决定）／节点不可恢复失败（投 `@node.failed` 给 supervisor 端口；**无 supervisor → 整个工作流冻结**，保留快照）。冻结是刻意的：宁可停住，也不静默丢节点或静默改图。

**持久化**

```ts
type Store = {
  load(id):    Promise<{ snapshot: Snapshot; version: number } | undefined>;
  save(id, snapshot, expectedVersion): Promise<number>;   // 版本不符 → 冲突报错
  drop(id):    Promise<void>;
};
```

- 快照 + 单调版本 + CAS。CAS 不是为并发（单写者），是安全网：部署重叠／重复 alarm 时会写失败而不是互相覆盖。
- **恢复契约（必须显式实现）**：启动时扫出 `pending` 非空的快照，重新投递。
- **节点类声明 state schema（zod）**，内核校验、落盘、载入再校验。状态必须可被 schema 描述（不能塞 Map / class 实例 / 闭包）。
- **迁移**：节点类自带 `version` + `migrate(state, from)`，载入时链式迁移；迁移不了（未知类／未知版本）→ **显式失败并冻结该工作流**。
- **转移日志是一等持久化**：`{seq, node, msg, emitted, at}`，带保留上限，**只做旁路**（不参与状态）。换来：卡住诊断（awaiting + pending 直接拼出"在等谁、等了多久"）+ **确定性重放**（step 纯 + 结果在日志里 → 时间旅行调试器）。

**LLM 侧**

- **内核不认识 LLM**。`arbetslag/agents` 提供 agent/subagent/tool 节点与模型无关的 history 协议；`arbetslag/openai` 提供 effect performer。
- **agent 的循环是图结构**：agent 节点只做"history + 新消息 → 回复或工具调用"；工具是节点，调用沿边出去、结果沿边回来，循环由回边构成。（现有代码里 `CompletionAgent` 只是 `{api, historyId, allowedTools}` 查表记录，那个循环根本不存在——仓库里 `packages/arbetslag/tests/probe.ts` 就是用户用来验这个洞的探针。）
- **工具 schema 沿端口向节点要**（`describe() → {name, description, inputSchema}`），模板里不复制 schema。理由：两份 schema 必然漂移，且漂移极隐蔽（模型按 B 版调用、工具按 A 版校验）。代价：改接线会改模型可见的工具集，定义校验要报"tools 端口连了不提供 describe() 的节点"。

---

## 4. 已被推翻的东西（别重提）

入口节点 / ingress / 渠道 / 传输归属框架 / 内核路由表 `key→workflowId` / 模板里的 `channels:` 与 `per:` / `dispatch` 动词 / `Event` 信封 / `forward` 结构字段 / 「一个工作流只服务一个会话」/ 数组位置即身份 / 从类名派生节点名。

一条通用教训：**凡"框架理应知道 X"的主张，默认先假设它错**，直到能证明 X 无法由调用方表达。

---

## 5. spawn + await 子 agent 的参考时序（**尚未核对**，概念 11 会用）

`SubagentTool` 是框架内置节点类，把"spawn + await + correlation"打包。

```ts
if (msg.type === "call") {
  const child = newId();
  return {
    state: { caller: msg.from, callId: msg.callId, awaiting: [child] },
    spawn:   [{ id: child, class: "builtin.agent", config: this.config.template }],
    connect: [{ from: child, to: this.id }],
    sends:   [{ to: child, message: { type: "inbound", content: msg.args.query } }],
    effects: [{ kind: "timer.wait", input: { at: Date.now() + this.config.timeoutMs } }],
  };
}
```

时序：`agent ─call→ SubagentTool ─spawn+sends→ child(agent)` → 子节点自己走 `model.complete` effect 往返 → `sends(final)` 回到 tool → tool 转 `toolResult` 给 caller，同时 `retire: [child]`。

子节点的 effect 往返与父节点的等待互不干扰——父的邮箱空着就是"阻塞"。并行 spawn N 个：`awaiting: [c1..cN]`，回一个减一，减到空才回 caller。超时与结果竞态：两个 handler 都要按 `awaiting` 是否仍包含该 child 做幂等判断。

---

## 6. 现有代码 → 新设计（重写时的对照，全部作废但能省一次考古）

| 现在 | 之后 |
|---|---|
| `Project`：数组位置即身份，`connections: [[i,j]]` | 工作流是持久状态，节点有稳定 id；边按端口命名 |
| `Actor`：Inbox/Outbox 建在 `ts-chan` 上，`process` 是 async | 节点是纯 step；邮箱由内核持久化管理 |
| `messageBox.ts`（Inbox/Outbox） | 出局 |
| `CompletionAgent {api, historyId, allowedTools}` = 查表记录 | agent 是真正有状态的节点，history 是它的私有状态 |
| `Runtime.completionAPIs[]` + `Project.sink()` 查表投递 | 出局；模型调用是 effect |
| `InputAdapter` / `OutputAdapter` 基类 + 用户构造 `WebhookServer` | 出局；图外只有触发器 |
| `Store {set, get}` | 快照 store（load/save-CAS/drop） |
| `classNameOf` / `nodeName` 派生名字 | 显式注册名 `defineNode({name: "builtin.agent"})`，否则改类名或 minify 让历史文件全部失效 |
| `toolsState` 字段 | 出局，并入各节点 state schema |
| `UserFunction` | 普通节点（step 包一个纯函数） |
| `node.ts` 隔离 Node-only 依赖 | 思路保留，扩展到 node/worker 两个入口 |
| `tests/smoke.ts` 的 10 条断言 | 估计 4 条可搬（背压、stop 语义、错误传播、序列化往返的精神），其余语义已变 |

反面样本的具体坐标（`code-review-arch` 用得上）：`src/project.ts`（位置即身份）、`src/index.ts`（`Runtime` 查表 + `classNameOf`）、`src/actor.ts` + `src/messageBox.ts`（async Actor）、`src/implementation/input/webhook.ts`（`WebhookServer` 靠构造注入）。

---

## 7. 环境 / 仓库事实

- `apps/telegram-bot/` **只剩 `node_modules`**，源码从未进过 git。残留依赖提示原始意图：`hono` + `@hono/node-server`、`yaml`、`neverthrow`、`date-fns`、`dotenv`。
- 核心包 `packages/arbetslag`：`ts-chan`、`nanoid`、`openai`、`type-fest`、`zod`；biome + tsdown + pnpm workspace。
- 未跟踪：`packages/arbetslag/tests/probe.ts`（用户的手工探针）。
- `/home/longfangsong/Projects/arbetslag/CONTEXT.md` 那个路径**不存在**（旧挂载点已消失）。文档一律写 `/Users/longfangsong/Projects/arbetslag/`。
- **代码文件目前不许动**（用户只放开了 `CONTEXT.md` 与 `docs/adr/`）。

---

## 8. 逐概念核对进度（主线）

已完成：**1. 模板（Definition）**

下一个及之后：

2. **工作流** ← 现在从这里开始。快照完整字段（形状 + 邮箱 + 节点 state + pending effects + awaiting + 版本 + templateName/Version）；id 规则（谁生成、格式、是否可由外部指定）；生命周期（创建/冻结/回收）；**`append` 与 `run` 的并发边界**（CAS 冲突重试、"投递了但没人 run"的悬挂态）；恢复契约；快照体积上限。
3. **形状**：节点 id 生成与稳定性；spawn/connect/retire 的原子应用与顺序；回收后 id 不复用；形状不变量（端口在发送端唯一等；注意"入口无入边"这条已随入口节点一起消失）。
4. **节点**：`step` 签名与不变量；state schema；config schema；`describe()`；注册名；version + migrate；失败判定；`initState(config)`。
5. **边 / 端口**：模板语法（`edges: [[a,b]]` 还是 `{from, port, to}`）；端口命名与唯一性；未连接端口的报错文案；`{port}` 与 `{to: NodeId}` 的共存规则。
6. **消息**：信封字段（`from?`）；保留 type 命名空间（`@effect.*` / `@await.*` / `@node.*` / `@intake` 之类）；`ephemeral` 属于投递元数据而非消息字段；body 可序列化边界。
7. **Effect**：kind 注册表；input schema；幂等键；取消协议（retire 带未完成 HTTP 的节点，宿主怎么停）；streaming 变体的三件事件。
8. **触发器**：调用形态（库 / CLI / app）；与其自有路由表的衔接；重复投递的去重策略；模板里的触发器声明段（内核不读）。
9. **出口节点**：与图其余部分的一致性校验；Effect 只发不收；多出口。
10. **静止 / Run**：工作集定义；步数预算；易失转移不落盘；Run 结束后宿主如何知道"还要不要唤醒"；awaiting + pending 的可观测报告格式。
11. **agent / subagent / tool 节点（agents 包）**：history 协议；工具发现；第 5 节那份 spawn+await 时序需要核对。
12. **supervisor / 冻结**：`@node.failed` 的载荷；冻结后的运维动作（修复、恢复、放弃）。

---

## 9. 尚未决定 / 已知脆弱（结构性，不是待办）

未定：

1. 包的物理布局与命名。
2. 模板里边与端口的具体写法。
3. Workers 映射：一个工作流一个 DO？DO alarm 怎么驱动"跑到静止"？一个 DO 能承载多少节点状态？
4. 转移日志的保留策略与截断阈值。
5. 观察面形态：CLI `inspect`？HTTP 端点？有没有 UI。
6. 在飞 effect 的取消协议。
7. 成本/用量记账放哪一层。
8. 多版本节点类并存（迁移期间的注册与解析）。
9. 现有 `smoke.ts` 哪些断言可原样搬。

已知脆弱：

- **静止 ≠ 死锁不可判定**。只能靠 awaiting + 超时 + 观测让它"看得见"。
- **单写者 + 每跳落盘**：慢 step 或高频消息流会让整份工作流变慢，且难以归因。

---

## 10. 用户的协作方式（最省时间的部分）

- **一轮一个概念，一轮一组问题，每条给推荐答案。** 不要铺开多个概念，不要给"如果你选 A 那么 B"的条件问题。
- **他会逐条反驳，且反驳往往是对的。** 本 session 我三次被他推翻：① "模板会让实例跟着变"（他举 PPT 模板：用模板新建后完全自由编辑，与模板无关）② "渠道应该算路由键"（他：这是 workflow 内部的事）③ "框架应该拥有入口"（他：做成 tool/CLI 就行，无非手动运行 vs 自动运行）。
- **抽象论证他吃不进，例子他能吃。** 他明确要求过"推荐请给出例子"。方案一律给 YAML / TS / 时序 / 表格的**具体样式**。
- **他会自己推翻自己上一轮的结论**（例：先问"入口要不要特殊"，下一轮说"入口根本不是节点"）。别为了连贯守着旧结论。
- **他会主动质疑术语**（"我感觉这更像模板"、"能否避免引入'地址'这个容易误解的术语"）。术语之争要认真对待：先承认论据错误，再给新论据或直接改。改名时同步改 `CONTEXT.md` 和 ADR。
- 说"先别写文件"就是字面意思——只写他点过头的文档。

---

## 11. Suggested skills

- **`grill-with-docs`**（`/Users/longfangsong/.pi/agent/skills/grill-with-docs/SKILL.md`）——**本阶段主技能**。逐概念核对就是它的 round loop。盲点：它假设"问事实是你的活"，而本项目的"事实"在用户脑子里，代码是反面样本。
- **`ask-user`**（`/Users/longfangsong/.pi/agent/npm/node_modules/pi-ask-user/skills/ask-user/SKILL.md`）——概念核对完需要他拍板时用。但他偏好**读长文本 + 逐条回**，多选弹窗不一定合适。
- **`prototype`**（`/Users/longfangsong/.pi/agent/skills/prototype/SKILL.md`）——遇到纸上说不清的点（快照体积、DO alarm 驱动静止、流式易失转移的手感）用。**先问**，因为动代码需要许可。
- **`handoff`**——每轮核对完优先更新 `CONTEXT.md`/ADR，不要造新文档类型。
