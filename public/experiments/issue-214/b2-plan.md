# Issue #214 技术方案：对话级 Practice 候选簿

**状态：待评审设计，未实施。** 本方案依据截至 2026 年 9 月 24 日的 Issue、当前仓库和官方 Hook 文档。按本次只读要求，方案仅在此交付；没有创建 OpenSpec 文件、修改代码、运行测试、提交或推送。

## 结论与边界

建议在 Lorelum CLI runtime 内定义 `ConversationPracticeLedger`：它保存某次任务会话中**被可靠观察到成功读取、以后可能仍相关**的 Practice 元数据。记录不是“已采纳”、不是效果评价，也不是 Practice 正文的副本。宿主 adapter 提供会话、Agent 归属与生命周期事实；CLI 统一负责候选记录和有界持久化；Pack/LocalStore 继续负责 canonical Practice。初版不拆独立 workspace package。Issue #214 要求的正是这项跨宿主合同，而非具体 Hook 实现。([github.com](https://github.com/lorelum/lorelum/issues/214))

**Codex 自动记账须设证据门槛。** 官方 OpenAI Hook 文档说明：子 Agent Hook 的通用 `session_id` 使用父会话 ID；`SubagentStart` 有 `agent_id`，但 `PostToolUse` 公布的字段没有发起 Agent ID，也没有与该 `agent_id` 直接配对的字段。因此不能仅凭同一个 session ID、同一 turn 或邻近事件，把一次 `lore get` 归给主 Agent或某个子 Agent。尚无真实 payload 证明存在另一条可靠关联路径；在证明之前，Codex adapter 对这类事件应跳过记账，而不是猜测归属。

本阶段只确定合同及后续证据门槛。不实现 Codex 或其他宿主的 Hook、不自动 query/get、不自动采纳或注入正文、不做 compact 后恢复、不引入本地 MCP，也不建设通用 Agent memory。这与 Issue 的非目标及当前 [agent-integration spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md) 一致。([github.com](https://github.com/lorelum/lorelum/issues/214))

## 已确认的运行链

目前 `lore get <practice-id>` 经 CLI 从选定 LocalStore 进行一次 canonical point read，成功 JSON 含 `practice`、`contentDigest` 和完整有序的 `sources`；失败与缺失有独立错误语义。Digest 是 canonical 内容的 SHA-256，不是签名；相同 ID、相同内容的多 Pack 来源会合并返回。不同命令调用之间没有固定 Store revision 的保证。依据：[get 命令](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.ts)、[结果 schema](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/result-schema.ts)、[CLI get 合同](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md)。

现有 Plugin 只为 `startup|resume|clear|compact` 的 `SessionStart` 调用 `lore hook codex`；CLI 只识别 `SessionStart`，注入 metadata-only Pack Catalog，失败输出 `{"continue":true}`。仓库中的 Hook 测试使用合成输入，并非主、子 Agent 真实 `PostToolUse` payload。依据：[Hook 配置](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/hooks/hooks.json)、[CLI Hook 实现](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts)、[Hook 测试](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.test.ts)。官方文档列出了 `SessionStart` 的这些 source，以及 `PostToolUse` 的 `tool_input`、`tool_response`、`tool_use_id`；这些字段说明了可调查什么，不等于证明复杂 shell 命令的输出可归属或可信。

## 建议的最小合同

以下是 **Proposed** 的 CLI 内部能力接口，不是宣称现有公开命令，也不预先规定各宿主的 Hook ABI：

```ts
interface ConversationScope {
  hostKey: string;              // 宿主命名空间，不参与 Practice 规则
  rootConversationRef: string;  // 宿主提供的不透明根会话引用
  epoch: string;                // CLI 为当前会话代际签发的令牌
}

type Reader =
  | { kind: "root" }
  | { kind: "child"; agentRef: string; parentRef: string };

interface ReadCandidate {
  practiceId: string;
  contentDigest: string;
  title: string;
  appliesWhen?: string;
  sources: readonly { packName: string }[];
  reader: Reader;
  recordedAt: string;
}

interface ConversationPracticeLedger {
  bindSession(input: HostLifecycleObservation): Promise<ConversationScope>;
  recordRead(input: {
    scope: ConversationScope;
    reader: Reader;
    observationRef: string;
    result: VerifiedGetProjection;
  }): Promise<"recorded" | "duplicate" | "stale" | "skipped">;
  list(input: {
    scope: ConversationScope;
    reader: Reader;
  }): Promise<readonly ReadCandidate[]>;
}
```

`VerifiedGetProjection` 由可信宿主 adapter 从**直接观察到的** `get` 成功结果提取：须确认调用完成、退出码为 0、完整 CLI envelope 为 `command:"get"` 且 `ok:true`，并校验 ID、digest、title 与全部来源；再仅传入所需元数据。普通 shell 文本、`echo` 出来的 JSON、转述、被取消的调用、非零退出、来源不明的嵌套工具输出，都不能凭字符串匹配成为读取证据。CLI 再做字段和大小校验，但不把宿主的断言误称为密码学证明。当前 `get` 的 `applies_when` 是必有字符串；候选摘要可以因预算省略它，不能虚构 Pack version——`get` 没有在同一快照中提供该字段。多来源记录全部 `packName`，不只取第一项。

`bindSession` 将宿主可证明的 `startup/resume/clear` 事实映射到 scope：新会话建立 epoch；resume 保留 epoch；clear 清除旧候选并轮换 epoch。clear 必须带可去重的生命周期事件身份和预期旧 epoch，重复 clear 不得删除 clear 后的新记录。每次 `recordRead` 必须携带**读取开始前绑定**的 epoch；迟到的旧 epoch 结果返回 `stale`。不能可靠建立此前绑定和事件对应关系的 adapter 不记录该事件。compact 在本 Issue 中只可保留原绑定，不消费或注入候选；恢复能力留待单独设计。

父子可见性采用单向继承：根 Agent 看自己的记录；子 Agent 看根、已确认祖先及自己的记录，不看兄弟分支。子 Agent 的读取只记在自己的分支，**不自动回写或提升为根 Agent 的候选**；需要提升时由根 Agent 自己读取。宿主须提供可信的父子关系；只有根会话 ID、没有发起者或父链时不得推断。候选按 `(scope, epoch, reader, practiceId)` 去重：同 ID 重读更新为最近一次成功观察到的 digest、标题和完整来源快照；不同 digest 替换旧版本，不并列冒充当前版本。子 Agent 视图中若祖先与自己都有同一 ID，优先呈现最近的自身分支记录，并保留其读取来源标识。

## 持久化、一致性与降级

建议复用项目已有的 Bun/SQLite 能力，在 CLI 所有的用户级状态目录中放置**独立的 ledger 数据库**；不得写入 LocalStore 的 `store.sqlite`、Pack artifact、Engine index 或 `--store-root` 所选目录。位置可由现有 `resolveLorelumPaths()` 派生为单独子目录；实际路径与权限在实施提案中固定并测试。[LocalStore ADR](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/adr/0007-engine-local-store.md) 与 [路径解析](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/config/src/paths/lorelum.ts) 提供现有边界依据。选择 SQLite 是为了让多个短命 CLI/Hook 进程对绑定、去重、更新和裁剪使用同一事务；不需要 Backend、服务进程或新依赖。

概念上只需 scope/epoch、已确认的 Agent 父链、候选行及近期 observation 去重键。一次成功记录在同一事务内检查 epoch 与父链、处理重复事件、upsert 候选并裁剪；并发结果以**事务提交顺序**定义“最近观察”，不假称真实读取的先后顺序。相同 `observationRef` 的近期重放幂等；同一键却携带不同结果应拒绝。会话隔离使用完整 `(hostKey, rootConversationRef)`，原始不透明引用不用于文件名或渲染输出。`hostKey` 是本机命名空间，**不是鉴权机制**。

记录只保留 ID、digest、title、可选 appliesWhen、Pack 名和读取元数据；不保存 body、anti-pattern、原始 Hook payload、transcript、绝对来源路径或模型输出。建议以“每根会话 64 条候选、每会话 256 个近期事件键、最多 256 个保留会话、空闲 30 天”为**待测起步参数**，并为单条标题、appliesWhen、来源数设置字节上限：超过上限跳过候选，不悄悄截断身份或来源。普通读写顺带清理过期记录，无后台常驻任务。渲染时再限制条数与文本预算，把 Pack 文本明确当数据而非指令。上述数字不是已确认的产品阈值，需用真实会话规模校准。

候选是历史读取快照，不随 Pack 升级或卸载自动变成新正文。消费者使用前仍须经当前 CLI `get` 核对 ID/digest；不一致或已缺失就不采用旧内容。Ledger 不参与 Pack mutation 的事务，故其失败不得回滚安装，也不得使原本成功的 `get` 失败。缺失候选返回空列表；数据库不可用返回可区分的 unavailable 供 adapter 静默降级；归属不明、取消、失败或旧 epoch 返回 skipped/stale，均不输出误导性的“已记录”成功。现有 Hook 的 fail-open 行为可作为宿主侧保持任务继续的参考，而非掩盖内部诊断。[Hook CLI 合同](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/hook.md)

## Codex adapter 的证据门槛

在另一个获准的验证阶段，使用隔离环境捕获并脱敏**真实** payload，至少覆盖：主 Agent 直接 `lore get`、子 Agent `lore get`、嵌套/复杂 shell、失败与取消、并发调用、clear 与 resume。逐例核对发起 Agent、根会话、父链、工具调用 ID、原始退出结果、事件先后与 clear 去重身份能否可靠关联。不能读取 transcript 或扫描全局会话文件来补猜归属；官方文档也不把 transcript 格式承诺为稳定 Hook 接口。若证据不足，Codex 自动记账保持关闭，但通用 Ledger 合同仍可供具有可靠事件身份的其他宿主使用。

## 取舍、阶段与验收

| 选择 | 判断 |
| --- | --- |
| Plugin 私有 JSON/“当前会话”文件 | 拒绝：状态所有权落在 Codex，跨宿主和并发/clear 语义会分叉。 |
| 写入 LocalStore 或 Engine retrieval state | 拒绝：候选是宿主会话状态，不是 canonical Pack 内容或可重建检索 index。 |
| CLI 内独立 Ledger 状态与接口 | 推荐：复用现有 CLI-first 边界，统一规则；代价是增加一个有清理责任的小型持久状态。第二个真实宿主需要共享内部实现时，再评估独立 package。 |

Issue #214 当前应先把上述决定和“Codex 归属尚无真实证据”的限制写入按仓库惯例评审的 OpenSpec proposal/design 与 delta spec；本次只读运行不创建这些文件。设计获批后，另立实施步骤完成 CLI Ledger、独立存储、协议文档和测试；Codex adapter 只有通过 payload 门槛才进入实现；compact 消费另行评审。仓库对这类跨边界合同变化要求先有 OpenSpec proposal，当前 [OpenSpec 配置](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/config.yaml) 也区分 Proposed 与现行合同。

后续验证以行为为准：成功 `get` 的元数据投影、同 ID 重读及多来源；失败/取消/伪造输出均无记录；根、子、兄弟会话可见性；clear 后迟到事件和重复 clear；跨进程并发与事件重放；容量/TTL 裁剪、数据库损坏或不可写时原始 `get` 仍成功；Pack 变更后候选只能提示重新读取。测试应使用隔离状态目录和合成 payload；Codex 归属结论另以脱敏真实 payload 证据支持。本次未运行测试或捕获 Hook payload，不能把上述验收写成已经通过。

**仍缺的决定性证据**是 Codex 实际能否提供可靠的“工具结果 ↔ 发起 Agent ↔ clear 前 epoch”对应关系。官方字段和现有合成测试均不足以证明它。Lorelum Skill 的一次定向检索返回 `backend.state-invalid`；遵守本次只读限制，未尝试修复 Backend 或改建索引，因此没有把未取得的 Practice 指引当作方案依据。当前运行也无法切换到 AGENTS.md 指定的 `gpt-6-sol/xhigh`，这一模型要求未能满足。

