# Issue #214 技术方案：对话级 Practice 候选簿

**状态：待评审设计，非实施方案批准。** 本次只读调查未修改文件、运行测试、提交或推送。建议先确定通用候选簿合同，再以真实 Codex Hook payload 证明自动记录可行；在归属证据不足时，不启用 Codex 自动写入。

## 结论与范围

在 `lore` CLI runtime 中定义一个独立的 `ConversationPracticeLedger`：它记录“某个可确认归属于本任务会话的 Agent，曾成功读取这一版 Practice”。记录仅供后续 Agent 判断是否值得重新 `lore get`，**不表示已采纳、有效、当前仍已安装，或必须遵循**。同一根会话下的主 Agent 与子 Agent 可以看到同一候选集合；host 负责证明会话和读取事件的关系，Lorelum 负责统一去重、持久化、保留和读取规则。这个边界符合 Issue #214 的目标与非目标。([github.com](https://github.com/lorelum/lorelum/issues/214))

本 Issue 的交付是可供后续实现依赖的合同。Codex `PostToolUse` adapter、compact 后的候选注入、自动 query/get、Practice body 注入、跨设备同步及本地 MCP 均不在本阶段。后续如获准实施，应按仓库的大变更规则建立 OpenSpec change；本次只读要求优先，因此没有创建文档。

## 已确认的依据与关键缺口

| 类别 | 依据 | 对设计的影响 |
| --- | --- | --- |
| Observed | `lore get` 从选定 LocalStore 做一次 canonical point read，成功输出 `practice`、`contentDigest` 和排序后的全部 `{packName, sourcePath}`；失败有独立 envelope。相邻两次命令没有跨命令快照保证。[实现](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/get/get-command.ts#L38)、[当前合同](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md#L20) | 候选摘要可取自一次成功结果；以后使用时仍须重新读取并比较 digest。 |
| Observed | `contentDigest` 是 canonical 内容的 SHA-256，**不是来源签名**。同一 ID 的多 Pack 同内容来源合并返回；当前 `get` 不给 Pack 版本。[当前合同](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md#L35) | 保留全部来源，不任选“主来源”；不能通过另一次 `pack list` 猜测读取当时的 Pack 版本。 |
| Observed | 当前 `lore hook codex` 只接受 `SessionStart`，只注入 Pack catalog；Plugin 也只配置了这一事件。[Hook 实现](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/hook/codex.ts#L105)、[配置](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/plugins/lorelum/hooks/hooks.json#L1) | 目前没有自动记录成功 `get` 的实现，不能把现有 catalog 当候选簿。 |
| Observed | OpenAI Docs 指出子 Agent Hook 使用父会话 `session_id`；`PostToolUse` 给出工具输入、响应及 `tool_use_id`，但其列明字段没有发起 Agent ID。`SubagentStart` 有 `agent_id`，文档未证明它能与每个后续 `PostToolUse` 无歧义关联。 | `session_id` 只能作为待验证的根会话线索，不能凭它判定一次 `get` 来自主 Agent。 |

**缺失证据是上线门槛，不是设计中的默认假设。** 尚无本次调查取得的真实、脱敏 Codex Hook payload，能够覆盖主/子 Agent 的成功与失败 `get`、复杂 shell 包装、取消、并发、resume 与 clear。公开文档说明事件形状和覆盖范围，也提醒部分工具路径可能不走默认 Hook；它不能证明某次实际调用的归属与完整可观测性。

## 责任与最小合同

- **Host adapter** 拥有宿主生命周期解释：提供稳定的 `hostKey` 和不解析其内容的根会话 `rootRef`，确认发起者 `readerRef`，把一次真实、成功的 `lore get` 结果转换成摘要。它不得仅以进程、工作目录、父会话 ID 或可伪造的 shell 文本推断发起者。
- **CLI runtime** 拥有候选簿状态和公开 JSON 命令：验证输入、隔离会话、去重、并发事务、clear、裁剪及读取。它不解释 Codex 专用字段；`hostKey` 只是命名空间，不参与 Practice 规则。
- **Pack / LocalStore** 继续拥有正文、canonical digest 和来源。候选簿既不写入 Store，也不参与 Engine 检索和索引；失效摘要不能成为正文的替代来源。这保留了现行 CLI-first 集成和 canonical 读取边界。[集成规范](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/agent-integration/spec.md)、[读取规范](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/openspec/specs/practice-read/spec.md)

建议的内部业务合同如下；名称和命令拼写可在 OpenSpec 评审时定稿，字段语义不应留给各 adapter 自行定义：

```ts
type Binding = {
  hostKey: string;       // 稳定的 host 命名空间
  rootRef: string;       // host 给出的不透明根会话引用
};

type Candidate = {
  practiceId: string;
  contentDigest: string;
  title: string;
  appliesWhen?: string;
  sources: readonly { packName: string; sourcePath: string }[];
  lastReadAt: string;    // CLI 提交记录的时间，不由 host 声称
};

type ReadObservation = {
  binding: Binding;
  epoch: string;         // 从 open 取得，并在工具调用开始前绑定
  eventRef: string;      // 同一次已验证工具调用的稳定标识
  readerRef: string;     // adapter 已确认归属；不需要持久保存或展示
  selectedStoreRoot?: string;
  candidate: Omit<Candidate, "lastReadAt">;
};

interface ConversationPracticeLedger {
  open(binding: Binding): Promise<{ epoch: string }>;
  record(input: ReadObservation): Promise<
    "inserted" | "refreshed" | "replaced" | "duplicate"
  >;
  list(binding: Binding, selectedStoreRoot?: string): Promise<{
    epoch?: string;
    candidates: readonly Candidate[];
    evicted: boolean;
  }>;
  clear(binding: Binding, expectedEpoch: string): Promise<{ epoch: string }>;
}
```

CLI 命令建议为 `lore ledger open|record|list|clear`，使用现有单行 JSON envelope、`describe` schema 和错误码惯例。[CLI 合同](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/README.md) 绑定信息及记录摘要经 stdin 传递，不把会话引用或工具输出放进命令行参数。`record` 只接收 adapter 提取后的字段，不接收或持久化完整 `practice.body`、anti-pattern 正文、Hook 原始 payload、transcript 或 shell 输出。它仍需对 ID、digest、来源、字符串和总字节数做严格校验；CLI 对本机调用者的声明做验证，但**不能凭一个调用者自报的 `readerRef` 实现安全认证**，归属保证由 host adapter 与其测试承担。

## 会话、来源与状态语义

会话隔离键是精确的 `(hostKey, rootRef)`；不同 host、不同根会话不共享候选。`rootRef` 是 opaque 值，不从其字符形状推断父子关系。adapter 只有在证明子 Agent 属于该根会话后，才传同一 binding。根会话内候选双向可见，包括兄弟子 Agent；读取列表不会自动注入正文或升级候选权重。

同一会话、同一选定 Store、同一 `practiceId` 只保留一条当前候选。再次成功读取相同 digest 时刷新读取时间与完整来源集合；digest 改变时替换旧摘要，不伪称旧版仍是当前候选。`sources` 保留 `get` 返回的**全部** Pack 名和相对路径；不推断 Pack 版本。来源变化但内容 digest 不变也更新来源快照。候选代表“当时读到的内容”；Pack 后续升级或卸载不会反向修改历史读取事实。消费者需要正文时执行 `lore get <practiceId>`，比较 digest；缺失或不一致则将候选视为过期线索，使用新读取结果自行判断，不能沿用旧摘要作当前事实。[当前 `get` 合同](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/docs/cli/get.md#L35)

`--store-root` 是现有 `get` 的选定 Store 机制。[解析实现](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/cli/src/store/storage-root.ts#L1) 建议候选另带由 CLI 根据规范化绝对 Store 路径计算的本机 `storeScope`，列表默认只返回同一 Store 的候选；不在输出或数据库中保存绝对路径。显式路径与默认路径指向同一规范化路径时得到同一 scope。它用于避免不同 Store 中同 ID 的候选混用，不是权限凭据，也不承诺解析符号链接别名。全局 `--store-root` **不搬迁候选簿数据库**。

`open` 对已有会话幂等，resume 和 compact 不清空；不存在时创建随机 epoch。`clear` 在一个事务中核对预期 epoch、删除该会话候选并轮换 epoch。一次工具调用必须携带调用开始前取得的 epoch：clear 后才完成的旧调用即使迟到也不得回填。缺失会话、epoch 不符或无法确认 reader/成功结果的 `record` 失败且不写入；自动 Hook 对 Lorelum 失败应让宿主继续，但不可把“未记录”显示成“列表确实为空”。`eventRef` 在保留窗口内去重，使同一 Hook 重试不重复刷新；不同事件按**成功记录的提交顺序**更新，不声称按工具开始时间排序。

## 持久化、保留与降级

建议由 CLI 模块拥有一份独立、仅本机用户可读写的 SQLite 文件，放在独立应用状态目录，**不在 Pack LocalStore、派生索引、Plugin 私有目录或全局“当前对话”文件内**。用 Bun 已有的 SQLite 能力和参数绑定语句；此固定小型状态不要求新增 workspace package、Backend daemon 或依赖。目录及文件须限制用户权限并拒绝不安全的文件类型/符号链接；跨进程写入用 SQLite 事务和有限 busy timeout。损坏或不可访问时返回明确的 `ledger.unavailable`，不自动删除数据库，也不影响 `lore get`。现有 Backend 已有 Bun SQLite 跨进程协调用法，但其状态和数据库不应与候选簿共用。[现有用法](https://github.com/lorelum/lorelum/blob/88e691ba939943e67793e92e1899387971dbce36/packages/backend/src/runtime/coordination/sqlite-exclusive-lock.ts#L1)

保留规则建议先作为**待校准参数**：每根会话最多 64 条、无新成功读取 30 天过期、另设全局记录/元数据字节上限；单条记录超限或来源过多时拒绝整条，不悄悄截去来源。写入事务内先按确定性顺序裁剪最久未读候选及过期会话；`list` 不续期。返回 `evicted` 提醒列表可能不完整。物理 SQLite 文件及日志文件的峰值占用、清理频率和上述起步值，需用目标会话/来源规模测试后定稿；不能仅以逻辑行数上限宣称磁盘已严格受限。宿主缺少可靠 `SessionEnd` 时仍能靠过期清理，不伪造结束事件；OpenAI Docs 也说明 Codex 的 `SessionEnd` 不适用于子 Agent，且可能延迟发生。

## Codex adapter 的启用门槛

先在**获准的后续验证阶段**以脱敏 fixture 捕获当前 Codex 运行形态中的 `SessionStart`、`SubagentStart`、工具调用前后及必要的 clear/resume 事件。逐例证明：

1. 一次成功 `lore get` 能由工具输入、退出状态和符合 schema 的单行 JSON 结果识别；失败、取消、超时、混合输出和复杂 shell 包装不会误记。不能可靠拆解的包装直接跳过。
2. 根会话、实际发起 Agent 与同一次 `tool_use_id` 能可靠关联，包括并发子 Agent。`SubagentStart.agent_id` 与 `PostToolUse` 之间不能只凭相同 `session_id` 或猜测的 `turn_id` 建关系。OpenAI Docs 已明确前者在子 Agent Hook 中是父会话 ID；工具 Hook 覆盖也不是所有路径的绝对保证。
3. 工具开始前的 epoch 能传到该工具的完成事件；clear 后迟到事件被拒绝，resume 保留原 epoch，无法证明这些关系时不启用自动记录。
4. 没有等价生命周期事件的其他 host 可以只使用手动/显式 CLI 合同，或不接入候选簿；不以模拟 Codex 事件换取表面一致。

若第 2 或第 3 项不能成立，**通用 CLI 合同仍可评审，但 Codex 自动记录暂停**。不要改为从 transcript、全局环境变量或成功 `get` 的 `session_id` 猜测主/子 Agent。

## 取舍与验证

推荐独立 CLI 状态，而非 Codex Plugin 私有数据库：前者让多个 host 共享同一语义和裁剪规则；后者会把通用候选定义锁进一个宿主。也不放进 LocalStore：它的生命周期按 Pack/Store 而非对话运行，混放会把短命、可丢弃的协作状态变成 canonical 内容的一部分。独立 workspace package、通用 Agent memory、事件流或服务化在第二个真实 host/明确规模需求出现前都缺少收益证据。SQLite 增加一份本机状态与迁移责任，但相比多进程 JSON 文件锁和崩溃恢复，事务边界更容易证明；这份代价需要在实施评审中明确接受。

后续验证应包括：

- 合同测试：多来源完整保留、同 ID 同/异 digest、不同 host/root/Store 隔离；列表无正文、无“adopted/effective”字段，后续 `get` 的缺失和 digest 变化正确视为过期。
- 持久化测试：多个 CLI 进程同时记录、重复事件、clear 与迟到事件竞争、进程在事务中退出、损坏文件、权限错误、过期及总量裁剪；错误时 `get` 仍正常。
- Adapter 证据测试：以上真实脱敏 Hook fixture 覆盖主/子 Agent、并发、resume/clear、失败/取消及不可解析包装；每个不能归属的案例断言**零写入**。
- 性能与隐私：按目标候选数量量测单次 record/list 的 p50/p95、进程启动开销和物理磁盘峰值；检查数据库、JSON 输出及错误日志不含 Practice body、原始 payload 或绝对 Store 路径。参数在测量后调整，不以本次未运行的测试作验收证据。

本次没有真实 Hook payload、第二个 host 的需求样本或候选规模测量；Lorelum Skill 的语义查询还遇到本机 `backend.state-invalid`，原始 trace 无记录，故没有取得可引用的 Pack Practice 指引，也未为此改动运行状态。**下一步是评审上述语义与边界，并单独授权脱敏 Hook 取证；不是直接实施 adapter。** 当前会话也没有切换主模型/推理档位的接口，不能声称已按仓库要求切换至 `gpt-6-sol` `xhigh`。