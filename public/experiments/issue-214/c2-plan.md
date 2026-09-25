# Issue #214：对话级 Practice 候选簿技术方案

*只读评审草案，2026 年 9 月 24 日。未修改文件、实施、提交或推送。当前运行时无法切换到仓库指定的 `gpt-6-sol/xhigh`；本方案不声称使用了该模型。*

## 结论

建议在 Lorelum CLI runtime 内定义一个持久、但刻意窄化的 `ConversationPracticeLedger`。它保存的是“某个 Agent 在任务会话中成功读取过、可能仍相关的 Practice”的**候选元数据**，不是采纳记录、效果证明或当前 Pack 内容的副本。宿主 adapter 提供经验证的会话归属与生命周期事件；CLI runtime 独占去重、隔离、裁剪和持久化规则；Pack/LocalStore 仍是 Practice 内容及来源的 canonical 所有者。第二个真实 Host 需要复用这套内部实现之前，不设独立 workspace package。

本 Issue 应先通过 OpenSpec 完成合同评审。此次按用户要求只输出实验草案，不创建 OpenSpec 文件，也不实现 Hook。尤其不能因为 Codex 存在某些 Hook 名称，就宣称已经能可靠捕获主 Agent 的 `get`：真实事件的成功状态、调用归属和 clear 前后因果关系仍是接入门槛。

## Observed：已经核实的边界

- 当前 Codex Plugin 仅注册 `SessionStart`，匹配 `startup|resume|clear|compact`；[Hook 配置](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/hooks/hooks.json#L4)调用 `lore hook codex`，输出 Installed Pack Catalog。[Hook 实现](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts#L13)只接受 `SessionStart`，读取 Pack details 后返回 metadata-only context；[主入口](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/main.ts#L32)为它提供独立于普通 CLI JSON envelope 的原始 ABI。现有 [Hook 测试](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.test.ts#L37)使用合成输入，并非 PostToolUse 或 SubagentStart 的真实归属证据。
- `lore get` 目前从选定 LocalStore 做一次 canonical point read。[命令实现](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.ts#L24)与[结果 schema](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/result-schema.ts#L9)给出 `practice.id/title/applies_when`、`contentDigest`、`sources[{packName,sourcePath}]`；来源按 Pack 名、Pack 内相对路径排序。`get` 的公开结果**没有 Pack 版本，也没有所选 Store 的标识**，不能在 Ledger 中臆造它们。[get 合同](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md#L23)还明确：一次读取内部一致，但两次命令之间不 pin Store revision。
- 当前 [agent-integration 规格](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md#L8)要求 CLI-first、Hook 不自动 query/get、无本地 MCP；[practice-read 规格](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/practice-read/spec.md#L8)把 canonical read 留在 Engine/LocalStore。[Codex 使用说明](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/plugins/codex.md#L35)描述的也是 Catalog，而非已读候选的自动捕获。
- Issue #215 才负责“可验证的主 Agent 成功 `get`”写入，且子 Agent 的读取默认不回流；#216 才在 SubagentStart 消费有界父候选提示；#217 将 compact 后消费明确列为 backlog。它们不是 #214 已具备的行为。

主 Agent 提供的定位线索与上述来源没有实质冲突，但有两个容易误读之处：`applies_when` 在当前 `get` schema 中是必有的字符串，Ledger 的 `appliesWhen` **可以省略**是本方案的摘要/预算选择；“来源 Pack 身份”不能从现有 `get` 结果扩写成 Pack 版本或跨命令仍有效的来源保证。

## Required：本阶段必须固定的合同

记录只说明 *read/considered candidate*。它不含完整 body、原始 prompt、query、transcript 或 shell 命令文本；读取候选也不执行 `query/get`。隔离键为 `hostKey + opaque root conversation reference`，其中 `hostKey` 只作命名空间，不改变 Practice 规则。父子可见性、重复读取、内容变化、clear/resume、并发、过期与降级必须有统一语义，而不是留给各 Plugin 自定。

Host adapter 仅转换宿主可信事件与上下文，并证明根会话、发起 Agent、父子关系及生命周期；CLI runtime 拥有 Ledger 状态和操作；Pack/LocalStore 不承担短期会话状态。缺少相应事件的 Host 可以不提供该能力，不能从 cwd、相同 session ID、transcript 或命令字符串合成一个“可信”事件。

## Proposed：记录和责任设计

一条记录是**一次成功读取所形成的候选版本**，而不是实时 Practice 对象：

| 字段 | 来源与语义 |
| --- | --- |
| `practiceId` | 成功 `get` 的结构化 `practice.id`；精确 ID，不从命令文本解析。 |
| `contentDigest` | 同一成功结果的 canonical SHA-256 digest；标识**读取时**的内容，不是签名或当前版本承诺。 |
| `title`、可选 `appliesWhen` | 同一结果的 `practice.title`、`practice.applies_when`；作为不可信的展示数据处理，不解释为给 Agent 的指令。 |
| `packNames` | 从同一结果的有序 `sources` 提取、去重后的 Pack 名集合；保留多来源事实，不随意选“主来源”。不保存机器绝对路径，不填造 Pack 版本。 |
| `observedAt`、内部提交序号 | CLI runtime 写入时生成，仅用于过期和确定性裁剪；不表示实际调用先后或 Practice 相关性。 |
| 会话 binding／generation | Host 证明的 `hostKey`、不透明根引用及当前生命周期代际；不是 Practice 内容。 |

多 Pack 提供相同 ID、相同 canonical 内容时，仍只有一个 `(practiceId, contentDigest)` 候选，`packNames` 列出所有已观察来源。相同 ID 再读到不同 digest 时，**允许两个有界版本并存**，而不凭异步 Hook 到达顺序把一个冒称“最新内容”；提示应显示 digest 的短标识或明确“读取时版本”，实际使用者重新 `lore get` 并比较完整 digest。相同键再次成功读取只更新该候选的观察时间和那次结果的来源集合；来源不跨 digest 求并集。这个选择牺牲一点容量，换取并发和延迟事件下不错误覆盖历史读数。[现有 get 结果与跨命令限制](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md#L23)

建议在 `packages/cli/src/` 内建立私有用例边界；以下是**待 OpenSpec 评审的示意接口**，不是现有 API，也不预设 Codex payload 字段名：

```ts
type Binding = {
  hostKey: string;             // adapter 固定的命名空间
  rootRef: string;              // Host 提供、CLI 视为不透明
  generation: string;           // CLI 在可信 lifecycle 转换时签发
};

type Candidate = {
  practiceId: string;
  contentDigest: string;
  title: string;
  appliesWhen?: string;
  packNames: readonly string[];
};

interface ConversationPracticeLedger {
  bind(event: VerifiedRootStartOrResume): Promise<Binding | "unavailable">;
  clear(event: VerifiedRootClear): Promise<Binding | "unavailable">;
  recordRootRead(
    binding: Binding,
    read: VerifiedSuccessfulRootGet<Candidate>,
  ): Promise<"recorded" | "skipped" | "unavailable">;
  listForRoot(binding: Binding): Promise<LedgerRead>;
  listForVerifiedChild(
    binding: Binding,
    relation: VerifiedParentChildRelation,
  ): Promise<LedgerRead>;
}
```

`Verified…` 表示 adapter 必须完成的证据校验，而非让 Ledger 信任任意 JSON 声明。具体 Host 事件解析、私有 ABI 承载方式和类型形状，要等真实 payload 后确定；普通 `get` 合同不应为此暗中增加副作用。#215 若无法取得结构化成功结果及可靠主 Agent 归属，就停在限制处，不能解析面向人的输出补齐。

### 会话、父子和生命周期

- **隔离与可见性。** CLI 以 `(hostKey, rootRef)` 找到会话，持久化时可对不透明引用做确定性摘要以避免明文出现在数据库或日志中；摘要不是加密保证。根 Agent 只写、读自己的根候选。经验证属于该根的子 Agent 可只读父候选，#216 再决定是否注入；子 Agent `get` 默认**不写回根候选**。未证实 parent-child relation 时返回不可用，不搜索“当前会话”。子 Agent 私有候选或显式提升规则另议。
- **start／resume／clear。** 可信 start 建立空代际；可信 resume 仅接回同一根引用、尚未过期的代际，找不到时返回空，而非从 transcript 重建；clear 原子废止旧代际并建立空代际。晚到的旧代际写入或读取必须拒绝。这里有一个真实接入门槛：adapter 必须能让每个事件携带或可靠恢复其**事件发生时**的代际，而不能在晚到事件处理时仅按 rootRef 查“现在的代际”；若宿主做不到，clear 后自动记录应禁用。现有 `SessionStart` matcher 并不等于这项因果保证。[现有 Hook](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts#L107)
- **SessionEnd 与异常退出。** 不依赖必达的结束事件：有可靠事件可标记 inactive；没有时依靠过期清理。resume 只恢复尚在保留期内的候选。Ledger 损坏、锁忙或版本不兼容时，读返回 `unavailable` 而非伪装成“空”；提示路径不注入，写路径不影响原始 `get`。
- **缺事件的 Host。** 不注册相应 adapter 能力即可。Host 可只有 Catalog、可只有经过证明的候选读取，不必为了统一接口伪造 PostToolUse、SubagentStart 或 compact。

### 持久化、并发和有界保留

建议使用 CLI 私有的 per-user SQLite 状态库，位于独立的应用 state 目录，**不放进 Plugin 私有目录、`--store-root` 指向的 Pack LocalStore 或 Engine index**；路径解析及权限在 OpenSpec 中明确。Bun/仓库已有 SQLite 使用先例，不需要为候选簿引入 daemon、同步服务或新数据库依赖。[CLI 与 Store 分层](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/AGENTS.md#L3) [现有 Store 路径](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/engine/src/local-store/lifecycle/local-store.ts#L31)

一笔事务完成 generation 校验、`(会话, practiceId, digest)` upsert 和裁剪；clear 也在事务中废止旧代际。并发 `get` 的最终展示顺序采用 Ledger 提交序号与 ID/digest 的固定 tie-break，不宣称是宿主实际阅读顺序。重试相同候选幂等地保留一条；不记录“阅读次数”或由重试推导的采纳程度。

容量建议先以内部可调实验参数起步，例如每根会话最多 **64** 条、同一 ID 最多 **2** 个 digest、全库最多 **256** 个近期根会话，空闲 **7 天**后过期，并分别限制单条标题、适用条件及来源集合的字节数。超过预算时按最早提交/最久未读的确定性规则淘汰；单条元数据过大则跳过写入并仅记非敏感原因，不能静默截断成虚假的“完整来源”。访问时惰性清理过期行并限制全库会话数，无需常驻清理进程。这些数字**不是已核准产品阈值**：后续用真实候选数、Hook 延迟、数据库大小和 #216 context 预算校准。候选读取与最终展示均有独立上限；“Ledger 中保留”不等于“一次全注入”。

## 取舍

| 备选 | 判断 |
| --- | --- |
| **推荐：CLI runtime 私有 Ledger + Host adapter** | 统一语义、事务和降级；复用既有 CLI-first 边界。代价是新增小型本地状态、生命周期迁移与清理测试。 |
| Codex Plugin 私有文件或全局“当前对话”文件 | 初期代码少，但无法可靠隔离 Host/根会话或处理并发和 clear；将通用规则锁进单一 Plugin，不选。 |
| LocalStore/Engine retrieval state 或独立 workspace package | 前者混淆 canonical Pack 与短期会话状态；后者在第二个真实 Host 前增加发布和依赖边界而没有证据，不选。 |

不为未知 Host 设计通用插件注册表，也不让 Hook 自动查 Store 核对每条候选：一次 `get` 后 Pack 可以更新或移除，候选应如实保留“当时读过”的事实；消费时重新 `get` 才能确认现在仍可用。若调用使用了自定义 `--store-root`，现有 `get` 输出不含 Store 身份，未来提示**不能保证**在另一 Store 中可重取；#215 的结构化关联证据需覆盖所选 Store，或在无法证明同一读取范围时抑制提示，而非从 shell 文本猜参数。[get 跨命令合同](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md#L45)

## 验收与缺失证据

**#214 的设计验收**：OpenSpec 明文固定上述候选语义、字段来源、多来源规则、职责、会话及代际隔离、父子只读范围、clear/resume、并发裁剪、过期和不可用结果；评审时用两 Host 同名 rootRef、两根会话同 ID、同 ID 不同 digest、并发重复写及 clear 后晚到写等表驱动案例检查合同。此次草案尚未成为 accepted spec。

**后续实现验收**：在隔离状态目录中做 SQLite 进程并发、崩溃重启、过期/容量、权限失败和 schema 不兼容测试；证实坏 Ledger 不改变原 `get` 退出码或内容。#215 还须测试成功、失败、取消、状态未知、复杂 shell 包装和主/子归属；#216 须以真实 Codex runtime 检验子 Agent 可见的是有界 metadata，而非 body 或自动 `get`。这些不是本次运行已通过的测试。

**接入前仍缺的 Host 证据**：在获准的隔离环境中采集并脱敏真实 Codex PostToolUse、SubagentStart、start/resume/clear 样本，覆盖成功/失败/取消、直接调用和 shell 包装、主 Agent 与子 Agent、并发和 late event；逐项证明可关联的结构化 `get` 结果与成功状态、可信 actor/root/parent、事件发生时代际，以及 Hook 输出的实际可见性。不保存原始 prompt、transcript、凭据或私人路径作为 Ledger 内容。官方字段表、相同 `session_id`、cwd 或 shell 字符串均不能替代这组运行证据。若关联条件失败，自动写入 **fail closed**；若候选读取或注入失败，宿主工作继续进行、不注入。

## Deferred：不随 #214 偷渡的工作

Codex PostToolUse 写入属于 #215，SubagentStart 有界提示属于 #216；compact 后恢复属于明确延后的 #217，还需要验证真实 compact 后、下一次模型请求前的注入时序和收益。具体 Host adapter、子 Agent 读取的私有持久化/提升、自动 query/get、采纳或效果判定、完整 body、通用 Agent memory/workflow engine、跨设备同步、本地 MCP，以及独立 workspace package 都不在此阶段。下一步是把本草案及上述证据门槛转成可评审的 OpenSpec proposal；只有合同获批且真实 Host 证据成立，才开启相应实现。

