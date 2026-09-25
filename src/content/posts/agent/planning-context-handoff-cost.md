---
title: Token 用得更多，账单反而更低：我拿 Lorelum 的 Issue 测了 12 次
published: 2026-09-25
description: "让低成本主 Agent 先找资料，再交给 Sol 做规划。围绕 Lorelum #214 跑了两轮三对三：完整链路里，研究优先组总 token 多约 18%，模型与搜索费用却低 38.2%。方案质量、翻车和完整 prompt 也一起公开。"
image: https://d988089.webp.li/2026/09/25/20260925065029387.avif
tags: ["Agent", "Context Engineering", "Codex", "成本优化", "Lorelum"]
category: Agent
draft: false
---

我挺喜欢用 Astra 处理难题，它确实强。但如果日常开发也一直让它当主 Agent，账单真的顶不住。网上常见的省钱办法是强模型负责规划、便宜模型负责执行；这当然比全程用强模型便宜，我也不觉得它错。只是对我来说还不够激进，也不够优雅。我想把分工再反过来试一次：让便宜模型当主 Agent，先把 Issue、代码、文档和外部资料找齐，强模型只在真正需要工程判断时接手。

:::tip[先看结果]

这次拿我自己的项目 [Lorelum](https://lorelum.com/) 做了十二次方案设计，规划端固定为 Sol xhigh；Astra 是我想解决的日常账单问题，不是这轮的受测模型。最接近实际开发的那组六次，研究优先的分工比简单委派低了 **38.2%** 的模型加搜索费用，方案均分 **89.3 对 87.3**。更有意思的是：省钱的那组总 token **反而多了约 18%**。另一轮只测 Sol 收到交接后的规划成本，降了 **29.1%**；那轮没把准备资料的成本算进去，所以两个百分比不能混用。这是一个 Issue、每组仅三次的结果，不是什么万能省钱公式，但足够让我想继续往下挖。

:::

## 这和 Lorelum 有什么关系

我做 Lorelum，本来就是因为 Coding Agent 很会写代码，却经常在工程判断上翻车：为了过测试改测试，修个小问题顺手扩大 scope，没看清项目已有约束就另起一套抽象。团队对这些坑有经验，可经验散在 PR 评论、聊天记录和人的脑子里，Agent 做决定时未必能找到。

Lorelum 把经验写成有适用条件的 Practice，再组织成可版本化、可分享的 Knowledge Pack。Agent 可以用 `lore query` 找到相关线索，用 `lore get` 读完整内容，然后自己结合当前项目判断。它现在支持本地语义检索、CLI / Skill 工作流和 Codex Plugin；Claude Code、Cursor 等也能通过 Skill 接入。项目还在 public alpha。

其中的 **Agentic Coding Pack** 盯的就是大家用 AI Coding、Vibe Coding 时反复遇到的问题。我希望它能跨模型、跨 harness 帮 Agent 少犯工程判断错误；这次实验，也是为了把「谁去查资料、怎样交接、强模型什么时候该停手」写成更靠谱的 Practice。光喊一句「便宜模型先查资料」没用：如果它漏掉关键 spec，强模型会回来重查；如果它把猜测当事实，后面可能设计得越认真、错得越远。下文第一次试跑就发生了后面这种事。实验还没有直接测 Lorelum 注入 Pack 的效果，后面的价差也不能写成「装上 Lorelum 就能省这么多」。

## 贵模型的钱花在了哪里

让强模型从零设计一个 Issue，它得先找 Issue 原文、spec、代码入口、测试，再去核对外部资料。它读完一份文件，工具把结果送回来，模型还要带着已经读过的上下文再请求一次。旧内容即使命中缓存，输入也不是免费的；刚读回来的工具结果则作为新输入计价。读十次文件，如果每次都等上一次结果再决定读什么，就可能多出许多次带着长前缀的模型往返。反过来，把**互不依赖**的文件集中在一次回复里并行读取，下一轮统一处理结果，文件内容还是要读，但之前的上下文不必为每一份文件分别再计一轮缓存输入。

:::note[缓存命中率高，不等于这部分免费]

这也解释了为什么缓存命中率已经很高，我还是想减少强模型的请求次数。拿这次实验所用的 [Sol 官方价格](https://developers.openai.com/api/docs/pricing) 举例，缓存输入是普通输入的 **1/10**。假设某段对话的缓存命中率是 **95% / 98% / 99%**，缓存部分仍分别占这一段**输入费用的约 66% / 83% / 91%**（不含输出和工具费）。少一次没必要的往返，省掉的那段长前缀不是零。比例也在变化：[DeepSeek V4.1 Flash](https://api-docs.deepseek.com/quick_start/pricing/) 的当前官方价格是 **1/50**，[MiMo V2.6 Pro](https://mimo.mi.com/models/zh-CN/mimo-v2.6-pro) 约为 **1/120**；便宜得多，但不能因此把 1/10 定价的强模型重复上下文当成免费。

:::

我 1 月那篇 [Context Caching](/posts/agent/codex-context-caching/) 讨论的是已有上下文怎样复用缓存；这次想试的是更靠前的一步：先让便宜模型收集、核对和裁剪证据，再一次性交给负责规划的 Sol。批量读取只是其中一种可能的省法，下面的十二次运行**没有单独隔离它的收益**。真正决定方案能不能少返工的，还是交接资料是否找对、有没有来源、哪些地方尚未证实。要把这件事做稳，我还想把检索范围、来源核对、未知项标记、交接后的停止条件和方案验收分别写成可复用的 Practice，按任务需要串起来。这只是整套省 token 工作流里很小的一环；一句「少调用工具」的 prompt 撑不起它。

## 实验选了什么任务

[Lorelum Issue #214](https://github.com/lorelum/lorelum/issues/214) 是一个设计题：怎样给一次对话维护 Practice 候选簿，让后续启动的子 Agent 有机会看到主 Agent 读过的相关内容。候选簿记录的是「成功读过、可能还相关」，不能写成「已采纳」，更不能写成「有效」。

难点都藏在边界里：一次 `lore get` 是主 Agent 发起的，还是子 Agent 发起的？子 Agent 能看到父级的什么？会话 clear 后，一个迟到的结果还要不要写入？同一个 Practice ID 更新了内容，digest 怎么处理？回答这些问题，得回去看 `get` 的返回、Hook 已经支持什么、LocalStore 的责任和现有 spec。#214 定合同；#215/#216 才轮到实际记录与子 Agent 提示；#217 的 compact 恢复还在后面。实验只要求产出方案，不写实现。

:::caution[第一次试跑：交接写错，方案跟着跑偏]

我在最初试跑时就犯了一个很蠢的错误：把 #214 当成「实现 Codex `PostToolUse` 捕获」，还把后续几个 Issue 的关系说反了。Agent 收到这份交接，认真地设计起 Codex 适配器和持久化，篇幅很长，方向全歪。不是模型凭空幻觉，是我把错误线索递到了它手里。

回头核对 Issue 后，我废掉了那次试跑。之前还有一批协作 subagent 中断、续跑，用量只能听它们自己报，没法拿来比。这两批都不是后面表格里的十二次；它们花的钱也没有偷偷算作某一组的费用。正式对照改用 Codex CLI 跑，至少能拿到每次的 usage 和完整输出。这个前史很重要：一次性交接要是交错了，少几轮工具调用根本不值一提。

:::

## 第一轮：只看 Sol 做规划

先不让便宜模型进来，只回答一个小问题：如果 Sol 开始规划前已经拿到相关线索，它自己的调用会省钱吗？

我固定同一个仓库 commit `88e691ba939943e67793e92e1899387971dbce36`、同一个 Issue、Codex CLI `0.155.1`，用 `gpt-6-sol / xhigh` 跑了六次。B1/B2/B3 是普通组，C1/C2/C3 多拿到我整理的交接。各组三次用同一份 stdin prompt，B、C 交错执行。全部只读，不改仓库，也不允许它们继续委派 Agent；产物是一份 Markdown 设计草案。

```sh
codex exec --ephemeral --disable multi_agent --disable multi_agent_v2 \
  --sandbox read-only -C <同一仓库> -m gpt-6-sol \
  -c 'model_reasoning_effort="xhigh"' --json -o <本次独立方案文件> -
```

两组都知道要遵守 `AGENTS.md` 和 Skill，不能读取其他试次的材料。不同之处在于 C 组多拿到了 #214 的范围、#215/#216 的分工、该去看的代码和测试入口，还有尚未验证的 Host 行为。我让它核对原文，不能照单全收。普通组的显式 prompt 只有 436 个字符，交接组 2,265 个字符；我要测的本来就不是「prompt 越短越省」，而是多花这点输入，能不能换来后面少绕路。两份完整 prompt 在文末。

六份 CLI 记录都有 `turn.completed`，也没有子 Agent。注意这轮只计 **Sol 从接到任务到交方案的费用**；我整理交接花了多少，表里没有。

### 账单比总 token 更值得看

下面把六次逐次数据列出来，再看两组总账。未缓存输入、缓存输入、输出分别计价；输出数字里已经包含推理 token。命令行读文件本身没有单独的工具费，读到的内容进入后续上下文照样花钱。Web Search 按每次 USD 0.01 计。

| 运行         |  未缓存输入 |      缓存输入 |       输出 |        命令 / 搜索 |   费用 USD |        方案分 |
| ------------ | ----------: | ------------: | ---------: | -----------------: | ---------: | ------------: |
| B1 普通      |     125,477 |       794,496 |     17,497 |             29 / 3 |     0.6148 |            92 |
| C1 交接      |     151,583 |     1,023,488 |     14,709 |            26 / 13 |     0.7850 |            85 |
| B2 普通      |     155,824 |       961,664 |     18,217 |             50 / 9 |     0.7762 |            87 |
| C2 交接      |      74,709 |       488,064 |     12,509 |             27 / 2 |     0.3921 |            89 |
| B3 普通      |     202,038 |     1,240,192 |     17,943 |             43 / 9 |     0.9215 |            85 |
| C3 交接      |      93,750 |       548,864 |     12,421 |             44 / 4 |     0.4615 |            91 |
| **B 合计**   | **483,339** | **2,996,352** | **53,657** |       **122 / 21** | **2.3125** | **均分 88.0** |
| **C 合计**   | **320,042** | **2,060,416** | **39,639** |        **97 / 19** | **1.6386** | **均分 88.3** |
| **C 相比 B** |  **-33.8%** |    **-31.2%** | **-26.1%** | **-20.5% / -9.5%** | **-29.1%** |   **+0.3 分** |

我用的是 [OpenAI 官方 GPT-6 Sol Standard 短上下文单价](https://developers.openai.com/api/docs/pricing)：每百万未缓存输入 USD 2、缓存输入 USD 0.20、输出 USD 10。原始 `input_tokens` **包含** `cached_input_tokens`，所以先减出未缓存部分；`output_tokens` **包含** `reasoning_output_tokens`，不要再加一次。六次的 cache-write 量都为 0。具体这么算：

```text
模型费用 USD = (未缓存输入 × 2 + 缓存输入 × 0.20 + 输出 × 10) / 1,000,000
搜索费用 USD = web_search 事件数 × 0.01
```

[官方工具价格](https://developers.openai.com/api/docs/pricing) 是每千次 Web Search USD 10，搜索内容进入上下文的 token 另按模型价格计。按记录中的 `web_search` 事件计次。 [CSV 原始拆分](/experiments/issue-214/usage-and-price.csv) 也放在这里，想自己核算可以直接下载。

| 三次合计            |         普通组 |         交接组 |       变化 |
| ------------------- | -------------: | -------------: | ---------: |
| 未缓存输入          |        483,339 |        320,042 |     -33.8% |
| 缓存输入            |      2,996,352 |      2,060,416 |     -31.2% |
| 输出（含推理）      |         53,657 |         39,639 |     -26.1% |
| 模型 token 成本     |     USD 2.1025 |     USD 1.4486 |     -31.1% |
| Web Search 调用     |             21 |             19 |      -2 次 |
| **模型 + 搜索成本** | **USD 2.3125** | **USD 1.6386** | **-29.1%** |

合起来，普通组模型费 USD 2.1025，交接组 USD 1.4486；把搜索加上，分别是 **USD 2.3125 和 USD 1.6386**，交接组低 **29.1%**。缓存输入也从约 300 万降到 206 万，少 **31.2%**；这和我想减少重复往返的方向一致，但不能把降幅全部归给「一次批量读文件」，两组收到的上下文线索也不同。总 token 少了 31.5%，但只报总 token 会漏掉最要紧的价格差：同样一个 token，未缓存输入、缓存输入和输出的单价完全不同。命令加搜索的次数从 143 降到 116，也不能直接折算成钱，因为每次读多少、之后又把多少历史带进下一轮，比次数本身更重要。

尤其是 C1，完全没按我期待的方向走。它比 B1 **贵了约 USD 0.1701（27.7%）**：命令少跑三次，网页却搜了 13 次，B1 只搜了 3 次。C1 的未缓存输入、缓存输入也都更高。C2/C3 后来各省了将近一半，才把组均值拉下来。给 Agent 找对入口，不等于它会知道什么时候停止调查。

即便暂时不看这次翻车，三对三的绝对差额也只有 **USD 0.6740**，平均每份方案 USD 0.2247。交接材料要是每次都由另一个昂贵 Agent 从零整理，花得比这还多，省下的这点 Sol 钱就没了。材料能跨任务复用，当然可以摊薄；这轮实验没有测复用。六次运行彼此重叠，文件时间也不能拿来推断哪组更快。

### 便宜的方案有没有变差？

便宜归便宜，方案要是写坏了也没意义。普通组均分 **88.0**，交接组 **88.3**；评审非盲、样本又小，我只把它当成「这一轮没有明显变差」的信号。具体怎么打分、每份方案有什么得失，感兴趣可以展开看。

:::details

<summary>展开第一轮六份方案的评分和短评</summary>

我按同一张五维量表评分：Issue 范围与阶段 20 分，现状和合同证据 20 分，接口及规则所有权 25 分，隔离/并发/生命周期/降级 20 分，验收与表达 15 分。评审者知道组别，而且是 AI 辅助的工程评审，不是独立盲评。

| 方案                                         | 分数 | 我最在意的得失                                                                          |
| -------------------------------------------- | ---: | --------------------------------------------------------------------------------------- |
| [B1 普通](/experiments/issue-214/b1-plan.md) |   92 | Store scope、多来源、clear 代际和降级较完整；把「不可用」与「没有候选」分清楚。         |
| [C1 交接](/experiments/issue-214/c1-plan.md) |   85 | 字段与生命周期完整，但出现损坏的 `([]())` 引用，generation/eventRef 等也较早定死。      |
| [B2 普通](/experiments/issue-214/b2-plan.md) |   87 | 多进程事务具体；较早引入 child 分支写入和事件去重表。                                   |
| [C2 交接](/experiments/issue-214/c2-plan.md) |   89 | 把 Observed/Required/Proposed/Deferred 分开；同 ID 双 digest 并存是需要评审的明确取舍。 |
| [B3 普通](/experiments/issue-214/b3-plan.md) |   85 | 清理与测试矩阵扎实，但公开 ledger 命令、根/子/兄弟共视超出首阶段边界。                  |
| [C3 交接](/experiments/issue-214/c3-plan.md) |   91 | 首版限定可信 root 写入、子 Agent 只读父线索；SQLite 路径和预算仍只是实验参数。          |

如果你想自己看，[B1](/experiments/issue-214/b1-plan.md) 和 [C3](/experiments/issue-214/c3-plan.md) 值得放在一起：两份都能进入评审，但对记录的责任、状态和验收写法并不相同。 [C1](/experiments/issue-214/c1-plan.md) 则有一个很扎眼的坏引用 `([]())`，也较早把一些设计选择定死。表里的六份原文都放了公开阅读版，只把本机绝对路径改成对应 commit 的源码链接；它们不是 Lorelum 已采纳的方案。

交接组拿到的也不是整仓代码，更不是一份现成答案。我写的是它容易找错的地方：#214 负责定义候选簿语义，#215/#216 才负责后续接入；一次成功读取可以留下 Practice ID、digest、标题和来源等元数据，不能推导出「已采纳」；CLI runtime、Host adapter 和 Pack/LocalStore 分别管什么。另外给出 `get`、Hook 和 spec 的入口，同时明说真实 Host payload、主子 Agent 归属和失败信号还缺证据。

它们照样得自己做设计选择。C2 想让同 ID 的两个 digest 在有限范围内并存，C3 想让首版只接受可信 root 写入，B1 更仔细地区分了「候选簿不可用」和「当前没有候选」。这些分歧不可能靠多塞几段 context 自动消失。交接能省的，是四处找入口、反复把无关材料带进下一轮的开销。

:::

## 第二轮：把找资料的 Luna 也算上

第一轮的漏洞太明显了：材料是我事先整理的，计算时却当它免费。于是我在同一个 Issue、同一个 commit 上又跑六次完整的 Luna→Sol 工作流。每次都让 `gpt-6-luna / max` 接收用户级任务指令，再起一个 `gpt-6-sol / xhigh` 子 Agent 做方案。

A1/A2/A3 的要求很简单：看 #214，用 Sol 规划。B1/B2/B3 则让 Luna 先查 Issue、仓库代码、测试和必要的官方资料，整理一份能核查的详细交接，再把方案交给 Sol。这回交接材料不是我替 B 写的。两组的 [完整用户 prompt：A 组](/experiments/issue-214/luna-workflow/simple-prompt.md)、[B 组](/experiments/issue-214/luna-workflow/research-first-prompt.md) 都能下载。

这里得承认 B prompt 还留着一句审计要求：最终把交给 Sol 的 prompt 和主要来源一起交回来。平时开发未必会加这句话，它也可能增加输出 token；当时的运行就是带着它完成的，我不能事后把附件里的原文改成更好看的版本。所以第二轮也不算完全无干预的自然开发。费用表已经包含它实际造成的消耗。

为什么主 Agent 用 Luna？主要因为这轮实验跑在 Codex 里。我本来更想试 DeepSeek V4.1 Flash，但用它委派 Sol 没跑出完整的对照数据，于是换成 Luna 把六次跑完。Codex 的 MultiAgent V2 委派使用 [加密内容协议](https://developers.openai.com/api/docs/guides/responses-multi-agent)；这批运行记录里的 `spawn_agent.message` 也只有密文，没法逐字导出 Sol 实际收到的委派文本。公开附件因此保留了完整的用户输入和方案产物，不会假装那六份子 Agent prompt 已经拿到了。

Luna 在这里就是一个低成本主模型的测试样本，不是我认为最好的选择。它这次的指令遵循也挺差，B1/B2 没交齐承诺的 spec/tasks；A1/A3 又各向 Sol 追问了两次，把强模型的额外往返加回来了。我想测的是「性价比模型先找资料，前沿强模型做规划」这套分工，没打算为了这篇文章把 Luna 的编排 prompt 调到完美。换个 harness，主模型和规划模型都可以换；也可以用别的多 Agent 组合，比如让 Claude Code 编排任务、Codex 承担其中的规划子任务。后面的数字只属于这次 Codex + Luna + Sol 的组合，别拿它当模型搭配的上限。

### Luna 多干活，Sol 少花钱

按 [官方 Standard 短上下文价格](https://developers.openai.com/api/docs/pricing)，Luna 每百万未缓存输入、缓存输入、输出分别是 USD 0.10 / 0.01 / 0.50；Sol 是 USD 2 / 0.20 / 10。Web Search 按 USD 0.01 一次。这里对 Luna 根线程、Sol 子线程分别取 usage，再加两边的搜索；没有把子线程用量误算到根线程里。输入包含 cached、输出包含 reasoning 的处理和第一轮一样。单位 USD：

| 运行         | Luna 模型费 | Sol 模型费 | 两边 Web 次数/费用 | 全链路合计 |
| ------------ | ----------: | ---------: | -----------------: | ---------: |
| A1           |      0.1030 |     1.0853 |          19 / 0.19 | **1.3783** |
| A2           |      0.0671 |     0.8793 |          35 / 0.35 | **1.2963** |
| A3           |      0.0776 |     1.0782 |          37 / 0.37 | **1.5258** |
| **A 合计**   |  **0.2476** | **3.0428** |      **91 / 0.91** | **4.2004** |
| B1           |      0.1060 |     0.6153 |          19 / 0.19 | **0.9112** |
| B2           |      0.1055 |     0.5518 |          37 / 0.37 | **1.0273** |
| B3           |      0.1228 |     0.2360 |          30 / 0.30 | **0.6587** |
| **B 合计**   |  **0.3342** | **1.4030** |      **86 / 0.86** | **2.5972** |
| **B 相比 A** |  **+35.0%** | **-53.9%** |  **-5.5% / -5.5%** | **-38.2%** |

三次合计，A 花 **USD 4.2004**，B 花 **USD 2.5972**，B 低 **38.2%**。有意思的不是 B 少用了多少 token：它的总 input+output **反而多约 18%**。B 让 Luna 多干活，Luna 模型费比 A 多约 USD 0.087；Sol 模型费却少了约 USD 1.640，两边 Web Search 还少 USD 0.05，最后净省约 USD 1.603。这就是为什么我更关心价格，而不是总 token 的百分比。 [逐线程 CSV](/experiments/issue-214/luna-workflow/usage-and-price.csv) 把各档 token、搜索和单次费用都列出来了。

再看 Agent 到底怎么做的。A 组自己选了 `fork_turns=all`，A1/A3 还各追问 Sol 两次；B 组自己选了 `none`，没有 follow-up。B 的 Luna 根线程跑了 **135 次本地命令**，A 只有 **78 次**，但 Sol 的未缓存输入从 A 的约 **87.3 万**降到 B 的约 **26.0 万**。这些都是运行时真实发生的选择，我没有在 prompt 里规定 `fork_turns` 的取值。B 三次都更便宜，但不能把这结果解释为「只改了交接资料、别的全固定」——委派方式和后续追问也跟着变了。

### 方案质量和运行时间

方案沿用第一轮的量表，A 均分 **87.3**，B **89.3**。B1/B2 的研究做得认真，最后却没交齐自己承诺的 spec/tasks。这两分的均分差距不值得拿来宣传「质量更好」，交付漏项也不能因为省钱就不算。六份方案的分数和原文放在下面，想细看再展开。

:::details

<summary>展开第二轮六份方案的评分与原文</summary>

同一个评审者非盲评分：A1/A2/A3 分别是 **91 / 85 / 86**，B1/B2/B3 是 **86 / 88 / 94**。A1 把 Store scope、clear 后迟到写和持久化规范冲突讲得很清楚；B3 查到已发布的官方 tag 上有可选的 `PostToolUse.agent_id`，但没假装它就等于实际收到的 Hook payload。

- A 组：[A1](/experiments/issue-214/luna-workflow/a1-plan.md)、[A2](/experiments/issue-214/luna-workflow/a2-plan.md)、[A3](/experiments/issue-214/luna-workflow/a3-plan.md)
- B 组：[B1](/experiments/issue-214/luna-workflow/b1-plan.md)、[B2](/experiments/issue-214/luna-workflow/b2-plan.md)、[B3](/experiments/issue-214/luna-workflow/b3-plan.md)

附件只替换了本机源码路径，方案内容没有替它们修改；这些是实验草案，不是 Lorelum 已接受的合同。

:::

时间这一栏我没拿来做结论。A3/B1/B2 分别跑了约 115 / 127 / 136 分钟，中间有 TLS 和 WebSocket 重连；那天我的网络不好，没法从墙钟时间分出模型规划、搜索和断线重试各占多少。

## 这次我会怎么用这个结果

在这个小仓库、同一个 Issue 的十二次设计里，「先找线索，再交给 Sol」确实比直接让 Sol 从头查更便宜，而且方案评分没有明显输掉。但第一轮 C1 更贵、第二轮 Luna 漏交材料，已经足够说明它不是一个无条件有效的开关。**29.1%** 算的是 Sol 单独规划，**38.2%** 算的是 Luna 加 Sol 的整条链路；这两个数字都不是 Lorelum 带来的产品收益。

我很想知道把它放进更大的仓库会怎样：搜索范围更大、反复读上下文的成本可能更高，交接的收益或许更明显；但交接遗漏一条关键边界，返工也可能更贵。这个猜想现在还没数据，不能替大仓库宣布胜利。至于让低价模型承担几乎全部 Web/context、把 Sol 的额外检索压得更低，是下一轮想测的更激进做法，**这次没有测**。

下一步是换几个 Issue 跑，看看这次的价差能不能复现，也想继续打磨 Agentic Coding Pack 里关于资料收集、交接和停止条件的 Practice。原始数据和方案都放在附件里；你可以换一套评审标准，或者只挑两个 Agent 的方案对着看。这十二次还不足以给出一个放之四海而皆准的百分比。

最后补一下近况，也解释为什么博客停更这么久：2 月到 6 月我在一家中厂实习，做 AI Native 全栈；3、4 月还在写 [《AI 原生工程》](https://ai.mengchen.icu/)；7 月开始做 Lorelum，7 月底去了九坤 Converge AI 做 Agent Infra。书只写完前两章，拿一个项目串起需求、技术栈、仓库规范和 spec-first，后面还是大纲。没继续写，是因为当时想展开的不少具体操作，很快被新模型和 Agent 工具内化成了能力；前两章关于怎么判断、怎么给项目立边界的内容，我仍然觉得值得读。 [书的仓库](https://github.com/TatsukiMeng/ai-native-engineering) 欢迎 Star。

至于 [Lorelum](https://lorelum.com/)，这次实验只是我打磨它的一步。Agentic Coding Pack 能不能真的让不同模型、不同 harness 的 Coding Agent 少犯错，还得靠更多人拿真实任务来试。如果你在用 Coding Agent，[装一下](https://lorelum.com/en/docs/)，告诉我哪条 Practice 帮了你、哪条只是在添乱；想贡献代码、测试或 Pack，[GitHub](https://github.com/lorelum/lorelum) 有 `good first issue` 和 `help wanted`。觉得这个方向值得做，也请给 Lorelum 一个 Star。

## 附：第一轮六次运行的显式 Prompt

下面分别是传给 B1/B2/B3 与 C1/C2/C3 的 **stdin 原文**，同组没有追加 follow-up。附件也有 [普通组原文件](/experiments/issue-214/baseline-prompt.md) 和 [交接组原文件](/experiments/issue-214/context-prompt.md)。CLI 的系统/开发者指令，以及适用的 `AGENTS.md`、Skill 不在 stdin 里；两组共享这些环境，所以这不是「模型看过的所有文本」。第二轮的两份用户 prompt 在前文附件，实际的 `spawn_agent.message` 因密文无法逐字公开。

:::details

<summary>展开普通组 B1/B2/B3 的 stdin 原文</summary>

```text
你是一次独立的技术规划运行。针对 lorelum/lorelum Issue #214（https://github.com/lorelum/lorelum/issues/214），在当前仓库产出可评审的技术方案。遵守当前适用的 AGENTS.md 和 Skill；只读调查，不修改任何文件，不实施、不提交、不推送。最终回答直接输出完整 Markdown 方案，说明关键依据、缺失证据、取舍和验证方式。

本实验明确禁止委派：不得启动 subagent、调用 spawn_agent/collaboration，不得递归启动其他 Codex 或代理。你独立完成。不要阅读本仓库的 `docs/research/issue-214-planning-handoff-experiment.md`、任何本实验 prompt/结果文件或 `/tmp/lorelum-214-*plan*.md`；这些是另一次试次的材料，不是 Issue 依据。不要为了测量而改变正常规划方法。
```

:::

:::details

<summary>展开交接组 C1/C2/C3 的 stdin 原文</summary>

```text
你是一次独立的技术规划运行。针对 lorelum/lorelum Issue #214（https://github.com/lorelum/lorelum/issues/214），在当前仓库产出可评审的技术方案。遵守当前适用的 AGENTS.md 和 Skill；只读调查，不修改任何文件，不实施、不提交、不推送。最终回答直接输出完整 Markdown 方案，说明关键依据、缺失证据、取舍和验证方式。

本实验明确禁止委派：不得启动 subagent、调用 spawn_agent/collaboration，不得递归启动其他 Codex 或代理。你独立完成。不要阅读本仓库的 `docs/research/issue-214-planning-handoff-experiment.md`、任何本实验 prompt/结果文件或 `/tmp/lorelum-214-*plan*.md`；这些是另一次试次的材料，不是 Issue 依据。

以下是主 Agent 已经收集的决策线索，供你一次性定位证据；它不是事实权威。必须以 Issue 原文、当前合同和源码核实，有矛盾以这些来源为准，并明确指出错误线索。Issue #214 的当前阶段是定义跨 Host 通用的 ConversationPracticeLedger（名称可调整）语义、归属、宿主绑定与最小合同：一条记录只表示某 Agent 在任务会话中成功读取、可能仍相关的 Practice，不是 adopted/effective。至少覆盖 practiceId、contentDigest、title、可选 appliesWhen、来源 Pack 身份；按 hostKey + 不透明的 root conversation reference 隔离。去重/更新、并发、父子可见性、clear/resume、清理和有界保留由 CLI runtime 统一定义。Host adapter 负责可靠的会话、父子、生命周期信息和事件转换；Pack/LocalStore 继续拥有 canonical 内容和来源。不得放在 Codex Plugin 私有状态、LocalStore/Engine retrieval state 或全局“当前对话”文件。第二个真实 Host 出现前，不预建独立 workspace package。

本 Issue 的明确非目标：不实现具体 Codex Hook 或其他 Host adapter；不自动 query/get/采纳，不存完整 Practice body，不做 compact 恢复，不引入本地 MCP、通用 Agent memory/workflow engine。#215 才是可验证的主 Agent 成功 get 写入，子 Agent get 默认不回流；#216 是后续 SubagentStart 的有界父候选提示；#217 的 compact 消费延后。真实 Codex PostToolUse/SubagentStart payload 是后续接入的证据门槛，不能从官方字段表、相同 session_id、cwd、transcript 或 shell 命令文本猜测一次 get 的主/子 Agent 归属。失败、取消、复杂包装或不能确认归属时应 fail closed；宿主缺少对应事件时保持能力可选。

已定位的当前仓库证据：`packages/cli/src/hook/codex.ts` 仅支持 SessionStart 的 metadata-only Catalog；`packages/cli/src/main.ts` 有 Hook 原始 ABI 路由；`packages/cli/src/get/get-command.ts` 和 `result-schema.ts` 返回 canonical Practice、contentDigest、有序 sources；`docs/cli/get.md` 说明同一次 point read 一致但跨命令不 pin Store revision。`openspec/specs/agent-integration/spec.md` 规定 CLI-first、Hook 不自动 query/get、无本地 MCP；`openspec/specs/practice-read/spec.md` 定义 get 合同；`docs/plugins/codex.md` 是当前 Plugin 工作流。阅读 `packages/cli/AGENTS.md`、`docs/AGENTS.md`，按需核对相关测试和 Issue #215/#216/#217。真实 Hook payload 和清晰的主/子归属尚未在仓库中得到证明；不要声称已有自动捕获能力。当前未跟踪 `packages/cli/benchmarks/` 与任务无关，不碰。

请把 Observed、Required、Proposed、Deferred 分开；给出字段来源与多来源规则、责任和最小接口、会话隔离/父子/clear-resume/TTL/并发/降级、少量真实备选、可核验验收及缺失的 Host 证据。设计性变化依仓库规则应先走 OpenSpec，本次仅输出实验草案。为了测试交接效率，优先一次集中或并行读取以上关键材料，非必要不重复广搜和预检；若线索不足或冲突，仍须额外核实，正确性高于减少调用。
```

:::
