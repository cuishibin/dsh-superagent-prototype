# 008 扩展示例手册

本手册展示如何扩展 `dsh-superagent`。它使用 [001 领域模型](001-domain_cn.md) 与 [002 授权](002-authorization_cn.md) 中的术语，以及 [007 架构](007-architecture_cn.md) 中的组件。

## 添加一个 Source

本示例实现一个名为 `webhook` 的 Source kind。它通过一个虚构的 CLI `hookq` 去读取一个假想中的消息队列，并把消息转换成 Triggers。除此之外，其他接口都来自 `dsh-superagent` 本身。

这个实现有四项职责：翻译事件、定义接收与授权规则、提供回复与上下文，以及维护轮询循环。下面的步骤会按这个顺序逐步构建。

示例假设这个 CLI 契约存在：

- `hookq pull --after <n> --json` 会打印位置 `n` 之后的消息，按时间从旧到新：`{"me": "<the agent's id on the queue>", "items": […], "next": <position>}`。每条 item 包含 `id`、`conversation`、`thread`（表示其所属 thread 的起始消息 id；若该消息就是 thread 起点则不存在）、`sender`（含 `id` 和 `bot`）、`mentions_me`、`text`、`at` 和 `url`。
- `hookq reply --conversation <id> --to <message id> --key <key> --text <text>` 会在某条消息下回复，且同一个 `key` 只会生效一次。如果目标 conversation 已被删除，它会在标准错误输出 `{"error": {"code": 410, "message": "…"}}`。
- `hookq thread --conversation <id> --root <message id> --json`，若附带 `--after <cursor>` 则表示读取后续一页。输出为一个 thread page，消息同样按从旧到新排列：`{"items": […], "next": "<cursor>"}`；若是最后一页，则没有 `next`。

把这个 kind 及其 plugin entry 写在 `src/sources/webhook.ts` 中，风格上可参考 `src/sources/github.ts`。如果这个 Source 以后继续扩展，也可以像 Feishu Source 那样，把翻译、Channel 和 polling 逻辑拆到不同模块。

### 1. 先把事件映射到领域模型

在写代码前，先选定稳定映射。因为 Trigger id、Thread id 和 Channel address 都会持久化保存，后续再改会导致已有记录断裂。

| 概念 | 在 webhook 队列中的对应物 |
| --- | --- |
| Trigger | 一条消息 |
| Thread | 某个 conversation 中的一条消息及其所有回复：`<conversation>:<thread 起始消息 id>` |
| Actor | 发送者在该队列中的 id，也就是 allowed list 中使用的那个 id |
| Channel address | conversation 以及回复要挂在哪条消息下 |
| Receiving scope | 一个 conversation，在 admission table 中占一行 |

### 2. 配置：schema、fields、facts 与 health

模块会用到下面这些导入：

```ts
import type { Context } from '@deepseek-ai/cordis'
import { setTimeout as delay } from 'node:timers/promises'
import { z } from 'zod'
import { key, threadKey, type OutboundMessage, type Trigger } from '../core/index.ts'
import { CliError } from '../infra/commands.ts'
import { KV } from '../kv.ts'
import { health, sourceContext, type SourceContext } from './host.ts'
import type { SourceKind } from './kind.ts'
```

```ts
const ConfigSchema = z.object({
  conversations: z.array(z.object({
    id: z.string().regex(/^[a-z0-9-]+$/),
    listen: z.enum(['mentions', 'all']),
  }).strict()).default([]),
  pollSeconds: z.number().int().min(10).max(3600).default(30),
}).strict()
export type WebhookConfig = z.infer<typeof ConfigSchema>

export const webhookKind: SourceKind<WebhookConfig> = {
  kind: 'webhook',
  title: { zh: 'Webhook 队列', en: 'Webhook queue' },
  description: { zh: '接收 webhook 队列里下列对话的消息。', en: 'Receives the messages of these conversations from the webhook queue.' },
  schema: ConfigSchema,
  fields: [
    { key: 'conversations', type: 'rows', label: { zh: '接收的对话', en: 'Conversations' }, columns: [
      { key: 'id', type: 'text', label: { zh: '对话', en: 'Conversation' }, pattern: '[a-z0-9-]+' },
      { key: 'listen', type: 'select', label: { zh: '接收', en: 'Receive' }, options: [
        { value: 'mentions', label: { zh: '仅 @', en: '@ only' } }, { value: 'all', label: { zh: '全部消息', en: 'All' } }] },
    ] },
    { key: 'pollSeconds', type: 'number', advanced: true, min: 10, max: 3600,
      label: { zh: '轮询间隔 / 秒', en: 'Polling interval / seconds' } },
  ],
  health: ['webhook'],
  // find, scope, channel, threadContext, policy and environment follow in the next steps.
}
```

- `kind` 是稳定的 Source id。它会出现在 Trigger 的 `source`、Thread key 前缀、`source:webhook` settings block 以及 Source Policy 中。
- `schema` 负责校验已保存的配置，并提供默认值。在操作员首次保存一个 instance 之前，registry 会按 schema 默认值创建一个 enabled instance。默认值应设计为“什么都不接收”；本例中默认就是空 `conversations`。
- `fields` 会自动生成 Source settings 表单，无需自定义 panel 代码。`rows` 字段会把第一列当作主键来编辑行；`advanced` 表示不常用项，默认折叠。如果该 Source 能把 id 解析成可显示名称，就实现 `refreshLabels`，并把这些名称保存到 `KV.sources.labels(kind)`；这样带 `choose` 和 `refresh` 的列就能使用它们。
- 可选的 `facts(deployment)` 可以展示由 setup 推导出的值，比如某个 CLI 当前对应的是哪个账号。当前 setup 只会检查 `SETUP_FIELDS` 中列出的 GitHub 与 Feishu 字段，因此本示例没有额外 facts。
- `health` 用于声明这个 Source 会使用哪些 health entries，初始状态都是 `starting`。registration 会占用这些名称，并拒绝与 `runtime`、`reply`、`worker` 或 effect kinds 等名称冲突。建议使用 kind 的名字，并把 `sync:<id>` 留给 Policy 同步使用。`error` 会让状态脚本退出 2，并使每小时检查通知操作员，因此只有在确实可能需要人工介入时才报告它。

### 3. 把一条消息翻译成 Trigger

先校验 CLI 的返回值，再做转换：

```ts
/** One message as `hookq pull` and `hookq thread` print it. */
const MessageSchema = z.looseObject({
  id: z.string().min(1), conversation: z.string().min(1), thread: z.string().optional(),
  sender: z.looseObject({ id: z.string(), bot: z.boolean().default(false) }), mentions_me: z.boolean().default(false),
  text: z.string(), at: z.string(), url: z.string().optional(),
})
type QueueMessage = z.infer<typeof MessageSchema>
/** The answer of `hookq pull`, whose items are read one by one, and of `hookq thread`. */
const PageSchema = z.looseObject({ me: z.string(), items: z.array(z.unknown()), next: z.number().int() })
const ThreadPageSchema = z.looseObject({ items: z.array(MessageSchema), next: z.string().optional() })
```

```ts
/** The Trigger of one queue message; the agent's own messages produce none. */
export function webhookTrigger(message: QueueMessage, me: string): Trigger | undefined {
  if (message.sender.id === me) return undefined
  const thread = `${message.conversation}:${message.thread ?? message.id}`
  return {
    id: key('webhook', message.id),
    source: 'webhook',
    threads: [{ source: 'webhook', id: thread }],
    actor: { source: 'webhook', id: message.sender.id },
    human: !message.sender.bot,
    ...(message.mentions_me ? { addressed: 'mention' as const } : {}),
    at: message.at,
    text: message.text,
    channel: { source: 'webhook', thread: `webhook:${thread}`,
      address: { conversation: message.conversation, message: message.id, ...(message.url ? { url: message.url } : {}) } },
    ...(message.url ? { url: message.url } : {}),
    meta: { conversation: message.conversation, messageId: message.id, root: message.thread ?? message.id },
  }
}
```

- **稳定 id。** 用队列消息 id 通过 `key` 生成 Trigger id。`Engine.receive` 对已存在的 id 会返回 `duplicate`，因此即使重复读取，也不会造成重复 accept。若以后需要把“编辑”视为新事件，可像 GitHub comment 的 `updated_at` 一样，把编辑版本号纳入 id。
- **Threads。** 第一个 Thread 是 primary，它拥有 Eval。额外 Thread 只提供 Correlation 上下文。具体使用什么 id 由 Source 自己决定；完整 key 会变成 `webhook:<id>`。
- **发送者。** `actor` 中使用 Source 自己的原生 id。对于 bot 或 CI，设置 `human: false`。被列入 bot 列表的 bot 最多只能成为 inferred candidate。若 actor id 为空（例如 CI），则会命中 admission table 中“未列入名单且未指向智能体”的那个 cell。只有在消息真的指向了智能体时，才设置 `addressed` 为 `mention` 或 `direct`；这样一个在允许名单中的人，才会得到 explicit strength。永远不要为智能体自己发出的消息生成 Trigger。
- **回复目标与元数据。** `channel` 用来标识该往哪里回复；`url` 用来链接回原始消息。`channel.thread` 用于给出站消息做顺序分组：同一个 Ticket 发往同一 conversation 的后续消息，会等待之前的消息发出或失败 3 次。其他 Ticket 不受影响。像 conversation、thread root 这类 Source 特有字段，都应放进 `meta`。`meta` 与 `text` 都是不可信数据，不能参与授权。

接入这个 kind 后，智能体就能通过 `orchestrator_read trigger/<id>` 和 `thread/<key>` 读取这些新 Trigger 和 Thread。它们暴露消息文本、disposition 以及相邻 Triggers。但一个新 kind 不会自动拥有内建 skill、原始服务记录资源，或 pull request binding 能力。

普通全文搜索无需额外代码即可工作。若用户会使用 Source 特有的引用方式（比如消息 id、URL、`#12`）进行搜索，就实现 `find(query)`。它应返回对应识别出的 Thread keys 和搜索 terms：

```ts
  find: query => ({ threads: [], terms: [...new Set(query.match(/\bm[0-9]+\b/g) ?? [])] }),
```

在这个示例中，查询里的队列消息 id（例如 `m42`）会被拿来搜索对应 Trigger。

### 4. Source 接收什么：scope

```ts
  scope({ meta }, config) {
    const conversation = config.conversations.find(item => item.id === meta.conversation)
    if (!conversation) return { listened: false, reason: 'conversation_not_listed' }
    return { listened: true, observed: conversation.listen === 'all', row: conversation.id }
  },
```

Engine 会通过 `SourceRegistry` 为每个新 Trigger 调用 `scope`。超出范围的消息仍应通过 `receive` 进入系统，并作为 filtered 被记录，而不是在 polling loop 里悄悄丢掉；这样面板才能解释“为什么被过滤”。

`observed: true` 表示该 Source 在这个范围里会接收未指向智能体的消息，因此 admission table 才有机会对它们生效。`row` 则标识 admission table 中对应哪一行。

revalidation 会用 Mandate 中保存的元数据再次调用 `scope`。因此 `scope` 只能依赖 `meta` 和当前配置。恢复只要求 `listened` 仍为真，不要求 `observed` 仍为真。若一个 kind 不实现 `scope`，则默认它接收自己产生的所有 Trigger，但对未指向智能体的消息一概视为不观察。

### 5. 回复：Channel

```ts
  channel: {
    allowed: (address, config) => config.conversations.some(item => item.id === address.conversation),
    send: sendWebhook,
    link: address => ({ url: address.url ?? '',
      label: { zh: `Webhook 对话 ${address.conversation}`, en: `Webhook conversation ${address.conversation}` } }),
  },
```

```ts
export async function sendWebhook(host: Pick<SourceContext, 'commands'>, message: OutboundMessage): Promise<'sent' | 'gone'> {
  const { conversation, message: to } = message.channel.address
  if (!conversation || !to) throw new Error('Missing webhook reply target')
  try {
    await host.commands.run('hookq', ['reply', '--conversation', conversation, '--to', to, '--key', message.id,
      '--text', message.text])
    return 'sent'
  } catch (error) {
    if (error instanceof CliError && error.code === 410) return 'gone'
    throw error
  }
}
```

- **发送幂等。** Outbox 会通过 `SourceRegistry.send` 调用 `channel.send`。如果 Source plugin 被卸载，消息会先等待。发送可能在远端已成功但本地没来得及确认时失败，因此重试不能制造重复消息。用 `message.id` 作为幂等 key：示例中的 `hookq` 支持 `--key`；GitHub 会写隐藏 comment marker；飞书支持 idempotency key。
- **永久失败和可重试失败要区分。** 成功返回 `sent`；只有在“永远无法送达”的情况下才返回 `gone`，例如示例里的 410。gone 消息会被保留为 dropped，不再重新排队，并会显示在 `GET sources` 和 Source 页面中。其他错误一律抛出，让 Outbox 从 5 秒开始重试，最长退避到 300 秒。`CliError.code` 是 CLI 自己输出里报告的那个错误码。
- **allowed destinations。** `allowed(address, config)` 只依赖“已保存的 address”与“当前配置”。凡是判断一个目标是否还允许发送所必需的事实，都应写进 address；例如飞书会记录是否为 direct chat。若目的地已不再被监听，则消息应被 dropped；而 Source 仅被 disabled 时，消息仍应等待。若不实现 `allowed`，则 Source 在 enabled 状态下默认允许所有地址。
- **links。** `link(address, links)` 用来生成面板中显示的 URL 与 label，最终由 `SourceRegistry.link` 使用。

### 6. 给 assessment 提供 Thread 上下文

```ts
  threadContext: {
    messageId: trigger => String(trigger.meta.messageId ?? ''),
    // A queue thread is flat: its root, and the message a reply answers, are on its first page, as on GitHub.
    read: async (commands, trigger, _part, cursor) => {
      const page = ThreadPageSchema.parse(JSON.parse(await commands.run('hookq', ['thread', '--conversation',
        String(trigger.meta.conversation), '--root', String(trigger.meta.root), '--json', ...(cursor ? ['--after', cursor] : [])])))
      const messages = page.items.map(item => ({ id: item.id, from: item.sender.id, at: item.at, text: item.text,
        ...(item.sender.bot ? { bot: true as const } : {}), ...(item.url ? { url: item.url } : {}) }))
      return { messages, ...(page.next ? { next: page.next } : {}), body: { messages } }
    },
  },
```

主动 assessment 需要读取 Trigger 周围的对话上下文。`orchestrator_eval` 会先读取一个初始窗口；`orchestrator_eval_read` 可以继续展开更多 page、root 或 parent message。可通过可选的 `threadContext` 接口来实现：

- `read(commands, trigger, part, cursor, config)` 返回一个 `ContextPart`。其中 `messages` 使用 `ContextMessage` 结构：`{id, from, at, text}`，并可附带 `bot`、`deleted`、`reply_to`、`url`、`path`、`line` 等字段。最后一页时省略 `next`。`body` 是真正展示给 assessing Session 的内容；本例中与 `messages` 相同。可选的 `status` 可用于表达特殊结果，例如飞书的 `chat_neighbours`。`part` 可能为 `thread`、`root` 或 `parent`。
- `messageId(trigger)` 用来在这些消息中找出 Trigger 自己对应的消息。
- `images(text)` 和 `image(…)` 适用于消息里可能携带图片的 Source，飞书就是这样；本示例省略。

若一个 kind 不提供 `threadContext`，那么 assessment 时就拿不到 Thread 上下文，只能读取 Trigger 自身的文本。

如果消息支持“回复某条更早消息”，可以额外实现 `repliesTo(trigger)`。它返回对应 Trigger 的 id，这样 follow-up notices 和 worker Inputs 在读取时就能显示“它回复了谁”。例如，若 `hookq` 带 `reply_to`，并在翻译阶段把它存进 `meta.replyTo`：

```ts
  repliesTo: trigger => typeof trigger.meta.replyTo === 'string' ? key('webhook', trigger.meta.replyTo) : undefined,
```

### 7. 谁可以发起工作：Source 的 Policy 部分

```ts
  policy: {
    schema: z.object({ allowed: z.array(z.object({ id: z.string().min(1), name: z.string().trim().max(100).optional() }).strict())
      .default([]) }).strict(),
    fields: [{ key: 'allowed', type: 'rows', label: { zh: '能发起工作的发送者', en: 'Senders who may originate work' }, columns: [
      { key: 'id', type: 'text', label: { zh: '发送者', en: 'Sender' } },
      { key: 'name', type: 'text', label: { zh: '名字', en: 'Name' }, optional: true }] }],
    explicit: { zh: '名单里的人在消息里 @ 智能体。', en: 'Someone on the list mentions the agent in a message.' },
    rows: stored => ConfigSchema.parse(stored).conversations.map(conversation => {
      const all = conversation.listen === 'all'
      return { id: conversation.id, label: conversation.id, note: all ? { zh: '全部消息', en: 'All' } : { zh: '仅 @', en: '@ only' },
        cells: { listedUnaddressed: all, unlistedAddressed: true, unlistedUnaddressed: all } }
    }),
  },
```

- `schema` 用来校验这个 Source 的 Policy 部分，包括 `allowed` sender ids。id 比较时大小写不敏感。admission tables 的校验是独立通用的。这个 Policy 片段保存在 `policy.sources.webhook` 下，并会参与 Policy version 的计算。
- `fields` 会生成 **Eval → On each Source** 下的配置表单，且从 `allowed` 开始。`explicit` 用于解释“怎样才算在这个 Source 上明确指向智能体”。可选的 `unnamed: {person, bot}` 可以为不透明 id（如飞书的 `ou_…`）提供展示名称；本示例省略，因为队列里的 id 可直接阅读。
- `rows(config)` 列出 admission scopes 以及对应可用的 cells。每个 row id 必须和 `scope` 返回的 `row` 保持一致。只有 Source 真正会接收未指向智能体的消息时，对应的未指向 cell 才可用。若某个 kind 不实现 `rows`，则使用默认 cells。
- 可选的 `bots` 用于列出有资格成为 inferred candidate 的 bot，结构与 `allowed` 相同。若不实现，则没有 bot 会被准入。
- 可选的 `sync: {label, ready(part), run(host, part, signal)}` 用来在启动时和点击 **Sync now** 时刷新成员同步。飞书就是用它同步 chat membership。失败会显示在 `sync:<id>`，并从 1 分钟开始重试，最长到 1 小时，直到成功。

### 8. 给 Executor 暴露变量

```ts
  environment: {
    variables: [{ key: 'DSH_SUPERAGENT_WEBHOOK_CONVERSATION', description: {
      zh: '接下工单的 webhook 对话；工单不是从这个来源来的时为空',
      en: 'The webhook conversation that started the ticket; empty when the ticket did not start on this Source' } }],
    values: ({ trigger }) => ({
      DSH_SUPERAGENT_WEBHOOK_CONVERSATION: trigger?.source === 'webhook' ? String(trigger.meta.conversation ?? '') : '' }),
  },
```

Execution settings 会显示 `variables` 中声明的变量，即使 Source 当前 disabled。只要 Source 启用，`values` 就会根据 deployment、config 和原始 Trigger 生成 Ticket 级别的具体值。

prepare command 通过环境变量读取这些值；worker 会在自己的 instructions 中读取这些变量的值和说明。变量名应使用 `DSH_SUPERAGENT_<KIND>_…` 前缀；若 Ticket 来自别的 Source，就返回空字符串；绝不能把凭据放进去。

### 9. Plugin entry 与 polling loop

```ts
export const name = 'dsh-superagent-webhook'
export const inject = ['orchestrator']
export function apply(ctx: Context): void {
  ctx.effect(() => ctx.orchestrator.sources.register(webhookKind), 'webhook: source kind')
  ctx.effect(() => {
    const source = sourceContext(ctx.orchestrator, webhookKind)
    const controller = new AbortController()
    // DSH ends its process on an unhandled rejection, so a failure that escapes the loop becomes the Source's health.
    const running = watch(source, controller.signal)
      .catch((error: unknown) => { source.report('webhook', { state: 'error', detail: String(error) }) })
    return async () => { controller.abort(); await running }
  })
}
```

这个 plugin 注入了 `orchestrator`。第一个 effect 负责注册 kind，并在销毁时自动注销。第二个 effect 用一个 `SourceContext` 启动 polling loop。销毁阶段会 abort 这个 loop，并等待它结束，包括在应用 stop 或 restart 时。

```ts
async function watch(ctx: SourceContext, signal: AbortSignal): Promise<void> {
  await ctx.ready
  // Setup restarts dsh-superagent once it resolves the accounts; until then the Source reads nothing.
  if (!ctx.configured()) { ctx.report('webhook', health('paused', 'setupPending')); return }
  let failures = 0
  while (!signal.aborted) {
    const config = ctx.config<WebhookConfig>('webhook')
    if (!config) ctx.report('webhook', health('paused', 'sourceOff'))
    else {
      const at = new Date().toISOString()
      try {
        const { read, unreadable } = await pollQueue(ctx, signal)
        failures = 0
        ctx.report('webhook', { state: 'ready', detail: `${read} messages, ${unreadable} unreadable`, at })
      } catch (error) {
        failures += 1
        // One network failure is retried on the next round; three in a row need a person.
        if (!signal.aborted && failures >= 3) {
          ctx.report('webhook', health('error', 'pollFailures', { count: String(failures), error: String(error) }))
        }
      }
    }
    await delay((config?.pollSeconds ?? 30) * 1000, undefined, { signal }).catch(() => {})
  }
}

/** Read the queue after the cursor, take in each message, then move the cursor past them. */
export async function pollQueue(ctx: SourceContext, signal: AbortSignal): Promise<{ read: number; unreadable: number }> {
  const after = ctx.state.get(KV.webhook.cursor) ?? 0
  const output = await ctx.commands.withSignal(signal).run('hookq', ['pull', '--after', String(after), '--json'])
  const page = PageSchema.parse(JSON.parse(output))
  let unreadable = 0
  for (const item of page.items) {
    if (signal.aborted) throw new Error('The webhook Source stopped during the read')
    const message = MessageSchema.safeParse(item)
    if (!message.success) { unreadable++; continue }
    const trigger = webhookTrigger(message.data, page.me)
    if (!trigger) continue
    await ctx.receive(trigger)
    const conversation = { label: { zh: '对话', en: 'Conversation' }, value: message.data.conversation }
    for (const ref of trigger.threads) ctx.threads.describe(threadKey(ref), { facts: { conversation } })
  }
  ctx.state.set(KV.webhook.cursor, page.next)
  return { read: page.items.length, unreadable }
}
```

- **setup 与开关。** 必须先等待 `ctx.ready`，且在 `ctx.configured()` 为 false 时什么都不要读。每一轮都调用 `ctx.config(kind)` 获取当前配置；Source 被关闭时，它会返回 `undefined`。
- **cursor。** 读取位置保存在 `ctx.state` 中，并且必须通过 `KV`（`src/kv.ts`）中定义好的 typed entry 来访问，比如 `webhook: {cursor: record<number>('webhook-cursor')}`。`tests/store.spec.ts` 要求 key literal 只出现在那个注册表里。外部包也可以通过导出的 `record` 与 `family` 定义自己的 key。把值设成 `undefined` 就是删除该记录；`ctx.atomic` 可把多次写入放进同一个事务。

  只有在“这个 cursor 覆盖的所有 Trigger 都已经成功写入系统”之后，才前移 cursor。若某一轮中途被打断，cursor 必须保持不变；由于 Trigger id 稳定，重放是安全的。若使用时间戳型 cursor，可使用 `src/sources/poll.ts` 中的 `cursorAt(started)`，它会向前多减 5 秒，以容忍远端写入延迟。
- **Thread 元数据。** 在 `receive(trigger)` 成功之后，调用 `ctx.threads.describe(key, {label, url, facts})` 补充该 Thread 的展示信息。`ctx.threads.patch(key, meta)` 会委托给 `threads.patchMeta`，用于设置或删除 Source 自己拥有的 metadata keys。两者在 Thread 不存在时都会静默忽略，包括 filtered Trigger 对应的 Thread。
- **逐项容错。** 对 malformed item 要计数并继续，而不是让整个 batch 失败。`src/infra/commands.ts` 中的 `validItems(item, list, tally)` 就是为这种模式准备的。若是对独立 Threads 进行轮询，则优先使用 `src/sources/poll.ts` 中的 `pollThreads`；它会把每个失败记录为 `pollError`。其中 `polled` 判断会考虑 live subscriptions，并且会保留“两周窗口”，即从“最后 Trigger 时间”和“Ticket 结束时间”中较晚者再往后两周。
- **health。** 只能通过 `ctx.report` 写入自己在 kind 中声明过的 health entries。`health(state, code, values)` 提供了固定消息，如 `setupPending`、`sourceOff` 和 `pollFailures`。一般应先累计“连续失败次数”，达到阈值后再报告 `error`。
- **abort。** 把 signal 传入 `ctx.commands.withSignal(signal)` 和 `delay`，并在逐条处理 item 的过程中间检查它；这样 stop 或 restart 才能立即终止 loop。

本示例没有发送 Source 自己产生的 notices。若某个 Source 需要这样做，可调用 `ctx.notify`；Outbox 会把它像普通 replies 一样投递出去。

本示例也没有通过 Thread 生命周期来决定 Ticket 是否结束。只有在某个 Source 的 Thread 生命周期本身就决定“工作是否完成”时，才应使用 `ctx.tickets.bindThread` 与 `threadClosed`，GitHub pull request 就是这种情况。真正何时结束 Ticket，仍然由 Engine 决定。

### 10. 注册这个 plugin

在 `tsdown.config.ts` 中把该模块加入自己的构建入口：

```ts
entry: { index: 'src/index.ts', 'github-watch': 'src/sources/github.ts', 'feishu-watch': 'src/sources/feishu.ts',
  'webhook-watch': 'src/sources/webhook.ts' },
```

然后在 `package.json` 的 `exports` 中补上 `"./webhook-watch": "./lib/webhook-watch.js"`，并像 Feishu 一样，把这个 plugin 加到 bundle 的 `cordis.patch.yml` 的 `insert` 列表中，初始设为 disabled：

```yaml
    - id: dsh-superagent-webhook
      name: dsh-superagent/webhook-watch
      disabled: true
```

在某个 profile 的 `cordis.patch.yml` 中，把 `id: dsh-superagent-webhook` 设置为 `disabled: false`。然后重新 build 并 restart。新的 Source 就会出现在 Sources settings 和 **Eval → On each Source** 下。

同时，记得把这个新模块和对应 spec 补进 [007 架构](007-architecture_cn.md) 的 module table 与 test table。`tests/docs.spec.ts` 会检查这两个索引是否完整。

### 11. 测试

创建 `tests/webhook.spec.ts`，模式可以照搬 `tests/sources.spec.ts`：使用一个临时目录中的真实 Store、带 `HealthBoard` 的 `SourceRegistry`、带 stub ports 的 Engine，以及通过 `sourceContextOf` 构造出来的 `SourceContext`。

通过 spy `Commands.run` 来 mock CLI 调用。由于 `withSignal` 通常会返回一个新的 `Commands` 实例，因此也要一并 spy 它。

```ts
function host(overrides: Partial<SourceContextDeps> = {}) {
  const store = new Store(join(mkdtempSync(join(tmpdir(), 'webhook-')), 'db.sqlite'))
  store.saveSettings(SettingsSchema.parse({
    sources: { webhook: { enabled: true, config: { conversations: [{ id: 'ops', listen: 'mentions' }] } } },
    policy: { sources: { webhook: { allowed: [{ id: 'alice' }] } } },
  }))
  // The registry claims the kind's health entries on the board, where the test reads them.
  const board = new HealthBoard()
  const sources = new SourceRegistry(() => store.settings(), board)
  sources.register(webhookKind)
  const engine = new Engine({ repo: store, scope: sources, policy: () => settingsPolicy(store.settings(), sources),
    assessor: { request: vi.fn() }, executor: { start: vi.fn(), deliver: vi.fn(), stop: vi.fn() },
    outbox: { send: vi.fn(), status: () => undefined, withdraw: vi.fn() } })
  const commands = new Commands(() => testDeployment(), {}, new AbortController().signal)
  const receive = vi.fn((trigger: Trigger) => { const result = engine.receive(trigger); engine.evaluateDue(); return result })
  // The Source's own notices go to the outbox, which keeps their receipts and sends nothing here.
  const deps: SourceContextDeps = { store, engine, outbox: { send: message => { store.receipts.add(message) } }, commands, sources,
    deployment: testDeployment(), ready: Promise.resolve(), configured: () => true, receive, ...overrides }
  return { ...deps, store, engine, commands, receive, health: board.all, ctx: sourceContextOf(deps, webhookKind) }
}
/** The queue as `hookq` answers it. */
function queue(h: ReturnType<typeof host>, answer: () => unknown) {
  vi.spyOn(h.commands, 'withSignal').mockReturnValue(h.commands)
  return vi.spyOn(h.commands, 'run').mockImplementation(async () => JSON.stringify(answer()))
}

it('takes in what the queue holds after the cursor, once, and moves the cursor after it', async () => {
  const h = host()
  queue(h, () => ({ me: 'agent', items: [message('m1'), { id: 'broken' }, message('m2', { conversation: 'elsewhere' })], next: 3 }))
  expect(await pollQueue(h.ctx, new AbortController().signal)).toEqual({ read: 3, unreadable: 1 })
  expect(await h.receive.mock.results[0]!.value).toBe('stored')
  expect(await h.receive.mock.results[1]!.value).toBe('filtered')
  expect(h.store.get(KV.webhook.cursor)).toBe(3)
  // An allowed sender who mentions the agent makes an explicit request.
  expect(h.engine.candidate(key('webhook', 'm1'))?.strength).toBe('explicit')
  await pollQueue(h.ctx, new AbortController().signal)
  expect(await h.receive.mock.results[2]!.value).toBe('duplicate')
  h.store.close()
})
```

`message(id, fields)` 用来构造一条队列消息。至少要覆盖以下测试点：

- 翻译逻辑：稳定 id、Thread、actor、`human` 与 `addressed`，以及智能体自己的消息不会变成 Trigger；
- 一轮 poll：Triggers 只被存一次，一条超出 scope 的消息被记为 `filtered`，cursor 会前移，允许名单中的发送者 mention 智能体会被评估为 `explicit`；
- 一轮被中途打断的 poll，应保持 cursor 不变；
- `send`：成功时返回 `sent`，只有 410 返回 `gone`，其他错误应抛出；CLI 错误可用 `commandError` 构造；
- Thread context：返回某一页的 messages 和下一页 cursor；参考 `tests/registry.spec.ts` 如何通过 `SourceRegistry.threadContext` 读取 kind 的 Thread；
- plugin：`apply` 会注册 kind；在 setup 尚未完成时报告 `setupPending`；其 disposer 会注销 kind；任何逸出 loop 的失败都应写入 Source health，而不是成为 unhandled rejection；
- Source 的 Policy 部分和它的 admission rows；可参考 `tests/registry.spec.ts` 与 `tests/policy.spec.ts` 对 Feishu Source 的测试方式。

### Checklist

1. 先映射 Trigger、Thread、actor、Channel address 与 receiving scope。
2. 写出 kind：`schema`、`fields`、`health`，若 setup 能推导出额外事实，则补上 `facts`。
3. 把事件翻译成带稳定 id 的 Trigger，并排除智能体自己的消息。
4. 在 `scope` 中，仅基于 `meta` 决定 Source 接收什么。
5. 提供一个 Channel：其 `send` 能保证每条消息只发一次，并且只有在目标对话永远无法接收时才返回 `gone`。
6. 如果 Source 支持读取 Thread，就为 assessment 提供 Thread context。
7. 为这个 Source 定义自己的 Policy 部分，至少包括 `allowed` 和 admission rows。
8. 注册它的 execution environment variables。
9. 编写 plugin entry 与 polling loop：等待 setup、响应开关、在 `receive` 成功后移动 cursor、对单项错误容错、报告 health、在 abort 时立即停止；并把 cursor key 注册进 `KV`。
10. 补上构建入口、导出项和 `cordis.patch.yml` insert，并把该 module 和对应 spec 补进 007 的表格中。
11. 按 `tests/sources.spec.ts` 测试其他 Sources 的方式来测试它。
