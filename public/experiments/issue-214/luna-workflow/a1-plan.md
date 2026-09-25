按你的要求，只启动了 **一个 `gpt-6-sol`、`xhigh` 规划 subagent**，没有二次委派。它完成了方案，我又对照当前源码、测试和仓库持久化规范做了复核。

Issue #214 的范围是定义跨 Agent 对话候选簿的职责、数据、隔离和生命周期；#215、#216 分别涉及自动记录和子 Agent 注入，#217 的 compact 恢复应继续延后。citeturn1view0turn1view1turn1view2turn1view3

**结论先说：**建议让 CLI runtime 统一定义候选簿语义，由 Host adapter 提供经过验证的根会话和 actor 身份。第一阶段只允许“已证明属于根 Agent 的成功 `lore get`”写入；可证明属于该根会话的子 Agent 可以读取，但不回写父簿。候选表示“读过、可能相关”，不是“已采纳”或“已验证有效”。

Codex 官方 Hooks 文档提供了有用但不足以替代真实运行证据的边界：文档称子 Agent Hook 使用父 `session_id`；`PostToolUse` 列出的字段没有 `agent_id`，而 `SubagentStart` 列有。当前仓库没有真实的 `PostToolUse` / `SubagentStart` payload 样本，因此在实际字段证明根 Agent 归属、成功结果和 clear epoch 之前，自动写入必须 fail-closed。citeturn3view0turn4view0

仓库采用 `spec-driven` OpenSpec，并要求文档中文、结构标题和 `SHALL/MUST` 关键字用英文。当前环境对仓库是只读权限，所以**下面是完整的待评审、待落盘草稿；没有创建文件、修改代码或运行实现测试**。建议目录为 `openspec/changes/conversation-practice-ledger-contract/`。

## `proposal.md`

# Proposal

## Why

当前 Lorelum Hook 只提供 Installed Pack Catalog，没有 conversation-level Practice 候选簿。主 Agent 曾成功读取的 Practice 可能仍与任务相关，但 CLI runtime 和 Host adapter 尚无共同合同来表示“读过但未证明采纳”的候选，也没有跨宿主、跨进程的会话隔离、父子可见、clear/resume、并发更新和保留规则。

这项能力必须区分“读取证据”和“采纳或效果证据”。候选簿不应成为第二份 Practice 正文库，也不应改变 Pack LocalStore、检索语义或 Host Plugin 的存储责任。

## What Changes

新增对话级候选簿的设计合同：

- 以 `hostKey`、opaque 根会话标识和会话 epoch 隔离状态；Store 范围作为候选记录的隔离维度。
- 只记录最小 Practice 元数据：ID、读取时的 content digest、标题、可选 `appliesWhen` 和完整的来源 Pack 身份。
- 规定相同 ID 的重复读取和 digest 更新如何去重、覆盖与排序。
- 规定根 Agent 写、已证明的子 Agent 读、clear/resume、迟到写栅栏、并发及有限保留的语义。
- 规定 Host adapter 必须用真实 Hook payload 证明来源、成功状态和会话关联；证据不足就不读写。
- 在物理持久化选型获 Owner 评审前，只冻结 service 与 storage port 的合同，不默认 CLI 可直接创建 SQLite、SQL 或迁移基础设施。

这是**新增产品行为的设计提案**，不是现有功能的记录。自动记录、子 Agent 注入和 compact 恢复不由本 change 实施。

## Non-Goals

- 不保存 Practice body、prompt、transcript、完整 `tool_response`、shell 命令或绝对文件路径。
- 不记录“已采纳”“有效”“完成”或质量评分。
- 不自动运行 `lore query` / `lore get`。
- 不将候选状态放进 Pack LocalStore、Engine retrieval state 或 Codex Plugin 私有文件。
- 不引入本地 MCP、第二套 runtime、跨设备同步或通用插件体系。
- 不把 #215、#216、#217 合并进本期实现范围。

## Risks

当前公开 Hook 文档没有证明实际运行时的 actor、`get` 成功结果和 clear 前后的 epoch 关联。Hook 文档字段只是接口描述，不能当作本机 payload 证据。候选簿的原子合同可以先评审，但自动记录仍以脱敏的真实 payload 证据为前置门槛。

SQLite + Drizzle 与 JSON + 原子替换/跨进程锁是两条可行路径，但当前没有明确归属 CLI conversation state 的持久化基础设施。物理选型属于 Owner review decision，未批准前不得默认为 raw SQLite in CLI 合规。

## Acceptance

评审后的方案应清楚定义候选含义、身份和 Store 隔离、重复/更新、多来源、根子会话可见范围、clear/resume、并发栅栏、保留策略、失败降级、物理存储决策以及真实 Hook 证据门槛。所有实现任务仍保持未开始。

## `design.md`

# Design

## Observed

当前 `lore get` 已有可复用的结构化数据：`practice.id/title/applies_when`、`contentDigest` 和 `sources[{packName,sourcePath}]`。成功结果由 CLI JSON envelope 输出；当前 sources 不包含 Pack version。digest 是 canonical JSON 的 SHA-256 标识，不是签名。候选簿不需要另造 `get` 结果格式，也不应复制 Practice body。  
证据：[get-command.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.ts:43)、[result-schema.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/result-schema.ts:9)、[protocol.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/output/protocol.ts:90)、[canonical-practice.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/engine/src/local-store/model/canonical-practice.ts:47)。

同一 Practice ID 的多个 Pack 来源只有在 digest 相同的情况下才合并为一个 Effective Practice；digest 不同会产生冲突。因此，候选记录应保留读取时的 digest 和该次读取的全部来源，而不能把多个历史读取的来源无限混成一个当前状态。  
证据：[effective-practices.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/engine/src/local-store/model/effective-practices.ts:60)。

当前 `lore hook codex` 仅支持 `SessionStart`，Plugin 也只注册该事件。现有 Hook 测试使用模拟输入，没有真实的 `PostToolUse` / `SubagentStart` payload。Hook 对错误采用退出码 0 和 `{"continue":true}` 降级，能作为后续非阻塞集成的惯例，但不能证明新事件合同。  
证据：[codex.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts:13)、[hooks.json](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/hooks/hooks.json:4)、[hook.md](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/hook.md:11)。

现有 agent-integration 合同要求 CLI-first，Hook 保持 metadata-only，且不自动 query/get；不提供本地 MCP。  
证据：[agent-integration spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md:8)。

## Required

候选簿需统一定义：

- 根会话和 Host 隔离，以及所选 Store 的隔离。
- Practice ID 去重和 digest 更新。
- 多来源 Practice 的身份呈现。
- 根子 Agent 可见范围和子 Agent 写入规则。
- startup、clear、resume、迟到回调的 epoch 语义。
- 并发写入、事务可见性和崩溃语义。
- 元数据预算、条目上限和过期清理。
- 任何缺少来源证明或 storage 失败时的 fail-open 行为。

## Proposed Ownership

Host adapter 解析宿主事件，并只在能够证明时提供根对话、actor、成功 `get`、目标 ID 和 Store 范围。它不得通过 `cwd`、transcript 路径、全局“当前会话”文件或 shell 文本推断身份。

CLI runtime 拥有通用候选簿的规则：身份、去重、更新、可见性、清理和失败语义。持久化 adapter 只实现原子读写，不解释 Practice 语义。Pack/LocalStore 仍拥有 canonical Practice 与来源数据；Hook 入口只解析输入、组装响应并 fail-open。第二个真实 Host 出现前，不另建 workspace package。

## Candidate Contract

候选字段建议为：

```ts
type RootBinding = {
  hostKey: string;
  rootConversationRef: string;
  epoch: string;
};

type CandidateReceipt = {
  practiceId: string;
  contentDigest: string;
  title: string;
  appliesWhen?: string;
  sourcePackNames: readonly string[];
  storeScopeKey: string;
};

type CandidateReadResult = {
  candidates: readonly CandidateReceipt[];
  omittedCount: number;
};

interface ConversationPracticeLedger {
  openRoot(input: {
    hostKey: string;
    rootConversationRef: string;
    reason: "startup" | "resume" | "clear";
  }): Promise<RootBinding>;

  recordRootGet(
    binding: RootBinding,
    receipt: CandidateReceipt,
  ): Promise<"added" | "replaced" | "unchanged" | "stale-binding">;

  listForProvenDescendant(
    binding: RootBinding,
    storeScopeKey: string,
    budget: { maxItems: number; maxUtf8Bytes: number },
  ): Promise<CandidateReadResult>;

  endRoot(binding: RootBinding): Promise<void>;
}
```

这只是内部设计合同，不是现有公共 API，也不应暴露接受任意 `--host`、`--session`、`--actor` 的通用写入命令。

候选唯一键建议为 `(root binding, storeScopeKey, practiceId)`：

- 同 ID、同 digest 的重复成功读取保持一条记录并更新最近成功观测时间。
- 同 ID、新 digest **替换**旧摘要，不保留版本历史；候选簿不是阅读审计日志。
- 每次写入保存那次 `get` 的来源 Pack 名完整集合，去重并稳定排序；不跨历史读取合并来源。
- `sourcePackNames` 是必需字段；若完整来源集合无法读取或超出记录预算，跳过整个候选，不能截取一部分来源造成假完整。`appliesWhen` 可整体省略，但必须能识别其被省略。
- 不存 `sourcePath`、Pack version、Practice body 或其他未必要字段；当前 `get` 结果不提供 Pack version，禁止自行推断。
- `contentDigest` 表示读取时的 canonical 内容标识，不证明该版本仍安装或仍是当前内容。需要使用时，子 Agent 仍通过正常 `lore get` 验证。

`storeScopeKey` 应是本机不透明范围键，不包含绝对路径，并从该次调用实际解析出的 Store 范围计算。若 Hook 无法证明此次 `get` 使用的是哪个 Store，候选不写入。Store identity 算法及其跨平台稳定性需纳入 #215 的证据门槛。

## Root Binding、Parent/Child 和 Lifecycle

`RootBinding` 的 epoch 用来阻止 clear 后的迟到写污染新对话：

- `startup` 仅在确认新根对话后创建 binding；重复 startup 不能被当成 clear。
- `resume` 只在确认同一 Host、同一根对话且仍在保留期内时沿用 binding，否则返回空簿。
- `clear` 原子旋转 epoch 并删除旧 epoch 候选。携带旧 binding 的迟到写入返回 `stale-binding`。
- Host adapter 必须让后续事件关联到它发生时的 epoch。若宿主只能给出“当前 epoch”，无法把旧 `PostToolUse` 与 clear 前的 binding 区分，则该宿主自动记录保持关闭。
- 子 Agent 必须被证实属于同一根会话和同一 Store scope 才能读取。子 Agent 自己的成功 `get` 不回写父候选簿。其他 Host/root、未证明关联或不同 Store scope 返回空结果。
- compact 状态恢复不属于本期。子 Agent 自有候选、向父显式提升和跨设备同步也延后。

根会话和 epoch 不应被拼成文件路径或写入诊断日志；持久化只使用安全的不透明 key。没有有效 binding 时，不得尝试读某个“最近的”ledger。

## Concurrency、Persistence 和 Retention

与后端选型无关的原子保证：

1. `recordRootGet`、`clear`、过期清理和容量裁剪在同一原子边界内提交。
2. 并发写按持久化提交顺序决定结果；读者只能看到完整的旧快照或完整的新快照。
3. clear 旋转 epoch 后，旧 binding 写入必须拒绝，不得重新创建旧候选。
4. 过期记录在读取时不可见，物理垃圾回收在后续 ledger 操作中完成；硬会话上限保证无活动期间磁盘记录数量仍有上界。
5. 事务或文件锁内不调用 CLI、Store、网络或模型。

初始保留值可作为评审用的**试行参数**，不是已测得的性能目标：每个根对话最多 32 个候选、候选元数据最多 64 KiB、最多保留 512 个根对话、根候选闲置 30 天过期。子 Agent 读取按调用传入 `maxItems/maxUtf8Bytes`；返回 `omittedCount`，且先按最近的根成功读取排序，再用 Practice ID 稳定打破并列。超出预算时可省略可选 `appliesWhen`；不可截断 ID、digest 或来源集合。最终阈值需通过真实 Hook 延迟、磁盘占用和 #216 上下文预算测量后冻结。

物理持久化在 #214 保留为 Owner review decision：

| 选项 | 优点 | 需评审的问题 |
|---|---|---|
| CLI 专属 JSON + 原子替换和跨进程锁 | 不新增关系型 schema / migration；service 和 adapter 可留在 CLI 内部。 | 必须证明锁竞争、旧锁回收、PID 重用、clear 与迟到写、rename 崩溃、Windows 行为和容量裁剪原子性；不能把它简化成“写临时文件再 rename”。 |
| 独立 SQLite + Drizzle adapter | 唯一键和事务直接支持并发 upsert、clear 与裁剪；必须是独立文件，不能并入 Pack LocalStore 或索引。 | 仓库现有 Drizzle connection/migrator 在 Engine，且文档规定不新建 package-local migration runner / 第二 connection factory，也不允许 CLI 命令直接发 SQL。必须明确复用或提取 persistence foundation，或显式修订该规则；不能默认 raw SQLite in CLI 合规。 |

仓库持久化规范规定 Engine 数据库共享 connection/migrator、关系型 CRUD 优先 Drizzle，并要求 CLI 使用 repository/adapter，而非直接 SQL。它没有定义新 conversation-owned 数据库的合规落点。故建议本方案先定语义和 storage port，把**物理后端及其代码归属列为实现前的 Owner 决定**；SQLite + Drizzle 是可评审优先候选，但不是已批准实施路线。  
证据：[persistence.md](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/development/persistence.md:19)、[CLI AGENTS.md](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/AGENTS.md:7)。

## Failure Semantics

候选数据是可选提示，不可影响原 `lore get` 或 Agent 工作流：

- 身份、成功状态、Store scope 或 epoch 无法证明：不读或不写。
- 候选簿损坏、锁忙、schema 不兼容、容量裁剪失败：跳过记录或返回空簿，发出不含敏感字段的诊断。
- 不把错误伪装成“没有相关 Practice”的检索结论，也不做自动 query/get。
- Hook 异常继续使用 fail-open 响应，不能改变已完成的 `lore get` 结果。
- 公开 Hook 是同步调用，因此仍需设短超时并测端到端延迟；fail-open 不代表零延迟。

## Open Questions

1. 真实 Codex payload 是否足以分辨 `PostToolUse` 的根 Agent / 子 Agent 来源、成功退出状态、Store scope 和 clear epoch？
2. 候选簿采用 SQLite + Drizzle 时，应如何复用现有 persistence foundation 而不将会话状态放入 Engine retrieval state？若 JSON，是否接受锁和崩溃回收的实现与验证成本？
3. 32/64 KiB/512/30 天的试行上限是否合适？需在 #216 的上下文预算和实际 Hook 运行成本测量后确认。

## Validation Plan

- Service contract tests：host/root/store 隔离；重复 ID；digest 替换；多 Pack 来源完整排序；预算超限；clear epoch；resume / TTL；子 Agent 只读；未知 binding 空结果。
- Persistence contract tests：并发 read/write/clear；崩溃重开；损坏或不兼容；过期和容量物理清理；不同平台文件/锁行为。
- Host contract tests：保存已脱敏的真实 payload fixture；覆盖根/子 Agent `lore get` 成功、失败、取消，简单命令和包装 shell，clear/resume、旧事件迟到及 `--store-root`。
- Hook integration tests：账本失败时原命令不变，子 Agent 继续；测量 Hook 启动与注入的 p50/p95 和上下文字节预算。

这些是计划中的可观察验收标准，尚未执行。

## `specs/conversation-practice-ledger/spec.md`

## ADDED Requirements

### Requirement: Conversation-scoped Practice candidate metadata

候选 SHALL 表示：根 Agent 在指定 `hostKey`、opaque root conversation 和 Store scope 中，成功读取过一个 canonical Practice 版本。候选 MUST 被呈现为“可能相关”而非“已采纳、有效或当前版本”。Ledger MUST 只保留 `practiceId`、读取时的 `contentDigest`、标题、可选 `appliesWhen` 和完整的来源 Pack 名集合及本地必要的时间/隔离元数据。候选 MUST NOT 保存 Practice body、prompt、transcript、shell 命令、完整 tool response 或绝对路径。

#### Scenario: Read-time snapshot is not an adoption claim

- **WHEN** 根 Agent 成功读取 Practice 并生成候选
- **THEN** 候选 SHALL 保存读取时的 digest 与摘要字段，且 MUST NOT 表示该内容已被采用、验证有效或仍是当前 Pack 版本

#### Scenario: Multiple identical Pack sources

- **WHEN** 一次 `lore get` 返回相同 ID、相同 digest 的多个来源
- **THEN** 候选 SHALL 记录全部唯一来源 Pack 名并稳定排序；不得拆成重复候选或丢失来源

#### Scenario: Candidate exceeds metadata budget

- **WHEN** 必需字段或完整来源集合超出单候选的字节上限
- **THEN** runtime MUST 跳过整个候选；可选字段可以整体省略并明确标记，不得产生不完整来源集合

### Requirement: Verified root binding and descendant visibility

Ledger operations SHALL require显式的 `hostKey`、opaque 根会话标识和 epoch binding。候选只对相同 Host、root、epoch 和 Store scope 可见。Host adapter MUST 在成功记录前证明 actor 为根 Agent、原命令成功且结构化 `get` 结果目标 ID 匹配；未经证明的身份或来源 MUST fail-closed。经证明属于同根的子 Agent MAY 读取候选，但首期 MUST NOT 将子 Agent 读取回写父 ledger。

#### Scenario: Unknown actor attribution

- **WHEN** Hook payload 无法证明某个成功 `get` 由根 Agent 发起
- **THEN** runtime MUST 不写入候选，并 MUST 不通过 cwd、transcript、shell 文本或全局当前会话状态猜测

#### Scenario: Proven descendant reads candidates

- **WHEN** Host adapter 证明子 Agent 属于同一 root 和 Store scope
- **THEN** runtime MAY 返回受限候选摘要；子 Agent 的读取 MUST NOT 修改根候选 ledger

### Requirement: Deterministic update, clear, and resume

每个 root epoch 内，候选 SHALL 以 `(storeScopeKey, practiceId)` 唯一。同 digest 的重复成功读取 SHALL 更新最近读取顺序但不得增加记录；不同 digest 的读取 MUST 替换该 ID 的旧摘要而非保留历史版本。`clear` MUST 原子旋转 epoch 并拒绝旧 epoch 写入；`resume` 只有在证明同一未过期 root 时 SHALL 恢复；身份不明时 SHALL 返回空簿。

#### Scenario: Delayed write from cleared epoch

- **WHEN** `clear` 已旋转 epoch 且旧 Hook 回调稍后尝试写入
- **THEN** runtime MUST 返回 `stale-binding`，且 MUST NOT 复活旧候选

#### Scenario: Resume after retention expiry

- **WHEN** root 会话恢复时其候选已过期
- **THEN** runtime SHALL 返回空簿，不得回退到另一个 Host、root 或 Store scope 的候选

### Requirement: Bounded candidate retention and fail-open behavior

Ledger SHALL 对候选数量、元数据字节数、会话数量和闲置时间施加有限上限，并在读取预算不足时确定性排序和报告省略数量。候选簿错误 MUST 不改变原始 CLI 命令结果或阻塞宿主继续运行。

#### Scenario: Ledger unavailable

- **WHEN** Ledger 读写失败或状态无法校验
- **THEN** Hook MUST 不注入候选、发出脱敏诊断并以 fail-open 行为继续，不自动执行 query/get

#### Scenario: Bounded read

- **WHEN** 候选总量超过 Host adapter 的读取预算
- **THEN** runtime SHALL 按最近成功根读取顺序返回受限子集和 `omittedCount`，稳定处理排序并保留完整必需字段

## `tasks.md`

# Tasks

## Design Review

- [ ] 评审根写、子读、不回流父簿的首期范围。
- [ ] 在实施前决定物理存储路径及其 persistence foundation 归属；如果选 SQLite，明确 connection/migration/db:generate 的合规方案；如果选 JSON，接受并验证跨进程锁、崩溃恢复和跨平台成本。
- [ ] 在隔离 Codex 运行中采集最小脱敏 payload 形状，覆盖 `PostToolUse`、`SubagentStart`、失败/取消、clear/resume、迟到回调和 Store override；不保留原始 prompt、完整 tool response、transcript 或私有路径。
- [ ] 验证 Host adapter 能在每个候选事件上证明 actor、成功 `get`、root epoch 和 Store scope；任何缺口都确认维持 fail-closed。
- [ ] 根据 Hook p50/p95、候选字节数及 #216 的上下文预算复核初始保留参数，再冻结 spec 的默认值。

## Follow-up Implementation (Separate Scope)

- [ ] #215：实现经证明的根 Agent 成功 `lore get` 自动记录；不可证明的事件不记录。
- [ ] #216：独立设计并验证 `SubagentStart` 的实际可见性、候选排序、上下文预算和无回流行为。
- [ ] #217：compact 后恢复留待单独批准的设计与证据评审。

---

### 评审结论与当前限制

实际源码支持提取 `id/title/applies_when/contentDigest` 和来源 Pack 名，Hook 当前仅有模拟 `SessionStart` 覆盖；官方文档与实际运行 payload 之间仍有待验证的差距。上面的设计把这差距保留为 #215 的 fail-closed 门槛，没有假定自动归属已经可用。

这次 Lorelum semantic guidance query 返回 `backend.state-invalid`，对应 trace 没有记录；一次明确的 offline keyword query 也失败且没有 trace 证据。我没有修复或重启 Backend。直接读取已知的 delegation Practice 成功，用来约束唯一规划 subagent 的交付范围。方案依据仍以当前仓库源码、spec、配置、Issue 和官方 Hooks 文档为准。

OpenSpec 生成流程要求提案文件落到仓库；受当前 **read-only** 权限限制，这些 artifact 未写入仓库，也没有测试或实施。评审时最需要确认的两项是：根写/子读策略，以及持久化选型和归属。若没有新的 payload 证据，Codex 自动记录仍保持关闭。

