# 002 授权

授权决定了：谁可以发起工作，以及这项工作可以包含哪些内容。本文档建立在 [001 领域模型](001-domain_cn.md) 之上。

宿主系统会强制执行允许名单、规则和 Mandate。消息、评论以及被引用的文档，都只是决策的输入；它们本身不能改变这些权限。

## 一个 Trigger 的两条路径

### Eval：判断是否启动工作

Source 会记录一条消息是否“指向了智能体”：`mention` 表示提到了 bot 或 agent account，`direct` 表示私信。gate 会检查发送者和该 Source 的 Policy：

- 一个在允许名单中的人，只要明确指向智能体，就构成显式请求。
- 人在已有 Ticket 的 Thread 上的 follow-up，按下面的规则处理。
- 其他人的消息则使用该接收范围对应的 admission table。
- 出现在 Source bot 列表中的 bot，可以产生 inferred candidate，但永远不能形成 explicit request。其他 bot 一律不准入。

### Subscription：继续已有工作

已订阅 Thread 上的每一条消息，都会作为上下文送达订阅该 Thread 的 Ticket，包括 bot 和 CI 发出的消息。这个投递过程绕过 gate，也不会授予新的权限。每个 Ticket 仍然受到自己 Mandate 的限制。

人在 Ticket 自己的 Thread 上发出的消息，归这个 Ticket 的 Executor 判断。同样地，如果主 Session 把一条消息 attach 到某个 Ticket，那么这条消息也归这个 Ticket 处理。在 Executor 持有这条消息期间，它不会进入 Eval，也不会生成 Decision。每个 Executor 要么在自己 Ticket 的 scope 内处理这个请求，要么把它 hand back 回去，由主 Session 再做路由。

只要 Ticket 仍然存活，这种 ownership 就一直存在，即使人已经移除了它的 Subscription。对于一个没有被人 stop、但正有消息等待恢复它的 ended Ticket，这种 ownership 同样成立。

主 Session 会收到一条 follow-up notice。在 Executor 仍然拥有这条消息期间，主 Session 唯一能做的事情是响应人的要求去 stop 工作；accept、attach、ignore 和 requeue 都会被拒绝。stop 仍然要按下文所说的 follow-up 强度经过 gate。

### Hand back 与释放 follow-up

如果某条消息要求的工作超出了 Ticket 的 scope，或者其实属于另一个 Ticket，Executor 会把它 hand back 一次。Ticket 保留自己的 Subscription。如果一个 attach 进来的 Thread 实际上并不相关，Executor 会把自己在这个 Thread 上持有的那些消息一并 hand back。

只有在没有任何 Ticket 还拥有某条消息时，这条消息才会进入一次新的 Eval。新的 candidate 中会带上 Executors 给出的原因。gate 会把它当作这个 Thread 上的普通消息来判断，主 Session 之后可以接受它、attach 到别处，或者忽略它。若 hand back 的消息原本并不归该 Ticket 所有，比如 bot 消息，或者人已经把它送进 Eval 的消息，那么不会重新排队任何东西。

当 Ticket 结束时，Executor 尚未读取或接手的 follow-up 会被释放。这包括：从未送达 Executor 的消息，以及那些在它最后一次 Report 或 turn end 之后才送达、且它既没有 acknowledgement、也没有 answer、也没有 hand back 的消息。每条被释放的消息，只要不再被其他 Ticket 拥有，就会进入 Eval。stop 只结束被 stop 的那个 Ticket 的工作。

一个被 stop 的 Ticket，不会自动把“stop 这件事本身”重新送入 Eval，除非另一个 Ticket 把这条消息 hand back 了。主 Session 在 stop 它时，可以把消息里剩余的内容当作一项新工作来接。

人在面板上也可以把一条仍被 Executor 拥有的 follow-up 强制送入 Eval。如果主 Session 随后又把它 attach 回一个 live Ticket，Executor 会再次收到它。对于那些没有有效判断覆盖的消息，面板同样提供重新评估入口。

### 仅上下文订阅

Executor 可以为了上下文去跟随一个相关 Thread，比如另一个 Issue 或 PR。这类 Thread 上的消息也会送达 Ticket，但不会赋予 authority，也不会提供 follow-up strength。即使人类消息发在这种 Thread 上，也不会让一个 ended Ticket 恢复，除非主 Session 显式把其中某条消息 attach 到该 Ticket。主 Session 发送的 note 也同样只是上下文。

任何没有送达任何 Ticket 的人类消息，以及那些其 Ticket 已被人 stop 的 Thread 上的消息，都会进入 Eval。bot 的消息也会独立走这条路径，不受它是否被送达已订阅 Ticket 的影响。

## 强度

| 强度 | 情形 |
| --- | --- |
| explicit | 在允许名单中的人明确指向智能体；或者在已有 Ticket 的 Thread 上进行 follow-up；或者 admission table 中某个 cell 设置为 explicit request |
| inferred | 不在允许名单中的人，在已有 Ticket 的 Thread 上 follow-up，且其 admission cell 不是 explicit；或者 admission table 中某个 cell 设置为 inferred candidate；或者 Source bot 列表中的某个 bot 在指向智能体，或处于 Source 接收所有消息的范围内发出消息 |
| not admitted | 其他 bot 的消息、admission table 忽略掉的一切、以及 accept mode 为 `off` 时的一切 inferred 情况 |

对于一个不在允许名单中的人，如果其对应的 explicit admission cell 被开启，那么即使是在已有 Ticket 的 Thread 上，这条消息也会被视作 explicit，并且在任何 accept mode 下都有效。否则，他的 follow-up 就是 inferred，在 `off` 模式下不准入。仅为上下文而跟随的 Thread 上的消息，不适用 follow-up strength，而是走 admission table。

## Admission

每个 Source 的 Policy 都有一张 admission table。它的行表示接收范围：

- **飞书：** 每个已配置的群、bot 会接收 mention 的其他群，以及私信。只有在飞书设置卡里的 **其他群** 设为 **仅 @** 时，其他群才会被接收。
- **GitHub：** 新 Issue、新 Pull Request、评论，以及默认分支上的 CI。

每一行最多有 3 个 cell。每个 cell 决定某类消息会成为 explicit request、inferred candidate，还是直接忽略：

| Cell | 可选值 | 默认值 |
| --- | --- | --- |
| 在名单中，但没有指向智能体 | inferred, ignore | inferred |
| 不在名单中，但指向了智能体 | explicit, inferred, ignore | ignore |
| 不在名单中，也没有指向智能体 | inferred, ignore | ignore |

只有在 Source 会接收该范围内“所有消息”时，未指向智能体的那些 cell 才可用，比如飞书群设成 **全部消息**，或者 GitHub 某个 receive-all 开关打开。CI 结果没有 sender，因此这一行只提供最后一个 cell。bot 消息是否可准入，则由 bot 列表决定，而不是这些 cell。

把“未列入名单但指向智能体”的 cell 设为 explicit，就允许该范围内的任何人通过 mention 智能体来发起工作。这在专门的支持群里很有用。inferred candidate 是否真的进入评估，仍然取决于 accept mode。对于 follow-up，通常会绕过 admission table，但若某个 explicit cell 适用，它仍然会赋予 explicit strength。

### Bot 与自动化评审器

宿主会把 GitHub Apps、用户名以 `[bot]` 结尾的账号，以及飞书 bot，都识别为 bot。某个 Source 可以列出诸如 `dependabot[bot]` 这样的 bot，让它们的消息有资格成为 inferred candidate。它们必须要么明确指向智能体，要么出现在 Source 会接收全部消息的范围中。它们不能发起 explicit request，也不能 stop Ticket。

如果一个自动化评审器使用的是普通账号，那么从宿主的角度看它依然算人。它的评论会送达已订阅 Ticket，也会生成 follow-up notice。团队应在 prompt 文本中告诉智能体，哪些普通账号实际上是自动化评审器。

## 路由 candidate

主 Session 用 `orchestrator_route` 来处理 explicit candidate。一条调用里可以包含多个 accept、attach、stop 或 ignore 决定。每一项都会独立处理：某一项被拒绝或字段缺失，不会影响其他项执行。

inferred candidate 通常都需要一次完整 assessment。但有一个快捷路径：如果从它的 card 上就能明确看出它根本不涉及任何启用中的规则，比如只是闲聊或无关讨论，那么主 Session 可以直接忽略它。此时 Decision 会记录 `ignoredFromCard`，以区别于经过规则 assessment 后的 ignore。主 Session 会从自己的 Policy prompt section 中读取当前启用的规则。

对 inferred candidate 的 accept，始终要求经过 assessment。对于没有被 gate 认定为 explicit 的 Trigger，直接 accept 会被拒绝。

## Accept mode 与规则

accept mode 是 Policy 的一部分：

| 模式 | 效果 |
| --- | --- |
| `off` | 所有 inferred 内容都不会成为 candidate，例如未被点名的消息、bot 的消息或不在名单中的人的 follow-up |
| `observe`（默认） | 会对 inferred candidate 做 assessment；如果 assessment 结果为正，只会记录成“would accept”的 ignore，不会真的接单 |
| `auto` | 对 inferred candidate 的正向 assessment 会直接 accept |

explicit request 在任意模式下都有效。建议一开始使用 `observe`，先检查那些被记录下来的 Decisions 是否合理，再切换到 `auto`。每次 assessment 都必须对每条启用中的规则判断一次，无论它是否匹配。一次 proactive accept 至少需要匹配上一条规则。

每条规则描述一种主动工作的类型。Ticket 的 Mandate 会记录匹配到的规则，Executor 也会收到这些规则的名字和指令。

对于某个 Ticket，如果某条规则定义了自己的流程或沟通要求，那么这些要求会覆盖默认值。比如某条“保持沉默”的规则可以抑制 acknowledgement；某条“需要 review”的规则可以指定必须由谁 review 和 approve 才能 merge。而那些适用于所有 Ticket 的要求，则应写进团队 prompt section。

## 策略版本

规则、accept mode 和每个 Source 的 Policy 共同构成 Policy version。每个 Mandate 都会记录自己是在什么版本下被接单的。

版本变更会丢弃正在运行的 Eval，并把它们的 Triggers 重新排队。它还会给每个受影响的 live Ticket 发送一个 policy-changed Input，用来说明：

- 某条已匹配的规则发生了变化（包括新文本），或者已不再适用；
- 请求者已经不再被允许发起工作；
- 主动工作已不再被接受，例如模式从 `auto` 改成了别的。

未受影响的 Ticket 不会收到这个通知。变更本身不会立即撤销 live Ticket 的 Mandate。

Source 配置和 Policy 是分开的。Source 配置决定什么会进入系统：监听哪些群、某个范围是接收所有消息还是只接收 mention。Mandate 会记录自己的 Source 以及接收范围。一个 Ticket 恢复时，要求 Source 仍在监听原始 Thread，但不要求它仍然接收那个范围内的所有消息。

reply 在自己的 Source 关闭时会等待；如果 Source 已经不再监听其目标对话，它就会被丢弃。

关于“Executor 对自己的结果能做什么”的限制，比如何时允许 merge，并不决定谁有权授权工作。那属于智能体的执行指令，而不是 Policy 的一部分。

## 恢复与停止

一个 ended Ticket 会保留自己的 Mandate。只要在它自己的某个 Thread 上有人发消息，它就可以在不经过新的 Eval 和 Decision 的情况下，以同一项工作恢复，不论发消息的人是谁。bot 和 CI 不会唤醒 ended Ticket。Executor 自己决定这条请求是否仍在 scope 内；如果不在，就在那一轮里结束。

只有三种操作会改变 Mandate：accept 创建它、人工 stop 撤销它、恢复前重新校验失败会使它失效。一个被人 stop 的 Ticket 会永久保持 ended，不再接收任何消息。之后若要继续相关工作，必须重新 accept。

### 恢复前的重新校验

Source 必须仍在监听原始 Thread。其余检查取决于 Mandate 的 basis：

- 显式 follow-up 形成的 Mandate 会通过。
- inferred Mandate 需要当前是 `auto` 模式，并且它匹配的规则仍然启用。
- 由 admission table 某个 cell 准入的 Mandate，要求在当前 table 和当前 sender 身份下，至少还能获得与当时同等强度。也就是说，如果当时是靠“名单内成员”这个 cell 被接纳的，那么后来把这个人移出名单，可能会使 Mandate 失效。
- bot 的 Mandate 要求该 bot 仍在 Source 的 bot 列表中。由 inferred table 或 bot 产生的 Mandate，也同样需要 `auto` 模式以及匹配规则仍启用。
- 其他 Mandate 则要求 sender 仍在该 Source 的 allowed list 中。

如果重新校验失败，Ticket 会保持 ended，并且不会接收那些正在等待它的消息。每条消息都会被当成“没有到达任何 Ticket”来处理；只要没有其他 Ticket 仍然拥有它，它就会进入 Eval。主 Session 不会收到“它已经到过某个 worker”这样的信息。即使是在一次 processing pause 之后恢复，也适用同样规则。此时 attach 回这个 Ticket 会被拒绝，进一步工作必须通过新的 accept 来发起。

- **人在 Thread 里说“stop”，本质上仍是一条普通 Trigger，** 由 Eval 决定 stop。
- **面板中的 stop 是一个命令。** 它直接作用于 Ticket，并被记录为人工 stop。
- **等待条件属于 Source。** 像“连续 N 小时无人认领”这样的条件，应由某个 Source 自己等待并再次检查；Eval 永远不会自己等待。

## 当 Eval 没有完成时

时间限制从 candidate 被送达到主 Session 时开始计时。等待投递的时间不计入其中；未送达的 candidate 永远不会超时。

一次超时的 Eval 会把其 Triggers 重新排队再试。连续两次重试后，第三次超时会记录为 `unavailable`。assessment 失败同样会记录 `unavailable`；之后的新 Trigger 仍然可以重新触发新的 Eval。若发生 Policy 变更，或某次 decision 使用的是已过时的 Policy，也会记录为 `unavailable`，并立即在当前版本下把该批次重新排队。

`evals` 健康项会报告过去一小时内仍未解决的 unavailable Decision。`scripts/dsh-superagent status` 也会包含这个错误。

面板只会在以下条件同时成立时，才要求人关注一个 unavailable Decision：它发生于最近 24 小时内、它仍是其 Thread 上最新的 Decision、尚未被 dismiss，且该 Thread 既未排队也未处于 evaluating 状态。此时人可以要求再次评估这个 Trigger，或者 dismiss 这个问题。
