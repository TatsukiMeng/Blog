已完成研究，并将证据包一次性交给唯一一位 **gpt-6-sol、xhigh** subagent；没有再委派其他 agent。Sol 给出了推荐方案，以及一套 `spec-driven` OpenSpec 草案：`proposal.md`、agent-integration delta spec、`design.md` 和 `tasks.md`。

先说明两个限制：当前工作区是只读的，所以这些草案只在下文提供，没有写入仓库；没有代码改动，也没有运行测试。另一次 Lorelum 语义检索返回 `backend.state-invalid`，我没有自动改用 keyword 或把未取回的 Practice 当作设计依据。

## 研究结论与证据边界

#214 本身是设计门槛：要求先定候选语义、字段来源、多来源处理、责任边界、隔离和生命周期，并要求提供真实 Hook payload 证据，或明确记录仍缺的关联能力。它当前没有 milestone、branch 或 PR。#215 才定义成功读取的写入路径，#216 才消费候选向子 Agent 提示，#217 的 compact 后恢复明确另行排期。#214 也把 #28、#32、#35 和 #204 标为相关背景，并要求候选不能被误解为采纳或强制工作。([github.com](https://github.com/lorelum/lorelum/issues/214))

仓库当前 `lore get` 已经输出单行机器可读 JSON；成功结果包含完整 Practice、`contentDigest` 和 `sources[{packName,sourcePath}]`。它没有 `--json` 命令选项，且结果里的 `body` 与候选元数据同在一个 envelope。因此 #215 所说的“不要解析默认人类可读布局”不代表当前 CLI 缺 JSON 合同；真正需要设计的是如何验证 Hook 输出并只提取允许进入候选簿的字段。[get-command.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.ts:24)、[result-schema.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/result-schema.ts:8)、[get.md](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md:20)

当前 Lorelum Codex Hook 只支持 `SessionStart`，处理 Pack Catalog，不处理 `PostToolUse` 或 `SubagentStart`；现有 Hook 测试使用合成 payload，并非真实宿主事件。[codex.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts:13)、[codex.test.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.test.ts:52)、[hooks.json](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/hooks/hooks.json:1)

Codex 官方 Hooks 文档确认：`PostToolUse` 在工具产生结果后触发，Bash 即使退出非零也会触发；它提供 `tool_name`、`tool_use_id`、`tool_input`、`tool_response`，但 `tool_response` 是工具相关的 JSON 值，文档没有定义 Lorelum `get` 在各种 shell 调用下的精确响应形状。文档还说子 Agent Hook 使用 parent `session_id`；`SubagentStart` 提供 `agent_id`/`agent_type`，可通过 `additionalContext` 给子 Agent 补充上下文；transcript 格式不是稳定接口。([learn.chatgpt.com](https://learn.chatgpt.com/docs/hooks))

对身份字段有一个重要的证据细化：官方 Codex `rust-v0.153.4` 的已发布 PostToolUse schema 已包含**可选** `agent_id`，当前官方 `main` schema 也包含它，main 源码会将子 Agent Hook context 转成该字段。官方 Learn 页面没有在 PostToolUse 字段表中列出 `agent_id`，并明确提醒 `main` schema 可能包含尚未进入 release 的字段。当前最新 release 页面显示为 `0.156.1`（2026-09-23 发布），但我没有取得该 tag 的 pinned schema，也没有捕获本次运行时的真实 payload。因此，不能把 `agent_id` 说成纯属猜测；尚待确认的是当前安装版本的实际行为、字段缺席是否足以判为 root，以及成功状态和 shell 返回结果能否可靠验证。([github.com](https://github.com/openai/codex/blob/rust-v0.153.4/codex-rs/hooks/schema/generated/post-tool-use.command.input.schema.json))

上游 Codex #44095 是用户提交的 correlation-key 提案，不是 API 合同。它报告 `session_id`/`turn_id` 不能用于重建并发 spawn 的精确父子边；但 #214/#216 若只按 root conversation 读取候选，不一定需要直接 spawn-edge。不能用时间、事件顺序或 transcript 猜关联。([github.com](https://github.com/openai/codex/issues/44095))

## Sol 的方案摘要

Sol 推荐：候选簿归通用 CLI runtime 管理，以**独立于 Pack LocalStore 和索引的本地持久状态**承载；优先考虑专用 SQLite 和内部 typed boundary，暂不新增公开 CLI ABI 或独立 workspace package。Host adapter 负责证明宿主事件、会话与 Agent 身份；候选只保留白名单元数据，不保存 Practice 正文。若 Codex 无法可靠验证唯一成功的 root Agent `lore get`，#215 对该宿主应保持关闭，不以 `session_id`、字段缺席或 shell 文本猜测补洞。

Sol 还提出同一 `practiceId` 的不同 digest 作为历史变体保留，并对 clear 后迟到事件采用 fail-closed 策略。这两项是**Sol 的设计推荐，不是现行仓库合同**，值得 Owner 审查；尤其不同 digest 的提示排序与 clear 后何时恢复写入，需要后续实测和明确语义。

## 完整 OpenSpec 草案

### `openspec/changes/conversation-practice-ledger/proposal.md`

~~~markdown
## Why

[Observed] 现行 agent-integration 合同只涵盖已发布 `lore` CLI 与宿主原生 Skills/Hooks；Codex Plugin 的 `SessionStart` Hook 提供 metadata-only Pack Catalog，不记录 `lore get`，也不在 `SubagentStart` 传递已读 Practice。`lore get` 已返回 canonical Practice、digest 与来源，但完整 body 与这些字段同在成功 JSON envelope 中；当前没有对话级候选状态。（`openspec/specs/agent-integration/spec.md`；`packages/cli/src/get/get-command.ts`；`packages/cli/src/get/result-schema.ts`；`packages/cli/src/hook/codex.ts`；Issue #214）

[Required] #214 先定义“任务会话中曾成功读取、可能仍相关”的候选，而非采纳、执行或有效性证明。#215 才考虑 root/main Agent 的成功读取写入；#216 才考虑子 Agent 在启动时取得可选、有界的候选提示。当前文档和 mutable main schema 尚不能证明实际 Codex 发布版可准确归属所有读取事件；不能把推测当作写入能力。

## What Changes

- 提议为现有 `agent-integration` 增加对话级 Practice 候选合同：仅保留必要元数据，按 `hostKey + opaque root conversation reference` 隔离，定义去重、多 digest/来源、父子可见性、clear/resume、并发、淘汰与失败语义。
- 规定唯一责任：Host adapter 解释宿主事件及身份；通用 Lorelum CLI runtime 持有候选规则与持久化；Pack/LocalStore 仍是 canonical Practice、来源和索引的所有者。候选簿不进入 Pack Store、Engine retrieval state、Codex Plugin 私有存储或全局“当前对话”文件。
- 为 #215/#216 规定可验证的 host 能力门槛及最小内部交接合同。没有可信成功/归属证据时不记录；没有可信 root 绑定或读取能力时不注入，普通 `lore get` 与 Agent 工作继续。
- 本 change **提出新的产品合同，不是记录现有行为**。当前 `SessionStart` Catalog 和 `lore get` JSON 合同不因本提案立即改变；本提案不授权实现或发布。

## Capabilities

### Modified Capabilities

- `agent-integration`: 在现有 CLI-first 集成边界内增加可选的对话级已读 Practice 候选合同。delta spec 只描述获批并实施后的可观察行为，不代表现行 `openspec/specs` 已具备该能力。

## Impact

- 受影响的后续实施范围：`packages/cli` 中的通用 runtime 与 Codex Hook adapter、对应测试和集成文档。#215 负责写入路径，#216 负责子 Agent 提示；是否增加独立 workspace package，待第二个真实 Host 出现后再评估。
- 不修改 Practice/Pack schema、`lore get` 现行公开结果、Pack LocalStore/Engine 索引或 Backend `instance.json`；不增加本地 MCP、自动 query/get、自动采纳、完整 body 注入、workflow engine、通用 memory 或跨设备同步。
- #217 的 compact 后恢复独立排期。本 change 只保证候选状态及身份合同不会预先阻断该后续设计；`SessionStart source=compact` 当前重新生成的 Pack Catalog 不能被描述为候选恢复。
- Owner 审查 #214 的语义与证据门槛后，才能决定进入 #215；#215 的实际成功归属与并发验证通过后，才能进入 #216 的真实 Codex 子 Agent smoke。
~~~

### `openspec/changes/conversation-practice-ledger/specs/agent-integration/spec.md`

~~~markdown
## ADDED Requirements

### Requirement: 对话级已读 Practice 候选语义

获批并实施此 delta 后，系统 SHALL 将候选仅解释为该 root 任务会话的 main/root Agent 曾成功读取、可能仍相关的 Practice；候选 MUST NOT 表示已采纳、已执行、有效或当前仍为 Store 的最新内容。每个候选 SHALL 仅保留 `practiceId`、`contentDigest`、`title`、可选 `appliesWhen`、读取时的来源 Pack 元数据及必要的本地保留元数据；MUST NOT 持久化完整 Practice body、prompt、transcript、私有 query 或原始 shell 输出。

#### Scenario: 成功且可归属的读取
- **WHEN** Host adapter 能同时确认唯一一次 `lore get <practice-id>` 成功完成、结果对应请求的 ID，且发起者属于该 root/main Agent
- **THEN** 通用 runtime 接受该次元数据候选；不会将其标成 adopted 或 effective

#### Scenario: 不可证明的读取
- **WHEN** 执行失败、取消、退出状态未知、输入/结果无法准确解析，或发起 Agent/root 归属不确定
- **THEN** 系统不写入候选，且不改变原 `lore get` 的成功、失败或退出语义

### Requirement: 根会话隔离与宿主责任

Host adapter SHALL 为受支持的事件提供稳定的 `hostKey` 和不从 cwd、transcript 路径、时间顺序或全局 current-session 文件推断的 opaque root conversation reference，并区分已证实的 root、child 与 unknown actor。`hostKey` SHALL 只用于命名空间隔离，MUST NOT 影响 Practice 规则。不能证明绑定的 Host/event SHALL 降级为“不记录/不注入”。

#### Scenario: 两个根会话具有相同 Practice ID
- **WHEN** 不同 `hostKey` 或不同 root reference 中读取相同的 Practice
- **THEN** 一方的候选不会出现在另一方的读取结果中

#### Scenario: 子 Agent 读取
- **WHEN** 可识别的子 Agent 执行 `lore get`
- **THEN** 默认不写回 root 候选簿；单有共享的 `session_id` 不足以改变这一规则

### Requirement: 候选去重、来源和有界保留

通用 runtime SHALL 以 `(hostKey, rootRef, practiceId, contentDigest)` 识别同一候选。同一 key 的重复成功观察 SHALL 不产生重复候选；同一 `practiceId` 的不同 digest SHALL 作为不同历史读取变体保留，直到按统一保留规则淘汰。来源 SHALL 表示读取时 `lore get` 报告的位置快照，而非发布版本、签名或当前安装状态。保留规则 SHALL 包含有限的单项、单会话和全局约束，以及确定的过期/淘汰顺序；具体数值须在实施前依据脱敏规模和真实 Hook 负载校准并文档化。

#### Scenario: 相同内容的多个来源
- **WHEN** 一次 get 对同一 digest 返回多个来源
- **THEN** 候选保留有界、去重、确定排序的来源集合，不把多个来源误认为多个 Practice

#### Scenario: 同一 ID 后来读到不同内容
- **WHEN** root 在同一会话中先后成功读取相同 ID、不同 digest
- **THEN** 两个历史变体可区分，系统不推断其中之一为已采纳或当前有效；展示时必须保留 digest 区分并受提示预算裁剪

#### Scenario: 超出保留限制
- **WHEN** 元数据单项超限或新增候选将超出会话/全局限制
- **THEN** 系统拒绝该单项或按已文档化的确定规则淘汰旧项，而不是保存 body 或无限增长；记录失败不影响 get

### Requirement: 生命周期、删除与并发一致性

对同一 root 的候选写入、clear/删除和保留裁剪 SHALL 作为原子状态变更处理；跨进程并发写入 SHALL 不产生部分候选或重复 key。正常 resume SHALL 读取同一已证实 root 的保留候选；clear SHALL 删除此前候选。若 Host 在 clear 后复用相同 root reference、又不能区分 clear 前后的事件，系统 SHALL 暂停该身份的自动写入，直到取得可证明的新会话 incarnation，避免迟到事件污染新会话。compact 不自动触发候选重新注入；后续恢复由 #217 单独定义。

#### Scenario: 并发相同候选
- **WHEN** 多个 Hook 进程同时接受同一候选
- **THEN** 持久化状态至多有一个该 key，读取者只看到提交完成的状态

#### Scenario: clear 与迟到事件
- **WHEN** clear 已删除候选，但仍无法判定一个随后到达的读取事件属于 clear 前还是后
- **THEN** 不把它写入新的会话视图

#### Scenario: 失败与用户删除
- **WHEN** 状态不可用、损坏、迁移失败、忙、磁盘已满，或用户要求清除已知会话
- **THEN** get 及普通 Agent 工作不受阻；受影响的候选读取返回空/不可用并不注入；已授权的删除不导致另一个 root 的候选被清除

### Requirement: 可选的子 Agent 提示与 CLI-first 边界

在 #216 实施之后，受支持的 Host adapter SHALL 只能从已证实的父 root 候选簿构建有界、metadata-only 的可选提示，标明“父 Agent 看过、可能相关、不完整、不强制、非采纳”。没有候选、无法证明 root 关联、超出可安全裁剪的预算或 ledger 错误时 SHALL 不注入且不阻断子 Agent。此合同 MUST NOT 自动 query/get、注入完整 body、要求子 Agent 遵从候选，或引入本地 MCP/stdio/MCP tools。

#### Scenario: 子 Agent 启动
- **WHEN** `SubagentStart` 有可靠 root 会话绑定且存在可展示候选
- **THEN** 子 Agent 可收到经有界裁剪的候选元数据；该事件自身不触发 `lore query` 或 `lore get`

#### Scenario: 宿主能力不足
- **WHEN** Host 缺少所需 Hook、root 关联或 context 交付能力
- **THEN** 仅该可选候选功能不可用；既有 CLI-first 使用方式保持有效
~~~

### `openspec/changes/conversation-practice-ledger/design.md`

#### 结论与阶段

本设计推荐把候选簿做成 `packages/cli` 通用 runtime 的**专用、持久、对话作用域状态**，以内部 typed 操作承接 Host adapter。推荐 SQLite 单文件的独立状态区，以唯一键和短事务解决多 Hook 进程并发；该区位于经配置解析的 Lorelum 应用状态根之下，**不跟随 `--store-root`**。最终默认子路径、权限实现和 Bun 平台行为需在 #215 实施前核实；这些是 Proposed，不是当前已有机制。Codex adapter 只将可信 Hook 证据转换成确认的观察，不持有表结构、淘汰规则或状态文件。#214 不实现。

[Observed] 当前 `lore get` 一次读取成功时，单行 JSON envelope 的 `data` 含 `practice`、`contentDigest`、`sources[{packName,sourcePath}]`；`practice` 含 `body`。普通 CLI 已是 JSON，不能把 #215 中“需避免解析人类布局”误读为现状需要 `--json`。（`packages/cli/src/get/get-command.ts:24-72`；`packages/cli/src/get/result-schema.ts:8-62`；`docs/cli/get.md:20-59`；`packages/cli/src/output/protocol.ts`）现有 Codex Hook 只支持 `SessionStart` Catalog，其他事件 fail-open；fixtures 是合成事件，未证明真实 `PostToolUse`/`SubagentStart` 形状。（`packages/cli/src/hook/codex.ts:13-17,89-115,137-141`；`plugins/lorelum/hooks/hooks.json`；对应测试）

[Required] Issue #214 六项验收分别是候选而非采纳、字段来源及多来源、唯一责任、隔离/父子/清理/并发/降级、真实 Hook 证据或明确缺口、后续 adapter 与 compact 的最小合同。本阶段提供可评审决定与证据门槛；#215/#216 后续独立授权实施。

#### 责任和最小交接

Host adapter 只解释宿主：事件字段是否可信、root 会话身份/clear incarnation、actor 是 `root-confirmed | child-confirmed | unknown`，以及从**唯一成功 get 的结构化结果**提取白名单。通用 CLI runtime 验证候选字段并独自管理写入、去重、读取、裁剪、clear 与持久化。LocalStore 继续给 get 提供 canonical 内容、digest、来源；候选簿从不是 Practice 内容的第二份事实来源。Hook 原始 payload 与返回给 Codex 的 context envelope 属于 adapter，不穿透到状态层。

最小包内合同示意（非现行 API，类型和调用方向先固定语义，不锁定文件名）：

```ts
type RootConversation = {
  hostKey: string;        // 宿主固定 namespace；不是业务策略
  rootRef: string;        // 宿主证明的 opaque root incarnation；仅作值，不拼接路径
};
type CandidateMetadata = {
  practiceId: string;
  contentDigest: string;
  title: string;
  appliesWhen?: string;
  sources: readonly { packName: string; sourcePath: string }[];
};
type ConfirmedRootGet = {
  root: RootConversation;
  candidate: CandidateMetadata;
  // 构造者必须已验证：唯一 get、明确 exit=0、匹配的 ok JSON、root actor。
};
type LedgerResult = { kind: "ok" } | { kind: "unavailable"; reasonCode: string };
interface ConversationPracticeLedger {
  record(observation: ConfirmedRootGet): Promise<LedgerResult>;
  listForChild(root: RootConversation): Promise<
    | { kind: "ok"; candidates: readonly CandidateMetadata[] }
    | { kind: "unavailable"; reasonCode: string }
  >;
  clear(root: RootConversation): Promise<LedgerResult>;
}
```

`ConfirmedRootGet` 仅由 adapter 在验证后构造，不能由普通 `lore get` 的成功结果自行构造。adapter 对不合格事件直接 `ignore(reasonCode)`，不调用 `record`。`SubagentStart` 调 `listForChild`，再由 adapter 编码宿主专用、metadata-only 的 context envelope；ledger 不认识 Codex JSON，也不执行 query/get。这里的“内部接口”不等于新增公开 `lore ledger ...` 命令或版本化宿主 ABI；第二真实 Host 若需跨包复用或外部插件调用，再根据其实际交接与兼容需求重开。代价是初版 Host 仍与 CLI 包同发，换来不凭猜测承诺长期公共协议。

元数据示例（只有这些白名单字段和 runtime 自己的保留元数据落库）：

```json
{
  "hostKey": "codex",
  "rootRef": "<opaque-root-incarnation>",
  "practiceId": "agentic-coding.example",
  "contentDigest": "<canonical-content-sha256>",
  "title": "示例标题",
  "appliesWhen": "当任务需要明确交接时",
  "sources": [
    {"packName": "example-pack", "sourcePath": "practices/example.yaml"}
  ]
}
```

`practiceId/title/appliesWhen` 来自成功 `get.data.practice.id/title/applies_when`；digest 来自 `get.data.contentDigest`，是 canonical 内容 SHA256，**不是签名**；来源来自该次 `get.data.sources`，按既有结果顺序验证、去重并有限保留。（`packages/cli/src/get/result-schema.ts:8-62`；`docs/cli/get.md:20-59`）`packName + sourcePath` 只证明读取时的来源位置，不能声称 Pack 版本、artifact identity 或全局唯一 publisher。若未来必须精确发布身份，先取得 Pack/LocalStore canonical 数据与公开 `get` 结果的版本化设计、迁移和兼容审批；不能在 ledger 中猜造版本字段。解析完整 get envelope 可在内存短暂出现 body，但转换输出、持久化、错误日志和 context 仅使用字段白名单；对来自 Pack 的文本做长度、控制字符和 context 转义约束，不把它当指令。

#### 身份、归属与实际 Hook 证据门槛

`rootRef` 应表达一次 root conversation 的 incarnation，而不是工作目录、transcript 文件或“最近一个会话”。正常 resume 若继续同一已证实 incarnation，读取原候选；new thread 必须是不同 rootRef。`SessionStart source=clear` 原子删除旧候选：若宿主仍使用同一个 `session_id` 且不提供能关联事件前后代际的标记，则 runtime 对该身份设置暂停写入状态，直至 Host 能证明新的 incarnation。否则迟到的 clear 前 PostToolUse 会污染新会话。compact 不清除候选，也不构成 #217 的恢复注入。异常退出后的状态按同一身份保留至过期/清理；不从新进程的 cwd 恢复身份。

Codex 官方 Learn Hooks 文档（2026-09-24 核对）描述 subagent hooks 使用 parent `session_id`、`PostToolUse` 可在 shell nonzero 后触发、`SubagentStart` 有 `agent_id/agent_type`；这**可能足以让 SubagentStart 按 root-wide ledger 读取**：本合同只问“属于哪个 root”，不问“直接由哪个父 agent spawn”，因此不要求 spawn-edge join。但这**不足以证明 PostToolUse 是 root 发起的成功 get**。官方 repo mutable main schema 中 optional `PostToolUse.agent_id` 不能代表已发版；即使缺席也不能推断 root。上游关于 spawn correlation 的用户报告只作风险线索，不当 API 保证。

#215 对一次写入必须逐项证明：受支持的真实 tool name；`tool_input` 可准确解析为**一次、直接、唯一**的 `lore get <literal-id>`（不从 grep/regex 猜复杂 shell）；exit 明确为 0；`tool_response` 能可靠提取唯一完整的 CLI `command=get, ok=true` JSON envelope，结果 ID 与参数一致且元数据有效；actor 明确 root；`session_id` 能证明 rootRef/incarnation。wrapped、pipeline、多命令、alias、复杂 quoting、失败、取消、exit 未知、响应不可读、ID 不符、child 或 unknown 都不写。未来可逐一扩展经过真实证据和解析测试的调用形式；不以 shell 输出形似成功 JSON 代替身份/退出证据。失败写入不得改变原 get 结果和退出码。

最小脱敏真实运行矩阵：root 直接成功 get、not-found/非零、取消/退出未知、child get、带引号的直接命令、wrapped/pipeline/多命令、并发 spawn 及多个 PostToolUse、`SubagentStart` 的共同 root 绑定、startup/resume/clear/compact 与进程重启。只捕获字段**形状、类型、关联与受控测试值**；不保存 body、prompt、transcript、私有 query、原始 shell 输出或完整个人 `session_id`。记录 Codex 安装版本、Host/插件版本和事件样本来源。必须在 #215 之前明确 `tool_response` 的 exit/stdout 形状、root/child 判别及 clear 代际；在 #216 之前证明 `SubagentStart` root 绑定与实际 additionalContext 到达新子 Agent。若无法证明，结论可以是“当前 Codex 版本不支持该自动能力”，现有手动 CLI 交接仍可用。

#### 存储、去重、并发与保留

推荐专用 SQLite（可评估 Bun 内建 SQLite，不预加依赖）而不是单个 JSON 文件：跨 Hook 进程的唯一键、短写事务、原子 clear 与读快照是实际需求。存储属于 CLI runtime 的专用对话状态区，使用 Lorelum 应用根的解析/权限惯例，但不把 `--store-root` 的 Pack 选择错误地当作会话状态根。（`packages/config/src/paths/lorelum.ts:9-15`；`packages/cli/src/store/storage-root.ts`；`docs/development/persistence.md:9-45`）`packages/backend/src/runtime/runtime-state.ts` 的 `instance.json` 是 daemon 身份/秘密的私有生命周期文件，不是可复用 ledger。#215 确认最终路径、跨平台 `bun:sqlite`/锁行为、权限、备份/删除说明与是否已有可复用安全文件处理；不把猜出的路径当现行合同。

概念上 `conversation(hostKey, rootRef, lifecycleState, updatedAt)` 和 `candidate(hostKey, rootRef, practiceId, contentDigest, title, appliesWhen, boundedSources, lastAcceptedAt)`；候选唯一键四元组。所有 key 部分仅作数据库值并受长度校验，绝不成为文件路径。相同 ID/digest 的重复事件 upsert 为一个候选；不设“使用次数”或采纳计数。不同 digest 并存为不同**历史读取变体**，不会凭提交顺序宣称较新的内容当前有效；展示时须带 digest 区分，不把相同 ID 多个变体合成当前版本。相同 digest 的多来源是一次候选的有界、排序、去重位置快照；再次读取可更新该快照，但不宣称旧来源仍安装。时间字段只是本机接受写入/保留排序时间，不是 Host 事件因果顺序。更强的精确事件去重/chronological latest 仅在 Host 提供稳定 event ID/sequence 且实际需要时重开。

在一笔短事务中验证身份状态、upsert、限制检查和淘汰；读者只见提交状态。设有限的单字段/单项、每会话候选数与元数据字节、全局会话数/元数据字节、闲置年龄；先删过期，再按最早本地接受时间及稳定 key 顺序淘汰，超大单项直接丢弃。具体值**不是本设计承诺**，须在 #215 根据实际候选尺寸、并发 Hook 调用频率、context 预算和本地磁盘表现测量校准，并记录测试及配置/默认值。SQLite 删除后的物理文件回收可能滞后：这些首先是逻辑容量界，磁盘满必须 fail-open；若需要硬物理上限，凭测量另定回收策略。`SubagentStart` 的独立 context 字节预算由 adapter 在序列化后的最终字符串上裁剪，稳定顺序选择条目并注明部分列表；不能把 runtime 的记录上限误当注入预算。

状态根目录及文件仅当前用户可读写，拒绝不安全的 symlink/权限/路径替换；数据库忙、版本不兼容、损坏、磁盘满和迁移失败返回内部 `unavailable`，不重建覆盖可恢复数据、不泄露原始 payload 或 rootRef 到日志。Schema 版本需在库内标记，迁移采用事务和可恢复备份流程；旧 CLI 遇到新 schema 不写坏，新版遇到无法迁移不注入。记录 reason code 而非文本/标识；Hook timeout 和同步执行成本在实际负载中测量。用户对已知 root 的删除是内部 `clear(root)` 同语义；若需要用户无需宿主身份即可选择性删除或脚本化删除，须先设计公开命令与鉴权/兼容；初版至少文档化停止相关进程后的**整个专用状态区**安全清除，不建议在数据库连接仍开放时手工删除 DB/WAL 文件。

#### 方案取舍、冲突与重开条件

1. **推荐：CLI 包内专用 SQLite + 内部合同。** 代价是本地 schema/迁移/权限和 SQLite 锁测试；它满足跨进程事务而不污染 canonical Store。若实测写入极少仍需跨进程原子性，单文件 JSON 的读改写锁和崩溃恢复并不更简单。若平台 SQLite 不可用或锁行为不满足测试，重开机制评估，不能悄悄降为全局 JSON。
2. **不选：Codex Plugin 私有状态或 cwd/current-session 文件。** 它复制规则、使 root 绑定含糊、第二 Host 不能复用，直接违反 #214 责任/隔离要求。只有产品明确放弃跨 Host 合同且重开 Issue，才考虑专属状态。
3. **不选：Pack LocalStore/Engine index 或 Backend `instance.json`。** 前者持有 canonical Pack 内容和可重建索引，`--store-root` 可切换；后者绑定 daemon 进程/秘密生命周期，均不是 conversation 短期状态。仅当产品重新定义 canonical ownership 并有迁移设计时重开。
4. **暂缓：公开 `lore ledger` CLI、独立 package、跨设备同步和压缩恢复。** 当前只有一个待验证 Host，内部调用足以服务 #215/#216；新增公开 ABI 会承诺尚未验证的字段与生命周期。第二真实 Host、明确的用户级选择性删除 UX、外部进程调用需求或 #217 单独批准时分别重开，不为想象的扩展先造框架。

来源强度与未闭合点：

- Lorelum 现行源码/测试证明 JSON get、Catalog-only Hook、错误 envelope 和 Store ownership；没有任何 ledger 行为或真实 Hook 事件测试。本文不声称现有测试已运行。
- Codex 当前官方 Learn Hooks 文档描述公共字段、PostToolUse 即使 nonzero 也触发、SubagentStart context 与同步/async 行为，但没有充分定义 shell `tool_response` exit 结构及 clear 后 identity。
- OpenAI 官方仓库 mutable `main` schema 包含 optional PostToolUse `agent_id`，**不等于**当前 stable 0.156.1 schema/运行行为；本次未取得对应 pinned schema 或本机真实脱敏捕获。
- 上游用户 issue 关于 spawn-edge 缺失是报告/提案，不能升格成 Host 合同；本 root-wide 读取不需要精确直接父边，但写入仍必须证明 root actor。
- Issue #215 中有关“默认人类布局”的表述与当前 `get` 单行 JSON contract 不一致；实施时用现有稳定 JSON envelope（或经批准的等价内部传递），只提取白名单，不加不存在的现行 `--json`。

#### 验收与阶段门

#214 的六项可评审验收：① 上述 spec 的 candidate/read-only 语义与负面断言；② get schema 到候选字段逐项可追溯、同 ID 多 digest/多来源可区分；③ adapter/runtime/LocalStore 各有唯一规则与状态所有者；④ 隔离、child 默认不回流、clear/resume、原子写/淘汰/不可用降级的状态场景；⑤ 带版本的真实 Codex payload 矩阵或逐项标记“未证实，不能启用”；⑥ #215 root 写入、#216 root-wide child 读取、#217 独立 compact 恢复均有最小依赖合同而没有提前实施。

Owner 批准 #214 且真实 payload 足以证明 root 成功归属后才进入 #215。#215 应有单元/集成测试：allowlist 排除 body、直接成功、失败/取消/exit unknown、child/unknown、复杂 shell、相同/不同 digest、多来源、两 root 隔离、并发多进程、clear 迟到事件、权限/损坏/忙/磁盘满/迁移与回滚；以独立实际 Codex smoke 证明可记录而 get 结果/退出不变。若关键 Host 证据不成立，应停止该 Host 自动写入而非降低判据。随后 Owner 单独批准 #216，并验证真实 SubagentStart 注入确实到达 child、不会自动 get、不会携带 body、无候选/错误/预算超限不会阻断。任何毫秒、条数、TTL、字节的默认预算在测量前均不承诺；#217 留待另案。
#### `openspec/changes/conversation-practice-ledger/tasks.md`

~~~markdown
## 1. 设计确认与宿主证据（#214 gate；不实施）

- [ ] 1.1 Owner 审查 proposal、delta spec 和 design，确认候选语义、唯一责任、digest 多变体、root identity/clear 暂停规则、内部接口及延后项；未批准前不视为当前合同。
- [ ] 1.2 以脱敏的真实 Codex 运行矩阵记录安装版本及 PostToolUse/SubagentStart/SessionStart 字段形状，逐项证明或标记无法证明：唯一 get 与 exit、root/child actor、共享 rootRef、clear incarnation、context 交付；不得保存原始 body/prompt/transcript/私有 query。
- [ ] 1.3 依照证据形成 #215 的 go/no-go 结论；不能可靠归属时记录 Host 暂不支持，不以猜测、时间顺序或 transcript 解析补洞。

## 2. 通用 runtime 与 root 写入（未来 #215；须另行授权）

- [ ] 2.1 核实 Lorelum 状态根、Bun SQLite 的目标平台/锁行为和权限处理；测量实际元数据尺寸/Hook 并发以确定有限保留默认值，记录理由与可恢复的 schema/迁移、回滚方案。
- [ ] 2.2 建立不依赖 `--store-root` 的独立状态，提供包内 `record/listForChild/clear` 语义；用原子事务验证唯一键、多 digest、多来源、并发、clear 和裁剪。完成对应测试后才标记完成。
- [ ] 2.3 在经真实事件证明的 Codex PostToolUse 形式中，仅把明确 root、直接且唯一、exit=0、ID 匹配的 get JSON 转成白名单候选；失败/child/unknown/复杂调用忽略，写入故障不改变 get。用回归测试和实际 Codex smoke 复核。
- [ ] 2.4 文档化状态权限、用户安全清除、失败降级、schema 兼容和物理文件回收限制；校验安装后的 CLI-first 行为仍不自动 query/get。

## 3. 子 Agent 读取（未来 #216；须 #215 验收及另行授权）

- [ ] 3.1 证明实际 SubagentStart 绑定同一 root，完成 metadata-only、可选、有界且可转义的提示；无绑定/无数据/超预算/超时/存储故障不阻断。
- [ ] 3.2 单元、集成与真实 Codex runtime smoke 验证 child 实际收到提示、不同 root 不串联、child get 不回流、不会自动 `lore get` 或注入 body；完成证据和文档后才标记完成。

## 4. 明确不在此 change 实施

- [ ] 4.1 #217 compact 后恢复须另行设计/批准；不得把现有 compact Catalog 重新生成冒充候选恢复。无需为了“完成”此项而实施 #217。
~~~

## 实际交给 Sol 的 prompt

交接设置：`gpt-6-sol`、`xhigh`；只这一位 subagent；明确要求它不得再委派。以下是实际发送的 prompt 原文。

~~~text
你是本任务唯一的技术方案规划 subagent；按要求不得再 spawn/delegate 任何 subagent。请基于下面已核对的证据，为 Lorelum Issue #214 产出完整、可评审的技术方案；不要实施、不要改任何代码或文件。当前工具权限是 read-only，所以不要运行任何写文件/OpenSpec 创建命令；请在回复中给出可直接保存的 OpenSpec proposal 内容及明确目标路径。只有当某项判断确实依赖本包未给出的关键事实时才做窄范围只读核实；不要从头漫游仓库，不要重复已给出的检索。正确性优先于省工具调用。

## 目标、范围和输出

目标 Issue：#214「定义跨 Agent 的对话级 Practice 候选簿与宿主会话绑定合同」，任务类型是 design（Issue 当前 Open、design label、无 milestone/branch/PR）。本阶段定义语义、唯一责任、身份/生命周期合同、错误与验收；不是实现功能。

请按仓库 `openspec/config.yaml` 的 `spec-driven` schema 规划并在回复中提供内容草案（中文；OpenSpec 结构标题和 SHALL/MUST 关键词保留 English）：
1. `openspec/changes/conversation-practice-ledger/proposal.md`
2. `openspec/changes/conversation-practice-ledger/specs/agent-integration/spec.md`（仅写 proposed delta，不把它说成现行 contract）
3. `openspec/changes/conversation-practice-ledger/design.md`
4. `openspec/changes/conversation-practice-ledger/tasks.md`（只是将来实施分段，不执行）

另外先用简短摘要给出推荐结论、最重要的事实/未知项及推荐依据。区分 Observed / Required / Proposed / Deferred。完整方案要能沿“现状证据→具体问题→目标→责任→合同与数据→失败/生命周期→取舍→可验证验收”复核。给少量有实质差异的候选方案，明确推荐及不选原因；不要只堆名词，不要把任务留给读者做架构决定。

不要预先把争议点写成既定架构答案。尤其要评估但不要猜定：CLI runtime 持久化的具体机制/路径、Host adapter 到通用运行时的最小接口、Codex `session_id`/`agent_id` 的成功归属判据、来源 identity 的粒度、同一 practiceId 多 digest/多来源时的合并/保留、去重/更新/淘汰/clear/resume 语义、跨进程并发、可选 host 的降级、是否需要 CLI 公开命令/版本化 ABI。对每个决定写出采用理由、代价、拒绝的替代方案和重开条件。不要发明规模、TTL、条数、字节/延迟预算；若给实验起点，要标成待校准参数并说明证据门槛。

## 明确目标与非目标

Issue #214 定义候选记录代表「Agent 在该任务会话成功读取过、可能仍相关的 Practice」，不代表 adopted / effective / 执行 / 接受；至少稳定定位 `practiceId`、`contentDigest`、`title`、可选 `appliesWhen` 与来源 Pack 身份。会话隔离目标是 `hostKey + opaque root conversation reference`；hostKey 是 namespace，不参与 Practice 规则。需要定义去重、更新、并发、父子可见范围、clear/resume、清理及有界保留。候选簿不得放完整 Practice body。

Issue 描述的责任目标是：Host adapter 提供本宿主 session/parent-child/lifecycle 信息并转换 Hook payload/context envelope；通用 Lorelum CLI runtime 拥有候选项持久化和 CRUD/候选语义；Pack/LocalStore 继续拥有 Practice/Pack canonical 内容、来源和索引。不得让 Codex Plugin 私有定义/保存候选簿；不得把 conversation 短期状态混进 Pack LocalStore、Engine retrieval state 或全局 current-conversation 文件。是否另建 workspace package 等第二个真实 host 后再评估。请把这些作为 Issue 要求/约束来核对，而非由此擅自发明多层抽象。

#214 非目标：不实现 Codex 或其他 Host adapter；不自动 query/get、自动采纳或注入完整 body；不做 workflow engine、通用 Agent memory、跨设备同步；不实现 compact 后恢复；不引入本地 MCP。保持仓库 CLI-first 现行边界。

## 已核对的阶段关系与 Issue 原文要点

- #214 是 design gate：先定义通用候选状态合同、会话绑定与 Host 能力证据。
- #215 是后续第一条写入路径：只有可确认成功且明确归属于 root/main Agent 的 `lore get` 才记录；失败/取消/退出未知/参数无法准确解析/无法关联一律不记录；子 Agent 的 get 默认不写回 root；写入失败不能影响 get；不解析默认人类布局，需使用稳定 JSON 契约或等价内部传递；不自动 query/get、存 body/prompt/transcript。
- #216 是后续 Codex adapter：在 `SubagentStart` 读取父 root 会话的 ledger，注入有界可选的 metadata-only 候选提示；提示明确“父 Agent 看过、可能相关、不完整、不强制、非采纳”；无候选/关联失败/超预算/ledger 错误不阻塞子 Agent；需要实际 Codex runtime smoke 验证子 Agent 确实收到提示且未自动 get。
- #217 是明确 Backlog/延期：root compaction 后恢复候选提示，独立设计，不混入 #214/#215/#216 首轮。
- #28 是长期任务关键时刻 Practice 检索/注入研究；#32 关注压缩前影响摘要，不等于 #217 的压缩后恢复；#35 约束不要让候选变强制工作或诱发过度实现；#204 是 material decision 时主动 re-query，关注当前新决策，不是复用“已读候选”。这些仅作背景，不扩大本阶段。

当前关联 Issue 正文：
- #214 https://github.com/lorelum/lorelum/issues/214
- #215 https://github.com/lorelum/lorelum/issues/215
- #216 https://github.com/lorelum/lorelum/issues/216
- #217 https://github.com/lorelum/lorelum/issues/217
- #28 https://github.com/lorelum/lorelum/issues/28
- #32 https://github.com/lorelum/lorelum/issues/32
- #35 https://github.com/lorelum/lorelum/issues/35
- #204 https://github.com/lorelum/lorelum/issues/204

## 本地 checkout 及权威顺序

当前仓库：`https://github.com/lorelum/lorelum/tree/88e691ba939943e67793e92e1899387971dbce36`，HEAD `88e691b`，detached clean checkout（只读）。`openspec context --json` 已确认此目录为 OpenSpec root，`members: []`、`status: []`；`openspec/changes` 无文件/无 active change；schema=`spec-driven`。不要读 archive 来补设计；当前 spec/contract、接受的 ADR、代码和 tests 才是依据。仓库 root AGENTS（用户已提供）规定 Large 的跨包 lifecycle/integration/public contract 要先 OpenSpec；不引入本地 MCP。OpenSpec proposal 是本次唯一交付，不实施。

已读指导：`.agents/skills/technical-design/SKILL.md`、其 `references/review-and-writing.md`、`.agents/skills/openspec-propose/SKILL.md`。技术方案需以证据推导、比较少量候选、明确 API/状态/错误/生命周期/兼容及验收；不要形式化加层。OpenSpec skill 当前只可用于结构参考，因 read-only 不能创建 artifact。

## 关键仓库文件及已确认用途/事实

1. `openspec/specs/agent-integration/spec.md`
   - 现行 CLI-first contract：已发布 `lore` CLI + host-native Skill/Hook；集成经 CLI list/query/get，不直读 LocalStore、不引擎/复制排序；Codex Hook metadata-only，不自动 query/get；不建本地 MCP/stdio/MCP tools。
   - 新 ledger 是新跨边界行为，当前 spec 尚无该能力；proposal delta 不能冒充 accepted spec。

2. `openspec/specs/practice-read/spec.md`
   - `lore get <practice-id>` 由 Engine 从选定 LocalStore 读单个完整 canonical Practice；非法 ID 在 Store I/O 前失败；合法缺失区分返回；canonical、digest、source 顺序校验；consistent snapshot + typed busy/recovery errors。

3. `packages/cli/AGENTS.md`、`packages/engine/AGENTS.md`、`docs/AGENTS.md`
   - CLI owns parsing/composition/protocol; Engine owns LocalStore canonical data/retrieval/persistence; Engine 不能依赖 CLI/Backend。所有 Store 命令共用 root resolver。SQL/persistence 的权限与测试约定见 Engine guidance。现行 spec 负责 contract，文档补充操作说明。

4. `packages/cli/src/get/get-command.ts`（约 24–72 行）、`packages/cli/src/get/result-schema.ts`（约 8–62 行）、`docs/cli/get.md`（约 20–59 行）
   - get 当前是无 command-specific options 的 `get <practice-id>`，通过 `getEffectivePractice()` 取一次；成功 `data` 直接含 `practice`、`contentDigest`、`sources[{packName,sourcePath}]`。practice 有 `id,title,stage,tech_stack,applies_when,severity,body,anti_patterns`。sources 可有多个相同内容的来源；Docs 定义按 Pack name/source path 排序。digest 是 canonical content SHA256，不是 publisher signature。`get` 自身不提供 Pack version/artifact identity，也不做跨命令 revision pinning。出错 envelope `ok:false,error.code`，退出 2；success 0。
   - `packages/cli/src/output/protocol.ts`、`docs/cli/README.md`：普通 CLI stdout 恒为单行 JSON envelope（成功 `command/ok/data`；失败 `command/ok/error`），不解析 message。#215 中“默认人类可读 get 文本”的说法与当前 CLI contract 不一致；这里已有机器 JSON 但其中含完整 body。方案应解析/消费稳定结构且只允许字段白名单进入 ledger，不能把 body 持久化。不要发明 `--json` 现状；get 的 schema 本身在 `describe get` 可发现。

5. `packages/cli/src/get/get-command.test.ts`、`packages/cli/src/get/store.integration.test.ts`
   - 已有测试覆盖 one-point read、公开结果 schema、ID 参数拒绝、not-found/busy/recovery/unexpected 错误、single line JSON、sources、canonical digest、失败退出语义。尚无 ledger 或 Hook-to-get 记录测试。

6. `packages/cli/src/hook/codex.ts`（13–17、89–115、137–141 行）、`packages/cli/src/hook/codex.test.ts`、`packages/cli/integration/scenarios/hook-codex.ts`
   - 当前 hook ABI 的 TypeScript event 仅 `SessionStart`；input 类型只暴露 `hook_event_name`；payload parser 接受任意 object 但处理只支持 SessionStart，读 Pack details 并返回 catalog。其他 event、无效 payload、Store error 均写 stderr、stdout `{ "continue": true }`，exit 0。unit/integration fixture 是合成 SessionStart，不是捕获的真实 PostToolUse/SubagentStart payload。

7. `plugins/lorelum/hooks/hooks.json`、`plugins/lorelum/README.md`、`docs/cli/hook.md`、`docs/plugins/codex.md`
   - 当前 plugin 仅 SessionStart matcher `startup|resume|clear|compact`；运行 `lore hook codex`；Hook 不 query/get；10s timeout、additionalContextLimit 5000；fail-open。现有 compact SessionStart 重新生成的是 Installed Pack Catalog，不是 ledger compact recovery。Codex plugin Hook 需审核信任；PostToolUse/SubagentStart 引入新生命周期调用频率与同步时延，需要专门预算/失败设计。

8. `packages/config/src/paths/lorelum.ts`（9–15 行）、`packages/engine/src/local-store/lifecycle/local-store.ts`（26–34 行）、`packages/cli/src/store/storage-root.ts`、`docs/development/persistence.md`（9–45 行）
   - `~/.lorelum` 是共享 Lorelum 根，`config.yaml` 和 LocalStore 默认均在该 root；`--store-root` 选 Pack Store，不等于对话状态 root。Engine 当前三类 SQLite 是 LocalStore、keyword、semantic index，分别有生命周期；SQL migrations/Drizzle 归 Engine 持有的数据。conversation ledger 不能混进这些 Store/index；仓库没有现行 generic conversation state store。

9. `packages/backend/src/runtime/runtime-state.ts`
   - 有后端 daemon 的私有 `instance.json`，保存进程/secret/model 等运行态，并验证文件权限/符号链接；这是进程身份/生命周期状态，不是可复用会话候选存储，不可当现成 ledger。

10. `openspec/config.yaml`、`docs/development/README.md`
   - 默认 OpenSpec `spec-driven`；设计/产物 Chinese，结构标题与 SHALL/MUST 英文；当前 authority 是 openspec/specs；unapproved proposals 不改 current contract。

**尚未运行测试**：本任务只读调查，无代码改动；引用现有 tests 作为当前证据，不声称 test pass。

## Codex 外部资料：只按来源强度陈述

优先级 1（当前官方 Learn docs，检索日期 2026-09-24）：
- `https://learn.chatgpt.com/docs/hooks`（原 codex hooks URL 重定向至此）
- Common input: `session_id`, `transcript_path`, `cwd`, `hook_event_name`, model/permission fields；官方文字说 subagent hooks 使用 parent session id。`transcript_path` 是便利字段，transcript format 不是稳定 Hook interface。
- `PostToolUse` 在 supported tool 产生 output 后触发，包括 Bash non-zero exit；有 `tool_name`, `tool_use_id`, `tool_input`, `tool_response`；因此事件存在不等于 get 成功。官方 schema 页面并未在可读 table 里定义 `tool_response` 对 shell 的精确形状/exit code；不要假定 raw stdout/exit status。
- `SubagentStart` 的 event-specific fields 有 `agent_id`、`agent_type`；Hook 可通过 `hookSpecificOutput.additionalContext` 给新 subagent 加 developer context，`continue:false` 不会阻止 subagent 启动。
- 默认同步 hook 会等结束；async hook 可后台跑，Codex 可并发 background hooks，session end 会取消 unfinished work。Hooks/插件需 review/trust。`SessionStart source` 有 startup/resume/clear/compact；文档并未定义这些情况对 ledger identity 清理/复用的全部语义。
- docs 的 Schemas 段明确：GitHub `main` branch schemas may include fields not in current release；当前 release behavior 应以该 docs page 为准。

优先级 2（OpenAI 官方 repo 的 mutable main schema，不能等同当前 release）：
- `https://github.com/openai/codex/blob/main/codex-rs/hooks/schema/generated/post-tool-use.command.input.schema.json` 当前 main schema 有 optional `agent_id`、`agent_type` 以及 session/tool fields；`agent_id` 不在 required 列表里。
- `https://github.com/openai/codex/blob/main/codex-rs/hooks/schema/generated/subagent-start.command.input.schema.json` 当前 main schema 要求 `agent_id`、`agent_type`、`session_id` 等。
- 与 Learn release 文档有 field coverage/version 差异。OpenAI GitHub latest stable release page 目前为 Codex 0.156.1（2026-09-23 release；main 有后续 commits）。本 pass 无法从 web 工具取回 v0.156.1 pinned generated schema，也没有捕获此次实际运行时 payload。因此：当前 stable 是否有 optional PostToolUse `agent_id`、root actor 如何表示、子 Agent PostToolUse 是否总携带 agent_id、非零命令如何呈现，都是未闭合的证据问题。不要按 main 推定已发版，也不要以 `agent_id` 缺席推断 root。

Release reference: `https://github.com/openai/codex/releases/latest`（当前显示 0.156.1 released Sep 23 2026）。

优先级 3（官方项目中的用户报告/提案，只作线索、不作合同）：
- `https://github.com/openai/codex/issues/44095` 标题 “Expose an explicit spawn-to-subagent correlation key”。作者报告其在 v0.153.4 验证时 root/child lifecycle 缺乏显式 spawn-edge join，建议新的 spawn correlation fields。它说明 parent-child association 不能用时间/顺序/transcript path/latest spawn 猜；但它是 issue proposal，不等于当前 API guarantee，也不能直接证明 #214 的 root-vs-child get attribution。

## 本阶段必须解决或显式留为 evidence gate 的设计问题

1. Candidate semantics、最小 metadata allowlist 与字段来源；title/appliesWhen/source identity/digest 来源；`get` 已含 body，必须避免 body/prompt/transcript/private query 落库或进 context/log。
2. `hostKey + opaque root conversation reference` 的具体类型/生命周期；Codex session id 是否满足 root isolation；clear/resume/new thread/child behavior；禁止 cwd/global current-session/transcript/shell output heuristics。
3. “成功且归属 root Agent 的 lore get”可验证条件：PostToolUse tool name/input/result/exit 证据；direct command 与 quoting/options/wrapped/multi-command；nonzero/unknown/malformed/inaccessible result 的处理；root/child/unknown 三态；若当前宿主无法证实则 fail-closed 并让相应能力停在证据限制，不用猜测补洞。
4. Host adapter—generic CLI runtime 的最小 API、输入/输出和版本兼容；考虑现有 CLI one-line JSON 与 Codex raw Hook ABI 是两个合同。只在稳定需求下增 public command/API；不因未来 host 强加通用 package/plugin framework。
5. Storage ownership/path/backend: 必须远离 Pack LocalStore/index；现有 `~/.lorelum` 混合 config/default Store 的事实、Backend `instance.json` 是另一种 state 的事实；比较最小安全可并发方案、权限、atomic/idempotent writes、cross-process behavior、cleanup/recovery。不要创建“global current conversation”状态；与 host identity 显式关联。
6. Dedup/update/multi-source: key 是否 `(hostKey, rootRef, practiceId, contentDigest)`；同 ID 更新、不同 digest 是否覆盖/并存；当前 `get` 多来源仅返回 packName/sourcePath 无 version/artifact identity。请解释 source identity 到何种粒度足够，以及若改 CLI public result 会引入何种 contract/version/migration。
7. 子 Agent 可见性：默认仅 main-agent read 记入 root；root candidate 可按需供 child read；child read 不回流；SubagentStart 安全注入 metadata-only、optional、有界、不自动 get；若只有 `agent_id` 但无可靠 spawn-edge，这对 Issue #216 是足够还是需要显式 parent link，必须把关系说清。
8. Lifecycle: clear/resume/compaction/异常退出、TTL/count/size bounds、用户删除、竞争及并发 Hook；#217 compact recovery 不可被本阶段实现，但本合同要让后续有最小稳定交接。
9. Failure/security/privacy: ledger unavailable/migration/corruption/full disk/busy/unsupported host/timeout 不影响 `lore get`/普通 Agent 工作；失败默认不注入，避免污染/重复；本地状态权限、路径重定向/符号链接和日志 redaction。
10. 版本边界与证据闭环：不要只引用文档；提出可复现、最小化、经脱敏的真实 Hook payload 捕获矩阵（成功 get/失败 get/子 agent get/直接与复杂 shell/clear-resume-compact/并发 spawn），指出本回合未有真实捕获，哪些决策只能按条件定义。按 Issue #214 的六条验收输入逐条给能评审/后续验证的 criteria。

## 方案需要保留的范围边界

- #214 是合同设计，不实现 hook/ledger/CLI command/package，不修改 Practice/Pack schema，不把现行 CLI Hook 能力改写成已支持。
- #215/#216 是依赖 #214 的后续实现阶段，不把其完整功能并入 #214 提案；#217 为后续独立设计/ backlog。
- 不引入本地 MCP、workflow engine、通用 memory、跨设备同步；不自动 retrieval/read/adoption。
- 说明需哪些证据/Owner approval 才能从 #214 进入 #215，再从 #215 到 #216；真实 payload 不满足条件时允许得出“host 暂不可支持此能力”的结论。

## 方案验收与回复格式

- 推荐明确；每个决定由可核查的本地路径/行号、Issue #、或官方 source 支撑。Facts/Cases 无法支持的必须标 unknown/assumption。
- 评估范围小的现实候选方案，拒绝理由绑定已确认事实；不发明未来要用的 service/plan/plugin abstraction。
- 包含最小稳定接口示意（签名/字段/调用方向）、candidate 示例（只含元数据）、持久化数据所有权、并发/清理/失败语义、兼容发布分段、测试和实际 Codex runtime 验收矩阵。
- AC 描述可观察合同，不以“新增类/文件”代替；无数据量/延迟要求就明确不承诺具体数字。
- 若来源冲突，列专门的“冲突与未知”小节，区分官方 release docs、mutable main schema、user-reported upstream issue、Lorelum 当前源码/测试。
- 最终要给出完整 Markdown artifact drafts；不要写文件、跑测试或实现。任何窄范围核实都记录查询和结论。
~~~

## 主要取证来源与最终限制

本地关键依据包括：[agent-integration spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md:1)、[practice-read spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/practice-read/spec.md:1)、[Codex Hook CLI contract](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/hook.md:1)、[Codex Plugin guide](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/plugins/codex.md:1)、[persistence guide](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/development/persistence.md:9)、[`~/.lorelum` path resolver](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/config/src/paths/lorelum.ts:9) 和 [Backend runtime state](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/backend/src/runtime/runtime-state.ts:11)。

外部来源以 OpenAI 官方 Hooks 文档和 OpenAI/Codex 仓库为主：官方文档列明 `PostToolUse`/`SubagentStart` 行为、parent `session_id`、Hook context 规则，以及 main schema 与当前 release 的界限；仓库 schema 可确认 `rust-v0.153.4` 已有可选 `agent_id`；最新 release 页显示 v0.156.1。上游 #44095 作为用户报告/提案，只用于指出直接 spawn 关联风险，不当作产品合同。([learn.chatgpt.com](https://learn.chatgpt.com/docs/hooks))

最后再明确：**真实 Codex Hook payload 尚未捕获**；当前 Lorelum Hook 测试是合成输入，不能证明运行时关联合同。已核到的旧记忆仅作搜索线索，相关结论均重新对照本 checkout 和当前官方资料。若要把这四份草案保存成 OpenSpec 文件，需要在具备写权限的后续任务中另行授权；本次没有创建或验证仓库 artifact。

