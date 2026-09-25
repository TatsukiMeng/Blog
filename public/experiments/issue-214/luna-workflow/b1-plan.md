完成了 Issue #214 的方案研究，并按要求只启动了一个 `gpt-6-sol` / `xhigh` 规划 subagent。它交付了完整的提案、设计、spec delta 和后续任务文本；我做了主线程复核。

先说明交付边界：工作区是只读的，所以我没有创建 OpenSpec 文件或改代码。`openspec status --json` 显示当前没有 active change；下面给出的是可供评审的完整方案内容，不是已写入仓库或已接受的规范。本轮也没有运行产品测试、采集真实 Codex Hook payload，或对 GitHub Issue 执行写操作。

## 核心结论

#214 应只定义“对话级 Practice 候选簿”的含义、会话绑定和责任边界，不把“已读取”说成“已采纳、有效或适用于新任务”。候选项只保存有限 metadata，不保存 Practice 正文、用户 prompt、transcript 或原始命令。

推荐保持三层职责：

- **CLI runtime** 拥有候选语义、校验、去重、持久化、清理和注入预算。
- **Host adapter** 提供有证据的 Host 身份、root/child 归属和生命周期信息；无法证明就不读、不写。
- **Engine / LocalStore** 继续只拥有 canonical Pack、Practice、来源与 retrieval state，不存短期会话状态。

首个 Codex adapter 可以评估扩展现有 `lore hook codex` raw Hook 入口，但这只是候选调用路径，不代表目前能可靠捕获成功 `lore get` 或关联 root/child。OpenAI 官方 Hook 文档确实列出相关事件字段和 `additionalContext` 出口，但没有替代真实 runtime 配对测试的证据。citeturn0view0turn2view2turn1view3

**进入 #215/#216 前，必须用隔离 fixture 在真实 Codex runtime 核实 `PostToolUse` 的成功结果、root-v-child 归属、取消/包装命令和 clear/resume 时序。** 如果不能证明，不得靠 transcript、cwd、全局 current-session 文件或 shell 文本猜测；对应自动记录/注入就保持可选或延期。

Issue 正文的阶段关系是：#214 定合同；#215 做主 Agent 成功 `get` 首条写入；#216 做 Codex 子 Agent metadata-only 注入；#217 compact 恢复继续延期。#204 的 material decision 主动 re-query 是互补能力，不能由候选簿替代。citeturn1view0turn1view1turn2view0turn2view1turn3view0

## 重要现状与未证实项

现有 `lore get` 已有稳定 JSON 合同：结果含完整 Practice、`contentDigest` 与 `sources: [{packName, sourcePath}]`；同 ID、同 digest 的多 Pack 来源会合并。digest 是 canonical 内容 SHA-256，不是签名；结果不含 Pack version。Hook 如果看到这份结果，必须在内存中只摘取允许的 metadata，不能把 body 持久化。[get 文档](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md:20) [CLI get handler](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.ts:38) [结果 schema](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/result-schema.ts:9)

仓库当前 Codex Hook 仅处理 `SessionStart`；Plugin 也只注册 `startup|resume|clear|compact` 这一匹配器。现有 Hook 单测和编译 CLI smoke 都使用手写 `SessionStart` payload，并非 Codex runtime 证据。[Hook CLI 合同](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/hook.md:1) [Hook 实现](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts:13) [Hook 单测](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.test.ts:52) [Plugin 配置](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/hooks/hooks.json:1) OpenAI 文档说明 `PostToolUse` 会携带工具输入/响应字段，`SubagentStart` 有 `agent_id` 并允许 `additionalContext`；文档还说明 subagent Hook 使用 parent session ID。这些是平台合同，不足以证明 session/turn 字段如何把具体的 `lore get` 绑定到 root Agent，尤其是嵌套子 Agent、clear 和迟到事件。citeturn0view0turn2view2turn1view3

另有一个需要评审解决的存储边界：Sol 推荐在 `~/.lorelum` 下另建 SQLite sidecar。它与 Pack 的 `store.sqlite` 分离，方向上符合 Issue；但仓库已有 SQLite/Drizzle 机制属于 Engine 数据库域，现行持久化指南说 CLI 不直接执行 SQL。因此 sidecar 不能被当作“直接复用现成数据库能力”：方案要明确 ledger adapter 的 owner、schema/migration、连接与权限策略，并比较小型原子文件方案。[Engine 持久化指南](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/development/persistence.md:1) [Engine 包边界](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/engine/AGENTS.md:1)

## 完整方案（Proposed）

### Proposal：定义对话级 Practice 候选簿与宿主会话绑定合同

这是**未来行为提案**，不记录现有能力，也不授权实现。

候选簿记录“某根 Agent 会话中，有证据表明成功读取过某 Practice；它可能仍与任务相关”。它不表示 Agent 采纳了 Practice，不表示该内容现在仍安装，也不证明它对当前任务有效。

候选字段拟为：

- `practiceId`：从 `get.data.practice.id` 取值。
- `contentDigest`：从 `get.data.contentDigest` 取值，标识读取当时内容。
- `title`、可选 `appliesWhen`：作为受限展示/相关性线索，不作为指令。
- `sources`：保留这次成功读取所返回的完整来源快照 `{packName, sourcePath}`；不添加 `get` 当前合同没有提供的 Pack version。
- `recordedAt`：本地持久化提交时间，只用于排序和过期回收，不声称是工具调用开始时间。

持久化必须排除完整正文、用户 query/prompt、raw Hook payload、命令行文本、transcript、绝对路径和凭据。候选只能指向 `lore get`，不能取代 Store 的 canonical 内容。

### 设计：合同、调用路径与责任

会话 scope 使用 `hostKey + opaque rootConversationRef`。`hostKey` 是 Host 命名空间；`rootConversationRef` 由 Host adapter 提供，必须不含 prompt、文件路径或用户信息。resume 只有在 Host 证明仍是同一逻辑会话时才沿用 scope；clear 必须建立新 scope，或有可靠机制失效旧代际并拒绝迟到事件。

最小内部接口可以是：

```ts
interface ConversationScope {
  readonly hostKey: string;
  readonly rootConversationRef: string;
}

type Actor = "root" | "child" | "unknown";

interface ReadObservation {
  readonly scope: ConversationScope;
  readonly actor: Actor;
  readonly observationId: string;
  readonly get: {
    readonly practice: {
      readonly id: string;
      readonly title: string;
      readonly applies_when?: string;
    };
    readonly contentDigest: string;
    readonly sources: readonly {
      readonly packName: string;
      readonly sourcePath: string;
    }[];
  };
}

interface ConversationPracticeLedger {
  recordSuccessfulGet(event: ReadObservation):
    Promise<"recorded" | "duplicate" | "skipped">;
  listForProvenChild(scope: ConversationScope):
    Promise<readonly Candidate[]>;
  clear(scope: ConversationScope): Promise<void>;
}
```

`actor` 未知或为 child 时不写入 root ledger。调用方必须证明 child 与 root 的关系后才能读取对应 ledger；不能让 child 任意指定另一个 scope。接口是内部设计示意，不是已发布 CLI API，也不是当前 Hook payload schema。

主 Agent 成功 `get` 只允许在结果成功、数据完整且 actor/root 归属可证时写入。非零退出、失败、取消、结果不完整、命令包装歧义或会话归属不明，都不记录，也不改变 `get` 原结果。子 Agent 的 `get` 不回写父 ledger。

同一 `practiceId`、相同 digest 的重复成功读取更新为**最近一次 get 的完整来源快照**，不把多次调用的 sources 做历史合并。同 ID 新 digest 建议替换旧候选，避免同一定位 ID 下保留互相矛盾的候选版本；“最新”定义为 ledger 提交顺序，不是工具调用开始顺序。若无法证明这个排序符合实际事件语义，则候选只能是弱提示，不得自动用于决策。

推荐把首次 Host 调用放在 CLI 的 raw Hook ABI 内，由 `lore hook codex` 根据事件分派；Plugin 只负责注册/转发 Hook，不自行持久化通用 ledger。新增公开 `lore ledger record` 一类命令会允许调用方自行声称“成功读取”，需要另一个可信合同；当前优先评估复用 Hook 入口，但若实际 payload 不足，必须重新设计显式、可验证的回执路径，而不是猜测。

### 持久化与保留策略

推荐 ledger 是 **CLI-owned、独立于 LocalStore 的本地状态**：不在 `store.sqlite`、Pack 快照、Engine retrieval state 或 Plugin 专属 `PLUGIN_DATA` 中，也不受 `--store-root` 改变。共享 `~/.lorelum` 路径可由 `@lorelum/config` 解析，但实际子目录和存储 adapter 是新责任，必须在评审中明确。

Sol 的候选是单独 SQLite sidecar，事务支持多进程原子写；替代方案是受限 JSON 文件加原子替换与跨进程锁。SQLite 是当前偏好的实现候选，不是已核实的仓库惯例。评审应先确认新 adapter 的模块 owner、schema/migration 和连接生命周期，且不得直接让 CLI command handler 执行 SQL。

可作为压力测试起点、而非最终产品阈值的参数：

- 每 scope 最多 32 条、序列化 metadata 最多 32 KiB；单条最多 4 KiB。
- `title` 最多 256 UTF-8 bytes、`appliesWhen` 最多 512 bytes、sources 最多 16 项；超过单条预算就跳过整条，不截断来源。
- 7 天未新增 root 成功读取后过期；读取或子 Agent 注入不刷新 TTL。
- 全局最多 1,024 scopes / 16 MiB 候选 metadata；写入前清理过期项，再按确定性规则淘汰最旧候选。
- 注入候选暂定最多 8 条、约 800 tokens，按最近成功记录与稳定 ID 排序；不依赖宿主帮 Lorelum 决定如何裁剪。

这些数值需用真实 payload 形状、候选数、metadata 大小、Hook 延迟和淘汰率校准，不能称为已有预算。若保存 `observationId` 用于并发/重放幂等，其数量和保留期也必须纳入同一总预算；Sol 初稿未给该去重数据单独的容量上限，这是需要补齐的设计点。

### 子 Agent 提示与失败语义

#216 才消费 ledger：只有在 root-child 绑定被证明时，才可向 child 注入有限 metadata；提示必须说明这是“曾读取、可能过期的候选”，不自动 `query/get`，不注入 body，也不把 Pack 提供的 title/appliesWhen 当成系统指令。

无候选、session 绑定失败、文件/数据库不可用、超预算、渲染失败，都应继续启动 child，且不给候选提示。写入失败也不能改变原 `get` 结果。Hook 必须保持 fail-open，不能用 `continue:false` 阻止已经完成的操作或 child 创建。Plugin Hook 仍受用户 review/trust 机制控制；部署侧应确认被信任的 Hook 配置确实包含该事件处理。citeturn1view3turn1view4

### 取舍与分期

| 选项 | 取舍 |
|---|---|
| 混入 LocalStore | 复用数据库方便，但会把短期 Host 生命周期、GC 与 canonical Pack/Store recovery 耦合，违反 Issue 的状态边界；不选。 |
| Plugin 自己保存 `PLUGIN_DATA` | 单 Host 方便，但 ledger 语义随 Plugin 私有化，不能形成跨 Host runtime contract；不选。 |
| CLI-owned 独立 sidecar | 符合 issue 的语义 owner 与 LocalStore 隔离；需新增存储 owner，SQLite/迁移成本需要明确。推荐方向。 |
| 小型 JSON + 文件锁 | 可减轻数据库/schema 成本，但跨进程原子更新、锁恢复和有界淘汰需要自己处理；作为对比项，不能仅因数据小就认定简单。 |
| 新建通用 workspace package | 目前只有一个 Host consumer，会提前固化尚未验证的抽象；待第二个 Host 有实际复用证据后再评估。 |

### 验收和后续阶段

#214 的评审应确认字段语义、scope 隔离、来源多源规则、同 ID 新 digest 策略、失败语义、并发/清理/预算边界，以及 #215/#216 的最小内部调用合同。真实 runtime 证据必须单列，不得用合成 fixture 代替。

后续验收建议覆盖：

- **CLI/runtime 单测**：只接收成功回执；body、prompt、transcript 不存储；失败/取消/child/unknown actor 不写；重复事件幂等；digest 更新与多来源 snapshot 规则明确。
- **持久化测试**：隔离 root、双进程并发、原子性、锁超时、崩溃恢复、过期与容量回收、异常降级；事件去重数据也有限。
- **Hook 合成测试**：验证协议解析和 fail-open，不声称证明真实 Codex 行为。
- **真实 Codex runtime 测试**：在隔离 Store 与无秘密 fixture 上核验 root/child 成功、失败、取消、直接及复杂包装、并发/迟到事件、nested child、clear/resume/SubagentStart 字段与输出配对；只保存匿名化字段形态和结论。

阶段边界为：设计审阅通过后，#215 才在有证据时实现 root 成功 `get` 首条写路径；#216 再在 SubagentStart 与父根关联证据通过后实现可选 metadata-only 注入；#217 compact 恢复仍独立延期。#204 的新 material decision re-query 不从 ledger 推断充分性。上述 Issue 页面给出的先后与非目标均以其正文为准。citeturn1view1turn2view0turn2view1turn3view0

## 主要取证来源

- Issue #214 与关联阶段：[#214](https://github.com/lorelum/lorelum/issues/214)、[#215](https://github.com/lorelum/lorelum/issues/215)、[#216](https://github.com/lorelum/lorelum/issues/216)、[#217](https://github.com/lorelum/lorelum/issues/217)、互补问题 [#204](https://github.com/lorelum/lorelum/issues/204)。#214 正文还列出 #28/#32/#35 作为研究背景，本方案没有把它们扩成当前实现范围。citeturn1view0turn1view1turn2view0turn2view1turn3view0
- Codex Hook 平台合同：[OpenAI 官方 Hooks 文档](https://developers.openai.com/codex/hooks)；Plugin trust 与 `PLUGIN_DATA`：[官方 Plugin 文档](https://developers.openai.com/plugins/build/plugins)。结论仅限官方 release 文档；未将文档字段描述冒充为本机真实 runtime 采集。citeturn0view0turn2view2turn1view3turn1view4
- 当前实现合同：[agent-integration spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md:8)、[practice-read spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/practice-read/spec.md:8)、[Plugin distribution spec](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/plugin-distribution/spec.md:15)、[get CLI 文档](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md:20)、[Hook 文档](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/hook.md:1)、[Hook 源码](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts:13)、[Hook fixture](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/integration/scenarios/hook-codex.ts:9)、[OpenSpec 配置](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/config.yaml:1)。
- Lorelum Pack 查询按 Skill 执行，但本轮 `lore query` 返回 `backend.state-invalid`，对应 trace 没有记录；一次明确的 keyword-runtime 诊断也失败，因此未读取到 Practice 正文。未把该检索失败作为架构证据；关键取舍来自当前 checkout、Issue 和 OpenAI 官方资料。

## 实际交给 Sol 的 prompt

下面是一次性交给唯一 subagent 的原始任务 prompt：

```text
你是本任务唯一的技术方案作者。请以 gpt-6-sol / xhigh 完成 Issue #214 的可评审技术方案；不得再委派任何 subagent。用户只授权研究和设计，不授权实现、改代码、创建/修改 OpenSpec 文件、提交、发 Issue 评论或其他外部动作。当前 sandbox 为 read-only；请只在回复中交付拟议文档全文。允许必要、只读、范围明确的核实，但不要重做下面已完成的宽泛检索；如果关键事实确实无法据现有材料判断，请清楚标为未证实/设计门槛。

## 目标与阶段范围
Issue #214: https://github.com/lorelum/lorelum/issues/214
中文标题「定义跨 Agent 的对话级 Practice 候选簿与宿主会话绑定合同」。这是设计阶段，不是实现任务。范围是定义一个跨 host 可复用、窄范围的 ConversationPracticeLedger（名称可调整）合同：条目表达某 Agent 在任务会话中成功读取、可能仍相关的 Practice；不得表达 adopted/effective；至少讨论 practiceId、contentDigest、title、可选 appliesWhen、来源 Pack 身份；使用 hostKey + opaque root-conversation reference 隔离；统一定义去重/更新/并发/父子可见性/clear-resume/清理/有界保留；不存完整 Practice body。

Issue 自身明确的非目标：不实现 Codex 或其他 Host Adapter；不自动 query/get、自动采纳或注入完整 body；不做 workflow engine、通用 Agent memory、跨设备同步；不做 compact 恢复；不引入本地 MCP。#214 设计要给 #215、#216 后续阶段提供最小合同，但不能把它们实现合并进本阶段。

## 阶段关系（已核对 Issue 正文）
- #214 是合同/所有权设计阶段（design 标签、open）。
- #215 依赖 #214，实现可验证的主 Agent 成功 `lore get` 首条写路径；失败/取消/退出状态不明、无法解析参数或会话无法关联都不记录；子 Agent 读取不回流父簿；不允许解析默认人类可读输出，必须有稳定 CLI 结果合同或等价内传；记录失败不可影响原命令。其非目标：自动 query/get、注入或 compact 恢复、改变 Practice/Pack schema。
- #216 依赖 #214/#215，是第一个 Codex adapter：SubagentStart 时按可靠根会话读取父簿，注入可选、有界、metadata-only 提示；没有数据、关联失败、预算超限、渲染失败均不阻塞启动；不自动 query/get，不做 compact 恢复，不做 MCP。
- #217 明确 Backlog/延期，处理 compact 后 root-session 提示恢复；当前不属于首轮。
- #204 是互补问题：新 material decision 时主动 re-query 与充分性判断；不是本 ledger 能力。Issue #214 正文还把 #28/#32/#35 列为研究背景，不要据此扩展当前范围。

## 已核实仓库证据（不要让读者重新搜索；请把这些作为设计输入并引用路径/行）
工作区：`https://github.com/lorelum/lorelum/tree/88e691ba939943e67793e92e1899387971dbce36`。当前没有 active OpenSpec change；`openspec/config.yaml` 默认 schema 为 `spec-driven`，所有 artifacts 要用中文，结构标题和 SHALL/MUST 保留英文。大改动要先经 OpenSpec proposal。建议在回复里给拟议的 proposal.md、design.md、spec delta、tasks.md 内容；spec delta 选择现有 `agent-integration` 还是新 capability 要由你论证，不预定。所有内容必须标为 proposed，不能写成已接受合同。

当前规范/ADR：
- `openspec/specs/agent-integration/spec.md:8-16,26-42`：本地 Agent integration 是已发布 CLI + host-native Skill/Hook；Plugin/Skill/Hook 通过 CLI 合同，不直接读 Store/导入 Engine/Backend；Catalog 只是 metadata，Hook 不自动 query/get；明确禁止本地 MCP。
- `openspec/specs/practice-read/spec.md:8-20`：`lore get` 是 Engine 从选定 LocalStore 对一个 canonical Practice 做一致 point read。
- `openspec/specs/plugin-distribution/spec.md:15-20`：Plugin 负责宿主上下文编排，不能复制检索 runtime 或持有 Store/Backend/ranking。
- `docs/adr/0007-engine-local-store.md:24-28,109-121`（Accepted）：LocalStore/Engine 拥有 `~/.lorelum` 的 Pack canonical 内容、manifest、journal、SQLite projection、digest、恢复等；不适合混入 conversation state。该目录现在有 `installed-packs.json`、`store.sqlite`、Pack snapshots、operations 等。
- `openspec/config.yaml` 的当前项目上下文强调 CLI-first、不建本地 MCP、当前 spec 才是行为合同；不要把 Proposed ADR 提升为已接受决定。
- `docs/adr/0004-agent-first-cli-protocol.md:1-16,37-49` 当前仍是 Proposed；稳定 JSON envelope 可由 CLI docs/code/tests 证明，但不要称这份 ADR 已接受。
- `packages/cli/AGENTS.md`：CLI owns command parsing/protocol/execution route，Engine/Backend 不依赖 CLI；CLI 变更要补命令/schema/error tests 与 docs。
- `packages/engine/AGENTS.md`：Engine owns canonical Practice/LocalStore/retrieval state，不依赖 CLI/Backend；新存储/持久化必须遵循现有 owner boundaries；测试用隔离临时 Store。
- `docs/AGENTS.md`：OpenSpec spec 是当前能力合同，docs/cli 等解释合同，ADR 仅 Accepted 才是持久决策，研究/归档是背景。
- 当前代码未找到任何 conversation ledger/session candidate store 类型或 OpenSpec capability。

`lore get` 已经有机器合同，不要无证据地假定需要新造：
- `docs/cli/get.md:20-59`：成功 protocol envelope 的 data 有 `practice`（包含完整 body）、`contentDigest`、`sources: [{packName, sourcePath}]`；digest 是 canonical content SHA-256，不是签名；同 id+同 digest 多 Pack source 会合并并按 Pack name/path 排序；分开的 CLI 调用不 pin revision；错误通过 `ok:false/error.code`、exit 2 表达。
- `packages/cli/src/get/get-command.ts:19-59`：GetCommand 只用 `LocalStore.getEffectivePractice` 一次，返回 Practice、digest、packName/sourcePath；失败映射 Store error。
- `packages/cli/src/get/result-schema.ts:9-60` 与 `packages/cli/src/get/get-command.test.ts:75-92,146-160`：结果 schema 明确含 body/digest/source，测试检查结果与错误。
- `packages/engine/src/local-store/model/types.ts:25-41`：EffectivePractice 有 practiceId/digest/content/source array；来源有 packName、相对 sourcePath、digest。
- 注意：当前 `get` schema 没有 Pack version 字段；Issue 的“source Pack identity”需解释为哪些稳定字段，不能猜。
- `packages/cli/src/output/protocol.ts:33-48,90-119`：常规 CLI 为 JSON success/failure envelope；`get` 人类可读 layout 的 Issue #215 措辞与当前已实现 JSON output 不完全相符，但 Hook Bash 的 tool_response 如何包装 CLI stdout 还没实测。指出此处“现有 CLI 稳定合同 vs Host Hook 实际暴露”两层差异，不要宣称已解决 Hook payload 问题。

当前 Codex Hook 产品事实：
- `docs/cli/hook.md:1-19`：`lore hook codex` 是单行 raw Codex Hook ABI；当前只支持 SessionStart；错误 fail-open 输出 `{"continue":true}`，不 query/get、不启动 Backend、不改 Store。
- `packages/cli/src/hook/codex.ts:13-24,89-115,137-149`：事件类型仅 `SessionStart`；Hook input 类型现在只保留 `hook_event_name`；SessionStart 调 `listPackDetails` 渲染 Catalog；错误不阻塞 Host。
- `packages/cli/src/hook/codex.test.ts:52-123`：只有手写的 SessionStart、malformed JSON、unsupported PostCompact、Store failure fixtures；不含真实 PostToolUse/SubagentStart runtime capture。
- `plugins/lorelum/hooks/hooks.json:1-19`：Plugin 只配 SessionStart，matcher `^(startup|resume|clear|compact)$`，执行 `lore hook codex`，additionalContextLimit=5000，Shell/Windows wrapper fail-open。
- `plugins/lorelum/scripts/hooks-config.test.ts:6-35,37-90` 和 `packages/cli/integration/scenarios/hook-codex.ts:9-51`：结构/ABI 集成测试也使用合成 SessionStart payload，不是 Codex runtime evidence。
- `docs/plugins/codex.md:1-39` / `docs/development/plugins.md:1-40`：CLI-first、Plugin 自己不读 Engine/Store，当前 catalog 是 metadata-only；测试 smoke 是手写 JSON + CLI source，不是运行中的 Codex。
- `packages/engine/src/local-store/lifecycle/local-store.ts:26-34`：默认 Store root 从共享 `@lorelum/config` 得到 `~/.lorelum`。`docs/configuration/README.md:1-25,29-43`：该根目前是 config/backend/model/store 相关目录，没有 general session ledger。

## Codex 官方资料（本轮已 Web Search 官方 OpenAI 文档，按 release docs 而非仓库 main schema）
- Hooks: https://developers.openai.com/codex/hooks
- Plugin hooks packaging/trust/data dirs: https://developers.openai.com/plugins/build/plugins
已核实的官方 release 文档合同：
1. 所有 command Hook 通过 stdin 收 JSON common fields `session_id`, `transcript_path`, `cwd`, `hook_event_name` 等；官方文档说 subagent hooks 使用 parent session ID。`transcript_path` 的具体 transcript format 不是稳定 Hook interface。
2. `PostToolUse` 有 `turn_id`, `tool_name`, `tool_use_id`, `tool_input`, `tool_response`；Bash 的 input 是 `tool_input.command`，工具响应是 tool-specific JSON，Bash 即使退出非零也会运行 PostToolUse；工具覆盖不是完整的保证。
3. `SubagentStart` 有 `turn_id`, `agent_id`, `agent_type` 等；stdout JSON `hookSpecificOutput.additionalContext` 会作为子 Agent developer context。`continue:false` 不会阻止子 Agent 创建。
4. 多个匹配 Hook 可以并发执行；plugin bundled Hooks 需先 review/trust；Codex Plugin Hooks 的 `PLUGIN_DATA` 是可写路径，但 Issue #214 约定不让 Plugin 私自定义/持久化通用 Ledger。
5. Hook output context 默认约 2500 tokens，多个 Hook/Plugin 累计；当前 Lorelum catalog handler 单独设置 5000。任何预算裁剪应由 Lorelum 决定并可复现，不能只假设宿主 budget 会替我们裁对。

## 最关键未证实项（不要写成已证实）
当前只有官方 ABI 描述，没有一次在真实 Codex runtime 上采集并匿名化的 `PostToolUse(lore get success/failure/cancel)` 与 `SubagentStart` 输入/输出配对。当前合成测试仅 SessionStart。特别不清楚：PostToolUse 上的 session_id/turn_id 是否足以区分 root main Agent 与 child Agent 的 `lore get`；session_id 是否能作为稳定 root conversation ref（尤其 nested subagents、clear/resume）；`tool_response` 是否可以无损且稳定地识别 CLI JSON success/data/exit status；Shell 包装/管道/取消如何表示。transcript、cwd、全局 current-session 文件、shell 文本猜测均不能作为未经证实的 fallback。方案必须设置 controlled runtime evidence gate；无证据则将对应 Codex 功能标 optional/deferred，不伪造关联。

## 要你完成的设计决策（给明确推荐 + 少量真实替代项，并讲清为什么）
1. Ledger 的身份/语义：read/considered candidate 边界；source of truth vs derived hint；每个字段的来源、可变性、隐私和多源语义。
2. Host-neutral contract：`hostKey + opaque rootConversationRef` 的类型/不变量；adapter 如何提供身份/actor/lifecycle；root vs child 的可见范围以及父/子读取是否回流。
3. 当下实现责任边界和最小 API：CLI runtime 与 Host adapter 的职责，Pack/LocalStore/Engine 明确不持有它；是否改现有 `lore hook codex` ABI 或引入独立 CLI operation，必须基于调用路径分析，不要预定。
4. 持久化候选对比：可行的本地位置/介质、与 `~/.lorelum/store.sqlite` 和 `PLUGIN_DATA` 的边界、跨进程并发/atomic/idempotent update、崩溃恢复、clear/resume/SessionEnd、TTL/条数/字节的有界策略。精确阈值若无证据，标成拟议起始值/校准方法。
5. 记录字段与更新规则：同 practiceId+同 digest 去重，多 Pack sources；同 id 新 digest 是替换/并存/保留历史需推荐且讲明语义；现有 get 没有 packVersion；禁止正文和私密输入进入 Ledger。
6. 失败与 Host 降级：只记录可证实成功、归属确定的 root/main `get`；失败/取消/未知/歧义不写且不改变原命令；查询/注入路径不阻塞 Host；无候选/不可关联/超预算安全不注入。
7. 有针对性的验收矩阵：CLI/runtime 单测、持久化/并发/恢复测试、Hook synthetic fixtures、真实 Codex runtime acceptance（成功/失败/取消/root-v-child/包装/clear/resume/SubagentStart），并明确哪一层证明什么。不要把本轮“没抓到 payload”伪装成通过。
8. 明确 #215、#216、#217 的依赖边界/重开条件：#215 实现主 Agent 写入；#216 有证据后只注入元数据；#217 compact recovery 继续 deferred。

## 输出格式
请用中文，按当前 `spec-driven` 的意图输出一份可评审、可实施但范围锁在 #214 的 OpenSpec 提案内容（在回复中给出文件标题和完整文本）：
1. `proposal.md`：what/why、目标、非目标、是否引入未来行为、phase gate。
2. `design.md`：现状→问题→目标→责任→合同/API/存储/错误/并发/生命周期→取舍→验证，区分 Observed/Required/Proposed/Deferred。对主要接口给最小 TypeScript interface / schema 示例，不要发明未经证据支持的 host 字段。
3. `specs/<合适 capability>/spec.md` delta：只写这项提案希望未来可观察的 SHALL/MUST 行为，保留现有 CLI-first 和禁止本地 MCP。选 `agent-integration` 或新 capability 时说明理由。
4. `tasks.md`：只列未来在设计审阅批准后才可启动的工作；把真实 Hook payload capture 作为进入 #215/#216 前的 gate。不要标完成，不要将 #215/#216/#217 的实现塞进 #214。

开头给一段明确推荐；对架构/存储/会话关联的争议不许照搬 Issue 的 ownership 句子当论证，也不许预先采纳我的任何候选方案，要按现状、反事实和可验证风险做判断。材料不足时写“未证实”及它改变哪些决策。精确引用 repo 文件+行号和官方 source；不要只留给实现者“以后再探索仓库”。不要声称改了任何文件或运行了任何测试。
```

最后，OpenSpec 提案的拟议 artifact 集合是 `proposal.md`、`design.md`、`specs/agent-integration/spec.md` delta 和 `tasks.md`；spec delta 与任务都应明确保持 Proposed/未开始状态。上面给出的方案文本是评审稿，SQLite sidecar 和数值预算仍是方案建议，不是已决策的实现细节。

