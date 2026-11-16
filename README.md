# dsh-superagent

> **Research prototype for reference only, not an official DeepSeek product.** This code grew out of our team's day-to-day development work, with much of the engineering done by coding agents. Treat it as research code, not production-ready software. It may still contain bugs, and its coding style and engineering practices may be inconsistent. Review and adapt it before using it in your own environment.
>
> We have adapted it for publication, so it is not identical to our internal version or the implementation described in *Exploring Super-agent Design: Towards a More Autonomous Agent System*. Some designs discussed in the article are directions for future development and are not implemented in this prototype.

dsh-superagent is a proactive coding agent for one GitHub repository, packaged as a bundle for [DeepSeek Harness](https://www.deepseek.com/en/harness/). It listens to GitHub and, optionally, Feishu. When someone allowed asks the agent for work, it opens a work item, works on it in its own agent session and directory, and answers in the original conversation, for example with a pull request. A web panel shows what needs a person.

It is also a way of working with an agent: **take as little human attention as possible, and give as much value as possible.** An agent that can act for a whole team can also waste a whole team's attention, so it acts only when asked or when a rule the team wrote allows it, and speaks only when someone must act or can use a result. [docs/000-philosophy.md](docs/000-philosophy.md) explains the principle and what it asks of the operator.

## Setup

Setup has many steps, so an agent does it with you. Ask your coding agent:

> Read SETUP.md in this repository and set up dsh-superagent with me.

It asks one short question at a time, shows the full plan once, and then installs and configures everything. [SETUP.md](SETUP.md) is what it follows.

## Documentation

Read the numbered documents in order; each builds only on the ones before it.

- [000 Philosophy](docs/000-philosophy.md)
- [001 Domain model](docs/001-domain.md)
- [002 Authorization](docs/002-authorization.md)
- [003 Execution](docs/003-execution.md)
- [004 Prompts](docs/004-prompts.md)
- [005 Panel](docs/005-panel.md)
- [006 Deployment](docs/006-deployment.md): manual setup, configuration, upgrades and troubleshooting
- [007 Architecture](docs/007-architecture.md)
- [008 Cookbook](docs/008-cookbook.md): adding a Source

## Development

`pnpm check` runs the typecheck, the tests and the build.

## 常见问题（FAQ）

**这是什么项目？**

dsh-superagent 是一个面向单个 GitHub 仓库的主动式编码 agent，以 DeepSeek Harness bundle 的形式运行。它监听 GitHub（以及可选的 Feishu），在获得授权后自动开工、在自己的会话和目录里完成工作，并回到原始对话中交付结果（例如一个 pull request）。详见上文“Documentation”一节。

**如何安装和配置？**

不需要手动执行繁琐步骤。请让你的编码 agent 阅读 [SETUP.md](SETUP.md) 并和你一起完成安装：它会一次只问一个简短的问题，展示完整计划后再执行。

**它什么时候会自己行动？**

只有两种情况：有人明确要求它做事，或团队编写的规则允许它行动。它只在有人必须处理或可以使用结果时才发声，以尽量少占用人的注意力。授权机制的细节见 [docs/002-authorization.md](docs/002-authorization.md)。

**支持哪些消息来源？**

目前支持 GitHub（Issue、PR 及其评论等）和 Feishu。如何新增一个 Source，见 [docs/008-cookbook.md](docs/008-cookbook.md)。

**部署、升级或遇到故障怎么办？**

参见 [docs/006-deployment.md](docs/006-deployment.md)，其中包含手动部署、配置、升级与故障排查的说明。

**这是官方产品吗？**

不是。这是一个供参考的研究原型，并非 DeepSeek 的官方产品，可能包含 bug，请先审查并适配后再用于你自己的环境。

## License

[MIT](LICENSE)
