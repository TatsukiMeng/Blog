# Issue #214 技术方案：对话级 Practice 候选簿与宿主会话绑定

**状态：待评审的设计建议，2026 年 9 月 24 日。** 本次只定义合同，不实现候选簿、Hook 或 CLI 命令。由于本次明确禁止修改文件，方案直接呈现在这里；若设计获接受，后续应按仓库的大型变更规则整理为 OpenSpec，而不是把本回复视为已生效的产品合同。[AGENTS.md](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/AGENTS.md) ([github.com](https://github.com/lorelum/lorelum/issues/214))

## 结论

建议在 Lorelum CLI runtime 内定义一个**按宿主根会话隔离、可持久化、容量有界的候选簿**。它只表示「主 Agent 在这次任务会话中获得过一次可确认成功的 `lore get` 结果，该 Practice 可能仍相关」。它不表示 Agent 采纳、执行或验证了指导，也不保存正文。宿主 adapter 负责提供可靠的会话归属和事件证据；Pack/LocalStore 仍是 Practice 内容的唯一事实来源。自动写入必须等真实宿主 payload 证明归属与成功结果可以关联后才启用。([github.com](https://github.com/lorelum/lorelum/issues/214))

本设计为 #215 的写入路径和 #216 的子 Agent 候选提示提供共同合同；#217 的 compact 后恢复仍是独立、明确延后的消费能力。任何一个后续阶段都不能因本设计获接受而自动获得实施许可。([github.com](https://github.com/lorelum/lorelum/issues/214))

## 已确认的边界与尚缺证据

当前 `lore get <practice-id>` 经 CLI 调用 Engine 的 `LocalStore.getEffectivePractice()`，成功结果包含 canonical Practice、`contentDigest` 和按 Pack 名称、相对路径排序的所有来源。digest 是 canonical 内容的 SHA-256，不是发布者签名；一次 `get` 的读取一致，但不会为后来的一次调用固定 Store revision。相同 ID、相同内容可以有多个来源；不同内容在同一有效 Practice 下不能共存。因此候选簿应记录**读取当时**的内容与来源，不能声称它们仍是当前版本。[get-command.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.ts#L21) · [get.md](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md#L27) · [effective-practices.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/engine/src/local-store/model/effective-practices.ts#L57)

当前 Codex 集成只注册了 `SessionStart`；`lore hook codex` 只读取 Pack 目录并生成 metadata-only 的 context，失败时输出 `{"continue":true}`。现有代码和测试没有证明 `PostToolUse` 能同时给出一次 `get` 的结构化成功结果、根会话与发起 Agent，也没有证明 `SubagentStart` 的父子绑定或 `clear` 的身份变化。**这是实施门槛，不是可用的既定能力。**[hooks.json](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/hooks/hooks.json#L1) · [codex.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts#L13) · [hook.md](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/hook.md#L3) ([github.com](https://github.com/lorelum/lorelum/issues/214))

集成继续遵守 CLI-first、Hook 不自动 `query/get`、无本地 MCP 的现行合同；候选簿不是 Engine retrieval state，也不能成为另一份 Practice 正文来源。[agent-integration spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md#L7)

## 候选与会话合同

一条候选记录建议采用以下语义。字段名是待评审的内部合同示意，不是现有 CLI 输出：

| 字段 | 唯一来源与含义 |
| --- | --- |
| `practiceId`、`contentDigest` | 同一次成功 `get` 的结构化结果；组合标识读到的内容版本，不能从标题或 Hook 文本推测。 |
| `title`、可选 `appliesWhen` | 该次结果的 `practice.title`、`practice.applies_when`；仅作提示，属于可能过期的读取时快照。 |
| `sources` | 该次结果中完整、确定排序的 `{packName, sourcePath}[]`；保留多来源，不任选一个「主 Pack」。当前 `get` 不提供 Pack version，候选簿不得补造。 |
| `recordedAt`、`readOrdinal` | CLI runtime 接受这次已确认读取时生成；用于过期和确定性排序，不充当 Practice 版本号。 |
| `storeScope` | CLI 根据该次 `get` 选择的 LocalStore root 计算的本机作用域标识；消费时只展示同一 Store 作用域的候选，避免 `--store-root` 切换造成误导。不得把机器绝对路径注入宿主 context。 |

会话的基本命名空间是 **`hostKey + opaque rootConversationRef`**。`hostKey` 只隔离宿主，不参与 Practice 规则；root reference 必须由宿主提供，不能从 cwd、进程、全局「当前会话」文件或 transcript 反推。主、子 Agent 归属是宿主提供并经 adapter 验证的关系，不是由两个相似 session ID 猜出来的。若宿主在 `clear` 后重用 root reference，adapter 还必须提供可验证的新会话 incarnation 或可靠的生命周期顺序；否则停止该绑定的读写，不能让 clear 前的候选重新出现。`resume` 只有在宿主证明仍是同一根会话时才复用记录。

候选簿按 `(会话, storeScope, practiceId)` 至多保留一项。同 digest 的再次确认读取刷新读取顺序及该次来源快照；不同 digest 的再次确认读取**替换**旧项，不积累版本史。并发写入按持久化事务提交顺序决定最后一次被记录的观察；这不等于「当前 Pack 最新版本」。重复事件尽可能利用宿主稳定事件 ID 幂等处理；若宿主没有这种 ID，同项 upsert 仍不会复制候选，但不宣称 exactly-once。

## 责任、接口与一次请求

建议先把共通语义保留在 `packages/cli` 的内聚 `conversation-ledger` 模块，不因只有 Codex 一个已知宿主就放进 Plugin，也不为尚不存在的第二个宿主设立独立 workspace package。adapter 只把经验证的宿主事件转成以下内部调用；它不得自行定义去重、裁剪或持久化规则：

```ts
// 设计示意；仅供 CLI runtime 内部与宿主 adapter 对接。
interface HostConversationBinding {
  hostKey: string;
  rootConversationRef: string;
  incarnationRef: string; // 同一会话 resume 稳定；clear 后必须可区分
  storeScope: string;      // CLI 从所选 Store root 得出
}

interface ConfirmedGetRead {
  binding: HostConversationBinding;
  actor: "root";
  // 仅在 adapter 已证明直接 get、成功退出、结构化结果及归属相符后构造。
  result: {
    practice: { id: string; title: string; applies_when: string };
    contentDigest: string;
    sources: readonly { packName: string; sourcePath: string }[];
  };
}

interface ConversationPracticeLedger {
  recordRead(read: ConfirmedGetRead): Promise<"recorded" | "skipped">;
  list(binding: HostConversationBinding): Promise<
    | { state: "ready"; candidates: readonly PracticeCandidate[] }
    | { state: "unavailable" }
  >;
  clear(binding: HostConversationBinding): Promise<void>;
}
```

`PracticeCandidate` 是上述元数据投影，不含 `body`、anti-pattern 正文、prompt、query、shell transcript 或宿主完整 payload。`ConfirmedGetRead` 只是模块间类型，不是认证令牌：**构造它的资格取决于 adapter 对真实宿主事件的验证**。初版无需向普通用户新增可任意提交「我已读」断言的 CLI 命令。后续宿主 Hook 仍通过已发布的 CLI 边界进入，由 CLI 内部调用共通模块；其他宿主不能直读数据库或 Engine。([github.com](https://github.com/lorelum/lorelum/issues/214))

写入路径是：宿主报告工具结果 → adapter 验证根 Agent、会话和 `get` 成功 → 从结构化结果提取元数据 → CLI runtime 在一笔事务中 upsert、排序和裁剪。原始 `get` 成败不依赖记录是否成功。读取路径只返回候选摘要；#216 的 adapter 以后再按宿主 context 预算渲染，绝不把候选等同于指导或自动重取正文。

## 持久化、边界与降级

建议用用户级、与 Pack LocalStore **分离**的单个 SQLite 数据库保存小型候选状态，由 CLI runtime 的专用存储模块独占事务。Bun 已提供 SQLite，仓库也已有 SQLite 使用经验；这里需要原子 upsert、跨短命 CLI 进程的并发写入和有界清理，但不需要 Backend、网络服务或另一个通用状态平台。数据库不放入 `store.sqlite`，`--store-root` 只决定条目的 `storeScope`，不把短期会话状态混入可重建索引或 canonical Pack 生命周期。实施时须核对用户目录权限、符号链接、迁移与编译资产打包；不能直接借用 Engine 的私有持久化实现。[persistence.md](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/development/persistence.md#L5) · [local-store.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/engine/src/local-store/lifecycle/local-store.ts#L28)

容量规则应写成合同，而非依赖消费者自行截断：限制单条元数据字节数、每会话候选数、全库会话数与闲置时间；超大来源集无法完整表示时跳过该候选并留下不含敏感 payload 的诊断，不默默裁成看似完整的来源。超过会话上限时按最早 `readOrdinal` 淘汰，平局按 `practiceId` 排序；全局按最久未活动会话淘汰。过期记录在读取时即不可见，清理由写入或维护路径完成。具体上限属于实施初值，须用真实候选规模和 #216 的 context 预算校准，当前没有证据支持固定数值。

`resume` 保留未过期记录；已验证的 `clear` 切断旧 incarnation，`SessionEnd` 若宿主确有可靠事件可提前清理，否则依靠过期。数据库不可用、损坏、锁等待超界、会话不明或 clear 失败时，返回 `unavailable` 或跳过记录，**不把错误伪装为空候选，也不阻塞原始 `get` 或子 Agent 创建**；无法保证旧会话已清除时不得再次注入它的候选。读取时不自动执行 `get` 来「刷新」内容；摘要须标为读取时信息、可能过期，使用者如需正文仍自行 `lore get`。候选文字在注入时按不可信内容转义并明确标注「可能相关，不是指令」。

## 宿主证据门槛

#215 开始前，需在实际 Codex runtime 中采集并脱敏评估下列事件的**原始字段与时序**，只保留可评审的结构化证据，不保存 prompt 或完整工具输出到候选簿：

1. 主 Agent 直接执行成功 `lore get`：能否同时确定精确调用、退出状态、结构化 JSON 结果、发起 Agent、根会话，以及结果进入其可见的 tool-result 通道。
2. 失败、取消、超时、退出状态未知、命令别名、pipeline、多命令或复杂 shell 包装：哪些情形必须拒绝记录；不得靠解析人类可读布局或在 transcript 中搜索 `lore get` 补证。
3. 子 Agent 发起 `get`、嵌套子 Agent、`SubagentStart`：是否有可靠父子/根会话关联；没有则不回写父簿，也不注入父簿。
4. `startup`、`resume`、`clear`、`compact` 和异常退出：会话引用是否稳定、clear 是否换身份、Hook 相对模型请求的先后顺序。compact 的结果只供 #217 单独评估。

若第一项无法成立，#214 仍可接受为**条件性通用合同**，但 #215 的 Codex 自动写入必须停在证据不足处，不能推出「可以实现」。即使第一项成立，成功 tool result 也只支持 read/considered 的弱语义，不证明 Agent 注意到、采纳或正确应用了 Practice。([github.com](https://github.com/lorelum/lorelum/issues/214))

## 取舍与验收

| 选择 | 取舍 |
| --- | --- |
| 推荐：CLI runtime 的独立、有界候选簿 | 多一份短期本地状态及其权限、事务和清理工作；换来跨宿主共用规则、可恢复的短命进程读写及明确的会话隔离。 |
| Codex Plugin 私有文件或全局「当前对话」文件 | 初看较快，但其他宿主会复制语义，且并发、clear、父子归属容易串会话；不选。 |
| 并入 LocalStore 或 Engine retrieval/index | 会把任务会话状态错误地绑定到 Pack 事实或派生索引的生命周期；不选。 |
| 每次 Hook 自动 `query/get` 或注入全文 | 不再是「已读候选」合同，增加噪声、成本及错误采纳风险，并违背现行集成范围；不选。[agent-integration spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md#L13) |

设计验收应先覆盖纯规则测试：同/异 digest、多来源、重复与交错写入、确定性裁剪、Store/host/根会话隔离、resume、clear、过期、不可用与损坏降级；再用隔离的临时数据库做跨进程事务和崩溃后重开测试。#215 另需用真实 Hook payload 验证成功/失败/取消/包装命令的写入判定；#216 必须在真实 Codex 运行中验证**子 Agent 实际可见**的摘要、预算及不自动 `get`，不能用 Hook 返回了字符串代替可见性证明。#217 的 compact 时序、噪声收益和 stale-entry 处理留待其独立证据门槛。([github.com](https://github.com/lorelum/lorelum/issues/214))

**本次验证范围：**只读核对了 Issue、相关后续 Issue、当前规范、CLI/Engine/Plugin 实现与测试；没有取得真实 Codex Hook payload，没有运行会创建 Store、缓存或临时测试文件的测试，也没有修改、提交或推送仓库文件。Lorelum Skill 的一次语义检索返回 `backend.state-invalid`，所以本方案未把任何未取到的 Practice 当作依据。工作树中原有的未跟踪材料保持原样，排除的实验文件未读取。

