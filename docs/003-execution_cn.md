# 003 执行

本文档沿着一个已被 accept 的 Ticket，说明它如何执行、如何交付、以及在失败后如何恢复。它建立在 [001 领域模型](001-domain_cn.md) 和 [002 授权](002-authorization_cn.md) 之上。

## Ticket 阶段

每个 open Ticket 在任一时刻都处于一个 stage，这个 stage 由它“当前在等待什么”决定。按照下表从上到下读取，第一条匹配的规则即为当前阶段。

| 阶段 | 条件 | 面板标签 |
| --- | --- | --- |
| person / delivery | 向 Executor Session 投递某个 Input 失败；它会在 2 秒后重试，之后按指数退避，最长到 1 分钟；投递成功、启动新的 Assignment，或 Ticket 结束，都会清掉这个状态 | Input undelivered |
| person / answer | 它当前有一个 Attention | Awaiting answer |
| person / review | 最新 Report 是 `ready` | Awaiting review |
| person / blocked, failed | 最新 Report 是 `blocked` 或 `failed`，且从那之后还没有人的消息送达该 Ticket | Blocked · Failed |
| working / queued | 还没有 Assignment，或者只有一个 pending Assignment | Queued |
| idle | 最新 Report 是 `waiting`，或者当前 Assignment 已结束 | Waiting for a Trigger |
| working | 其他所有情况 | Working |

`person` 开头的阶段表示：没有人工输入，Ticket 无法继续。只要有新的人的消息到达，一个 `blocked` 或 `failed` 的 Ticket 就会被放行给它的 Executor，于是进入下一条匹配的阶段，通常会变成 `working`。

`idle` 表示这个 Ticket 已经结束了当前这一轮，没有留下待回答的问题。此时，只要订阅 Thread 上来了新的 Trigger，它还可以继续。

## Executor

每个 Ticket 会有一个名为 `worker-<ticket id>` 的 Executor Session，以及一个以该 Ticket 的 short name 命名的目录。目录创建在配置好的 execution root 下；默认值为 `<stateDir>/work`。

Executor Session 以操作员的操作系统用户身份、在无需批准的情况下执行命令。它们使用 `dsh-superagent-worker` preset，包含工作所需工具，并会在前台 shell 命令运行 5 秒后把它移入后台 job。这样 worker 就能在下一步接收新消息，而不是被一个长命令阻塞。

主 Session 使用 `dsh-superagent-router`，同样有 5 秒的 shell 和 job wait 限制，但工具集更小，只用于路由。Session 在创建时会绑定 preset，之后一直沿用。worker 生成的 subagent 则使用 DeepSeek Harness 的 `standard` preset。

### 后台工作与 worker slot

如果一个 turn 结束时，仍有 shell job 或 subagent 在后台运行，那么 Ticket 会继续占用自己的 worker slot，并保持在 working 状态。此时不会汇报成 waiting，主 Session 也收不到 waiting report。job 或 subagent 完成后，会以同一个 slot 唤醒 Executor，并把结果交给它。只有在某一轮结束且没有任何后台工作时，Executor 才会正常汇报 waiting。

每一轮 turn 结束后，后台工作最多允许继续 30 分钟。达到这个上限时，shell jobs 会被 kill，Ticket 被汇报为 waiting。报告中会包含 Executor 最后说过的话，以及哪些 jobs 被杀掉、还有哪些 subagents 仍在运行。subagent 可以继续存在。被杀掉 job 的结果，会在 Executor 的下一轮 turn 中被读取。

如果把工作 handoff 给人，slot 会立即释放，而此前的 Report 仍然有效。无论是 `ready` 之后、带问题的 handoff，还是 `blocked` / `failed` 之后的 handoff，只要从那之后没有新的 Input 到达，都适用这一规则。

发生 handoff 时，subagent 会停止。shell job（例如为了 review 暴露出来的一个服务器）则可以一直运行到 Assignment 结束。它们不占用 slot，也不受 30 分钟限制，包括之后的 turn 也是如此。job 完成不会主动唤醒 Executor；Executor 会在下一次 turn 中读取其结果。

人也可以直接进入 Executor Session 自己开启一轮 turn。此时后台工作的规则完全相同。

当一个 Assignment 结束时，它的 shell jobs 会被 kill，但不会唤醒 Session。其 subagents 会被释放；如果这些 subagents 完成后唤醒了 turn，这些 turn 会被丢弃。替换用的新 Assignment 及其等待中的 Inputs，会在这段关闭过程完成之后再启动。

在 DeepSeek Harness 进程内 stop 或 restart 这个插件时，也会走同样的关闭路径。那些尚未真正进入 Session history 的 Inputs 会被保留下来。关闭时会最多等待 10 秒，等待每个 worker 的后台工作结束；如果是进程重启，那么这些后台工作会被直接终止。一个在忙碌中被中断的 Assignment，会带着 recovery note 被重新启动。

worker 的 subagents（包括嵌套 subagent）使用 DeepSeek Harness 的 `standard` preset，但继承 worker 的权限。它们只保留 worker 允许使用的那些工作工具，并且在 Ticket 结束后失去这些工具。`standard` preset 下，shell 命令会在 60 秒后移入后台。subagent 自身结束时，它的 jobs 会被杀掉。

### 准备 Ticket 目录

在第一个 Assignment 之前，如果 Ticket 目录还不存在，就会在 execution root 中用 `sh` 执行一次 prepare command，最多运行 10 分钟。如果没有配置 prepare command，worker 就会在一个空目录里启动。

prepare command 失败会让 Ticket 失败，并记录其输出。之后如果发生 retry，或者人在该 Ticket 某个 Thread 上发消息，就会再次尝试 prepare。最常见的 prepare command 是用 `git worktree` 建出一个 checkout。某个 Pull Request 被认为属于这个 Ticket，当且仅当它的 branch 的最后一个 path segment 与 Ticket 的 short name 一致。

### 并发

从 launch 开始（包括 preparation），或从某次 delivery 唤醒一个 idle 的 Executor 开始，Ticket 就会占用一个 worker slot。在活跃 turn 整个期间，它也一直占用该 slot。

配置的并发上限作用于这些 slot；另外还会额外保留 1 个 slot 用于人的消息或状态请求。这类唤醒在正常 slot 已满时仍可使用这个额外 slot，并且会优先于新的 launch 获得下一个释放出来的 slot。因此，新工作不会阻止人继续推进一个已有 Ticket。

其他 Inputs，比如 CI 结果，只能等待普通 slot。一个 launch 若没有空余 slot，就会在 `working / queued` 状态下等待。多个 launch 可以并发运行，所以某个慢 prepare command 只会拖慢它自己对应的 Ticket。

### 执行变量

prepare command 和 Executor instructions 会收到同一组 execution variables。prepare command 通过环境变量读取；Executor 在自己的 prompt 中读取这些变量的值及说明。它们都不包含凭据。

| 变量 | 注册方 | 值 |
| --- | --- | --- |
| `DSH_SUPERAGENT_DIR`, `DSH_SUPERAGENT_NAME`, `DSH_SUPERAGENT_TICKET` | Executor | Ticket 的目录、short name 和 id |
| `DSH_SUPERAGENT_BLOCKED_COMMANDS` | Executor | Executor 不允许运行的命令 |
| `DSH_SUPERAGENT_GITHUB_REPOSITORY` | GitHub | Source 正在监听的仓库，格式为 `owner/name` |
| `DSH_SUPERAGENT_GITHUB_ACCOUNT`, `DSH_SUPERAGENT_GITHUB_COMMIT_EMAIL` | GitHub | agent account 以及它的 commit email |
| `DSH_SUPERAGENT_GITHUB_NUMBER`, `DSH_SUPERAGENT_GITHUB_PR_HEAD` | GitHub | 启动该 Ticket 的 Issue 或 PR 编号；若来源是 PR，则还包含它的 head ref |
| `DSH_SUPERAGENT_FEISHU_BOT`, `DSH_SUPERAGENT_FEISHU_CHAT` | Feishu | 发起该 Ticket 的 bot，以及对应的群 |

Ticket 级别的值来自它的目录和原始 Trigger。其他值来自当前 deployment settings。只有面板可以修改执行环境设置；发给主 Session 的文本，不能改变实际配置的 shell command。

worker 通过 agent account 的 `gh` 访问 GitHub，通过 bot 的 `lark-cli` 访问飞书。被 block 的命令如果按名字调用，会直接失败。

这些 wrapper 和目录只是默认路径，不构成安全边界。worker 仍然可以通过绝对路径执行程序。

## 合并

是否启用 auto-merge，由 **Delivery requirements** 提示词控制。宿主本身不带任何 merge 专用判定逻辑。

默认情况下，Executor 不会以 agent account 的身份去开启 auto-merge，因为 GitHub 会把 merge 归属给“开启 auto-merge 的那个账号”。因此，如果需要 auto-merge，Delivery requirements 里必须指定另一个可用于 merge 的账号 CLI。若没有，就由人手动 merge。

默认指令只会在以下情况下允许 auto-merge：PR 是为该 Ticket 打开的、它已经 ready、所需 approvals 覆盖了 head commit，并且 additions 加 deletions 的总行数少于 200。在推送新的 revision 之前，Executor 会先撤销 auto-merge。

不带正文的 review（例如 approval）也会作为 Trigger 到达，并可唤醒一个正在等待 review 的 Executor。需要注意，这些规则约束的是智能体，而不是 GitHub 自己的 merge 规则。如果你想真正强制 approvals，应在 branch protection 中要求它们，并启用 stale approvals 的 dismissal。若想完全依赖人工 merge，则可以移除 merge CLI 或删除 auto-merge 那一段提示词。

## Ticket 如何结束

- **没有 Pull Request 的 ready：** 一次 `ready` Report 会把 Ticket 以 completed 结束。
- **无事可做：** 如果 Executor 发现并不存在值得让人继续行动的工作，比如请求无法复现，或者已经被别的东西取代，那么它会报告 `dropped`。Ticket 会直接结束且不发消息，除非某条匹配规则要求必须发一条；之后若人在它的 Threads 上再次发消息，它还能恢复。

### Pull requests

一旦 Executor 报告它，Ticket 就会开始跟随自己起始来源的那个 pull request。之后，只要某个 pull request 是从它的分支上打开的，并且被 Executor 报告或被 GitHub Source 发现，它也会继续跟随。被 reopen 的 pull request 会重新被跟随。

当它自己打开的 pull requests 全都不再处于 open 状态时，这个 Ticket 就算做完了。如果它自己没有打开任何 PR，那么它所跟随的所有 pull requests 都必须关闭，它才算完成。此时 Ticket 会以最后一个相关 close 的 outcome 结束，并继续保留自己的 Subscriptions，以便后来人的评论还能恢复它。

但是，只要恢复、重试或恢复后重新执行出来的工作还没结束，close 就不会立即结束 Ticket。若 Executor 已经 acknowledgement 或 answer 了一个它拥有的 follow-up，但从那之后还没有做出非 `progress` 的 Report，也没有在读取那条消息后结束一个 turn，也同样不会结束。之后只要出现一个新的 `ready` Report，并且同样满足上面的 pull request 完成规则，Ticket 才会真正结束。

聊天类 Channel 会收到一条关于 pull request 最终结果的通知。GitHub 自身会在 PR 上显示这个结果。若移除 Ticket 对某个 pull request 的 Subscription，它也会停止跟随这个 pull request。

因此，`Awaiting review` 的准确含义是：已经出现 `ready` Report，并且这个 Ticket 关联着 pull request。

### Follow-up 与上下文 Thread

Executor 负责处理自己 Ticket 自有 Thread 上的人类 follow-up。它会处理 scope 内请求，并把超出 scope 或属于别的 Ticket 的请求 hand back。只有当没有任何 Executor 还拥有这条消息时，它才会成为主 Session 的 candidate。除此之外，主 Session 对 follow-up 唯一能做的事，就是应人之请 stop 对应工作。

一个 Input 只有在 Session 把它真正拿进某一步处理，并且此后产生了 Report 或结束了一个 turn，才算“已读”。当 Ticket 结束时，那些未读、且没有被 acknowledgement、没有被 answer、也没有被 hand back 的消息，会按 [002 授权](002-authorization_cn.md) 中的描述重新回到 Eval。已读但未采取行动的消息，比如一句感谢，则不会。

仅为上下文而跟随的 Thread 也会把消息送进来，但不会让 Ticket 拥有这些 follow-up，也不会让 ended Ticket 恢复。这些消息可以被独立评估。

只要 live Ticket 仍订阅某个 Thread，Source 就会持续轮询它；在“最后一条 Trigger 的时间”和“订阅它的 Ticket 结束时间”这两者中较晚的那个时刻起，再额外保留两周。两周之后，消息就只会通过普通 intake 到来：GitHub notifications 或飞书实时事件。

## 失败与恢复

### Executor blocked 或 failed

当 Executor Session 报告 `blocked` 或 `failed`，或者它启动失败／运行报错（这同样会记录成 `failed`），Ticket 就不会再接收自动 Input。

如果失败属于暂时性问题，比如模型提供方或网络的短暂故障、HTTP 408/429/5xx、连接被拒绝/重置/超时，那么系统会先自动重试 3 次，并带指数退避：launch 会重新执行；若是 worker 的 turn 失败，则会告诉它继续；但 prepare command 失败不会自动重试。

一旦真正进入 failed 状态，CI 和 bot 的消息仍会记录在 Ticket 上，但不会投递，也不会唤醒 Session。

其工具仍然可用，因此人仍可以直接进入 Session 内继续处理。

一个 launch 若失败，不会自行再次发起。

只要发生以下任一情况，这种 hold 就会结束：人的消息到达、后续出现一个既不是 `blocked` 也不是 `failed` 的 Report，或执行一次 retry。

这一状态完全由已存储的 Reports 和 Inputs 推导出来，因此重启后仍然保持。

### Retry

当 open Ticket 处于 `person / delivery`、`person / blocked` 或 `person / failed` 时，可以使用 retry。retry 的补救方式始终是：创建一个新的 Assignment，重新启动它的 worker，并把当前等待中的 Inputs 带进去。

一旦一个人的消息已经到达一个 blocked 或 failed Ticket，说明 Executor 已经拿到了这条消息，因此 Ticket 会显示为 working；如果是 failed launch 再次启动，则显示为 queued。在这种情况下，除非消息投递本身失败，否则不再提供 retry。

retry 会结束当前 Assignment，并在同一个 Executor Session、目录、分支和 pull request 上启动一个新 Assignment。

Executor model 只在 Session 创建时生效；一个已存在的 Session 会保留它原先的 model。

### Restart

运行中的 Assignments 在重启时会以 “host restarted” 结束；随后会按 Ticket 的先后顺序，用 recovery note 启动新的 Assignment。这个 recovery note 会提醒 Executor：在重复任何外部动作前，先检查目录、git 历史和 GitHub 状态。

那些已经处于 waiting、ready、blocked 或 failed 的 Assignments，会保持原样；但如果在那条 Report 之后又有新的 Input 到达，它们也会被重新启动。

Sessions、目录、Subscriptions、待处理消息以及下文提到的 stored work，在 restart 之后都会保留下来；同时也会保留“每个 Executor Session 到底把哪些消息真正纳入了自己的 history”这项记录：某一步一旦 claim 了消息，并在模型调用前把它放入 history，这件事就会被记录。于是，若一条消息是被一个被重启截断的 turn 拿走的，那么只要后面跟着一个 Report 或 turn end，它就会算已读；相反，如果某一步只 claim 了它，但在写入 history 之前就被 cancel 或报错截断，那么它就算未读。

无论是 retry 还是 restart，新 Assignment 都会再次接收那些“已投递到被替换 Assignment、但 Executor Session 其实还没真正拿走”的消息；那些消息可能是在 cancel 或中断步骤中被丢掉的。若重启或崩溃恢复后，Session 里仍残留一份旧副本，那么在新 Assignment 的第一条消息送达之前，这份旧副本会先被移除，保证重新投递后只存在一份。

### Stored work

凡是 `dsh-superagent` 在一个 decision 之后要做的事，比如把 candidate、follow-up 或 report relay 给主 Session，launch 一个 Assignment，或投递一个 Input，都会先作为“待办工作”存储下来，然后再异步执行。

如果 relay 或 delivery 失败，比如目标 Session 不可达，它们会在 2 秒后重试，然后按指数退避，最长到 1 分钟；launch 如果失败，则会让 Ticket 失败，除非前面所说的那种暂时性故障。

每一种 stored work，只有在“同类工作里有一项连续失败 3 次”之后，才会在 overview 的 system problems 中报告错误；唯一例外是 Input delivery 的失败，它会显示在 Ticket 自己身上。

发往主 Session 的 relays 保持原始顺序，notices 也保持原始顺序；如果某一项失败了，会阻塞同类中排在它后面的项。

### Source failure

GitHub Source 会分别读取每个 Issue、pull request 和 Thread：如果某次读取失败，会把错误保存在该 Thread 上，并在下一轮 poll 继续读取；这个失败也会累计到 Source 自己的错误计数中。默认情况下，连续 3 轮 poll 都失败，Source 才会报告错误；只要有一次成功 poll，计数就归零。

由于 GitHub 对“无权访问私有仓库”也会返回 404，因此 404 也会被重试。

如果某个已通知的 Issue 或 PR 的读取持续失败，那么在它第一次读取失败一天之后就会被 drop；如果 GitHub 直接报告它已被删除，则会立即 drop。被 drop 之后，它不再导致 poll 失败，但对应 Thread 仍然保留这个错误，Source 状态里也会标出它。

Source 的读取位置保持不变，因此之后只要再次有对应 notification 到来，或它的 Thread 被单独 poll，就会补读此前错过的内容。

飞书事件消费器会报告所有失败；对订阅 topic 的轮询，也会把每次失败记录在各自 Thread 上。

GitHub 列表或飞书页面中的 malformed item 会被跳过，并由 Source 状态计数。

### Channel failure

外发消息也会先存储，再作为 stored work 发送；发送失败会从 5 秒开始重试，并按指数退避，最长到 300 秒。

同一个 Ticket 发往同一个 conversation 的消息，会按排队顺序发送，因此 acknowledgement 一定先于后面的 answer 或 result：如果同一 Ticket、同一 conversation 中前一条消息尚未发出，那么后一条会等待，直到前一条连续失败 3 次并被标记为问题。

其他 Ticket 的消息，或者不属于任何 Ticket 的消息，不会被它阻塞。

只要某条消息已经有失败尝试，它就会作为系统问题显示，直到发出成功或被撤回。

只有在以下情况下，消息才会被真正 drop：它的 Source 已不再监听这个 conversation，或者 Source 明确回答这个 conversation 永远无法接收该消息；若 Source 只是暂时关闭，消息会一直等待。

GitHub 上的 deleted Issue 或 locked conversation，飞书上的 recalled message 或已不可再找到的消息，都会触发这种“永远无法送达”的判断。

若目标 review comment 已不存在，那么回复会退回到 pull request 的主 conversation 中。

一条被 drop 的消息仍保留记录：何时被 drop、为什么被 drop、属于哪个 Ticket。这样它就不会再次被重复排队；同时，一条被 drop 的 acknowledgement 会被读作 dropped，而不是 sent。

### Pause processing

pause processing 会停止 dispatch、新的 Evals、Eval timeout、Input delivery 和 replies；但一个已经在进行中的 launch 会停在那儿，直到 processing 恢复。

Sources 仍然继续记录新的 Triggers，正在执行的 Executor turn 也不会被打断。

如果人在 pause 期间给一个 ended Ticket 发消息，那么这条消息会先等待：pause 结束后的第一次 pass 中，如果该 Ticket 的 Mandate 在当时的 Policy 下仍然有效，它就会被恢复。

处于进行中的 Eval 在 pause 期间不会超时；它的 deadline 会顺延 pause 的总时长。

### Invalid configuration

如果某个已保存的 Source configuration 不通过该 kind 的校验，那么这个 Source instance 会被禁用。
