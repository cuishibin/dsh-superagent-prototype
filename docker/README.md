# Docker 部署须知

> 本仓库当前不包含 Dockerfile 或 compose 文件。下面的说明与部署方式无关，适用于任何方式（包括在容器中）运行 dsh-superagent 时的检查步骤。

## OAuth 登录完成后如何验证 GitHub Source 已 ready

dsh-superagent 的 GitHub Source 依赖 agent GitHub 账号的 CLI（配置方法见 [docs/006-deployment.md](../docs/006-deployment.md) 的「Agent GitHub account」一节）。用该 wrapper 完成 `gh-agent auth login`（OAuth 登录）后，按以下步骤确认 GitHub Source 已进入 ready 状态：

1. **触发重新检查。** OAuth 登录发生在应用之外，面板不会自动感知。若 Overview 仍在提示设置未完成，在 **Settings → Deployment** 页点击 **Check again**（对应管理 API 的 `POST setup/check`）重新运行 setup。

2. **在面板查看。** 打开 Superagent 面板的 **Source** 页，GitHub Source 的状态应显示为 `ready`。`starting` 也属正常（仍在启动中）；`error`、`setup`、`missing` 分别表示轮询出错、仍在等待设置、尚未配置。若提示 `The agent GitHub CLI cannot read its account`，说明登录的不是 agent 账号，需要用 wrapper 重新 `auth login`。

3. **通过管理 API 查看。**

   ```sh
   curl -s -H 'x-orchestrator-request: 1' http://127.0.0.1:<port>/orchestrator/api/sources \
     | jq '.[] | select(.kind == "github") | .state'
   ```

   输出 `"ready"` 即已就绪。请求必须发往 `127.0.0.1` 或 `localhost` 并携带 `x-orchestrator-request: 1` 头。

4. **用脚本检查。** 在检出目录运行 `scripts/dsh-superagent status`：退出码 0 表示各部分正常（Source 仍在启动也算正常）；退出码 2 表示有部分（如 `github`）报错，输出中 `errors` 会列出出错部分及详情；退出码 3 表示仍在等待设置，需要回到第 1 步。

5. **端到端验证。** 用允许列表（Policy → On each Source）中的 GitHub 账号，在新 Issue 或评论里 `@<agent login>` 发起一条请求；一个轮询周期（默认 1 分钟）内 Overview 应出现对应 Ticket。注意：通过编辑已有 Issue 正文添加的 @ 提及不会被接收，请发新评论。
