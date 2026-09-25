# Issue #214 技术方案：对话级 Practice 候选簿

> 状态：**实验草案，供评审；不是已接受的 OpenSpec，也不是实现结果。** 本次只读调查，未修改仓库文件、运行测试、提交或推送。跨 Host 状态、生命周期和合同属于仓库定义的大型变更；若方案获准，应先转为 OpenSpec proposal，再讨论实现。当前运行无法手动切换到仓库要求的 `gpt-6-sol` / `xhigh`，因此不声称已使用该档位。

## 结论与范围

建议在 **CLI runtime 内定义 `ConversationPracticeLedger`**：它保存某个 Agent 在一段任务会话中*成功读取过、可能仍相关*的 Practice 元数据。记录是读取时的候选快照，不是“已采纳”“有效”或“当前 Pack 内容”的证明。Host adapter 只负责把可信的会话、Agent、父子和生命周期事件转换为通用输入；Pack/LocalStore 仍是 Practice 内容与来源的唯一事实来源。状态不归 Codex Plugin、Engine retrieval state 或 LocalStore 管理，也不预建独立 workspace package。这个责任划分与 Issue #214 原文一致。([]())

本阶段**定义合同，不接入 Hook**。#215 才尝试记录可验证的主 Agent 成功 `get`；#216 才在真实 `SubagentStart` 中消费有界父候选；#217 的 compact 后提示明确留在 backlog。它们都不能反过来充当 #214 已有实现的证据。

## Observed：已核实的现状

- 当前 [Codex Hook 实现](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts)只接受 `SessionStart`，调用 `listPackDetails` 渲染 metadata-only Catalog；[CLI 入口](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/main.ts)把 `hook codex` 作为原始 Hook ABI 单独路由。[Plugin Hook 配置](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/hooks/hooks.json)仅匹配 `startup|resume|clear|compact` 的 `SessionStart`。现有测试验证 Catalog 和失败时 `continue: true`，**没有**候选簿、`PostToolUse` 捕获或子 Agent 归属测试。
- [当前 `get` 命令](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.ts)一次读取一个 effective Practice；[结果 schema](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/result-schema.ts)提供 `practice.id/title/applies_when`、`contentDigest` 和全部 `{packName, sourcePath}`。[Engine 合并规则](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/engine/src/local-store/model/effective-practices.ts)要求同 ID 的多 Pack 内容 digest 一致，并按 Pack 名、相对路径排序来源。[`get` 合同](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md)只保证**本次** point read 一致；下一次命令可能看到不同 Store revision。`get` 不返回 Pack version，digest 也不是签名。
- [agent-integration 当前 spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md)要求 CLI-first、Hook 不自动 query/get、无本地 MCP；[practice-read spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/practice-read/spec.md)保护 canonical point read。这两项合同不能因新增候选状态而被绕开。
- OpenAI Docs 描述了 `PostToolUse`、`SubagentStart` 和 `SessionStart` 等 Hook 的可能字段与时机，也说明子 Agent Hook 可使用父 session id、不同工具路径的覆盖并不相同。**官方字段表不是本项目真实调用的归属证据**；相同 `session_id`、`cwd`、transcript 路径或 shell 命令文本都不足以证明是哪一个 Agent 成功读取。

主 Agent 提供的线索总体与原文及代码相符；需要纠正的潜在推论是：**“Host 文档列出 `PostToolUse`/`SubagentStart`，所以现在能够自动捕获主 Agent 的 `get`”不成立**。当前 Plugin 根本未注册该写入路径，真实 payload 的成功结果和主/子关联也尚未验证。([]())

## Required：本阶段要定下的合同

候选簿必须以 `hostKey + opaque root conversation reference` 隔离根会话；`hostKey` 只作命名空间，不参与 Practice 规则。一条记录至少保留 Practice ID、读取时 digest、title、可选 appliesWhen，以及**当次读取返回的所有来源 Pack 身份**，绝不保存 body。统一规定去重、更新、并发、父子可见性、clear/resume、过期和有界保留；缺乏可靠 Host 关联时不写入、不注入，但不能阻塞原始 Agent 工作。Issue #214 要求的是这些可供后续 adapter 使用的语义和最小合同，而非本轮 Hook 实现。([]())

## Proposed：记录、责任和最小接口

字段从一次已验证的成功 `get` 结果提取，不从默认展示文本、transcript 或重新执行一次 `get` 拼装：

| 候选字段 | 来源及含义 |
| --- | --- |
| `practiceId`, `title` | 同一次 `get.data.practice` 的 `id`、`title` |
| `contentDigest` | 同一次 `get.data.contentDigest`；仅标识**当时** canonical 内容 |
| `appliesWhen?` | 同一次 `practice.applies_when`；空串可在候选摘要中省略 |
| `sources[]` | 同一次 `get.data.sources` 的完整有序 `{packName, sourcePath}` 集合；`packName` 是来源 Pack 身份，路径仅为 Pack 内相对路径；不补造 version |
| `actorRef`, `readAt`, `eventRef` | adapter 证明的 Agent/事件关联，以及 CLI runtime 的记录时间；均不是 Practice 内容字段 |

同 ID、多 Pack 的内容相同才会形成当前 effective Practice；候选簿存**当次返回的整组 sources**，不任选“主 Pack”，也不把不同时间的来源集合求并集。后来读取到同 ID、新 digest 时，同一 Agent 分区内以新快照替换旧快照；digest 相同但来源变化时，也用新读取的整组来源替换。候选最多代表“最近被簿记的一次可信读取”，不宣称它是 Store 的最新版本；需要使用正文时 Agent 仍自行 `lore get` 并检查当前结果。[`get` 文档](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md)与 [多来源测试](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/store.integration.test.ts)支撑这一快照边界。

以下是**包内设计接口示意，不是已存在 API**；`VerifiedGetReceipt` 是 adapter 在验证原调用成功、结构化结果和归属后才可提交的窄输入：

```ts
interface ConversationBinding {
  hostKey: string;
  rootRef: string;             // Host 的不透明根会话引用
  generationToken: string;     // CLI runtime 为本次生命周期代际签发
  actorRef: string;            // Host 内不透明 Agent 引用
  role: "root" | "child";
  parentActorRef?: string;     // child 必须有可验证的父子链
}

interface VerifiedGetReceipt {
  eventRef: string;            // 经证实稳定的调用标识，用于重放去重
  practiceId: string;
  contentDigest: string;
  title: string;
  appliesWhen?: string;
  sources: readonly { packName: string; sourcePath: string }[];
}

interface ConversationPracticeLedger {
  transitionRoot(event: {
    hostKey: string;
    rootRef: string;
    kind: "start" | "resume" | "clear" | "compact" | "end";
  }): Promise<{ generationToken: string } | undefined>;

  recordRead(
    binding: ConversationBinding,
    receipt: VerifiedGetReceipt,
  ): Promise<"recorded" | "duplicate" | "ignored">;

  listCandidates(
    binding: ConversationBinding,
    budget: { maxItems: number; maxCharacters: number },
  ): Promise<{ items: readonly CandidateSummary[]; truncated: boolean }>;
}
```

CLI runtime 统一校验字段、事件幂等、代际与父子权限，拥有独立的本机持久化和裁剪规则。Host adapter 负责证明 `rootRef`、`actorRef`、父子链、调用成功与 `eventRef`，把 Host 事件映射到接口；**不能**把未知字段填成“当前对话”默认值。`get`/LocalStore 不承担候选写入事务。第一条 #215 路径只提交已证明属于 root Agent 的 receipt；若未来记录 child 读取，应隔离在该 child 的 actor 分区，默认不提升到父候选视图。#216 的 child 读取视图只取经验证父链对应的 root 候选，不把其他子任务的探索混入。

## Proposed：生命周期、一致性与降级

建议用 CLI 管理、**与 Pack LocalStore 物理分离**的本机状态文件；`--store-root` 仍只选择 Practice Store，不能暗中选择或写入候选簿。起步可用 Bun 已采用的 SQLite 能力处理跨进程事务与唯一约束，不增加数据库服务或第三方依赖；具体跨平台状态目录须在 OpenSpec 中固定，不能落入 Plugin 私有目录。逻辑唯一键为 `(hostKey, rootRef, generation, actorRef, practiceId)`，另以 `(generation, eventRef)` 拒绝同一成功事件重放。一次事务完成事件去重、候选更新、限额裁剪；列表读取一个提交后的快照。并发到达的不同 digest 按**提交顺序**保留最后一份可信 receipt，并明确不保证实际工具调用的因果先后。

- `clear`：同一根引用也开启新 generation，旧候选立即不可见。延迟到达的旧 generation receipt 必须被拒绝；若 Host 无法让一次读取与 clear 前后的代际可靠关联，就暂停该 Host 的自动写入，而非在 clear 后误记旧读取。根引用变更时天然隔离。
- `resume`：仅在同一根引用、未过期 generation 上继续；找不到或已过期就返回空簿，绝不从 cwd/transcript 重建。`compact` 不清空、不自动注入；`end` 标记生命周期但不立即删除，以免正常结束/闲置事件妨碍可用的 resume。
- 清理与预算：建议初始可调实验值为每根会话最多 **64** 条、最多 **512** 个保留会话、闲置 **7 天**及绝对 **30 天**过期；这些不是已测得的需求阈值，OpenSpec 评审时应按实际规模和磁盘预算校准。超限按最近可信簿记时间淘汰，平手按 ID 确定性排序；巨大或不合法的单项宁可不记，不能静默截断来源集合后冒充完整来源。`listCandidates` 在预算内先保留 ID、title、digest 及“来源是否完整”的标识，再省略可选描述、裁掉整项，返回 `truncated`；具体 Host 文案和 Catalog 共同预算留给 #216。
- 失败路径：无可信归属、调用失败/取消、退出状态未知、结构化结果不完整、旧代际、存储忙碌或损坏时，不作候选写入；可选消费返回空且不阻塞原 `get` 或子 Agent。错误可在本机记录不含正文、prompt、原始 payload、路径及会话引用的分类计数；不能把“跳过写入”报成成功捕获。

这是一份有意窄的本机候选状态，不是同步、任务记忆、自动检索或 workflow engine。#217 即使将来使用它，也只能消费候选摘要，不能把 compaction summary 或候选 digest 当作当前 Store 事实。

## 取舍

| 备选 | 得失与结论 |
| --- | --- |
| **CLI runtime 的独立候选簿（推荐）** | 统一跨 Host 语义、事务和清理，延续 CLI-first；代价是新增小型本机状态及迁移/损坏恢复责任。 |
| Codex Plugin 私有文件 | 接入快，但会把通用规则和状态绑定到一个 Host，其他 Host 无法共享合同；不选。 |
| 混入 LocalStore/Engine retrieval state | 可复用现成 Store，却会把会话暂态和 Pack canonical 数据、派生索引的生命周期混为一谈；不选。 |

独立 workspace package 现在没有第二个真实 Host 的复用证据，先留在 CLI 包内；当第二个 adapter 确实需要同一规则且包依赖方向仍合理时再评估抽取。没有必要为这组窄规则引入新的持久化框架或本地 MCP。([]())

## 验收与仍缺的证据

**合同级验收**可先用伪造的、明确标为测试输入的 Host binding 做纯规则和隔离测试：同事件重放、同 ID/同 digest、多来源变化、digest 更新、不同 host/root/actor、父子与兄弟不可见、clear 后旧事件、resume/compact、TTL/限额、并发进程、SQLite 崩溃重开与损坏降级；断言存储和输出都不含 body。预算选择应有确定性测试，且不能暗示被裁掉的 sources 是完整列表。这些是**后续实现的验收设计**，本次没有运行测试。

**真实 Host 门槛尚未满足**：须在获准的独立验证中，对成功、失败、取消、复杂 shell 包装、长时运行/轮询的 `get`，以及 root/child `PostToolUse`、`SubagentStart`、startup/clear/resume/compact，采集最小化、脱敏的真实 Hook payload；逐项证明调用结果、退出状态、稳定事件 ID、发起 Agent、根引用、父子关系与 clear 前后代际能否关联。尤其要证明能从可信结构化结果提取同一次 `get` 的元数据，不能解析人类可读布局或凭命令文本猜测。若证据不成立，#215/#216 应停在“该 Host 不支持自动捕获/提示”，保留手工 `lore get` 和现有 Catalog，不制造伪等价能力。([]())

下一步仅是评审本草案及 Host 证据门槛；获批后按仓库规则建立 OpenSpec proposal。本次 Lorelum Skill 的语义检索遇到 `backend.state-invalid`，原 trace 返回空记录，因此没有把任何未取到的 Practice 当作方案依据；这也不是候选簿能力验证。

