# Agent runtime overview

English | [中文](agent-runtime-overview.zh.md)

This page summarizes the runtime, communication paths, and deployment boundaries behind DeepSeek Harness.

## Runtime

DeepSeek Harness runs on vendored Cordis. The runtime is a plugin tree rather than a privileged monolith: services, tools, persistence, model adapters, and the default agent driver are all composed through Cordis configuration. The core execution path is the `Agent` interface plus the default `agent-loop` implementation, while `web` and `headless` profiles assemble different product surfaces over that same spine.

See [architecture.md](../../../architecture.md), [core.md](../../../subsystems/core.md), and [packages/core/agent/README.md](../../../../packages/core/agent/README.md).

## Sandbox and execution world

The shipped base bundle mounts a local sandbox provider and a sandbox-policy service by default. That default policy starts from `workspace-write`, and confined subprocess execution is expressed as a file-effect policy, not a full container boundary. The built-in sandbox seam is same-world confinement only: Linux uses Bubblewrap or Landlock, macOS uses Seatbelt, and Windows uses the ACL restricted-token runner. Containers, microVMs, and remote executors are not backends of `ctx.sandbox`; a deployment that needs those replaces the relevant capability providers instead.

See [sandbox.md](../../../subsystems/sandbox.md), [packages/sandbox/sandbox/README.md](../../../../packages/sandbox/sandbox/README.md), and [packages/bundle/base/cordis.patch.yml](../../../../packages/bundle/base/cordis.patch.yml).

## Agent-to-agent communication

Agent-to-agent work goes through the subagent seam, not through an implicit side channel inside the driver. The parent agent starts or resumes a child through `ctx.subagents`, and continuable child messages are routed through the child agent's inbox FIFO. Child-to-parent reporting uses the explicit report channel, while interrupt and catalog-style operations are exposed as optional control tools. Authority is checked from durable parentage and live ancestry rather than from message attribution alone.

See [subagent.md](../../../subsystems/subagent.md) and [packages/subagent/README.md](../../../../packages/subagent/README.md).

## Console and UI communication with an agent

The control plane and observation plane are separate. For local or in-process clients, a UI plugin sends work directly through `agent.followup()`, `agent.steer()`, or `agent.inject()`. For web or remote clients, the request first crosses the shared `/api` route through the Connection and API gateway layers, then resolves to the target session or live agent on the host side.

Rendering does not depend on a synchronous request returning a whole answer. Durable conversation facts arrive through `session/event`, and host push frames or session projections carry derived state such as queue, title, or other current per-session values.

See [extension-cookbook.md](../../../cookbook/extension-cookbook.md), [api-gateway.md](../../../api-gateway.md), [session-projection.md](../../../subsystems/session-projection.md), and [packages/host/apiproxy/README.md](../../../../packages/host/apiproxy/README.md).

## Kubernetes and distributed deployment

The harness can run inside a container and can be deployed on Kubernetes as a single service instance. That is the most natural fit for the current architecture, especially when one process owns the live agent registry and its local persistence or mounted storage.

The project does not ship a ready-made multi-node control plane where several replicas transparently share one live runtime. Live agents are process-local, and the default persistence and sandbox layers are geared toward a single host process rather than cluster-wide coordination.

Distributed execution is possible through capability replacement rather than through horizontal scaling of the default runtime. Examples include remote subprocess providers such as E2B and out-of-process subagent providers. To build a truly distributed deployment, an operator still needs session affinity, shared durable storage, failover rules, and remote execution providers.

See [packages/core/agent/README.md](../../../../packages/core/agent/README.md), [packages/session/session-persistence-sqlite/README.md](../../../../packages/session/session-persistence-sqlite/README.md), [packages/e2b/subprocess-e2b/README.md](../../../../packages/e2b/subprocess-e2b/README.md), and [packages/shell/shell/README.md](../../../../packages/shell/shell/README.md).