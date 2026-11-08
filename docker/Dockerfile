# syntax=docker/dockerfile:1.7

# 这个镜像面向 dsh-superagent 的 Linux 运行环境：
# - 以 Node 24 为基础，满足 package.json 的 engines.node 要求
# - 预装构建和运行阶段最常见的系统工具，如 git / python3 / tmux
# - 预装项目文档中固定版本的 DSH CLI，便于后续在容器内继续做 profile 和 plugin 配置
FROM node:24-bookworm-slim

# 避免 apt 在构建中弹出交互式提示。
ENV DEBIAN_FRONTEND=noninteractive

# 安装这个仓库最常见的基础依赖：
# - bash: 方便在容器里交互调试
# - git / openssh-client: 代码拉取、worktree 和常见仓库操作
# - python3: 项目自带服务脚本和周边工具会用到
# - tmux / procps: dsh-superagent 的服务脚本依赖它们管理和探测进程
# - curl / jq / ca-certificates: 常见 API 调试和 HTTPS 请求工具
# - less / xz-utils: 方便查看日志和处理常见压缩包
# - gnupg: 用于添加 GitHub CLI 官方 APT 仓库签名
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
    bash \
    ca-certificates \
    curl \
    git \
    gnupg \
    jq \
    less \
    openssh-client \
    procps \
    python3 \
    tmux \
    xz-utils \
 && rm -rf /var/lib/apt/lists/*

# Debian 自带的 gh 版本偏老，不满足文档要求的 `gh >= 2.48`，
# 而且会导致 `gh api --slurp` 不可用。这里改用 GitHub 官方仓库安装新版 CLI。
RUN mkdir -p /etc/apt/keyrings \
 && curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg -o /etc/apt/keyrings/githubcli-archive-keyring.gpg \
 && chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" > /etc/apt/sources.list.d/github-cli.list \
 && apt-get update \
 && apt-get install -y --no-install-recommends gh \
 && rm -rf /var/lib/apt/lists/*

# 启用并固定 pnpm 版本，使其与仓库 packageManager 声明保持一致。
RUN corepack enable \
 && corepack prepare pnpm@11.7.0 --activate

# 预装文档中使用的 DSH CLI 版本。
# 这里用全局安装，是为了让容器启动后可以直接执行 dsh 相关命令。
RUN npm install -g @deepseek-ai/dsh@0.2.1-rc.3

# 使用非 root 用户运行，减少后续在挂载目录里生成 root 权限文件的概率。
RUN useradd --create-home --shell /bin/bash app

USER app

# 约定好 DSH 与 pnpm 的家目录，便于后续挂载卷做持久化。
ENV HOME=/home/app \
    DSH_HOME=/home/app/.dsh \
    PNPM_HOME=/home/app/.local/share/pnpm \
    PATH=/home/app/.local/share/pnpm:/home/app/.local/bin:$PATH

# 让镜像默认工作在仓库根目录，便于继续执行 pnpm build / dsh plugin 等命令。
WORKDIR /workspace/dsh-superagent

# 直接复制整个仓库并在构建阶段完成依赖安装与构建，
# 这样镜像 build 成功就意味着当前代码至少可以在 Linux 用户态完成基础构建。
COPY --chown=app:app . /workspace/dsh-superagent

# 使用锁文件做可重复安装，并在镜像构建时先完成一次 build。
# 这里不自动执行完整运行配置，因为 GitHub / Feishu 账号、DSH profile 和 stateDir
# 都依赖你自己的环境与凭据，不适合硬编码在基础镜像里。
RUN pnpm install --frozen-lockfile \
 && pnpm build

# 默认进入交互 shell，便于后续手动继续执行 dsh 配置、登录和启动命令。
CMD ["bash"]
