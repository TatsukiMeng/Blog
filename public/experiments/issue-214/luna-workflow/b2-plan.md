已完成研究并调用**唯一一个 `gpt-6-sol / xhigh` 方案子代理**；它没有再委派。下面先给出完整 Proposed 方案，再列我复核后认为需要评审者明确拍板的事项，最后附实际交给子代理的 prompt 和主要取证来源。

**没有实施，也没有写仓库文件。**当前工作区只读，因此不能把方案保存为 OpenSpec artifact；方案正文可供评审，建议后续可写入时放到 `openspec/changes/conversation-practice-ledger-contract/`。

## 方案结论

建议 #214 定义一个**宿主中立、由 Lorelum CLI runtime 拥有**的对话级 Practice 候选簿。候选只表示“主 Agent 曾在这个根会话里成功读取，可能仍相关”，不表示已采纳、有效、适用于当前子任务，也不表示子 Agent 已收到或采用。

Sol 建议由宿主 adapter 提供可核实的会话、角色和生命周期绑定，CLI runtime 负责候选语义、持久化、去重、裁剪和读取；不在 `lore get` 内暗中增加写入。只有真实运行能证明一次 `lore get` **完整成功、可归属于主 Agent、并绑定到正确根会话代际**时，#215 才启用自动记录；#216 也要等父子绑定和提示可见性得到验证后，才注入有界候选 metadata。证据不足时不写、不猜、不注入，不改变原 `get` 结果或阻塞子 Agent 创建。

这一阶段只做 #214 合同设计；#215 负责记录，#216 负责 Codex `SubagentStart` 提示，#217 的 compact 后候选恢复延期。候选簿不替代 #204 对新实质材料的重新检索决策。#214 与这些阶段的边界见对应 Issue。citeturn2view0turn2view1turn2view2turn2view3turn2view4

## Proposed 技术方案

### 现状与约束

当前 `lore get` 从所选 LocalStore 做一致的 Practice point read，返回完整 canonical Practice、`contentDigest` 和来源 Pack 的名称与相对路径；它没有宿主会话身份或候选写入。digest 是 canonical content 的 SHA-256，不是发布者签名；get 结果也不跨命令锁定 Store revision。[实现](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.ts:19>)、[结果 schema](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/result-schema.ts:8>)、[CLI 文档](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md:22>) 规定了这些行为。

当前 Codex Hook 只支持 `SessionStart`，调用 CLI 生成 Pack Catalog；Plugin 配置只注册 `startup|resume|clear|compact`，输出上限配置为 5000。Hook 代码没有使用 session/agent 身份字段，异常时 fail-open。现有 Hook 单元测试和 Plugin 配置测试覆盖的是当前 Catalog 行为，不是真实 Codex 事件样本。[Hook 实现](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts:13>)、[Plugin hooks.json](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/hooks/hooks.json:1>)、[Hook 文档](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/hook.md:1>)。当前集成边界要求 CLI-first、Hook metadata-only，不自动 `query/get`，也不增加本地 MCP。[agent-integration spec](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md:8>)、[plugin-distribution spec](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/plugin-distribution/spec.md:15>)。

### 候选语义与可见性

一次候选记录只证明：一次完整、可归因的成功 `lore get` 曾发生。它不证明 Agent 已采纳或执行 Practice，不证明内容仍是当前版本，不证明适用于子任务，也不证明子 Agent 已收到提示。后续 UI/context 必须保持“可能相关且不完整”的语义。

初始记录范围限定为经核实的根会话主 Agent 读取。子 Agent 读取不回写父簿；兄弟 Agent 不相互供给候选。嵌套子 Agent 只有在能可靠关联根会话和当前 epoch 时，才可读相同根账本快照。无法确认身份关系时返回空提示，不从 cwd、全局“当前会话”文件、transcript、shell 正则或未文档化字段推断。

### 最小内部合同

以下是审查用的内部接口示意，不是已有 CLI 公共命令。`Verified*` 类型不能仅靠传入 `verified: true` 证明可信；adapter 必须先依据真实宿主事件验证数据。

```ts
interface ConversationKey {
  hostKey: string;
  rootConversationRef: string; // 宿主提供的不透明根会话引用
}

interface BoundConversation {
  key: ConversationKey;
  epoch: string; // clear/reset 的会话代际栅栏
}

interface CandidateInput {
  practiceId: string;
  contentDigest: string;
  title: string;
  appliesWhen?: string;
  sourcePacks: readonly string[];
}

interface CandidateSnapshotItem extends CandidateInput {
  firstRecordedAt: string;
  lastRecordedAt: string;
}

interface ConversationPracticeLedger {
  record(
    read: VerifiedRootRead,
    candidate: CandidateInput,
  ): Promise<"added" | "repeated" | "replaced">;

  listForChild(
    start: VerifiedChildStart,
  ): Promise<readonly CandidateSnapshotItem[]>;

  clear(reset: VerifiedRootReset): Promise<void>;
  expire(now: Date): Promise<void>;
}
```

一次目标运行链为：

1. 主 Agent 获得原 `lore get` 返回值。
2. 宿主 adapter 验证事件、角色、根会话及完整成功结果，只投影必要 metadata；失败、截断或无法归属时丢弃。
3. CLI runtime 将 metadata 写入对应根会话和 epoch 的候选账本。
4. 子 Agent 启动时，adapter 验证父子/根关系；CLI runtime 返回一致的候选快照，adapter 再在宿主预算内渲染提示。
5. 任一阶段失败都不改变原 `get` envelope、退出码或子 Agent 启动路径；Hook 不执行新的 `query/get`，不读 LocalStore，也不保存完整 Practice body。

### 状态所有权、去重与生命周期

Sol 推荐候选语义留在现有 CLI runtime 边界，不为目前只有一个实际 Host 的状态抽取新 workspace package。存储建议为与 Pack 数据独立的 SQLite 数据库，例如 `~/.lorelum/conversation-state/ledger.sqlite`；不复用 `store.sqlite`、manifest、Pack artifacts、Engine schema、retrieval indexes，也不随 `--store-root` 变化。相同 `(practiceId, contentDigest)` 幂等更新；同一 ID 出现新 digest 时替换旧候选；多 source 保留当次读取中所有去重、稳定排序的 Pack 名称，不伪造 get 未返回的 Pack version。

生命周期建议：

- `clear`：提升 epoch 并清空候选；旧 epoch 的迟到写入拒绝。
- `resume`：相同根身份且未过期时保留候选。
- `compact`：不清除、不恢复候选；该能力留给 #217。
- `SessionEnd`：可作为清理提示，但不作为唯一清理机制；后续 ledger 操作执行过期检查。

建议作为**待校准起步参数**：每个会话最多 32 条、最多 512 个未过期会话、14 天无活动过期、单条序列化 metadata 不超过 4 KiB；注入最多 8 条或 1024 UTF-8 字节，并服从更小的宿主预算。候选按最近记录时间和稳定 ID 顺序裁剪。它们不是现有负载测量或已批准产品默认值，必须在 #215/#216 以实际规模、延迟和提示质量校准。

### 主要失败边界

官方 Codex Hook 文档支持 `PostToolUse`、`SubagentStart`、`SessionStart` 等事件；`SubagentStart` 文档列出 `agent_id`、`agent_type`，且可返回 `additionalContext`。通用字段包含 `session_id`，文档说明 subagent hook 使用 parent session ID。`PostToolUse` 可提供 `tool_name`、`tool_input`、`tool_response`、`tool_use_id`，也会在工具非零结果后触发。它证明 Hook 有可行入口，但没有在文档层面单独保证调用角色、完整 `lore get` 结果、shell 复合命令归属或嵌套 subagent 的根绑定；必须用真实运行证据确认。citeturn3view0turn3view1turn3view2

尤其要在 #215 核实 clear 竞争：若 `PostToolUse` 只携带 clear 前后相同的 session ID，clear 之前开始、之后才结束的 `get` 可能被误记到新 epoch。只有 Hook payload 可携带/关联读取开始时的 epoch，或事件顺序有充分保证，才能自动记账；否则该场景必须拒记。不要从文档字段缺失推断运行时字段绝不存在，也不要从可能性推断运行时已可靠。

### 方案取舍与需确认项

Sol 推荐独立 SQLite，而非逐会话原子文件：后者需要额外解决跨进程写入、全局过期容量、clear 竞争与崩溃清理；共用 Pack Store 或 Engine 会把短期对话状态放错所有者。Bun 官方资料说明其 SQLite API、事务能力与编译执行支持；当前 Engine 也已使用 `bun:sqlite`。这证明 SQLite 可行，但不证明独立 CLI sidecar 是唯一或已接受方案。citeturn0search1turn0search0

有三项需要评审前明确：

1. **路径归属：**提议路径位于 `.lorelum` 下，但 `.lorelum` 当前也是默认 LocalStore 根；它是否符合“不得混入 Pack LocalStore”，取决于评审将其理解为逻辑独立状态库，还是要求物理路径也分离。应确认安装、reindex、卸载、备份和 Store 移动场景下的边界。
2. **持久化实现：**CLI 当前没有通用 ledger 状态库或直接 Drizzle 依赖。方案选择 SQLite 后仍需明确由哪个模块管理 schema/migration、使用何种 SQLite adapter，以及 CLI 与 Engine 的依赖界面；不要仅凭“Bun 支持 SQLite”跳过这项判断。
3. **PostToolUse 的开销和顺序：**Codex 会对匹配工具调用执行该 Hook；如果按 Bash 匹配，可能涉及每次 Bash 调用的 Hook 启动成本。同步记录有机会保证候选先于随后 subagent 可见，但需要 timeout/延迟门槛；异步记录则可能产生 read-after-write 竞态。方案应把该测量纳入 #215/#216 验收，不预设 synchronous 或 asynchronous。

这三项不阻止 #214 的合同文本评审，但在实现 #215/#216 前需有明确决策或实测结论。候选 Hook 必须 fail-open；错误时不得返回会替换原工具结果或阻止子 Agent 的响应。宿主输出应转义/约束来自 Pack 的标题和 `applies_when`，并保持提示有界。

### 后续阶段与验收

| 阶段 | 主要验收 |
|---|---|
| #214 合同评审 | 定义记录语义、最小 metadata、Host/root scope、父子可见性、digest 更新、多 source、clear/resume/expiry、并发与拒记规则；确认不改变现行 `get`，不越过 CLI-first/metadata-only/no-MCP 边界。 |
| #215 记录 | 先采集脱敏真实事件：成功、不存在、错误、取消、截断、单命令/复合 shell、主/子/嵌套 Agent、clear/resume 与迟到事件。然后测试成功写入、未知归属拒记、幂等、digest 替换、多来源、clear 栅栏、TTL/容量、数据库忙/损坏/只读、`--store-root` 隔离及原 get envelope/退出码回归。 |
| #216 注入 | 实测 `SubagentStart` 父/根/嵌套绑定和上下文可见性；测试父子、兄弟、嵌套、异根隔离、空簿、预算裁剪、注入 metadata 无正文、无自动 `query/get`、无父簿回写、CLI/账本故障不阻塞。 |
| #217 延后 | 仅在写入可靠、compact 时序与根关联实测成立且有噪声/预算评估后另行设计。当前 SessionStart 恢复 Pack Catalog 不算候选恢复。 |

拟议 OpenSpec artifact 为：

- `proposal.md`：说明这是新增行为，不是记录已有能力；列出 why/what changes/impact。
- `design.md`：保留本方案中的 Observed / Required / Proposed / Deferred、合同、数据边界、取舍、失败语义和证据门槛。
- `specs/conversation-practice-ledger/spec.md`：新增候选簿能力 delta，不直接编辑当前 accepted spec。
- `tasks.md`：分别列 #214 文本评审、#215 写入、#216 注入，并将 #217 标为 Deferred。

## 方案依据与研究边界

- #214 当前 OPEN，内容要求的是候选合同设计，不是自动注入功能已经存在。#215/#216/#217 的阶段顺序与职责分离已按当前 Issue 核对。citeturn2view0turn2view1turn2view2turn2view3
- 仓库当前 `openspec/changes/` 没有 active change；`openspec/config.yaml` 要求中文 OpenSpec artifacts、spec 只写 observable behavior、design 区分现状与迁移决定。主要现行证据包括 [agent-integration spec](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md:8>)、[practice-read spec](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/practice-read/spec.md:8>)、[get 测试](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.test.ts:77>)、[Hook 测试](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.test.ts:44>) 及 [Plugin 配置测试](<https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/scripts/hooks-config.test.ts:6>)。本轮检查了代码与测试，没有执行测试。
- Lorelum semantic query 返回 `backend.state-invalid`，原 trace 没有可用记录；显式 keyword 诊断也失败，且 trace 为空。依照 Lorelum recovery 规则，没有预检或重启 Backend。直接 `lore get` 成功读取了与委派 handoff、最小设计、范围边界和验收证据相关的完整 Practices。
- 本方案文字是评审稿，不是已落盘的 OpenSpec 文件，也没有修改 Issue 或代码。

## 实际交给 Sol 子代理的 prompt

调用参数：一个子代理，`gpt-6-sol`、`xhigh`，未继承无关上下文；其任务消息原文如下。

<details>
<summary>展开查看完整 prompt</summary>

```text
你是本次唯一的技术方案作者。请只基于下列用户授权、Issue 事实和已核对仓库证据，完成 Lorelum Issue #214 的可评审技术方案。不要实现代码、不要运行会产生外部状态的操作、不要更新 GitHub Issue/PR、不要提交；也不要再 spawn/delegate 子代理。可为未被此证据包覆盖且会改变结论的事实做必要的窄范围核实，但不要从头漫游仓库或重复 broad search。正确性优先于省 token。

## 任务 / 权限边界
用户要求：为 https://github.com/lorelum/lorelum/issues/214 做可评审技术方案，不实施。方案作者必须是一个 gpt-6-sol、xhigh subagent；本任务就是这一个 subagent。用户要求 main agent 先完成取证并给你足够的决策材料，不要只传搜索关键词。所有 OpenSpec 正式 artifact 用中文；OpenSpec 结构标题和 SHALL/MUST 保留英文（见 `openspec/config.yaml`）。

当前 workspace 在受限只读 filesystem 下；不要尝试写仓库文件。请在最终响应中给出可以评审的完整中文方案文本，并说明如果后续允许写入，建议的 OpenSpec artifact 路径/组成。不能把只读限制伪装为已经写了文件。请将方案视作 Proposed，不把未来行为说成已实现或 Accepted。此设计请求授权规划，不授权实现。

## Issue 原文和阶段关系（截至 2026-09-24）
#214 目前 OPEN，title: `[Design] 定义跨 Agent 的对话级 Practice 候选簿与宿主会话绑定合同`。正文原话的核心合同：
- `lore get <practice-id>` 仅证明某个 Agent 曾成功读取/考虑过一条 Practice；不证明已采纳、有效，也不证明适用于当前子任务。候选簿是“可能相关、不完整”的上下文信号，不是事实清单或 workflow engine。
- 先定义 host-neutral `ConversationPracticeLedger` 语义与 session binding contract，再由 #215/#216 分别实现记录和注入。
- 必须定义记录语义、最小字段、session scope、读取与子 Agent 可见性、并发/幂等/dedupe、内容更新/多 source 行为、clear/resume/过期清理、失败不阻塞已存在任务、与 Pack/LocalStore 的边界；标明永不保存完整 Practice body/raw prompt/transcript/private query。
- 必须以 `hostKey + opaque root-conversation reference` 隔离；明确 host adapter/CLI runtime 的职责以及不能由 Host metadata 推断的内容。
- 不得把候选状态混入 Pack LocalStore、Engine retrieval state 或 global current-conversation file；不新增 local MCP/MCP-backed Plugin；不把候选提示当作对新实质材料重新 query/get 的替代。
- Acceptance: 写出 host-agnostic ledger/session identity interface 与数据模型；定义 parent/child visibility、session reset/expiry、bounded retention、失败隔离；列出最小端到端实现阶段、测试和验证。
- #214 Issue 正文还明确：记录 ledger 不得改变已成功的 `lore get`；候选元数据应至少支持 `practiceId`、`contentDigest`、可选 `appliesWhen`、来源 Pack identity；它与 #28/#32/#35/#204 相关，但不合并这些 Issue 的各自职责。

阶段链必须保持分开：
- #214 是本次唯一设计范围。
- #215 `[Feature] 记录可验证的主 Agent Practice 读取到对话候选簿`：依赖 #214，要求只记录可验证成功且可归因的 main-Agent `lore get`，幂等去重 `(practiceId, contentDigest)`，不写完整正文/原始查询/会话正文；失败、不完整、无法归因应不记录且不能改变 `lore get` 原结果；测试成功/不存在/失败/截断/取消/复用 digest 和多源。
- #216 `[Feature] Codex 在 SubagentStart 注入父会话的 Practice 候选提示`：依赖 #215；仅对子 Agent startup 注入父会话已记录候选 metadata、bounded/optional/fail-open；不自动 `query/get`、不注入 body、子 Agent 读取不回写父账本；验证父子/嵌套/并发/无关联/预算/CLI 不可用。
- #217 是 `[Backlog] context compaction 后恢复对话级 Practice 候选提示`，明确延期；#214/#215/#216 不能把它偷并为 compact recovery 阶段。已有 SessionStart 的 Pack Catalog 恢复不是候选簿恢复。
- #204 是针对新实质材料/约束/范围变化重新检索的独立决策；候选账本不得抑制 re-query。
- #28/#32/#35 是关联但独立的任务（respectively broad Practice-judgment context, pre-compaction capture/recovery line, scenario/scope work）。#214 不应重定义它们。

Issue sources to verify if needed:
- `https://github.com/lorelum/lorelum/issues/214`
- `https://github.com/lorelum/lorelum/issues/215`
- `https://github.com/lorelum/lorelum/issues/216`
- `https://github.com/lorelum/lorelum/issues/217`
- `https://github.com/lorelum/lorelum/issues/204`
- Related independent #28/#32/#35.

## 当前仓库证据（2026-09-24 workspace；不得用旧 checkout/旧计划覆盖）
工作区根：`https://github.com/lorelum/lorelum/tree/88e691ba939943e67793e92e1899387971dbce36`。`git status --short --branch` 显示干净、`HEAD (no branch)`（系统临时目录权限警告未影响 status 输出）。`openspec/changes/` 下没有 active changes，只有 archive；不要把 archived artifact 当成当前合同。只要引用这些文件，用它们说明代码/合同，不要要求 main 重读整仓：

1) `openspec/specs/agent-integration/spec.md`
- L8–16：当前本地 integration 是 CLI-first；Plugin/Skill/Hook 必须通过 CLI JSON contract，不得直读 LocalStore/导入 Engine/Backend/复制 retrieval 语义；Catalog 是 routing metadata；Hook 保持 metadata-only，不能自动 `lore query`/`lore get`。
- L37–42：明确禁止 local MCP、stdio、MCP tools、MCP-backed Plugin；只有独立批准的 remote-retrieval 才重议。
- 设计 ledger 需要解释：metadata-only 对“Hook 做身份传递/ledger record”究竟允许到何种程度；不能静默弱化现有规则。
2) `openspec/specs/practice-read/spec.md`
- L8–20：`lore get` 是经 Engine 对选定 LocalStore 的一致、完整 canonical point read；输入和错误合同受保护。
- L33–38：keyword query 和派生 index 仍从 Store snapshot 得出，不能成为 canonical state。
3) `openspec/specs/plugin-distribution/spec.md` L15–20：Plugin 通过已安装 CLI；Plugin 不持有 Store/Backend/ranking。
4) `packages/cli/AGENTS.md`：CLI 是解析/组合/JSON envelope 与 route 选择边界；不能把 retrieval/Store 规则移入 CLI；每个 CLI behavior 需 colocated bun:test，current contract changes 要改对应 CLI docs。
5) `packages/engine/AGENTS.md`：Engine owns canonical LocalStore/retrieval/derived state；Store root 的 Pack/index 内容不承载 model/runtime state；若触及 SQLite/Drizzle 要遵守 Engine persistence 规则。该指引加强“不放 Engine/LocalStore”，不意味着应在 Engine 下加 conversation ledger。
6) `openspec/config.yaml`：语言 Chinese；Proposal 要说明是记录现有还是新增行为；Specs 只写 observable behavior；Design 分开 observed/migration/deferred，requirements 映射现行 code/test；archive 仅为历史证据。当前 OpenSpec process 由 repo instructions 要求，不能创建 competing docs hierarchy。

### 读取路径和现状
- `packages/cli/src/get/get-command.ts` L19–22/L38–58：`lore get` 依赖 `LocalStore.getEffectivePractice`，成功返回完整 canonical Practice、`contentDigest`、sources 的 `packName`/相对 `sourcePath`；没有 host/session identity，也没有写候选状态。
- `packages/cli/src/get/result-schema.ts` L8–60：公开 get schema 包含完整 `practice.body` 与 `anti_patterns`、digest、sources。
- `docs/cli/get.md` L22–49：正常成功 envelope `ok:true`，退出 0；失败 `ok:false`，退出 2；完整正文；多个 Pack 相同 ID+相同 digest 会返回确定排序的全部来源；sourcePath 是相对路径，不提供 Pack version；`contentDigest` 是 SHA256 canonical content，不是 publisher signature。
- `packages/cli/src/get/get-command.test.ts`：对有效结果返回完整 Practice/digest/sources；invalid ID 不开 Store；不存在 -> `practice.not-found`；Store/runtime errors 映射稳定且不泄露内部路径；describe contract；没有 ledger/side effect coverage。
- `packages/cli/src/hook/codex.ts` L13–17：Hook 类型只声明 event 字段；L~77 onward `createCodexHookResponse` 只支持 `SessionStart`，调用 list-details renderer。运行层收到 JSON but does not consume session/agent fields; fail-open writes `{\"continue\":true}` and returns exit 0. `run()` raw Hook path is intercepted in `packages/cli/src/main.ts`.
- `packages/cli/src/hook/codex.test.ts`：只测试 SessionStart Catalog、bad JSON/unsupported event/Store error 的 fail-open 和 `--store-root`; 不覆盖 SubagentStart/PostToolUse/ledger identity.
- `plugins/lorelum/hooks/hooks.json`：唯一注册 `SessionStart`, matcher `^(startup|resume|clear|compact)$`, `lore hook codex`, `additionalContextLimit:5000`，Unix 与 Windows wrapper 都 fail-open。
- `plugins/lorelum/scripts/hooks-config.test.ts` L6–35：测试现有唯一 SessionStart hook、生命周期 matcher、5000 context limit，且禁止 PostCompact event；测试 L37 onward 对 forwarding/error wrapper 做结构级/进程级 mock，不是实际 Codex Hook event capture。
- `docs/cli/hook.md` L1–30：当前 `lore hook codex` 是版本化 ABI，只支持 SessionStart、只输出 Catalog context，不自动 query/get、不开 Backend、不改 Store；invalid/error emits continue:true。
- `docs/development/plugins.md` 和 `docs/plugins/codex.md`：source boundary CLI-first；CLI owns protocol and Catalog renderer, Plugin owns lifecycle matching/Skill; compact currently restores only Catalog via SessionStart source `compact`; current host Hook shouldn't be conflated with candidate ledger.
- `packages/config/src/paths/lorelum.ts` / config tests：default product data root is `~/.lorelum`; `packages/engine/src/local-store/lifecycle/local-store.ts` L26–34 confirms `defaultStorageRoot()` points to that root. `lore get` allows `--store-root` to select another Pack LocalStore. This does NOT prescribe ledger path; candidate session-state must not silently follow arbitrary `--store-root`.
- `packages/cli/package.json`: CLI owns command surface and depends on `@lorelum/config`, `@lorelum/engine`, but no dedicated candidate/state module exists. Root workspaces are `packages/*`, `apps/*`; there is no second local Plugin directory in this checkout beyond `plugins/lorelum`.
- Repository-wide `rg` for candidate ledger / `ConversationPracticeLedger` / `SubagentStart` / candidate terms excluding archived OpenSpec returned no current implementation/spec. Current specs are only agent-integration/practice-read/plugin-distribution; no active OpenSpec change.

### Existing code contract boundary and design tension
Current `get` is a read-only public command by behavior and spec. If design suggests hidden auto-recording inside get, it must explicitly reconcile that public read contract, the `lore get` must not change result requirement in #214/#215, local CLI contract/spec, and host identity absent from its args. Alternative host-event side-channel or explicit metadata-only runtime command also needs evidence and tradeoff; do not assume one is safe. Likewise #216 Hook can call `lore` CLI but must not bypass it, auto query/get, or hold Store.

### Storage path/persistence evidence and limits
- `.lorelum` is user root for Pack LocalStore/config; current Engine LocalStore has transactional/journal lifecycle/SQLite/Drizzle at its own boundary. Do not equate “same top directory” with “same ledger/LocalStore,” but explicitly assess physical and ownership separation.
- No reusable CLI conversation-state store or public generic state path contract found. Existing `packages/config/src/paths/lorelum.ts` only resolves the root and config file. CLI itself does not currently own a durable runtime DB. Engine uses Drizzle for LocalStore; it must not own conversation semantics.
- Hence exact storage medium, location, lock/transaction mechanism, TTL/cap/cleanup policy are design decisions. Compare the smallest plausible options (e.g. isolated atomic per-conversation files vs a separate transactional store) against concurrent command/hooks, crashes, cleanup, boundedness, permissions, migration, and package ownership. Do not copy an older Proposed sketch or assert one approach was accepted.

## Official external evidence (web checked 2026-09-24; primary source only)
Official OpenAI Codex Hooks docs: `https://developers.openai.com/codex/hooks` (page may redirect/auto-localize; use the English canonical URL/section headings and quote only paraphrases):
- Event list currently includes `PostToolUse`, `SessionStart`, `SessionEnd`, and `SubagentStart`.
- Common input documents `session_id`; it says subagent hooks use the parent session ID.
- `SubagentStart` event-specific input documents `agent_id` and `agent_type`; the event supports `additionalContext` injection.
- `PostToolUse` input documents `tool_name`, `tool_input`, `tool_response`, and `tool_use_id`; it fires after tool execution including non-zero command outcomes. The listed schema does not separately document a `lore get` success boolean, caller role, shell-level composition semantics, or guarantee complete output capture. Do not claim runtime attribution is solved from public schema alone.
- Official docs support feasibility of hook-triggered context/side-channel, not task-text retrieval. `SubagentStart` listed fields do not include subagent task prompt text; exact root mapping for nested events and `session_id` stability across clear/resume needs actual runtime evidence.
- `SessionEnd` is supported, but docs describe it for the main thread, with lifecycle triggers; do not treat it as guaranteed crash-proof cleanup. TTL/cleanup fallback remains a design decision.
Citations observed in web search during parent turn: issue sources and OpenAI official hook page; you may re-open just those exact URLs if you need a narrow check. No third-party Hook documentation should override the official page.

## Lorelum retrieval guidance already checked
Injected Catalog indicates `agentic-coding` Pack is relevant. Main agent attempted one targeted semantic query; it failed `backend.state-invalid`, then trace read returned `records:[]`, `missingEvidence:[]`, `truncated:false`. After reading the installed Skill recovery reference, explicit `--mode keyword` diagnostic also failed `runtime.unexpected` with trace id, and its same-trace logs were empty. Do not preflight/restart/repair Backend. Direct `lore get` worked for these complete Practices:
- `agentic-coding.context.give-delegated-agents-decision-context`: handoff must carry user-controlling specification, task outcome, unfindable repo facts, constraints/non-goals, decisions/rejections, scope, recipient deliverable/checks/escalation; don't send only task label; receiver still verifies current files.
- `agentic-coding.implementation.choose-smallest-sufficient-design`: first list behavior/rules and reject deficient options; count responsibility, state, indirection, fallback and I/O costs; require a present reason for extra layer/state; do not remove required protection just to minimize code.
- `agentic-coding.planning.decide-scope-and-stop-conditions`: label required/optional/out of scope/unresolved; each required behavior gets observable finish condition; keep deferred set and replanning evidence explicit.
- `agentic-coding.verification.map-evidence-to-acceptance`: evidence supports only criteria it actually observed; identify untested behaviors rather than activity totals.
Retrieval gap is a local tool/backend issue; none of it proves no relevant Practice exists. Do not let it block solution drafting.

## Required plan content / decision work
Produce a decision-bearing proposal, not a broad architecture wishlist. Mark Observed / Required / Proposed / Deferred clearly. The proposal must cover:
1) Brief conclusion and rationale; scope; non-goals; issue/workstream mapping (#214 versus #215/#216/#217/#204).
2) End-to-end current vs target lifecycle: successful attributed read → record or reject → lookup under parent conversation → host adapter builds bounded advisory context. Distinguish read, considered, adopted, effective, and received.
3) A minimal host-neutral interface and data shape sufficient to review: host scope (`hostKey`, opaque root conversation reference and any justified epoch/reset token); candidates keyed/deduped by Practice identity/digest; source provenance; timestamps; read API visibility; record/clear/expire semantics; typed errors/fail-open. Avoid full implementation or invented protocol fields.
4) Design comparisons and one justified recommendation on ownership/storage, proving it neither enters Pack LocalStore/Engine retrieval state nor becomes Codex-owned semantic state. Assess command/public contract and how record operation preserves `lore get` read result. Explicitly note CLI metadata-only/side-effect tension.
5) Bounded retention/expiry, concurrent writes, reset/resume fencing, digest changes/multi-source, invalid/ambiguous payload, Store/runtime failure. Any numeric budget must be labeled proposed/testable and include calibration basis, not presented as observed.
6) Parent-child and nested visibility: define which parent's ledger can be read and reject sibling/other-conversation leakage. Candidate hints remain optional/incomplete and never suppress #204-style re-query.
7) Explicit unknowns/evidence gates: real payload capture for success/failure/not-found/cancelled/truncated `lore get`; multi-command/composite shell; main vs subagent and nested subagent; clear/resume/late event ordering; complete PostToolUse result and SessionStart/SubagentStart data. State exactly which next phase must verify these before automatic write/injection. Don’t use cwd, global current-session file, transcript, shell regex or task text from undocumented fields as identity/evidence.
8) Later tasks and test matrix mapped to acceptance, clearly assigning #214 (contract/design only), #215 (record evidence), #216 (Codex injection), #217 (deferred compaction recovery). Include failure-open behavior without changing existing `get` results.
9) Required files/artifacts as Proposed only. If OpenSpec is the repo process, suggest a self-contained change under `openspec/changes/<clear-slug>/` including `proposal.md`, `design.md`, `tasks.md`, and only necessary spec delta(s); do not edit `openspec/specs/*` now or create parallel docs hierarchy. Keep tasks about later implementation, not claim complete.

Do NOT assume or pre-choose:
- JSON vs SQLite vs other persistence; physically sibling under `.lorelum` versus another app-data path;
- ledger state in CLI vs a shared new package, or CLI command name/API;
- `get` side effect vs Host PostToolUse handler vs explicit `lore` record command;
- TTL, entry cap, clear epoch/token model, or digest-upgrade replace vs coexist;
- whether a Codex Hook payload is sufficient for reliable successful main-Agent attribution or nested root binding.
Compare real viable options against present evidence and #214’s required scope. You can recommend one only after explaining the evidence and consequences, and can leave an item deferred only with a concrete evidence gate.

## 输出格式
用中文输出完整方案，简洁但足以实施/评审，正文应可直接成为 OpenSpec `design.md` 内容；再附拟议的 `proposal.md` 摘要、spec delta 文本和后续 `tasks.md`（如需要），并将实际需要审批的未决事项单列。引用上述具体路径/行或官方文档标题/章节。每项验收都映射到行为与可证明的 test/evidence；不要写实现完成声明。你的答复一次成形；如存在关键不可裁决项，报告事实/选项/影响，不向用户开启多轮搜索流程。
```

</details>

## 主要取证来源

- GitHub Issue #214 原文，以及 #215（成功读取记录）、#216（subagent 注入）、#217（compact 候选恢复延期）和 #204（新实质材料重新检索）的分工核对。citeturn2view0turn2view1turn2view2turn2view3turn2view4
- OpenAI 官方 Codex Hooks 文档：确认 `SubagentStart`、`PostToolUse`、会话字段、`additionalContext`、`SessionEnd` 生命周期，以及这些公开字段不能单独证明本项目所需的真实读取归因。citeturn3view0turn3view1turn3view2
- Bun 官方 SQLite 和编译执行文档：验证独立 SQLite 在当前运行时中可行，但不证明候选簿一定应采用它。citeturn0search1turn0search0
- 本地已检索到的完整 Practices：delegated-agent 决策上下文、最小充分设计、范围/停止条件、证据映射。语义 query 和 keyword query 均有运行时错误；这不证明没有其他相关 Practice。

