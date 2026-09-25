已完成 Issue #214 的只读技术方案草案。唯一规划 subagent 按 `gpt-6-sol / xhigh` 执行，未再委派，也没有实施或修改文件。

核心建议是：候选簿由 Lorelum CLI runtime 统一拥有，使用独立于 Pack/LocalStore 的本机状态；Host adapter 只提供可验证的会话与生命周期信息。候选记录只表示“曾成功读取、可能仍相关”，不代表采纳或有效。

最重要的实现门槛是 Codex 的实际 Hook 证据：官方文档列出的 `PostToolUse` 字段没有 Agent 身份字段，子 Agent Hook 的 `session_id` 又可能是父 session id，因此目前不能仅凭会话标识证明一次 `lore get` 来自主 Agent。必须先做脱敏实机 payload spike；验证不通过时，#215 的自动写入保持关闭，不用 cwd、transcript 或 shell 文本猜测归属。citeturn0search0turn1view1

下面按仓库默认 `spec-driven` 结构交付完整 proposal 草案。当前工作区只读，因此没有在仓库创建 OpenSpec 文件。

---

# `openspec/changes/conversation-practice-candidate-ledger/proposal.md`

## Why

主 Agent 读取的 Practice 可能对后续协作 Agent 有参考价值，但“读取过”不能证明“已采纳”或“对当前子任务仍有效”。需要定义一份跨 Host、按根对话隔离、可过期的轻量候选簿，同时不把短期会话状态混入 Pack/LocalStore。

Issue #214 要求明确记录字段、宿主会话绑定、去重与并发、父子可见性、清理和隐私边界，并以真实 Hook payload 作为可靠归属的依据。citeturn1view0

## What changes

新增对话级 Practice 候选簿合同：限定为可验证的主 Agent 成功读取；保存候选摘要而非 Practice 正文；由 CLI runtime 管理生命周期、持久化和清理；Host adapter 提供经过验证的身份及 Hook 事件。

这是**新增行为**，不是对当前实现的描述。设计评审通过不等于授权实现。

## Non-goals

不自动执行 `lore query` 或 `lore get`；不自动采纳 Practice、不注入完整正文；不实现通用 memory、workflow、跨设备同步或本地 MCP；不在此变更实现 Codex adapter；不实现 compact 恢复。#215 的记录、#216 的子 Agent 提示和 #217 的 compact 恢复仍是后续独立范围，其中 #217 明确延后。citeturn1view1turn1view2turn1view3

---

# `openspec/changes/conversation-practice-candidate-ledger/specs/agent-integration/spec.md`（delta 草案）

## Requirement: Conversation-scoped Practice candidate ledger

支持候选簿的 Host integration MUST 使用 `(hostKey, opaqueRootConversationRef)` 隔离对话状态。Host 无法可靠提供根对话身份或事件归属时，MUST 不写入候选，并 MUST 不影响普通 CLI/Host 操作。

### Scenario: Record a verifiable main-Agent read

- **WHEN** Host 能证明调用来自根对话中的主 Agent，且一次精确的 `lore get` 已成功返回符合当前 CLI 合同的结果
- **THEN** runtime MUST 记录该读取的候选摘要、Practice digest 和来源 Pack 身份
- **AND** MUST NOT 将其表示为已采纳、已生效或对所有子任务适用

### Scenario: Reject incomplete or ambiguous reads

- **WHEN** 调用失败、取消、返回非零退出、输出不完整、命令不可可靠识别，或 Host 无法证明 Agent/会话归属
- **THEN** integration MUST 不写入候选，且 MUST 保持 `lore get` 和 Host 原有行为不变

### Scenario: Share parent candidates read-only

- **WHEN** 已验证的子 Agent 在同一根对话启动
- **THEN** Host MAY 向子 Agent 提供有界候选摘要
- **AND** 子 Agent 的读取 MUST NOT 自动写回父候选簿

### Scenario: Apply lifecycle boundaries

- **WHEN** 同一对话 resume
- **THEN** runtime MUST 保留该对话的候选
- **WHEN** 对话 clear
- **THEN** runtime MUST 开始新 epoch，并拒绝旧 epoch 的迟到写入
- **AND** compact MUST NOT 被视为本变更已经支持候选恢复

### Scenario: Keep candidate state private and bounded

- **WHEN** runtime 持久化或呈现候选
- **THEN** MUST 不保存 Practice body、原始 prompt、transcript 或原始 Hook payload
- **AND** MUST 对状态数量和保留时间设置有界策略

---

# `openspec/changes/conversation-practice-candidate-ledger/design.md`

## 现状、问题与证据

当前 `lore get <practice-id>` 通过 selected LocalStore 读取一个 canonical Practice，成功结果包含完整 `practice`（含 body）、`contentDigest` 和全部 `{packName, sourcePath}` 来源；digest 是 canonical 内容 SHA-256。不同 `get` 调用之间没有版本钉住，所以候选簿只能记录“读取时观察到的版本”，不能保证当前 Store 仍有相同内容。[`get-command.ts`](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.ts:24) · [`get.md`](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md:20)

当前 Codex Hook 只支持 `SessionStart`、只消费 `hook_event_name` 并生成 Pack Catalog；Plugin 仅在 `startup|resume|clear|compact` 匹配这个事件。它没有候选簿、主/子 Agent 归属或持久化能力，也不会自动 query/get。[`codex.ts`](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts:13) · [`hooks.json`](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/hooks/hooks.json:3) · [`hook.md`](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/hook.md:9)

现有 Agent integration 合同要求 CLI-first、Hook metadata-only、不自动 query/get，并排除本地 MCP；候选簿应作为新的会话状态能力扩展这条边界，不改变 Pack/Engine 的 canonical 数据责任。[`agent-integration/spec.md`](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md:8)

Codex 官方 Hook 文档列出了 `PostToolUse` 的工具事件字段，但没有列出 Agent 身份；文档还指出子 Agent Hook 使用父 session id。`PostToolUse` 也不能单独等同于成功完成。它说明归属证据存在缺口，但不能代替 Issue 要求的真实运行 payload 验证。citeturn0search0

## 责任与数据模型

- **Host adapter** 负责解析宿主事件、验证根对话与主/子 Agent 身份、关联同一工具调用的前后事件，并确认工具完成及输出符合 CLI JSON 合同。Host 无法证明这些条件时返回 unsupported，不猜测。
- **Lorelum CLI runtime** 负责绑定根对话、epoch、候选 upsert、并发、读取、过期和清理。状态保留在 CLI-owned 独立存储，不进入 Engine retrieval state。
- **Pack/LocalStore/Engine** 继续负责 canonical Practice、digest、来源和检索 index。候选簿不是内容事实来源。

候选记录建议使用下列最小模型：

```ts
type ConversationScope = {
  hostKey: string;
  opaqueRootConversationRef: string;
  epoch: string; // runtime 生成；clear 时更新
};

type PracticeCandidate = {
  practiceId: string;
  contentDigest: string;
  title: string;
  appliesWhen?: string;
  sources: readonly { packName: string }[];
  firstObservedAt: string;
  lastObservedAt: string;
};
```

`practiceId` 与 `contentDigest` 组成候选版本键；`sources` 从同一次成功 `get` 的全部来源提取并按 `packName` 去重。当前 `get` 不返回 Pack 版本，因此不补造版本信息；默认不存 `sourcePath`、body 或 anti-pattern 内容。如果同一 ID 观察到不同 digest，允许在有界期限内并存，不根据 Hook 到达顺序断言哪个是当前版本；消费方若要实际使用，仍通过当前 `lore get` 重新读取。

接口是 runtime 内部示意，不是已发布 CLI/API：

```ts
interface CandidateLedger {
  bindRoot(scope: Omit<ConversationScope, "epoch">): Promise<ConversationScope>;

  beginObservedGet(input: {
    scope: ConversationScope;
    toolEventRef: string;
  }): Promise<{ observationToken: string } | { unsupported: true }>;

  recordSuccessfulGet(input: {
    observationToken: string;
    result: PracticeCandidate;
  }): Promise<"recorded" | "duplicate" | "stale" | "unsupported">;

  listRootCandidates(scope: ConversationScope): Promise<readonly PracticeCandidate[]>;

  clear(input: {
    scope: ConversationScope;
  }): Promise<"cleared" | "stale">;
}
```

Host adapter 不应只传一个可伪造的 `actor: "main"` 标志。它应在 CLI 内部构造不可由普通外部调用者随意拼装的 verified observation；只有在 `PreToolUse`/`PostToolUse` 能可靠关联、身份可证明且成功结果可验证时才创建该 observation。找不到可靠 pre/post 关联或 epoch 时不得记录。

## 会话可见性与生命周期

作用域固定为 `(hostKey, opaqueRootConversationRef)`，不使用 cwd、Store root、transcript、global current-session 文件或 shell 匹配推断。Host identity 不可用时该 Host 的候选功能关闭。

- 主 Agent 经验证的成功 `lore get` 是首轮唯一写入来源。
- 根 Agent 启动的直接子 Agent 可以读取父根候选，但只读；子 Agent 读取不回流写入父簿。这与 #215 的单向约束一致。嵌套子 Agent 只有在实机证明可以映射到同一 root 时才支持；否则不提供共享候选。citeturn1view1turn1view2
- `resume` 保留当前 epoch 与候选；`clear` 以 compare-and-swap 推进 epoch、清除旧候选；旧工具事件携带的 epoch 不匹配时拒绝迟到写入。
- `compact` 不改变候选状态，也不在本变更恢复候选。当前 `SessionStart` 对 compact 的处理仅恢复 Pack Catalog；候选恢复留给 #217。citeturn1view3

## 存储、并发和保留

推荐本机独立 SQLite sidecar，位置放在用户级 `.lorelum` 根目录下的新 `state` 目录中，与 `--store-root` 及 Pack LocalStore 完全分离。现有配置路径只确认 `.lorelum` root，并未定义候选状态路径，因此目录名属于 Proposed，不是当前合同。[`lorelum.ts`](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/config/src/paths/lorelum.ts:9)

最低表结构可为：

- `conversation_scope(scope_key, current_epoch, last_active_at)`
- `candidate(scope_key, epoch, practice_id, content_digest, title, applies_when, pack_names, first_observed_at, last_observed_at)`
- `observed_tool_event(scope_key, epoch, tool_event_ref, outcome, expires_at)`

候选唯一键为 `(scope_key, epoch, practice_id, content_digest)`；工具事件键为 `(scope_key, epoch, tool_event_ref)`。候选 upsert、事件去重、epoch clear 与 prune 均放在短 SQLite 事务中。采用独立 DB 是 Proposed 存储决策，不意味着 DB 归 Engine 所有；实现时应核对 Bun SQLite 与依赖声明，不把 Engine 的内部 DB 封装当作 CLI 的隐式依赖。

不推荐每根对话 JSON 文件：read-modify-write、多个 CLI Hook 进程并发、clear 与迟到写入都需要额外锁及崩溃恢复语义。也不选择 Plugin 私有存储、LocalStore 表、daemon/MCP 或通用 memory：它们分别破坏跨 Host 的状态所有权、canonical 与短期状态边界，或增加当前没有证据支持的基础设施。

清理合同应包括：clear 立即移除当前 epoch 候选；无活动对话按 TTL 过期；每个对话候选数、未完成事件数及全局 scope 数均有限；写入时在事务内先删过期数据，再按确定顺序淘汰最久未观察项。具体数值不凭空设定；应在实现 PR 中结合候选规模、并发 Hook 数及 SQLite 延迟定稿，并保证不成为用户可调的公共配置，除非后续出现真实需求。

## 错误、隐私和降级

一个读取只有同时满足以下条件才可写入：能关联同一 Host 工具事件；确认来自主 Agent；命令为可识别的单个 `lore get <exact-id>`；Host 能确认成功退出并完整取得结果；结果通过当前 JSON schema 校验，`command/ok` 正确、返回 ID 匹配请求，并包含 digest 与来源。`PostToolUse` 到达本身不是成功证据；非零退出、取消、poll、复杂 shell、多命令拼接、输出截断、归属不明或 schema 不匹配均不写。官方文档说明工具事件与结果需结合判断，并且部分非零退出也会触发后续 Hook。citeturn0search0

持久化与 context 输出中不出现 Practice 正文、prompt、transcript、原始 Hook payload、绝对路径或 shell 输出。DB 文件采用仅当前用户可访问的权限；日志只记录错误类别和计数，不记录会话/工具标识或候选内容。标题和 `appliesWhen` 按不可信文本处理，呈现时清理控制字符并限制长度。候选 DB 忙、不可读或损坏时仅关闭候选功能，不阻塞 Codex，也不影响 `lore get` 原结果。

## 前置证据门槛与分阶段实施

1. **Hook evidence spike（先于自动写入实现）**：在目标 Codex 版本中脱敏采集 `SessionStart(startup/resume/clear/compact)`、`PreToolUse`、`PostToolUse`、`SubagentStart`；覆盖主/子 Agent 成功、失败、取消的 `lore get`，单命令与复杂 shell，poll/非零退出，并发及 clear 后迟到事件。只保留字段形状和结论，不把原始 payload 写入仓库。验证根 session、Agent 归属、`tool_use_id` 关联、完成状态、stdout JSON 保真及 clear fence。无法证明就明确把 Codex 自动写入标成 unsupported。
2. **#214 合同评审**：评审范围、字段、单向读写、epoch、独立存储、bounded cleanup 和 unsupported/fail-open 语义。该方案批准只意味着设计对齐。
3. **后续获批的 runtime 实现**：建立独立存储与内部接口，覆盖跨进程事务、重复事件、候选去重/多来源/不同 digest、不同 Host/root 隔离、clear 与 stale epoch、TTL/配额、DB 忙或损坏，以及隐私断言。
4. **#215 获批且 spike 通过后**：只接主 Agent 成功读取路径；验证失败、取消、子 Agent、模糊命令和输出不完整均不写，且原 `get` 输出与退出码不变。
5. **#216 获批后**：验证候选在真实 `SubagentStart` 后确实可见、父子隔离、提示有界且 optional；不把“注入”描述为“采纳”。#217 compact 恢复不属于本轮。citeturn1view1turn1view2turn1view3

重开 JSON/SQLite 或引入更大抽象的条件应是：实测证明并发和恢复需求不同于预期、出现第二个真实 Host 实现，或数据库依赖成本明显超出问题本身。

---

# `openspec/changes/conversation-practice-candidate-ledger/tasks.md`（草案）

- [ ] 评审并确认 host-neutral 责任边界、候选语义和 parent-to-child 只读可见性。
- [ ] 执行并记录脱敏 Codex Hook payload spike；若主/子归属或成功结果不能可靠确认，明确记录 Codex 自动写入为 unsupported，不绕过门槛。
- [ ] 在独立 CLI-owned state 中实现有界候选簿；不改 Pack/LocalStore 与 `lore get` 现有合同。
- [ ] 验证并发 upsert、重复 Hook、不同 digest、多来源、resume/clear/旧 epoch、TTL/配额及数据库异常降级。
- [ ] 仅在证据门槛通过且 #215 另行获批后，接入主 Agent 读取记账。
- [ ] 仅在 #216 另行获批后，接入经验证的子 Agent 提示；compact 恢复继续由 #217 管理。

## 评审时需要定下的事项

1. 是否接受“当前 Host 证据不足时自动记录保持关闭”，而不是以近似关联换取覆盖率。
2. 是否接受独立 SQLite sidecar；具体 Bun/SQLite 依赖和路径在实现前复核。
3. 清理 TTL 和数量上限的数值在实现阶段以实测定稿，还是希望本次先指定一个显式默认值。
4. 子 Agent 提示是否只支持已验证的直接子 Agent；嵌套 Agent 默认不共享，直到有根会话映射证据。

本轮 Lorelum semantic query 返回 `backend.state-invalid`；按技能要求读取了原 trace，但没有可用记录。显式 keyword 诊断也失败。因此我没有声称成功检索到 Lorelum Practice，也没有操作 Backend、model 或 index。方案依据是当前 checkout、Issue #214–#217 和官方 Hook 文档；没有运行实现测试。

**未落盘说明：**当前 workspace 是只读的，所以这些 OpenSpec 内容目前只是完整草案，不是仓库中的 change。若你要在仓库继续推进，下一步需要可写 worktree；我可以在授权后把这四份 planning artifacts 写入，不触碰实现。

citeturn1view0turn1view1turn1view2turn1view3turn0search0

