# 内置 OMP Agent

Status: draft
Translation: current

[English](builtin-omp.md)

## 场景

已经安装 Oh My Pi（OMP）的用户可以在 Lody 内置 Provider 列表中选择 OMP，通过 Lody 新建和
恢复会话、使用 Lody 交互界面，并查看模型归属正确的 token 与费用。用户不需要填写 custom
ACP command，Lody 也不启动 OMP 自带的 `omp acp` 服务。

## 职责

Lody 随包提供 Apache-2.0 的 `acp-extension-omp` adapter。Lody 通过 ACP 和 provider-neutral
的 `acp-extension-core` 合约与它通信；adapter 持有 OMP 子进程，通过下列方式使用 OMP RPC
v1/v2：

```text
<Lody Node> <acp-extension-omp entry>
  -> omp --mode rpc
```

adapter 只启动 `omp --mode rpc`；任何额外启动参数（包括 Lody 自有 extension）都要先有经审核的
adapter 合约。adapter 不得以 OMP 自带 ACP 服务作为实现。首版使用用户已安装的 OMP executable，不把 OMP
本体打进 Lody managed-runtime archive。

adapter 是 ACP seam 上的 deep module。OMP 专属 RPC frame、session file、event 和 extension
UI message 都封装在内部；Lody consumer 只按广告的 ACP/Core capability 分支，不识别 OMP 私有负载。

## 能力真实性

每项可选 Core 能力独立广告。没有等价稳定 OMP RPC 或 extension interface 时保持不支持；
adapter 不得用 slash-command text、修改 private file 或 optimistic success 伪造控制面语义。

首版目标包括：

- 标准 ACP session new/load/resume、prompt/cancel、模型与思考级别配置、图片和 tool update；
- adapter 只在 initialize-time runtime probe 已证明 OMP host-tool bridge 后接受标准 ACP `mcpServers`；
  ACP 不存在 `mcpCapabilities.stdio` 广告位。带版本的 Core
  `agentCapabilities._meta.lody.stdioMcp` capability 决定 host 是否传递 built-in 和 external MCP
  servers；没有它时 host 不传递任何 MCP server。只有 client 广告 form support、每个 OMP ask 都可无损
  编码为现有 Lody AskUserQuestion form 且 runtime typed-UI bridge 已证明时，才启用结构化提问；
- Core 用量账本、上下文占用、压缩生命周期和同一回合 steer 各自通过独立门禁后才广告；adapter 把
  subagent lifecycle 映射为标准 ACP `tool_call`/`tool_call_update`，并携带有界的 `_meta.lody.task`；
  lifecycle 是整个 `subagents` capability 的父门禁，list/output 再各自通过门禁后增加。

除非契约测试证明 OMP 有等价原语，首版不广告 rate limits、goal control、subagent cancel、
`sessionHistory`、`worktreeProject` 和 `forkAtTurn`。只读 history 还必须证明有界分页、稳定
cursor、未知记录保留，并且不会恢复或修改源 session。

## 用量账本

只有 OMP 提供所需 operation identity 时，账本才来源于已完成的 OMP assistant/model
operation，不能使用 `get_session_stats` 的整段 session 总量。每个 operation 保留稳定 OMP
identity、`provider/model` key、互斥 token bucket 和报告的 cost。一个
`SessionUsageAccumulator` 覆盖 ACP accounting lifetime，发出分模型累计 `modelUsage`；host
不把连续 snapshot 相加。

`get_state.contextUsage` 和 `get_session_stats.contextUsage` 只用于标准 ACP `usage_update`
上下文仪表，不替代分模型账本。replay 不产生用量。Core
`SessionUsageUpdate._meta.lody.usageScopeId` 是有界的累计计量范围；host 校验它并据此派生
accounting side-channel identity。对 OMP，缺失或非法 scope 必须丢弃 accounting notification，
不能 fallback 到 ACP session id。每个新的 adapter accounting lifetime 使用永不复用的新 scope，
因此重启后的新用量不会被旧峰值吞掉。若无法证明 operation 合约，V1 阻塞并等待 OMP upstream
interface；缺少 Core per-model usage 的 builtin OMP 不符合本 Spec。

## Runtime 与配置

OMP 虽由用户安装，Provider 身份仍是 builtin。唯一允许的 stored override 是严格校验、非空的
`ompPath`；它、目标机有效环境中的 `OMP_PATH`、以及同一有效 PATH 的 `omp` 依次选择 executable。
一个在 probe 和 launch 中同样应用 login-shell/default-PATH policy 之后运行的 resolver 返回 canonical
descriptor。capability-cache identity 包含 adapter version、canonical path 和可信 version/file identity；
descriptor 不可验证时，probe 结果只在当前请求使用，绝不写入共享 capability cache。

凭据留在 OMP 自身存储或环境变量中，不复制到 Lody 配置。创建时始终验证 runtime compatibility；只有
OMP 提供稳定、无费用、无 session 副作用且有明确 provider/model scope 的 readiness primitive 时才检查
凭据。否则 setup 不得把 ACP startup error 分类为 authenticated 或 awaiting-auth；首次模型调用 auth
error 返回目标机设置指引和 retry 路径。

adapter 注入一个 Lody 自有 OMP extension。它保留原生 typed `ask` 和其他全部 active tool，
把 typed ask UI request 转换为 ACP elicitation，独立处理 tool approval，并且只转发有文档合约的
lifecycle/control event。无法建立必需 bridge 时 session setup 失败，不能启动半集成会话。

## 分阶段宿主接入

经审核的不可变 adapter 制品发布并完成回读之前，Lody 既不在 runtime manifest 中固定 adapter，也不打包它。
此阶段 builtin `omp` 只启动本地提供的 adapter 构建：`LODY_OMP_ADAPTER_PATH` 从 Agent 配置环境或目标机
有效环境读取，必须是已存在 adapter 入口文件的绝对路径，否则在 spawn 之前启动失败。capability-cache
identity 对该路径做摘要，因此重新构建 adapter 后需要显式刷新。Provider setup 把缺失的 adapter 路径或
`omp` executable 报告为 runtime-unavailable，且从不下载两者。

此阶段 adapter 不广告任何可选能力。它接受 host 的 MCP server 列表但不转发给 OMP，所以上文「没有
`stdioMcp` 时 host 不传递 MCP server」的规则尚未执行。用量账本在上文 scope 规则实现之前沿用共享的
builtin 路径。

## 分发与贡献

adapter 位于独立 Apache-2.0 仓库并依赖公开的 `acp-extension-core` package。MPL-2.0
`jiwangyihao/omp-acp` 只作协议研究证据，不复制实现。分阶段接入之后，Lody 只消费固定且带校验和的 adapter 制品，从不消费 adapter workspace；OMP 本体外置。

社区贡献先用 Issue 确认 adapter owner、仓库 URL、submodule policy 和贡献规模，再拆分实施，
让 adapter 合约先于聚焦的 Lody host 接入接受审核。Lody PR 遵循外部贡献的 assignment 与
对话分享规则。

## 验证

发布证据包含合成协议测试、针对每项广告能力的受控真实模型 OMP 18.2.8 contract probe、
adapter ACP smoke、usage/resume/restart 回归、Lody focused tests、public-boundary、创建时 setup
probe，以及 development/production 两条打包 CLI 启动探针。除非实际运行相应环境，这些检查
不能证明真实 provider 账单精度或跨平台支持。

## 证据

- `packages/acp-extension-core/src/{capabilities,methods,session,usage}.ts`
- `packages/acp-extension-pi/src/connection.ts`
- `apps/cli/src/agent/{setting,managed-agent-runtime}.ts`
- `packages/shared/src/ai.ts`
- 2026-09-23 对 OMP 18.2.8 执行的 RPC ready/state/stats/model/command/subagent 探针
