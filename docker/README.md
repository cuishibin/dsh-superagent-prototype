# Docker 使用说明

这个目录提供的是一个 **dsh-superagent 的基础 Linux 运行镜像**，适合在 Docker 里继续完成：

- `pnpm` 依赖安装与构建
- `dsh` profile 初始化
- plugin 挂载
- 后续的 GitHub / Feishu CLI 登录与配置

当前镜像 **不是开箱即用的完整生产镜像**。原因是下面这些信息都依赖你的实际环境，不适合写死在镜像里：

- GitHub agent 账号登录态
- 主 Session / worker 使用的模型凭据（例如 `DEEPSEEK_API_KEY`）
- 可选的飞书 bot 登录态
- `~/.dsh` 里的 profile 与 state
- 具体的 `stateDir`
- 目标仓库与 Policy / Execution / Prompts 设置

## 1. 构建镜像

在仓库根目录执行：

```bash
docker build -f docker/Dockerfile -t dsh-superagent:dev .
```

如果你之前已经构建过旧镜像，建议重建一次，确保容器里的 GitHub CLI 升级到文档要求的版本：

```bash
docker compose -f docker/docker-compose.yml build --no-cache
```

如果你更希望把端口、卷和工作目录固定下来，也可以直接使用本目录下的 `docker-compose.yml`：

```bash
docker compose -f docker/docker-compose.yml build
```

## 2. 运行容器

Linux / macOS：

```bash
docker run --rm -it \
  -p 3082:3082 \
  -v "$(pwd)/docker/data/dsh-home:/home/app/.dsh" \
  -v "$(pwd)/docker/data/gh-agent:/home/app/.config/gh-agent" \
  -v "$(pwd)/docker/data/local-bin:/home/app/.local/bin" \
  -v "$(pwd)/docker/data/lark-bot:/home/app/.config/lark-bot" \
  -v "$(pwd)":/workspace/dsh-superagent \
  dsh-superagent:dev
```

PowerShell：

```powershell
docker run --rm -it `
  -p 3082:3082 `
  -v "${PWD}/docker/data/dsh-home:/home/app/.dsh" `
  -v "${PWD}/docker/data/gh-agent:/home/app/.config/gh-agent" `
  -v "${PWD}/docker/data/local-bin:/home/app/.local/bin" `
  -v "${PWD}/docker/data/lark-bot:/home/app/.config/lark-bot" `
  -v "${PWD}:/workspace/dsh-superagent" `
  dsh-superagent:dev
```

说明：

- `docker/data/dsh-home` 目录用于持久化 `~/.dsh`，可以直接在宿主机查看和编辑
- `docker/data/gh-agent` 目录用于持久化 GitHub CLI 的 `gh-agent` 登录态，避免容器重建后重新登录
- `docker/data/local-bin` 目录用于持久化 `gh-agent` 这类本地 wrapper 脚本
- `docker/data/lark-bot` 目录用于持久化飞书 bot 的登录态；暂时不用飞书时可以保留为空目录
- `docker/data/model.env` 目录文件用于持久化本地 Docker 运行时的模型 endpoint 与 API key，默认由 `docker-compose.yml` 注入到 `dsh-superagent` 容器
- 仓库目录挂载到 `/workspace/dsh-superagent`，方便你在宿主修改代码后，容器里直接继续构建和调试
- `3082` 是文档里默认使用的面板端口；如果后面改了 profile 配置里的端口，这里也要一起调整

如果使用 Compose，可直接启动：

```bash
docker compose -f docker/docker-compose.yml run --rm dsh-superagent
```

或者先在后台启动容器，再进入它：

```bash
docker compose -f docker/docker-compose.yml up -d
docker compose -f docker/docker-compose.yml exec dsh-superagent bash
```

## 3. 容器内建议执行顺序

容器启动后，进入仓库根目录，再继续执行：

```bash
pnpm install
pnpm build
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

然后再按照文档继续：

1. 配置 `~/.dsh/profiles/superagent/cordis.patch.yml`
2. 准备 GitHub agent 账号与 `gh-agent`
3. 配置主 Session / worker 可用的模型凭据
4. 可选：准备飞书 bot 与 `lark-bot`
5. 配置 Source / Policy / Execution / Prompts
6. 启动服务：

```bash
scripts/dsh-superagent start
scripts/dsh-superagent status
```

### 模型凭据

`dsh-superagent` 的 GitHub Source 能把 mention 读进来，但真正把 candidate 路由成 ticket，还需要主 Session 能正常调用模型。

如果没有给当前 DSH profile 配置模型凭据，运行时通常会出现类似报错：

```text
llm-deepseek: no API key for provider route "deepseek-official"
```

这时你会看到：

- GitHub notification 已经到达
- trigger / thread 已经入库
- 但最后落成的是 `unavailable(timeout)`，而不是 ticket

处理方式二选一：

1. 在 DSH 的 Web Models 页面为当前 profile 写入对应 provider 的凭据
2. 在启动容器前，通过环境变量注入，例如：

```powershell
$env:DEEPSEEK_API_KEY = "your-key"
docker compose -f docker/docker-compose.yml up -d
```

如果用 Compose 长期运行，建议把这类模型凭据放进你自己的环境管理方式中，不要写死到镜像或仓库文件里。

当前仓库的 Docker Compose 已经预留了一个本地未跟踪文件：

```text
docker/data/model.env
```

对于 OpenAI 兼容接口，可按下面方式填写：

```dotenv
# OpenAI 客户端会自行拼接 /chat/completions，所以 base URL 应写到 /v1 为止
OPENAI_BASE_URL=http://host:port/v1
OPENAI_API_KEY=your-key
```

## 4. 这个镜像里已经包含什么

- `node:24-bookworm-slim`
- `pnpm 11.7.0`
- `@deepseek-ai/dsh@0.2.1-rc.3`
- `gh`（通过 GitHub 官方仓库安装，满足文档要求的 `2.48+`）
- `git`
- `python3`
- `tmux`
- `curl`
- `jq`
- `procps`

## 5. 还需要你自己补什么

- 目标仓库访问凭据
- 模型提供方凭据（例如 `DEEPSEEK_API_KEY`）
- `stateDir` 等部署配置

这个目录现在已经包含 `docker-compose.yml`，并且默认持久化了 `~/.dsh`、GitHub `gh-agent` 登录态、wrapper 脚本目录，以及可选的飞书 bot 配置目录。

## 创建 GitHub agent 登录态 wrapper
在容器里执行：
```bash
mkdir -p /home/app/.local/bin
mkdir -p /home/app/.config/gh-agent
cat > /home/app/.local/bin/gh-agent <<'EOF'
#!/bin/sh
GH_CONFIG_DIR="$HOME/.config/gh-agent" exec gh "$@"
EOF
chmod +x /home/app/.local/bin/gh-agent
```


```bash
/home/app/.local/bin/gh-agent auth login
/home/app/.local/bin/gh-agent api user --jq .login
```

登录完成后，GitHub token 通常会保存在：

```bash
/home/app/.config/gh-agent/hosts.yml
```

在当前 Docker 挂载方式下，它对应宿主机目录：

```bash
docker/data/gh-agent
```
