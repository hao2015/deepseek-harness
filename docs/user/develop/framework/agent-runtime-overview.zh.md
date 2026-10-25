# Agent 运行时概览

[English](agent-runtime-overview.md) | 中文

本页汇总 DeepSeek Harness 的运行时骨架、通信路径和部署边界，作为前面几类常见问题的中文速查页。

## 运行时骨架

DeepSeek Harness 建立在 vendored Cordis 之上。它的运行时不是单体核心，而是一棵通过配置装配的插件树：服务、工具、持久化、模型适配器和默认 agent driver 都是插件。核心执行面是 `Agent` 接口和默认 `agent-loop` 实现；`web` 与 `headless` 只是叠在同一条主干之上的不同产品组合，而不是两套不同 runtime。

参见 [architecture.md](../../../architecture.zh.md)、[core.md](../../../subsystems/core.zh.md) 和 [packages/core/agent/README.md](../../../../packages/core/agent/README.zh.md)。

## Sandbox 与执行环境

默认的基础 bundle 会挂载本地 sandbox provider 和 sandbox-policy service。默认策略从 `workspace-write` 起步，受限子进程执行表达的是“文件效果”边界，而不是完整的容器安全边界。内置 sandbox seam 只处理与宿主共享文件系统和内核的限制：Linux 使用 Bubblewrap 或 Landlock，macOS 使用 Seatbelt，Windows 使用 ACL restricted-token runner。容器、microVM 和远程执行器都不是 `ctx.sandbox` 的后端；如果部署需要这些能力，应替换相关 capability provider，而不是把它们塞进默认 sandbox seam。

参见 [sandbox.md](../../../subsystems/sandbox.zh.md)、[packages/sandbox/sandbox/README.zh.md](../../../../packages/sandbox/sandbox/README.zh.md) 和 [packages/bundle/base/cordis.patch.yml](../../../../packages/bundle/base/cordis.patch.yml)。

## Agent 之间如何通信

Agent 之间的委派走 subagent seam，而不是走 driver 内部的隐式旁路通道。父 agent 通过 `ctx.subagents` 启动或恢复子 agent；可继续执行的子 agent 后续消息统一进入子 agent 自己的 inbox FIFO。子到父的显式回报使用 report channel；中断、列举等操作则通过可选 control tools 暴露。权限判断依赖 durable parent 关系和 live ancestry，而不是仅看消息归因。

参见 [subagent.md](../../../subsystems/subagent.zh.md) 和 [packages/subagent/README.md](../../../../packages/subagent/README.zh.md)。

## 控制台或 UI 如何与 Agent 通信

这里要分控制面和观察面。对于本地或同进程客户端，UI 插件会直接调用 `agent.followup()`、`agent.steer()` 或 `agent.inject()` 把输入送回 agent。对于 Web 或远端客户端，请求会先经由 Connection 和 API gateway 的共享 `/api` 路由，再在 Host 侧解析到目标 session 或 live agent。

渲染面并不依赖一个同步 RPC 一次性返回完整答案。持久的对话事实通过 `session/event` 到达；队列、标题等派生状态则通过 host push frame 或 session projection 提供给客户端。

参见 [extension-cookbook.md](../../../cookbook/extension-cookbook.zh.md)、[api-gateway.md](../../../api-gateway.zh.md)、[session-projection.md](../../../subsystems/session-projection.zh.md) 和 [packages/host/apiproxy/README.md](../../../../packages/host/apiproxy/README.md)。

## 能否跑在 Kubernetes 或分布式环境上

这个 harness 可以放进容器里，也可以作为单实例服务部署在 Kubernetes 上。这也是当前架构最自然的落点，尤其是在一个进程拥有 live agent registry，以及本地持久化或挂载存储的情况下。

仓库当前并没有现成的多节点控制平面，让多个副本透明地共享同一个 live runtime。live agent 是进程本地的，默认持久化层和 sandbox 层也更偏向单宿主进程，而不是集群级协调。

如果要做分布式执行，当前更现实的路径是替换 capability provider，而不是直接对默认 runtime 做横向扩容。现成例子包括 E2B 这样的远程 subprocess provider，以及进程外 subagent provider。如果要把它做成真正的分布式部署，还需要额外补齐 session affinity、共享持久化、故障切换规则和远程执行 provider。

参见 [packages/core/agent/README.md](../../../../packages/core/agent/README.zh.md)、[packages/session/session-persistence-sqlite/README.md](../../../../packages/session/session-persistence-sqlite/README.md)、[packages/e2b/subprocess-e2b/README.md](../../../../packages/e2b/subprocess-e2b/README.md) 和 [packages/shell/shell/README.md](../../../../packages/shell/shell/README.zh.md)。