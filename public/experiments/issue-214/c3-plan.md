# Issue #214 技术方案草案：对话级 Practice 候选簿

**结论：**在 `packages/cli` 内定义 `ConversationPracticeLedger` 的通用语义和持久化边界。它保存一次**可验证的成功读取**留下的小型候选快照，供同一任务会话后续按需参考；它不保存正文，也不表示 Practice 已被采纳或证明有效。Host adapter 负责证明“谁、在哪个根会话、成功读取了什么”，CLI runtime 负责记录、隔离、裁剪和读取；Pack/LocalStore 仍是内容与来源的事实来源。本方案仅供评审，不构成实现授权。

## Observed：当前合同与实现

- 当前 Codex Plugin 仅为 `SessionStart` 的 `startup|resume|clear|compact` 注册 Hook，注入 Installed Pack Catalog。CLI 的原始 Hook ABI 只处理 `SessionStart`；收到其他事件会降级为 `{"continue":true}`。它没有已验证的 root/child 关联或自动捕获 `get` 的路径。依据：[hooks.json](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/hooks/hooks.json)、[codex.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts)、[main.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/main.ts)、[Hook 测试](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.test.ts)。
- 当前工作树的 `get` 成功 JSON envelope 已有 `practice.id/title/applies_when`、`contentDigest` 和有序的 `sources[{packName,sourcePath}]`。同一条结果来自一次一致的 point read；不同命令不共享被 pin 住的 Store revision。相同 ID、相同内容由多个 Pack 提供时，Engine 返回一份 canonical Practice 和全部有序来源；内容冲突则不是可合并的有效来源。依据：[get-command.ts](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.ts)、[结果 schema](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/result-schema.ts)、[get 合同](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md)、[来源合并](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/engine/src/local-store/model/effective-practices.ts)、[集成测试](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/store.integration.test.ts)。
- 现行集成合同要求 CLI-first、Hook 不自动 query/get、无本地 MCP；`practice-read` 规定 canonical、一致且有界的读取。这些合同没有定义对话候选状态。依据：[agent-integration spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md)、[practice-read spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/practice-read/spec.md)、[Codex 工作流](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/plugins/codex.md)。

交接线索与上述范围没有实质冲突，但**“Codex 有潜在 `PostToolUse` 事件，所以现有 Hook 已能确认主 Agent 的一次成功 `get`”是错误推论**。当前 ABI 与测试均不证明该关联；相同 `session_id`、cwd、transcript 或 shell 命令文本也不能补足证据。Issue #214 本身要求先评估真实 payload。

## Required：本阶段要定下的合同

记录的含义固定为“某 Agent 在此任务会话中成功读取过、可能仍相关的 Practice”。`read/considered candidate` 与 `adopted/effective` 必须在类型、文档和日后提示文案中分开。会话至少以 `hostKey + opaque root conversation reference` 隔离；去重、更新、父子可见性、clear/resume、并发及有界保留不能散落在 Host Plugin 中。Issue #214 要求给后续 adapter 和消费方一个最小合同，但明确不在本 Issue 实现它们。

## Proposed：责任、数据与最小接口

| 责任 | 唯一所有者 | 边界 |
| --- | --- | --- |
| 事件解释与归属 | Host adapter | 从真实事件确认成功状态、发起 Agent、根会话、父子关系和生命周期；不能确认就不提交观察值。 |
| 候选语义与状态 | CLI runtime 内的 ledger 模块 | 校验候选快照，执行隔离、去重、事务、裁剪、过期、clear 和读取；不依赖某个 Host 的字段名。 |
| Practice 事实 | Pack/LocalStore，经现有 `get` 合同暴露 | 保有当前正文、digest 与来源；ledger 不成为第二份 canonical Store，也不在提示读取时自动 `get`。 |

建议初版作为 `packages/cli` 的内部能力，由后续 Host adapter 通过已发布 CLI 的 Hook 路由接入；不预设独立 workspace package，也不新增面向任意调用方的“宣称已读”公开命令。下面是**待 OpenSpec 定稿的内部接口示意**，不是已存在 API：

```ts
type RootBinding = {
  hostKey: string;
  rootConversationRef: string; // 宿主提供的不透明值
  lifecycleEpoch: string;      // 能区分 clear 前后的可信会话代次
  storeScope: string;          // 本次 get 所选 Store 的内部作用域
};

type ReadCandidate = {
  practiceId: string;
  contentDigest: string;
  title: string;
  appliesWhen?: string;
  sourcePackNames: readonly string[];
};

interface ConversationPracticeLedger {
  recordVerifiedRootRead(
    binding: RootBinding,
    candidate: ReadCandidate,
  ): Promise<"recorded" | "unchanged" | "rejected-stale">;
  listRootCandidates(binding: RootBinding): Promise<readonly ReadCandidate[]>;
  clearRoot(binding: RootBinding): Promise<void>;
}
```

`recordVerifiedRootRead` 的调用前提是 adapter 已从**同一次**成功 `get` 的结构化结果确认归属与结果；runtime 仍复核 ID、SHA-256 digest 形状、非空来源、字符串长度和作用域。不可通过再执行一次 `get` 拼接字段：两次命令可能读到不同 revision。现有 JSON envelope 可作为候选数据来源，**但 Hook 能否完整、可靠地拿到该 envelope、退出状态和发起者尚未证明**；不能把 shell 包装文本解析当成结构化合同。依据：[输出协议](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/output/protocol.ts)、[get 合同](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md)。#215 也明确禁止解析面向人的输出。

字段按读取时快照解释：`practiceId`、`title`、非空时才保留的 `appliesWhen` 来自 `data.practice`；`contentDigest` 来自 `data.contentDigest`；`sourcePackNames` 来自同一结果的全部 `sources`，排序、去重后保存，**不随意选一个“主 Pack”**。只存 Pack 名，不声称知道读取时 Pack version：现有 `get` 结果没有该字段，事后另读 Pack list 会产生跨 revision 拼接。过期、升级或卸载后，这些字段仍只表示“当时读到”，不是当前可安装性证明；使用者需要时自行重新 `get`。正文、原始 prompt、transcript、shell 输出和私有查询均不得入簿。

会话主键由 `hostKey` 与根引用构成；持久化时可对不透明根引用生成本机内部键，避免把原值写进文件名、提示或日志。`storeScope` 进一步过滤所选 Store，避免一次显式 `--store-root` 读取泄露到使用默认 Store 的提示；它不改变 Issue 要求的 Host/根会话隔离。CLI 可使用独立于 `store.sqlite`、manifest 和索引的 runtime SQLite 文件，例如配置根目录下专属的 `runtime/conversation-ledger.sqlite`，权限限制为当前用户；`--store-root` 只选择内容 Store，不迁移 ledger 文件。该路径与安装、reindex、恢复互不参与事务，具体落点须在 OpenSpec 审查。现有项目已在测试中使用 Bun SQLite，无需为此引入新依赖；独立文件避免把短期会话状态混入 Engine 的 canonical/派生数据。

**写入与并发规则：**初版每个 `(Host 根会话代次, storeScope, practiceId)` 最多一项。相同 ID/digest 的再次可验证读取更新最近观察顺序；digest 变化时以**事务提交顺序**替换快照，而非声称它一定是最新 Pack 内容；digest 不变、来源变化也以本次完整来源集替换。单个 SQLite 事务完成 upsert、裁剪和读取顺序号分配，并发写与 clear 串行。若 Host 没有稳定事件 ID，无法识别“旧事件在新内容之后延迟重放”与真正再次读取旧内容；不能许诺超出证据的精确事件时间序。#215 接入前应验证事件唯一性及重试行为，必要时把去重事件键纳入合同。

**父子与生命周期规则：**初版只接受可验证的 root Agent 写入。后续子 Agent 可以在可靠父子绑定下只读父根候选；子 Agent 自己的 `get` 默认不回写父簿，兄弟之间也不传播。`resume` 保留同一代次；`compact` 不清空也不在本阶段消费；`clear` 原子清空该根候选并推进代次，旧代次迟到写入必须被拒绝。若宿主沿用同一个根引用，却无法证明 clear 的代次或事件先后，不能靠查“当前会话”猜测，应关闭该 Host 的写入/提示能力直至关联成立。异常退出和 `SessionEnd` 不立即删除，以便可靠 resume；读取、写入时机会性清理过期记录，缺少 lifecycle event 的 Host 可以不启用相关能力。

**有界与降级：**建议在 OpenSpec 中将初始上限列为可调实验参数，例如每根会话最多 32 项、最近读取后 7 天过期，并设置全局会话数及单项元数据字节上限；先完整保留一项的全部来源，超限则拒绝该项而非伪造不完整来源。按最近观察顺序、再以 ID 作确定性 tie-break 裁剪，清理操作不启动后台 daemon。ledger 损坏、忙碌、超预算或关联失败时，记录路径不改变原始 `get` 的结果，提示路径返回空且不阻塞 Host；内部诊断不得带出会话引用和候选正文。#216 的最终 context 预算还须与 Catalog 合并评估，不能靠截断 Practice ID 或注入 body 填满预算。

## 取舍与 Deferred

推荐独立的 CLI-owned ledger SQLite：它用一个明确的事务边界处理跨 Hook 进程并发、clear fencing 和裁剪，代价是独立 schema、权限与故障恢复测试。每会话 JSON 文件看似轻量，但会把并发覆盖、原子替换、全局清理和代次处理重新写成文件协议。Plugin 私有文件虽接线短，却不能给第二 Host 共享语义；Engine/LocalStore 则把短命 Host 状态带进 canonical Store 生命周期，均不选。第二个真实 Host 出现后，再凭其接入成本决定是否抽出 workspace package，而不是提前猜测稳定的跨包 API。

本 Issue 不实施 Codex 或其他 adapter、不自动 query/get/采纳、不存 body、不引入本地 MCP，也不实现 compact 后恢复。#215 在真实事件证据成立后接入主 Agent 写入；#216 才在 `SubagentStart` 消费有界父候选；#217 明确是延后的 compact 消费，仍需独立证明时序、预算、陈旧项处理和实际收益。

## 缺失证据、验收与下一步

目前缺的是**真实 Codex runtime 的脱敏事件样本和关联实验**：成功、失败、取消及退出状态未知的 `get`；直接调用与复杂 shell 包装；root/child 各自调用；`SubagentStart` 的父根绑定；同引用下的 startup、resume、clear 与延迟回调。需确认 Hook 实际收到的结构化结果、事件顺序及 Agent 归属，不能只援引官方字段表。本次只读调查没有进行该实验，因此不能宣布 #215/#216 的自动路径可行。

可核验的方案验收是：用合成 Host envelope 对 root 隔离、不同 Store、同/异 digest、多 Pack 来源、并发 upsert/clear、迟到事件、resume、TTL/上限及失败降级建立确定性测试；证明候选从未包含 body，且 ledger 读写不改变 LocalStore、`get` 输出和原命令退出码。随后以真实 Codex 样本验证关联前提；若前提不成立，验收结果应是**能力保持关闭并记录限制**，不是猜测填补。#216 的注入可见性和预算、#217 的 compact 效益分别留给各自 Issue 的真实运行验收。

下一步是把经评审确认的合同写成 OpenSpec proposal，再讨论实现；仓库规则将这类跨边界生命周期与可观察合同变化归为需先提案的大变更。本次按要求未修改、实施、测试、提交或推送任何文件。Lorelum Skill 的语义检索遇到 `backend.state-invalid`，原调用 trace 没有记录，因此没有把未取得的 Practice 当作设计依据。当前运行也无法切换至仓库要求的 `gpt-6-sol xhigh`；本草案不声称经过该模型路线审查。

