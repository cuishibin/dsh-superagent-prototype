# 006 部署

本指南说明如何安装、配置并运行 `dsh-superagent`。它建立在 [001 领域模型](001-domain_cn.md) 到 [005 面板](005-panel_cn.md) 之上，覆盖账号、启动检查、升级和排障。

## 运行的是什么

- **一个专用的 DeepSeek Harness profile**，本文默认命名为 `superagent`。建议基于 Web profile 模板创建。它会加载 bundle，并在 `127.0.0.1` 上提供自己的面板。应与日常工作的 profile 分开：该插件会修改进程级 `PATH` 和 git 设置，而且它自己的进程需要独立重启和升级。
- **一个智能体 GitHub 账号。** 智能体通过这个账号评论、推送分支并创建 Pull Request，底层依赖一个只以该账号登录的 GitHub CLI。
- **可选：** 一个飞书 bot，用于接收聊天请求。

插件配置保存 state 目录和机器限制。其他内容都通过面板完成配置：Sources 用来设置仓库和账号，Policy 用来管理允许名单，Execution 用来指定 worker 目录和命令。

### 权限与隔离

bundle 定义了 `dsh-superagent-worker` 供 Executors 使用，以及 `dsh-superagent-router` 供主 Session 使用。它们会出现在 DeepSeek Harness 的 preset 列表里，但只供 `dsh-superagent` 内部使用。不要把它们拿去给个人 Session 选用，也不要禁用它们对应的配置项；应用依赖这些 preset 来创建和恢复自己的 Sessions。

在 Web UI 中打开一个空白的 managed Session 时，系统可能会先选中 profile 的默认 preset。`dsh-superagent` 会在它的第一次 turn 之前恢复成自己角色对应的 preset。

这两个 preset 都以操作员的 OS 用户身份、在 `danger-full-access` 模式下运行，并且能读取被订阅 Thread 上的消息。账号 wrapper 只是默认选用了智能体身份的凭据，并不构成进程级隔离。worker 仍然可以按路径执行程序，也可以读取操作员 home 目录。若要真正隔离，应使用一个仅保存智能体凭据的独立 OS 账号。

## 前置条件

- Node.js 22.19+ 或 24+、pnpm、git 2.32+、Python 3.8+ 和 tmux
- GitHub CLI (`gh`) 2.48 或更新版本。Source 配置中的 CLI 字段必须指向 `gh` 或 `lark-cli` 的 wrapper；宿主依赖它们的子命令和输出格式。
- 一个位于 github.com 的仓库；不支持 GitHub Enterprise Server。若希望每个 Ticket 都有单独 checkout，需要保留一个本地 clone，prepare command 可以基于它创建 worktree。如果该仓库使用 Git LFS，还需安装 `git-lfs`。
- 一个已为 DeepSeek Harness 配置好的模型提供方。`dsh-superagent` 默认使用该 profile 的默认模型，除非 Eval 或 Execution 设置中显式指定了别的模型。
- 若要接入飞书：安装 `lark-cli`（测试版本为 1.0.90）以及一个带 bot 的飞书 / Lark 应用。

## 账号

### 智能体 GitHub 账号

1. 为智能体创建一个 GitHub 用户账号，并授予目标仓库写权限。GitHub App 不能充当智能体，因为 `dsh-superagent` 需要读取该账号的 notifications。
2. 为这个账号准备一个只使用它自己凭据的 GitHub CLI wrapper：

```sh
mkdir -p ~/.local/bin
cat > ~/.local/bin/gh-agent <<'EOF'
#!/bin/sh
GH_CONFIG_DIR="$HOME/.config/gh-agent" exec gh "$@"
EOF
chmod +x ~/.local/bin/gh-agent
gh-agent auth login
gh-agent api user --jq .login
```

请使用智能体账号完成登录；最后一条命令应打印出该账号的 login。`dsh-superagent` 也是通过同样方式读取该 login 的。

### 飞书 bot（可选）

1. 创建一个带 bot 能力的自建应用。为其授予接收消息事件、读写聊天消息、读取群成员等权限，并发布应用版本。
2. 为这个 bot 准备一个专用的 `lark-cli` 配置，并创建 wrapper：

```sh
LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config init --app-id <cli_…> --app-secret-stdin
LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config strict-mode bot --global
cat > ~/.local/bin/lark-bot <<'EOF'
#!/bin/sh
export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent"
exec lark-cli "$@"
EOF
chmod +x ~/.local/bin/lark-bot
```

第一条命令会从标准输入读取 app secret。必须使用独立配置目录：`lark-cli` 对每个 app id 只支持一个 profile，而你的个人配置很可能已经以用户身份登录了同一个 app。

宿主会拒绝一个“还能以用户身份行动”的 CLI。它会从 bot 配置中读取 app id、飞书 / Lark 品牌，以及 bot 的 `open_id`。

两个配置会共享 keychain 中保存的 app secret。如果你只想移除 bot 配置，直接删掉它的目录即可；不要运行 `lark-cli config remove`，因为那会影响共享 secret。

## 安装

在一台还没有 `superagent` profile 的机器上，只需执行一次：

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
git clone <this repository> dsh-superagent && cd dsh-superagent
pnpm install && pnpm build
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

上面 pin 的版本是公开可用的；其他版本也可以，但必须满足 `package.json` 中 `engines.dsh` 的要求。

第四条命令会基于 DeepSeek Harness 的 Web 模板创建 `~/.dsh/profiles/superagent`。第五条命令会安装本 bundle。若想把默认的 `~/.dsh` 目录迁走，可以设置 `DSH_HOME`。

只有在“全新安装”时才运行创建 profile 的命令。如果 profile 已存在，再运行这条命令会直接替换它，并删除它的 `cordis.patch.yml`，连同其中的 machine 配置一起丢失。升级时绝不要这么做。

## 机器配置

把 [examples/profile.cordis.patch.yml](../examples/profile.cordis.patch.yml) 复制到 `~/.dsh/profiles/superagent/cordis.patch.yml`，直接替换该文件，并把所有 `<placeholder>` 填好。若接入飞书，还要把其中的 `dsh-superagent-feishu` 条目也启用。`webserver` 条目决定面板端口。

| Key | Required | 含义 |
| --- | --- | --- |
| `stateDir` | yes | 绝对路径：用于存放数据库、锁文件、`worker-bin` 下的命令 shim、智能体自己的 git 配置，以及默认的 `work` Ticket 目录。升级前应备份它 |
| `promptMaxChars`, `admissionContextMaxPages`, `admissionContextTimeoutMs` | no | 允许保存的最大 prompt 配置长度，以及一次 assessment 最多读取多少 Thread pages、最长读取多久 |

第一次启动后，可以在 **Settings → Deployment** 或 DeepSeek Harness Plugin Manager 中编辑 machine limits。修改会无须重启立即生效。`stateDir` 不能在那里更改。若值非法，或存在未识别的配置 key，启动会被阻止，日志中会指出具体 key。

## 启动与停止

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

`start` 会在 tmux session `dsh-superagent`（若 profile 改名，则为 `dsh-<profile>`）中运行 `dsh --profile superagent`。它会等待最多 1 分钟，直到 runtime 变为 ready，或者报告缺少 setup。首次运行时，它会直接打印缺的配置。它也会拒绝重复实例，包括那些不是由本脚本启动的实例。

`stop` 与 `restart` 会请求优雅关闭，并最多等待 5 分钟，让 Executors 自己收尾。它们不会强杀进程，也不会碰其他 tmux session。

`status` 会输出进程 id、健康状态和 setup problem。若使用了不同 profile 或端口，可通过 `--profile` 和 `--url` 指定，或设置 `DSH_SUPERAGENT_PROFILE` 与 `DSH_SUPERAGENT_URL`。

`status`、`start` 和 `restart` 的退出码含义如下：

| Code | Meaning |
| --- | --- |
| 0 | 服务运行正常 |
| 1 | 服务未运行、运行了两份、无响应，或 runtime 本身报错；重启可能能修好 |
| 2 | 服务在运行，但 runtime 以外的某个部分（如某个 Source、或某个 Policy sync `sync:<source id>`）报错；重启无效 |
| 3 | 服务在运行，但仍在等待 setup；重启无效 |

一个尚在启动中的 Source 仍算正常运行。飞书 bot 若由于 setup 无法检查，也同样算正常运行。

`status` 会在 `errors` 中返回失败的 parts 及其细节，在 `setup` 中返回尚未满足的 setup 要求。`start` 和 `restart` 会把这些问题直接打印成文本行；若在等待期内服务始终起不来，则退出 1。只有显式的 `stop` 才会暂停自动恢复。

每个 profile 的日志和控制文件都保存在 `.artifacts/` 下：

- 启动日志：`dsh-superagent-<profile>-<date>-<time>-<pid>.log`，其中 pid 属于脚本本身。只保留最新 20 份。
- 控制文件：`control/profiles/<profile>/`，其中包含 stop 标记、锁文件和进程 id。

打开日志里打印出来的 URL，然后在侧边栏中选择 **Superagent**。

### 每小时检查

应在你平时使用的 DeepSeek Harness profile 中创建“每小时检查”，而不是放在 `superagent` profile 本身里。它必须在智能体服务宕掉时依然继续运行。可以让那个 profile 里的智能体，用下面这段 prompt 创建一个 hourly schedule，并把 checkout 路径替换成实际值：

```text
Check dsh-superagent: run `scripts/dsh-superagent --profile superagent status` in <checkout>. Exit 0: do nothing and say nothing. Exit 2: do not restart; tell me each part under `errors` with its detail, unless you already told me about the same ones. Exit 3: do not restart; tell me what `setup` says setup needs, unless you already told me. If it reports manually_stopped: true, leave it stopped. Otherwise (exit 1) run `scripts/dsh-superagent --profile superagent --respect-stop restart`, which exits the same way: 0 means it recovered, so say nothing; 2 and 3 are reported as above; if it exits 1 or fails otherwise, read the newest .artifacts/dsh-superagent-superagent-*.log and the Troubleshooting section of docs/006-deployment_cn.md, fix what you can, and tell me what was wrong and what you did.
```

`--respect-stop` 会保留一次人工 stop 的效果。只有 `stop` 会写这个标记；`start` 会清掉它。`restart` 不会写这个标记，因此若是一次 restart 在关闭阶段超时，每小时检查仍可以继续尝试恢复它。

## Settings

只有 setup 成功后，polling 和实际工作才会开始。Overview 会列出缺失的要求，并链接到对应 setting。任何 settings 变更都会再次运行 setup。如果你是在应用外部修复了问题，比如重新登录某个 CLI，那么应点击 **Check again**。

1. **Sources。** 设置 GitHub 仓库（`owner/name`）和智能体 CLI，比如 `/home/me/.local/bin/gh-agent`。请求通常通过智能体账号的 notifications 到达。一个新的 Issue，或一条 mention 了智能体的新评论，会在下一轮 poll 中到达；如果你只是编辑已有 Issue 正文、在里面补上 mention，是收不到的。应当新发一条 comment。对于飞书，则需要设置 bot CLI、监听的 chats、每个 chat 是收“全部消息”还是“仅 mentions”，以及是否接收 direct messages。
2. **Policy。** 系统内置的两个示例规则可以编辑或删除。在 **On each Source** 下，给 GitHub accounts 和飞书用户添加允许名单；在名单里出现至少一个人之前，任何请求都不会成为 Ticket。对于飞书，也可以选择一个 chat，在启动时或点击 **Sync now** 时把其成员同步到允许名单。
3. **Execution。** 配置目录根和 prepare command。若想让每个 Ticket 都有自己的仓库 checkout，可以这样写：

```sh
git -C /home/me/src/widgets fetch --no-tags "https://github.com/$DSH_SUPERAGENT_GITHUB_REPOSITORY.git" "${DSH_SUPERAGENT_GITHUB_PR_HEAD:-HEAD}:refs/dsh-superagent/$DSH_SUPERAGENT_TICKET" &&
git -C /home/me/src/widgets worktree add -b "agent/$DSH_SUPERAGENT_NAME" "$DSH_SUPERAGENT_DIR" "refs/dsh-superagent/$DSH_SUPERAGENT_TICKET"
```

像你个人账号登录过的 CLI 这类命令，应放进 blocked commands。
4. **Prompts。** 设置团队区块：GitHub disclosure line、review bots、commit author，以及 auto-merge 条件。

### Management API

面板本身也是通过一组本地 HTTP API 在工作的，智能体也可以调用这些 API。`SETUP.md` 就是用它们来保存 settings 的，而服务脚本会用 `GET state` 读取状态。

请求应发送到 `http://127.0.0.1:<port>/orchestrator/api/`，并带上 `x-orchestrator-request: 1`。host 必须是 `127.0.0.1:<port>` 或 `localhost:<port>`。缺少这个 header，或 `Origin` 来自其他站点，都会收到 403。

通过 API 保存的权限与面板相同。它们可以修改 execution environment 和 setup account fields，而主 Session 工具触发的保存则不可以。

智能体可以依赖下面这些 route 和字段。`src/app/api.ts` 中的其他 routes 是面板自己的实现细节，可能随面板一起变化。

| Request | Answer |
| --- | --- |
| `GET state` | 各页面共同读取的状态：`setup`（其中包含 `problem` 和 `feishuProblem`，为 `null` 或一个 setup problem，带 `place`、`code`、`values` 与 `detail`；setup 成功后还会有 `accounts`）；`health`（按 part 分组，包含 `state`、`detail` 和 `at`）；`load`（Executor Sessions 的 `running`、Tickets 的 `queued` 和 `maxWorkers`）；`paused`；以及 `masterId` |
| `GET settings` | `blocks`：`policy`、`eval` 和 `executor`，每个都含 `value` 和 `revision`；`sources`：所有 Source instances，各自带 `id`、`kind`、`installed`、其 block 的 `value`（格式为 `{"enabled": …, "config": {…}}`）、`revision`、kind 对应的 form，以及 setup `problem`；还有 `policyVersion` 和 `environment`（执行环境的 `root` 与 `variables`） |
| `GET sources` | 各 Source 的状态：`state`（`missing`、`off`、`error`、`setup`、`starting`、`paused` 或 `ready`）、它各个 parts 的 `health`、setup `problem`、最近一天的 `stats`、一天内被 `filtered` 的 Triggers，以及一天内被 `dropped` 的 messages（最新在前），每条 dropped message 带 `channel`、`preview`、可选 `ticket`（`id` 与 `title`）、可选 `thread`，以及 `dropped.at` 与 `dropped.reason`（`channelGone` 或 `channelIgnored`） |
| `PUT settings/<block>` with `{"value": …, "revision": n}` | 保存一个 block：返回 `saved`、该 block 新的 `revision`、`policy_changed` 和 `restarting`。`value` 必须是整个 block 的完整值，格式与 `GET settings` 一致；缺失的 key 会回落到默认值 |
| `POST executor/pause` with `{"paused": true}` or `false` | 暂停或恢复 processing：返回 `paused` |
| `POST setup/check` | 重新运行 setup，也就是重启 `dsh-superagent`：返回 `restarting: true`。若当前 setup 并无可修复问题，则会被拒绝 |
| `POST policy-sync/<source id>` | 立即同步该 Source 的允许名单：返回 `policy`，即该 Source 的策略部分保存后的值 |
| `GET prompts`, `PUT prompts` with `{"config", "baseRevision"}` | 读取 prompt 配置：返回 `current`（含 `revision` 和 `config`）、`defaults`、`sections`（team sections 的顺序）、`skillName`、保留的内建 skill 名称 `builtin`，以及 `skillPrefix`。或保存新的 prompt 配置：返回新 `revision` |
| `POST tickets/<id>/retry`, `POST tickets/<id>/stop` | 与面板一致，对指定 Ticket 进行 retry 或 stop |

health entries 按职责分组：

| Entry | Reports |
| --- | --- |
| `runtime` | 应用本身 |
| `models` | 正在等待模型预算重试的 Sessions |
| `evals` | 最近一小时内 unavailable 且后来没有 judgement 或 dismissal 的 Evals |
| `assessment`, `launch`, `delivery`, `relay`, `notice`, `reply` | [003 执行](003-execution_cn.md) 中描述的 stored work；其中 delivery 统计等待中的工作，reply 覆盖 Channel 消息 |
| `worker` | Executor Session 的事件处理；只有首次失败后才出现 |
| `github`, `feishu`, `history` | Source 轮询、事件和历史恢复 |
| `sync:<source id>` | 允许名单同步 |

一次拒绝会带 HTTP status 以及 `error` 文本：

- 400：值非法（带 `issues`，每项含 `path`）、账号检查失败（带 `setup`）、固定拒绝（带 `code` 和 `values`），或 body 不是 JSON / 过大。
- 404：目标不存在。
- 409：revision 过期。
- 500：其他任何错误。

几个 block 的结构如下：

- `source:github`, `source:feishu`：格式为 `{"enabled": true, "config": {…}}`。GitHub 的 config 包括 `repository`、`cli`，以及 `observeIssues`、`observePulls`、`observeComments` 和 `observeCi` 开关，默认关闭；它们分别用于接收仓库中每个新 Issue、新 PR、新 comment，以及默认分支 CI。Feishu 的 config 包括 `cli`、`chats`（格式 `[{"chatId": "oc_…", "name": "…", "listen": "mentions"}]`，也可为 `"all"`）、`directMessages`，以及 `otherChats`（`"none"` 或 `"mentions"`）。
- `policy`：格式为 `{"rules": […], "sources": {"github": {"allowed": [{"id": "<login>"}], "bots": [{"id": "dependabot[bot]"}]}, "feishu": {"allowed": [{"id": "ou_…", "name": "…"}], "syncChat": "oc_…", "admission": {"oc_…": {"unlistedAddressed": "explicit"}}}}}`。`bots` 列出允许成为 inferred candidate 的 bot。`admission` 的 key 对飞书可为 chat id、`other` 或 `direct`，对 GitHub 可为 `issues`、`pulls`、`comments` 或 `ci`；未设置的 cell 会使用默认值（见 [002 授权](002-authorization_cn.md)）。
- `eval`：包含 `mode`、`timeoutSeconds` 和 `model`。
- `executor`：包含 `model`、`maxWorkers`、`directory`、`prepare` 和 `blockedCommands`。processing pause 本身不在这个 block 里，而是单独的 `paused` 设置，由 `POST executor/pause` 修改。

修改 repository 或 CLI 会先触发相应 setup check。若检查失败，会返回 400，并在 `setup` 中说明细节。若账号修改成功，或当前 setup 本来就处于等待状态，则保存会返回 `restarting: true`。重启期间 API 会暂时不可用。

```sh
api=http://127.0.0.1:3082/orchestrator/api
rev=$(curl -s -H 'x-orchestrator-request: 1' $api/settings | jq '.sources[] | select(.id == "github") | .revision')
curl -s -X PUT -H 'x-orchestrator-request: 1' -H 'content-type: application/json' $api/settings/source:github \
  -d "{\"value\": {\"enabled\": true, \"config\": {\"repository\": \"acme/widgets\", \"cli\": \"$HOME/.local/bin/gh-agent\"}}, \"revision\": $rev}"
```

## Setup 检查

setup 使用的是 DeepSeek Harness 的 subprocess service，与 worker shell 相同。它会移除所有名字中包含 `KEY`、`PASSWORD`、`SECRET` 或 `TOKEN` 的环境变量。

setup 会验证 GitHub repository 与 agent 的 push 权限，写入 worker environment，并检查 git 是否只会在 `github.com` 上使用由 `dsh-superagent` 提供的凭据配置。飞书 setup 失败只会禁用 Feishu Source，不会影响其余部分。

setup 所写入的进程环境包括：

- **位于 `<stateDir>/worker-bin` 中的 command shims**，并被放在 `PATH` 最前面：`gh` 会调用智能体 CLI，`lark-cli` 会调用 bot CLI，被 blocked 的命令则直接失败。shim 在执行目标程序前，会先把自己的目录从 `PATH` 中移除。
- **位于 `<stateDir>/worker-gitconfig` 中的 git 配置**：它会覆盖操作员的 global 和 system git 配置。它会把智能体 CLI 设置为 `github.com` 的 credential helper，把 GitHub 的 SSH 地址改写为 HTTPS，并定义 Git LFS filters。个人 git 设置，比如 HTTP 代理，不会生效。这里不会设置 commit identity；git 永远不会提示输入凭据；同时还会移除 `GH_TOKEN` 和 `GITHUB_TOKEN`。

如果 `worker-bin` 或 `worker-gitconfig` 中存在不是 `dsh-superagent` 自己写入的文件，setup 会被阻止。若某个 Ticket 目录本身就位于一个其 git 配置中额外携带凭据的仓库之内，该 Ticket 也会失败。

## 试运行

使用一个被允许的 GitHub 账号，创建 Issue 或发一条包含 `@<agent login> <request>` 的 comment。通常在一个 polling interval（默认 1 分钟）内，Overview 中就应出现一个 Ticket。智能体会确认请求，在自己的 Session 和目录中完成工作，并把结果投递回原始对话。对于飞书，则是在已配置的群里 mention bot。

如果你已经看到 GitHub notification、trigger 和 thread，但一直没有 Ticket，而是稍后在 Overview 的 problems 中出现 `eval-unavailable` / `timeout`，优先检查主 Session 的模型凭据。常见表现是主 Session 每次收到 candidate 后都会起一个 turn，但马上因缺少 provider 凭据而失败，例如：

```text
llm-deepseek: no API key for provider route "deepseek-official"
```

这种情况下，GitHub Source 本身通常是正常的，卡点在“candidate 送达主 Session后，主 Session 无法调用模型完成 `orchestrator_route`”。应先在当前 DSH profile 的 Models 页面写入对应 provider 的凭据，或在启动环境中提供相应环境变量，然后再重新验证。

## 升级

1. 等待，直到没有任何 Executor 正在 turn 中途。
2. 备份 `<stateDir>/orchestrator.sqlite`。
3. 执行 `git pull && pnpm install && pnpm build`。
4. 执行 `scripts/dsh-superagent restart`，并检查状态。

一次 restart 会结束正在运行的 Assignments，并带 recovery note 启动替代者。团队文本和被覆盖的 prompt parts 会被保留；那些没有 override 的 parts 会使用新默认值。升级后，检查那些被标记为 **Default updated** 的 parts。

被新版本迁移过的数据库，旧版本无法再打开。因此回滚必须依赖升级前备份。若你升级了 DeepSeek Harness 本身，也需要重启当前运行的实例，这样面板才能加载匹配的新 artifacts。

### 智能体 preset

worker 与 router preset 都会在 shell 命令运行 5 秒后，把它转入后台 job。Session 若找不到自己的 preset，将无法加载。因此，只要还有 Sessions 在使用，就必须保留 `cordis.patch.yml` 里的 `dsh-superagent-worker` 与 `dsh-superagent-router` 条目；否则恢复时会报 `agent-preset/not-found`。

## 排障

setup problem 会显示在 overview 中，并附带修复链接；面板会用当前语言展示，日志和 API 则会显示下面这些英文文本。

| 症状 | 原因与修复 |
| --- | --- |
| `The agent GitHub CLI is not set.` | 在 GitHub Source 卡片中设置智能体 CLI |
| `The agent GitHub CLI was not found.` | 使用 wrapper 的绝对路径，或者一个在 `PATH` 中可找到的命令名 |
| `The agent GitHub CLI cannot read its account` | 对该 wrapper 执行 `auth login` |
| `@… cannot read …` 或 `@… cannot push to …` | 给智能体账号该仓库的写权限 |
| `The GitHub Source names no repository.` | 在 GitHub 卡片里填写 `owner/name` 格式的 repository |
| 某个 Ticket 失败并显示 `The prepare command failed: …` | 输出中会直接说明原因，例如 clone 路径写错；修正命令后 retry 该 Ticket |
| `git does not read the github.com credential configuration dsh-superagent writes` | 安装 git 2.32 或更新版本 |
| 某个 Ticket 失败并显示 `The git configuration of the repository in … sets …` | 从该 clone 的 `.git/config` 中移除指定 credential 或 `http.*.extraheader` 设置 |
| `… was not written by dsh-superagent` | 把对应文件从 `<stateDir>/worker-bin` 或 `<stateDir>/worker-gitconfig` 中移走，或选择新的 `stateDir` |
| `The bot lark-cli allows user identities` | 让 Feishu Source 指向一个 bot-only 的配置，参见上文飞书 bot 配置方式 |
| `The bot lark-cli does not work.` 或 `The bot identity of app … is …` | 检查该配置的 app id 与 secret |
| `The Feishu bot is not a member of the chat.` | 在群设置中把 bot 加进群；下次重试会成功 |
| `status` 退出 2 | 某个 part 报错，重启无效：`errors` 会列出每个 part 及其 detail；每个 part 一旦再次成功，就会自行清掉对应错误 |
| `status` 退出 3 | setup 仍在等待：`setup` 会说明缺什么，overview 也会链接到能修复它的 setting |
| `status` 退出 1 | 服务未运行、重复运行、无响应，或 runtime 本身报错；输出和最新日志会说明是哪种情况 |
| `… timed out after 90 s`、`… was stopped by …`、`… exited with code …` 或 `… could not run: …` | 某个 CLI 执行失败，但自己没打印原因；应以操作员用户手工执行该程序，以查看它自己的报错 |
| 面板显示 `dsh-superagent is not answering` | 它正在重启，或者由于配置问题根本没启动起来；DeepSeek Harness 日志里会说明原因 |
| 日志里出现 `dsh-superagent configuration has keys it does not use` | 移除被点名的 keys；日志会同时说明这些值现在该放在哪里 |
| 日志提示插件与当前运行的 dsh 不兼容 | 安装一个满足 `engines.dsh` 范围的 DeepSeek Harness 版本 |
| 升级 DeepSeek Harness 后，面板显示 `Failed to load plugins` | 重启当前运行实例 |
| mention 了智能体却毫无反应 | 检查发送者是否在该 Source 的 allowed list 中、Source 是否已开启、processing 是否未被 paused。对于 GitHub，“编辑已有 Issue 正文后补 mention”不会被接收到；应新发一条 comment |
| GitHub mention 已生成 trigger / thread，但最后变成 `eval-unavailable` / `Assessment timed out`，没有 ticket | 先检查主 Session 是否具备模型凭据。若 orchestrator Session 的 turn end 原因类似 `llm-deepseek: no API key for provider route "deepseek-official"`，说明 Source 已正常送达 candidate，但主 Session 无法调用模型完成路由。应在当前 DSH profile 的 Models 页面或启动环境中补齐对应 provider 的 API key，然后重试 |
