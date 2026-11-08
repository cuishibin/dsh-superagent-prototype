# 005 面板

在 DeepSeek Harness Web 侧边栏中打开 **Superagent**，即可监督这个智能体。面板会优先展示需要人处理的工作，其次才是系统健康和历史信息。本文档建立在 [000 理念](000-philosophy_cn.md) 到 [004 提示词](004-prompts_cn.md) 之上。

## 三个问题

打开面板，本质上是在按顺序回答三个问题：

1. 现在有没有什么需要我立刻处理？
2. 系统现在是否健康，且正在做正确的事情？
3. 某一项工作从 Trigger 到 Eval 再到 Ticket 的完整历史是什么？

## Tabs

这些标签页沿着工作流组织：**Overview**、**Source**、**Trigger**、**Eval**、**Execution** 和 **Settings**。在 repository 与 account 的 setup 成功之前，Overview 会直接显示缺少哪些条件，并给出跳转到相应 setting 的链接。

| 标签页 | 显示内容 |
| --- | --- |
| Overview | 需要人处理的 Tickets、系统问题、过去 24 小时的活动计数以及 Executor 负载。每个区块都可以跳到更详细页面 |
| Source | Source 状态、setup 问题以及所有非零计数。Filtered Triggers 和 dropped messages 采用与 Overview 相同的 24 小时时窗 |
| Trigger | 可按 Subscribed、Evaluating、24 小时内有新 Trigger、或 All Threads 过滤 Thread。每个 Thread 都显示其 Triggers（最新在前）、Decisions 和 Subscriptions |
| Eval | Decisions，以及它们的 reason 和 matched rules |
| Execution | Executor 状态、Pause processing 控制，以及可按 Open、Pending、Working、Waiting for a Trigger 或 Ended 过滤 Tickets。Ticket 详情会展示 actions、authorization、waiting Inputs、Subscriptions 和 Assignments |
| Settings | Sources、Eval、Execution、Prompts 和 Deployment。每个 group 都包含自己的保存历史 |

### 查看单条记录

一条被 dropped 的 message，会链接到它所属的 Ticket；如果没有 Ticket，则链接到对应 Thread。Source 页面会解释，是 conversation 消失了，还是 Source 已经不再监听它。

如果某条人的 follow-up 仍然归某个 Executor 持有，它会被标记为 **Delivered to Ticket**，而不是进入 Eval。Thread 页面还会显示 hand-back 的原因，以及那些仅作上下文的 Subscriptions。**Evaluate again** 可以把这类 follow-up 重新送进 Eval，或者重试一个没有有效 judgement 覆盖的 Trigger：比如被 Source 过滤掉的、得到 unavailable Eval 的，或者虽然被 delivered 但既没有非 stop 的 Decision、也没有 Executor acknowledgement 的。

Ticket 详情会展示原始 Trigger、Decision 及其 reason、scope 和 Policy version。Assignments 和次级细节会折叠在需要立即操作的信息下方。对于 failed Input delivery，或一个因 blocked / failed Report 而在等待人的 open Ticket，会提供 **Retry**。已结束的 Ticket 会显示诸如 **Pull request merged** 或 **Stopped by a person** 这样的 outcome。

## Settings groups

- **Sources：** 每个 Source 一张卡片，包含开关、字段和已验证的 account facts。GitHub 还提供对 new Issues、pull requests、comments 和 default-branch CI 的 receive-all 开关。轮询和读取限制放在 Advanced 中。
- **Eval：** 先显示 Policy version 和 rules，然后是 **On each Source** 与 Eval settings。每个 Source 标签页中都包含它的 allowed list、membership sync、admission table 和 bot list。Sync 会在启动时以及点击 **Sync now** 时运行。admission table 会区分接收范围、sender 是否在名单中，以及消息是否指向智能体。Eval settings 用来选择 accept mode、timeout 和 assessment model。
- **Execution：** worker model、并发数、目录根、prepare command 和 blocked commands。点击某个可用变量，可以把它直接插入 prepare command。
- **Prompts：** **Edit** 用于修改 team sections、内建 parts 和 skills。**Full prompt** 显示组装后的完整文本，并可与已保存文本或默认文本进行比较。**History** 列出所有版本。已编辑的内建 parts 会被标记为 changed，并可单独 reset；模板会列出它们要求保留的占位符。

  如果升级导致某个被 override 的默认值发生变化，面板会把它标记为 **Default updated**（默认内容已更新）。它会在团队文本上方比较旧默认值与新默认值。如果旧默认值当时没被记录下来，则退而比较“团队文本”和“新默认值”。这时可以选择 **Keep my text** 或 **Use the new default**。那些已被删除的 part 的 override，会列在 tab bar 下方，并在下次保存时被移除。
- **Deployment：** 显示启动问题，并提供 **Check again**；同时展示 state directory 和 machine limits。保存后的 limits 无需重启即可生效。DeepSeek Harness Plugin Manager 暴露的是同一页内容。

## Search

顶部搜索可以按文本、id、飞书 message id，或已存储的 link（例如 GitHub comment URL）来查找 Trigger。一个飞书链接如果原始消息里没有带出，则只有在宿主在当前进程生命周期中读取过它之后，它才会变得可搜索。

搜索也支持通过 `#12` 这类引用或 GitHub URL 查找 Thread，也支持按 title 或匹配到的 Thread 来查找 Ticket。

## 原则

1. **只聚焦重要的事。** 每个元素都必须回答前面三个问题中的一个，否则就应移除。只有当某个数字会改变监督者的行为时，它才应该出现。
2. **定义必须精确。** 各种分类都必须具备可检验定义、互斥且覆盖所有情况，正如 Ticket stage 那样。一个标签应该说明“它在等谁、需要什么”，而不是暴露内部状态码。
3. **每个概念只有一个归属页面。** Source 状态属于 Source；Threads 和 Triggers 属于 Trigger；Decisions 属于 Eval；Tickets 属于 Execution。Settings 也按它所配置的概念分组。Overview 只负责汇总和链接，而不是变成另一个详情页。
4. **按领域结构组织。** 面板使用领域术语。内部 identifier 只在折叠细节中出现；只要能解析出可读名称，就用名称替代 identifier。
5. **状态必须诚实。** 一个 count 应当等于它所链接列表的长度，任何状态都不应和其他事实互相矛盾。
6. **默认保持安静。** 健康状态不应占额外空间。问题必须是可行动的；偶发一次网络错误还不够格。那些需要人处理的行，应直接给出操作入口。页面上的解释文字控制在一句话之内，更详细内容放进 `(?)` tooltip。破坏性操作必须要求确认。
7. **控件保持一致。** 使用 DeepSeek Harness 的设计 token 和控件行为。相同的信息在面板中的外观应保持一致。每个 settings block 只有一条 save bar，并且只有在有变更、正在保存或保存失败时才显示。
