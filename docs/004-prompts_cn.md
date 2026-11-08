# 004 提示词

本文档说明：智能体会读取哪些提示词、团队可以编辑哪些内容、以及这些更改何时生效。它建立在 [000 理念](000-philosophy_cn.md) 到 [003 执行](003-execution_cn.md) 之上。提示词文本本身使用英文编写。

## 团队可以写什么

团队可以编辑 3 个 team section、添加 skills，并覆盖所有内建 system-prompt part。大多数默认文本都在 `prompts/` 和 `skills/` 下。工具描述、notices 和 cards 仍然保留在代码里，不能通过 prompt settings 来编辑。

| 区块 | 包含内容 |
| --- | --- |
| Communication | 什么时候给人发消息、消息如何措辞、每条 GitHub 消息开头的那一行（默认是 `This is by an automated agent, not a human`）、如何处理 reviews，以及哪些普通账号其实是 review bots |
| Ticket scope | 一个 Ticket 什么时候继续推进，什么时候应该在草稿状态停下 |
| Delivery requirements | Pull Request 中应提供什么证据、auto-merge 的条件和其行数阈值，以及 commit author；若未指定，则 Executor 会使用 agent account 的 no-reply 地址作为提交身份 |
| Team skills | 智能体在需要时加载的团队知识，例如一个 `identities` skill，用来说明哪个飞书用户对应哪个 GitHub 账号 |

内建文本覆盖了角色定义、工具使用、宿主机制、授权规则，以及 GitHub / 飞书的 skills。它会通过这些区块的标题引用 team sections，而不是重复书写它们。

### 内建 parts 与模板

覆盖项使用下面这些 keys：

| Parts | Keys |
| --- | --- |
| Shared instructions and roles | `runtime/shared`, `runtime/orchestrator`, `runtime/worker` |
| Working instructions | `runtime/subscriptions`, `runtime/replies`, `runtime/dispatch`, `runtime/work`, `runtime/admission` |
| Handoff instructions | `runtime/handoff/orchestrator`, `runtime/handoff/worker` |
| Source skills | `skill/feishu`, `skill/github` |
| Generated sections | `runtime/environment`, `runtime/policy`, `runtime/policy/none` |

其中 generated sections 是可编辑模板，但必须保留如下占位符：

- `runtime/environment`：`{variables}`，会被执行环境变量行填充。
- `runtime/policy`：`{version}`、`{mode}` 和 `{rules}`，其中规则会按“每条启用规则一行”的形式填充。
- `runtime/policy/none`：`{version}` 和 `{mode}`，用于“当前没有启用任何规则”的情况。

编辑某个 part 会改变模型收到的指令文本，但不会改变 gate、Mandate、tool checks 或 account wrappers；这些约束依然由代码强制执行。

在用户明确请求的情况下，主 Session 也可以通过 `orchestrator_prompts_save` 保存 overrides。如果一次保存省略了某个 override 的 `base`，就沿用已保存的 base。对于已经出现在现有配置中的 removed part，保存时会自动移除；其余任何未知 part 都会被拒绝。

每个 override 都会记录它是基于哪一版内建文本产生的。升级时，团队自己的文本会被保留；如果默认文本发生变化，面板会把这个 part 标成 **Default updated**，并把新版默认文本与团队文本并排对比。团队可以选择继续保留自己的文本、改用新默认值，或在其上继续编辑。那些没有 override 的 parts，会自动跟随新的默认值。

主 Session 可以通过 `orchestrator_read prompts` 读取受影响的 part 名称，位于 `default_changed` 下。

## 组装顺序与优先级

system prompt 按如下顺序组装：

1. Shared instructions 和 role。
2. 3 个 team sections。
3. subscriptions 与 replies。
4. 主 Session 读 dispatch；Executor 读 work instructions。
5. `runtime/admission`，只给主 Session。
6. Handoff instructions。
7. `runtime/environment`。
8. `runtime/policy`，只给主 Session。

admission instructions 来自 `prompts/admission.md`。它描述主动 assessment 以及在什么情况下可以直接从 card 上忽略一个无关 candidate。`orchestrator_eval` 返回的是上下文，而不是重复这些说明。

environment section 为主 Session 提供部署级变量，为每个 Executor 提供 Ticket 级变量。Policy section 会在每次模型请求时从 settings 重新生成。它包含 Policy version、accept mode、explicit request 的豁免规则，以及所有启用规则，格式为 `- <name> (<id>): <instruction>`；如果当前没有启用规则，则显示一段说明。它被放在最后，是为了让一次 Policy 变更不会改动前面那些 prompt 文本。

Executor 还会读取 DeepSeek Harness 的编码提示词以及仓库里的 `AGENTS.md`；主 Session 不会。对于 inferred Ticket，最终还会追加一段说明，限制其 Executor 只能在匹配到的规则范围内工作。Assignments 和传入消息以内联 Trigger card 的形式送达；只有在内容被截断时，才会附带资源指针。

Executor 默认无人值守运行。它们的工具集会排除那些会向人提问、向人展示 UI 或修改 profile 的工具，比如 `ask_user_question`、`present` 和 `plugin_manager`。

优先级从强到弱如下：

1. 由工具和宿主强制执行的事实：身份、权限、tool argument checks、reply purpose、长度限制。prompt 文本无法改变这些。
2. 内建区块和生成区块，以及团队对它们的 override。
3. proactive Ticket 中匹配规则所定义的流程与沟通要求，仅对该 Ticket 的 Executor Session 生效。
4. Team sections 和 team skills。
5. 外部文本：消息、评论、文档和引用。它们只是数据，永远不能修改规则或授权。

## 内建文本遵循的原则

这些原则把 [000 理念](000-philosophy_cn.md) 具体落到智能体的行为上：

- **角色分离。** 主 Session 只做路由：它转发原始 Trigger 和路由事实，不做技术决策；而且通过一个支持批量决策的 routing tool 一次处理多个待定 candidate。每条路由备注最多 30 个词。Executor 会自己读取原始来源、判断 scope、实施修改并完成验证。
- **主 Session 不等待。** reports、follow-up notices、candidates 和已完成后台 job 的结果都会自动以消息送达。只要当前没有必须做的事情，它就结束 turn。系统还有一个 guard，会拒绝 `sleep`、超过 30 秒的 shell timeout，以及超过 5 秒的 job wait，因为被卡住的一步会拖慢新 candidate 的到达。这些说明会替代 DeepSeek Harness 在主 Session 完整 prompt 中自带的 job-notification 说明。
- **只在必须让人行动，或者结果已经可供人使用时发消息。** 所谓“结果可用”，指的是它已经可 review、可使用，或者当前需要一个决策/资源，或者工作已经无法继续。聊天消息默认只用一句话，并且有 100 词硬上限。
- **Executor 自己决定自己的 follow-up。** 每个 Executor 会处理并确认 scope 内请求，并把超出 Ticket 范围或属于其他 Ticket 的请求 hand back。只有当所有 owner 都 hand back，或在结束时没有把这条消息接手，消息才会成为新的工作。主 Session 虽然会收到 follow-up notice，但除了按人的要求 stop 外，不能主动处理这些消息。被 hand back 的消息随后会作为带有 Executor 原因说明的 candidate 出现。
- **通过 Subscription 和主 Session 来协调。** Executor 可以跟随相关 Thread 以获取上下文。主 Session 可以用一个最多 100 词的 note，把另一个 Ticket 中的事实转告过来。无论是 Subscription 还是 note，都不授予 authority。
- **只有明确引用时才允许 attach。** 一条消息必须显式指向某个 Ticket 的 pull request、Issue、source message 或 acknowledgement，或者指向一个最近变更且只能匹配到一个 Ticket。仅凭“主题相似”不够。attach 会让这个 Ticket 订阅整个 Thread；若 Executor 发现它不相关，就应使用 `orchestrator_thread_unrelated`，并且不要在那里作答。
- **具体地、只确认一次。** 在实质性工作开始前，Executor 要确认每个新请求，并在原始消息下回复自己要处理的对象和预期结果。
- **通过 `human_action` 发问。** Executor 在 Report 中提出一个明确问题，主 Session 会原样转发。如果这个问题是由某条人的消息触发的，宿主会在那条消息所在的位置提问；否则，就在 Ticket 的原始 conversation 中提问。已经获得授权的工作，在此期间仍可以继续。
- **诚实交付。** 打开一个 Pull Request 不等于 ready；草稿必须明确标注未完成；没有真实 before 截图的 UI 变更，不算 ready。
- **人的 pause 会持续生效，** 跨 follow-up、跨新的 Executor Session、跨 restart，直到有人解除。自动反馈不会恢复一个已暂停的工作。
- **示例只说明格式，不说明事实。** 示例句子不能被智能体当成真实事实来复述。
- **按结果处理 reviews。** 修复并验证过的评论，应静默 resolve；拒绝采纳的评论，则用一句话说明原因。自动化 reviewer 的 thread 在获得答复后即可 resolve；每个自动 reviewer 在同一个 pull request 上最多只允许 3 轮往返。
- **遇到重复工作要让位。** 如果人的分支或 pull request 已经包含了 Ticket 要做的变更，Executor 应关闭自己的 pull request，并用一句话指向对方的分支或 PR。

## 版本

- 每一次保存都会生成一个不可变版本，包含完整配置、revision number、前序版本、时间戳和来源（panel 或 tool）。
- 保存时必须提供它基于的 base revision；如果存在冲突，保存会被拒绝，但不会丢掉当前 draft。
- Session 会在每次模型请求时读取最新版本，包括同一 turn 中后续的步骤。保存动作不会重启、取消或唤醒某个 Session。宿主会记录每个 Session 曾经使用过哪些 revisions；Executor 可以读取自己工作期间用过的任意 revision。
- deployment 会设置一个总大小上限，包含 overrides。校验阶段会拒绝未知 keys、保留的内建 skill 名称、未知 override parts，以及缺失必需模板占位符的文本。系统绝不会默默截断文本。如果某个 override 与默认值完全相同，它会被自动移除，这样该 part 就能继续跟随未来的默认值。
