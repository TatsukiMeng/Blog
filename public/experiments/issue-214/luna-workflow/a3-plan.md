已完成：只启动了一个 `gpt-6-sol / xhigh` 规划 subagent；没有委派给其他 agent，也没有实施或改动文件。下面是按仓库 `spec-driven` OpenSpec 结构整理的完整方案，可供评审。

**交付限制：**当前工作区为只读，不能把这些内容写进仓库；因此我把拟议的四份 OpenSpec 工件正文直接交给你。Issue #214 当前仍为 Open，且要求候选语义、字段来源、所有权、隔离与生命周期合同，以及真实 Hook payload 证据或明确记录能力缺口。([github.com](https://github.com/lorelum/lorelum/issues/214))

## 结论

建议定义一个由 Lorelum CLI runtime 拥有、与宿主无关、短寿命且 metadata-only 的对话级候选簿。Host adapter 负责提供可验证的宿主会话、Agent 归属和生命周期证据；runtime 负责候选语义、隔离、去重、并发、裁剪和清理；Pack/LocalStore 继续拥有 canonical Practice 内容及来源。

候选只表示“已成功读取过、可能仍相关”，不表示采纳、执行或有效。当前最大的门槛不是选 JSON 还是 SQLite，而是能否证明一次 `lore get` 完整成功、确实属于主 Agent，并与正确的根会话和生命周期 epoch 关联。证明不了就不记账，也不阻塞原命令。

本次没有捕获真实 Codex Hook payload，因此**目前不能声称 Codex 已具备可靠的自动记账能力**。先做受控、脱敏的 payload evidence spike；若不能证明归属与成功，#215 的自动写入路径保持 unsupported。Issue #215 负责验证后的写入，#216 负责后续 Subagent 注入，#217 是明确延后的 compact 恢复；候选簿不替代 #204 的主动 re-query。([github.com](https://github.com/lorelum/lorelum/issues/215))

## 拟议工件一：`proposal.md`

# 定义对话级 Practice 候选簿与宿主绑定合同

## Why

主 Agent 在任务中成功读取 Practice 后，这条线索可能对后续协作有用。但现有集成只注入 Pack Catalog，并没有对话级候选状态。若把“已读”当成“已采纳”、按相同 `session_id` 猜测发起者，或由 Plugin 私自保存状态，就可能产生错误的跨 Agent 传播。

## What Changes

本 change **引入拟议的新产品合同，不是记录已有功能**。它定义跨宿主候选语义、字段与来源、会话隔离、父子可见范围、可验证写入、生命周期、有限容量及错误降级，并规定 Codex payload 验证门槛。

本 change 只交付设计和证据结论，不交付 runtime、CLI 命令或具体 Host adapter 实现，也不把 Proposed delta 写成当前已生效的合同。

## Scope

- 由 CLI runtime 拥有通用候选语义与生命周期。
- Host adapter 仅提供可验证的 host、root/child、工具调用和 lifecycle 证据。
- Pack/LocalStore 继续拥有 canonical Practice、Pack 来源及检索索引。
- 无法可靠确认成功、归属、epoch 或 Store 范围时 fail closed：不记录、不跨会话读取。
- 本次计划真实 payload evidence spike；若证据无法证明关联能力，记录具体缺口，不伪造“已支持”。

## Non-goals

不实现具体 Codex 或其他宿主 adapter；不自动 query、自动 `lore get` 或推断采纳；不保存 Practice body、prompt、私有 query、shell/transcript 或原始 Hook payload；不改变 Practice/Pack schema；不引入本地 MCP、Agent memory 系统或跨设备同步；不在本期实现 compact 后的候选恢复。

## Review and Acceptance

评审需确认候选语义、字段来源和多来源处理、三方职责、隔离/可见性、清理/并发、错误降级及真实 Hook payload 证据，或明确承认当前缺少哪些可验证关联字段。通过设计评审不代表候选簿已经实现，也不授权实施 #215–#217。

## 拟议工件二：`specs/conversation-practice-ledger/spec.md`

以下是 **proposed delta**，不是当前合同；只有后续实现、验证并经正式流程接受后，才可同步到 `openspec/specs/`。

# conversation-practice-ledger Specification

## ADDED Requirements

### Requirement: Candidate semantics and metadata

候选簿 SHALL 只表达 read/considered candidate，不得表达 adopted、effective 或完整 guidance。候选项 MUST 包含精确 `practiceId`、读取时的 `contentDigest`、title 和至少一个来源 Pack；`appliesWhen`、相对 `sourcePath` MAY 保留。候选簿 MUST 不存储 Practice body、原始 prompt/query、shell 文本、transcript 或原始 Hook payload。

#### Scenario: Multiple sources for the same content

- **WHEN** 同一 ID、同一 canonical digest 来自多个 Pack
- **THEN** 候选簿 MUST 将其视为一个内容候选，并保留有界、确定排序的 provenance。

#### Scenario: Content changes under the same ID

- **WHEN** 同一会话读取到相同 Practice ID 的不同 digest
- **THEN** 候选簿 MUST 保留读取时的内容身份，且 MUST 不把旧 digest 表述为当前 Store 内容或 publisher signature。

### Requirement: Verified and attributable reads only

Host adapter SHALL 提供可验证的 root、发起 Agent、工具调用关联和 lifecycle 证据；runtime SHALL 只接受能证明为完整成功的 Lorelum `get` JSON 结果。调用方声明的成功/归属标志、单独的 `session_id`、可显示的 JSON 或 shell 文本 MUST NOT 作为证明。归属、完整性、退出状态、epoch 或可能影响复用的 Store 范围任何一项无法验证时 MUST 不记账；记账失败 MUST 不影响原 `lore get` 或阻塞宿主。

#### Scenario: Successful root read

- **WHEN** 主 Agent 的完整成功 `get` envelope 可关联到根会话、调用时 epoch 与可验证的 Store 范围
- **THEN** runtime SHALL 记录经 schema 校验的候选 metadata。

#### Scenario: Failed or ambiguous read

- **WHEN** 调用失败、取消、输出截断、shell 包装无法证明实际执行，或无法确认发起 Agent
- **THEN** runtime MUST 不从命令文本或表面 JSON 猜测成功。

### Requirement: Conversation and actor isolation

候选簿 SHALL 以 `(hostKey, opaqueRootRef, epoch)` 隔离；`hostKey` 仅作命名空间。root 只能读取该根会话中由可信路径记录的候选。child 只有在 parent/root 关系和 epoch 均经过验证时，MAY 读取父候选的有界视图；child 自己的读取默认留在 child-local 范围，不能自动回流父候选簿。身份、父子关系或 epoch 不完整时 MUST 返回空候选，不得按相同 `session_id` 合并。

#### Scenario: Actor relationship is unknown

- **WHEN** runtime 收到无法验证 actor/root/parent 关系的请求
- **THEN** runtime MUST 返回空候选，并让宿主继续。

### Requirement: Epoch fencing and bounded retention

可信 clear SHALL 开启新 epoch 并令旧 epoch 候选不可见；可信 resume MAY 恢复同一根会话的未过期 epoch；compact MUST NOT 被视为候选恢复。候选总量、字节数与保留时间 SHALL 有有限上限，裁剪 SHALL 确定且不得产生半条候选。旧 epoch 的迟到或重放事件 MUST NOT 污染新 epoch。宿主 context 或 ledger 不可用时 MUST 返回更少或零候选，且 MUST 不阻塞 Agent。

#### Scenario: Delayed result after clear

- **WHEN** clear 前发起的读取在 clear 后才收到完成事件
- **THEN** runtime MUST 使用调用时 epoch 验证并忽略旧 epoch 结果。

## 拟议工件三：`design.md`

# 对话级 Practice 候选簿技术设计

### Observed：当前代码与证据界限

当前 Codex Hook 只接受 `SessionStart`，输入接口只声明 `hook_event_name`，执行 Pack details 并返回 Catalog；错误时写 stderr 诊断并返回 `{"continue":true}`。[`codex.ts`](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts:13) 的 Hook config 只注册 `startup|resume|clear|compact`，当前 `additionalContextLimit` 为 5000。[`hooks.json`](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/hooks/hooks.json:2)

`lore get` 精确读取一次 Practice，成功结果包含 canonical Practice、`contentDigest` 和来源；schema 对完整 body 也有定义。[`get-command.ts`](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.ts:24) [结果 schema](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/result-schema.ts:9) CLI-first、Hook metadata-only、不自动 query/get、不新增本地 MCP 是当前集成合同。[agent-integration spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md:8)

官方 Codex 文档列出 common `session_id`，并说明 subagent Hook 使用 parent session ID；`SubagentStart` 有 `agent_id`、`agent_type`，而 `PostToolUse` 提供 `tool_use_id`、`tool_input`、`tool_response` 等字段，但文档未列出发起 Agent 身份或独立退出状态字段。`SessionStart` 的 `source` 才能区分 startup/resume/clear/compact。它们是候选字段，不是真实运行 payload 证据，也不能证明事件完整性或调用来源。([developers.openai.com](https://developers.openai.com/codex/hooks))

### Proposed：责任和最小内部接口

| 责任方 | 唯一职责 |
| --- | --- |
| Host adapter | 从真实 Hook 事件验证 root/child、工具调用和 lifecycle 关联；输出宿主 context envelope |
| CLI runtime | 验证 CLI 结果、管理 ledger 语义、去重、隔离、裁剪和过期 |
| Pack/LocalStore | canonical Practice、digest、Pack 来源和索引 |

根绑定只使用 `hostKey + opaqueRootRef + epoch`。当前 `get` 结果没有可信 Store identity，因此 `storeScope` 不是现成字段；必须在 evidence spike 中确认能否从实际 invocation 与 Store-root resolver 得到稳定、私有的范围标识。若可能跨 Store 误用而又无法验证范围，则限于可证明的默认 Store，或关闭写入/复用，不能把绝对路径存入候选或日志。

最小内部边界：

```ts
type CandidateRecipient =
  | { kind: "root"; rootRef: string }
  | { kind: "child"; agentRef: string; parentRootRef: string };

type CandidateReadRequest = {
  binding: RootBinding; // hostKey + opaqueRootRef + epoch
  recipient: CandidateRecipient;
  budget: CandidateBudget;
};

type ValidationResult =
  | { kind: "accepted"; read: ValidatedRead }
  | { kind: "unsupported"; reason: EvidenceGap };

function validateObservedGet(
  hostEvents: CorrelatedHostEvents,
  cliOutput: ObservedCliOutput,
  lifecycle: TrustedLifecycleState,
): ValidationResult;

interface ConversationPracticeLedger {
  recordValidatedRead(read: ValidatedRead):
    Promise<"recorded" | "duplicate" | "ignored">;
  listCandidates(request: CandidateReadRequest):
    Promise<readonly Candidate[]>;
}
```

这里的 `ValidatedRead` 不能靠调用方提供的 boolean 声明来成立。adapter/runtime 必须基于真实 Hook 事件关联、调用时 epoch、完整 CLI JSON envelope、成功状态与可验证来源构造它。若 shell 可以 `echo` 出同样的 JSON，或 `PostToolUse` 只有人类可读文本，返回 `unsupported`，不记账；不得解析 wrapper、cwd、全局会话文件或 transcript 来补证。

`Candidate` 保存 `practiceId`、读取时 digest、title、可选 `appliesWhen`、来源 Pack 和必要的相对 `sourcePath`，以及首次/最近有效读取时间；不保存 body、prompt、query、命令、transcript、完整 Hook payload 或绝对路径。内容去重键是 `(practiceId, contentDigest)`：同 digest 的多 Pack 来源合并为一个候选；同 ID 不同 digest 可以并存至版本上限。重复 `eventRef` 完全幂等且不刷新最近读取时间；新读取可更新时间。metadata 当作不可信文本转义并标注“可能相关、不完整、未证明采纳”。

读取接口必须显式接收 recipient：只有证明属于绑定 root 的 root，或其 parent/root 关系及 epoch 可验证的 child，才能读取对应视图。子 Agent 自己写入的候选默认不进入父视图；关系不匹配时返回空。#214 规定合同，#216 才实现 child 读取和提示。

### Lifecycle、存储及暂定限制

可信 clear 原子切换 epoch，旧候选立即不可见；resume 只有确认同一 root 与未过期 epoch 才恢复；compact 不隐式恢复候选。若要防止 clear 前发起、clear 后完成的读取，必须能把调用开始时 epoch 关联到完成事件；不能做到就关闭该路径。SessionEnd 只作尽力清理，不作为唯一过期手段。官方文档也说明 SessionEnd 不运行于 subagent，且可在会话结束/空闲后才运行。([developers.openai.com](https://developers.openai.com/codex/hooks))

容量和过期的暂定起始参数如下，均为实验值，不能直接写成不可变的 current spec：

| 维度 | 暂定值 |
| --- | ---: |
| 每个 root + epoch 候选数 | 32 |
| 同一 Practice ID 的 digest 数 | 2 |
| 每会话 metadata | 16 KiB |
| 每项来源数 | 8，超过时记录遗漏数 |
| title / `appliesWhen` / `sourcePath` | 160 / 240 / 256 字符 |
| idle / absolute TTL | 7 天 / 30 天 |
| 全局容量 | 128 个 root 会话 / 2 MiB metadata |

按每 ID 版本、会话数量、字节和全局容量依次确定性裁剪；不输出半条记录。先裁掉可选描述/来源与旧候选；若最小合法条目仍超限则丢弃。重复事件不延长 TTL；时钟异常不能超过 absolute TTL。

当前 5000 是 `SessionStart` 的 `additionalContextLimit`，不能直接当作 SubagentStart 可用预算。未来可用最多 800 字符作为 renderer 的待测上界，但在验证真实 SubagentStart 输出预算、与其他 context 的叠加及 Agent 确实可见之前，**实际注入预算为零**。([developers.openai.com](https://developers.openai.com/codex/hooks))

参数通过隔离的代表性任务和测试 Pack 校准：记录 metadata 字节及会话候选数的 p50/p95/p99、同 ID 更新与多来源分布、并发写入、clear/resume 间隔、实际 context 可见长度；用 1/8/32 候选和 1/4/8 来源验证边界与裁剪率。未完成校准前不启用自动提示。

### Storage 选择

候选数据不应写入 Pack LocalStore、Engine retrieval state、Plugin 私有文件或全局“当前会话”文件，也不需要独立服务或新的 workspace package。

- **JSON snapshot** 结构较简单，但 atomic rename 只能避免半写文件，不能解决多 Hook 进程的 read-modify-write 丢更新；还需跨平台锁、崩溃恢复及权限处理。
- **独立 SQLite 状态文件** 是 #215 的优先评估项，可将 epoch 检查、event 幂等、upsert 与有界裁剪放在事务中；但 #214 不冻结物理格式。仍需在目标编译平台验证 driver、锁等待、文件权限和恢复行为。

选择物理格式留给 #215，条件是先证明真实并发/Store 来源，再验证跨平台实现。持久化失败、数据库 busy/corrupt 或权限错误时，返回空候选并让宿主继续，不影响原 `get`。

### 必经 payload evidence spike

在未来获准的隔离 Codex 测试会话中，使用非敏感测试 Pack 验证下列场景；只保留字段白名单、类型、脱敏关联 token、状态/长度/截断标志、事件顺序和预期归属，不保存原始 payload、Practice body、shell 文本、路径或凭据：

1. root 的成功、失败、取消 `lore get`；child 的 `get`；root/child 并发读取。
2. 直接命令、shell wrapper、管道、嵌套调用、连续/并发命令，以及伪造但结构看似成功的 JSON 输出。
3. PreToolUse/PostToolUse 调用关联、调用开始时 epoch 与 clear 后迟到完成。
4. startup、clear、resume、compact 的真实顺序和字段；SubagentStart 与 root 的关联；SessionEnd 延迟或缺失。
5. selected Store 来源是否可验证，默认与显式 `--store-root`、别名/符号链接及 Store 切换是否能隔离。
6. Hook 是否能观察到完整 CLI envelope、真实成功状态、发起 Agent、工具调用来源及 child context 是否实际可见。

每条 fixture 记录 host/runtime 版本、事件序号、脱敏字段、预期 actor/root/epoch、验证结果和证据缺口。结果只能是：

- 证据同时证明来源、完整成功、actor、root、epoch 与必要的 Store 范围：#215 才可评估自动写入。
- 任一关键关联不能证明：记录具体缺口，将该 adapter 的自动写入/读取标为 unsupported；不要用更多文本解析绕过证据门槛。

## 拟议工件四：`tasks.md`

以下任务仅用于 #214 的证据和设计交付；全部未执行。

- [ ] 1.1 在获准的隔离 Codex 会话捕获成功/失败/取消、root/child、shell wrapper、并发、lifecycle 与迟到事件；生成字段级脱敏 fixture，确认没有保存正文、shell 内容或私有标识。
- [ ] 1.2 判定每个场景能否同时证明调用来源、完整成功、actor/root/epoch 与 selected Store 范围，并记下无法证明的字段及原因。
- [ ] 2.1 评审候选语义、字段来源、多来源合并、digest 更新、父子读取可见性、隔离与 fail-open/fail-closed 行为。
- [ ] 2.2 根据 spike 分布校准暂定条数、字节数、字段长度和 TTL；比较 JSON snapshot 与独立 SQLite 的并发、恢复、权限与跨平台成本。
- [ ] 2.3 检查 proposal/spec/design/tasks 是否一致，确保 #215、#216、#217 仍是后续独立阶段；Owner 评审通过后才可进入后续实施流程。

## 评审时需要决定

1. 是否批准“不能证明就不记账”的门槛；若 payload spike 失败，接受明确记录 Host 能力缺口，不以解析 shell/transcript 补齐。
2. 是否接受暂定容量与 TTL 作为 #215 的实验起点，而不是 current spec 的固定值。
3. 是否同意 #215 优先评估独立 SQLite，但等 Store identity、并发和跨平台证据后再选定物理格式。
4. 是否接受 SubagentStart 未经真实验证前注入预算为零，且 compact 候选恢复继续归 #217。

这次没有运行 Hook、捕获真实 payload、改代码或创建 OpenSpec 文件；没有执行自动化测试，因为本次是设计交付且工作区只读。Lorelum Skill 的语义查询遇到 `backend.state-invalid`，显式 keyword 查询也失败；我没有操作 Backend。随后读取了可确定 ID 的 delegated-agent context Practice，并用它约束唯一规划 subagent 的目标、证据边界、non-goals 与交付格式。

