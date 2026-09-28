# 内置 OMP Agent 实施计划

Status: proposed
Translation: current

[English](2026-09-23-builtin-omp-agent.md)

## 摘要

直接启动 OMP 自带 ACP 服务或登记 custom command 都无法形成可信的 Lody 内置 OMP 集成，
因为两条路径都会绕过 Lody Core 扩展行为和用量合约。本计划建立一个独立的 Apache-2.0
适配器，由它持有 `omp --mode rpc`，再以聚焦的小范围改动完成 builtin 注册。每项被广告的
ACP/Core 能力都必须通过 OMP 黑盒契约测试；没有等价原语的能力保持 absent。

## 目标、架构和约束

**目标：** 让 OMP 与 Pi、Codex 同级出现在 Lody 内置 Provider 中，具备可靠会话行为、
Lody 原生交互界面，以及按模型归属正确的 token/费用统计。

**架构：** Lody 通过 ACP/Core 与 `acp-extension-omp` 通信；适配器使用 OMP RPC v1/v2，
并注入一个显式的 Lody OMP 扩展。适配器是唯一理解 OMP 私有负载的 seam。OMP 本体外置。
可执行文件选择顺序固定为：验证后的 `ompPath`、继承的 `OMP_PATH`、目标机 PATH 上解析出的
`omp`。

**技术栈：** Node.js 22、TypeScript、`@agentclientprotocol/sdk`、`acp-extension-core`、
Zod、Vitest、OMP JSONL RPC 和 Lody pnpm workspace。

实施使用测试驱动开发；完成声明前使用 `verification-before-completion`。不得复制 MPL-2.0
`jiwangyihao/omp-acp` 的实现，可以采用黑盒协议事实和独立编写的测试。

本计划放在所属 Agent Note 中，不使用 `docs/plans`：Lody 要求跨模块决策进入
`.agents/notes`，并禁止新增第二套根文档树。

## 进展快照（2026-09-26）

- Task 0 正等待 maintainer 确认 adapter owner、仓库 URL、submodule policy、isolation model 和
  assignment。Lody Issue #930 仍为 open；计划中的 `LodyAI/acp-extension-omp` 仓库尚不存在。
- candidate 仓库 `dustPyrotechnic/acp-extension-omp` 的 `feat/bootstrap` 分支已到
  `a678bb4`：Node/pnpm metadata 与 lockfile、已核验 Apache-2.0 license、fake child-process ready-frame
  红绿覆盖、fail-closed real-contract 入口、package smoke 和三 OS fake-smoke CI matrix。CI run
  `36367716998` 已在 macOS、Linux、Windows 全部通过。它仍是个人 candidate，不是已接受的官方
  source。
- 无凭据隔离 OMP 18.2.8 探针确认 canonical executable 为
  `/opt/homebrew/Cellar/omp/18.2.8/bin/omp`；ready frame 广告 protocol 1、支持 `[1,2]`、1 MiB
  单帧上限和 64 MiB 重组上限。探针还证明 state/stats/models/commands response correlation、stdin
  EOF clean exit、SIGTERM exit 143，以及 `--no-extensions` 下仍存在 unsolicited builtin extension
  event。它没有运行模型，也不构成 Task 1R 证据。
- Task 6A 的 accounting 部分已在上游完成：Core 0.1.8 发布了 bounded
  `_meta.lody.usageScopeId`，Lody PR #970 已按 restart-safe accounting 消费它；旧的本地
  `accountingId` 提案已被取代。
- Task 6A 剩余的 `stdioMcp` 合约已有基于 Core 0.1.8 的本地红绿实现，typecheck 和全部 5 个
  Core tests 通过；仍需 review、release/tag，并记录 immutable version/commit/checksum。
- Task 1 已有部分 synthetic 实现证据：bounded UTF-8 JSONL framing、带 adapter 自有 1 MiB/64 MiB
  ceiling 的 ready negotiation，以及四种无凭据 query 的 deterministic request/response correlation。
  最终 suite 为 39 tests 全绿并通过独立 Spec/quality review；red-first 失败已在实现运行中报告，但未作为
  repository artifact 保留。session/prompt/cancel/resume/terminal 与 usage 合约仍未实现。
- Tasks 2–5 与 6B–11 尚无实现证据；尤其没有 credentialed real OMP contract run、release-ready
  adapter package、Lody host wiring、Lody packaged smoke 或 UI 验证通过。

## 交付门禁与结构

1. host 开发前确认贡献路线和 adapter 归属。
2. restart-safe accounting 使用已发布的 Core 0.1.8 usage scope；Task 5 开始前发布剩余的
   provider-neutral `stdioMcp` capability（Task 6A）。
3. adapter 固定已发布 Core 版本，独立审核并 tag。
4. Lody 同时更新 Core/adapter submodule pointer、host parser 与 persistence，并闭合开发/生产打包。
5. 只有稳定 RPC 缺少能力时才向 OMP 上游提接口。

adapter 先完成审核和 tag，再提交 host PR。以下标为条件能力的接口，门禁通过前不得出现在
initialize 响应中。

### Task 0：确认归属并初始化 adapter 仓库

**未来 adapter 文件：** `package.json`、`pnpm-lock.yaml`、`tsconfig.json`、
`vitest.config.ts`、`src/index.ts`、`test/fixtures/omp-rpc-process.mjs`、`README.md`、
`LICENSE`、`.github/workflows/check.yml`。

**步骤：**

1. 建立 Lody Issue，写明架构、能力矩阵、预计 host diff、adapter owner/URL、submodule 路线、
   isolation model、tag/checksum 标准和 OMP 本体外置。host 改动超过社区限制时，先取得 assignment。
2. 确认 LodyAI 接受指定仓库与 pinned submodule source，并记录 adapter 是普通 workspace member 还是
   isolated package；不得让 host 依赖未接受的个人仓库位置。
3. 初始化 Node 22、package manager、ACP SDK/Core/Zod/Vitest、TypeScript、lockfile，以及 `check`、
   `build`、`test`、`smoke`、`contract:real`、`pack` scripts。
4. 先写 fake-runtime ready-frame 红测，再使 adapter 根目录 `pnpm test`、`pnpm build` 变绿。建立
   `ubuntu-latest`、`macos-latest`、`windows-latest` fake-smoke CI matrix；real contract job opt-in
   且 fail-closed。
5. 提交：`chore: bootstrap OMP ACP adapter`。

### 阶段测试门禁

每个阶段先运行命名的失败测试并记录 red，再写最小实现。合成 fixture 只证明合成合约，绝不替代
需要的真实 runtime 门禁。

| 阶段 | Red/green 证据 | 进入下一阶段条件 |
| --- | --- | --- |
| 0 — 归属/引导 | fake ready-frame，adapter 根目录 `pnpm test`、`pnpm build` | maintainer 接受 owner、仓库、submodule 与 isolation model；三 OS fake-smoke matrix 已定义 |
| 1 — RPC 合成 | `pnpm test -- test/contract/omp-rpc` | 仅 wire parser 变绿，不解锁 V1 |
| 1R — 真实 baseline | fail-closed `pnpm contract:real`，OMP 18.2.8 | 已记录 new/prompt/cancel、exact load/resume、terminal order、per-model operation usage 成功；缺 binary/credential 即阻塞 |
| 6A — Core release | Core 0.1.8 usage tests，加 focused `stdioMcp` 红绿 compile-contract，再运行 `npm run typecheck && npm test` | 已发布 `usageScopeId` 保持完整且 reviewed/tagged release 含 `stdioMcp`；只有 Task 5 等待该 release，Task 6B 可直接使用 Core 0.1.8 |
| 2–5、6B–7 — adapter | focused runtime/session、extension、ACP、MCP/elicitation/permission、usage、Core suites | 每个命名 suite 变绿；optional capability 仅在真实 probe 成功后广告 |
| 8 — adapter package | `pnpm check && pnpm build && pnpm test && pnpm smoke && pnpm pack`、CI matrix | manifest/tarball/CI 都绿；这是唯一 adapter tag 门禁 |
| 9–10 — Lody host/UI | focused Vitest、fake-OMP dev/prod bundle smoke、setup/component suites | cache、setup、packaged asset、UI 均通过且不声称 real-provider 结果 |
| 11 — 集成 | root public-boundary、check、format、i18n、docs、packaged probe | 协议/包/UI/真实 provider（如实际运行）证据分层齐全 |

### Task 1：冻结 OMP RPC 与扩展合约

**Adapter 文件：** `src/runtime/omp/{frames,schemas}.ts`、`test/contract/omp-rpc/*.test.ts`、
`test/contract/real-omp/` fixtures。

1. 为 ready/version 协商、correlation、frame limit、malformed JSON、unknown event、进程退出、
   forward-compatible field 写红测，运行 `pnpm test -- test/contract/omp-rpc` 记录 red。
2. 冻结 state/stats/models/commands/new/switch/prompt/steer/follow-up/abort 及 terminal order；覆盖
   exact identity、unsafe switch rejection、replay、prompt admission、abort acknowledgement、idle、delivery drain。
3. 冻结 host-tool、typed UI、subagent subscription/list/output、extension frames；要求 stable id、
   bounded payload、cancel/disconnect、sanitization。只有全部 native ask kind 都能无损映射为既有
   AskUserQuestion form 时才认为 elicitation 可行。
4. 让 `pnpm test -- test/contract/omp-rpc` 变绿并提交 `test: freeze OMP RPC contract`；这只证明 wire。
5. 对 fixed OMP 18.2.8 和显式 provisioned credential 实现 fail-closed `contract:real`。它必须证明
   new/prompt/cancel、exact load/resume、terminal delivery order、completed operation identity/provider/
   model/disjoint tokens/cost；缺 binary/credential 必须 nonzero。V1 baseline implementation 前记录 green。
6. 单独 probe host-tool、AskUserQuestion-form typed UI、steering acknowledgement、compaction、subagents。
   baseline failure 停止 V1；optional failure 只移除对应 capability。

### Task 2：实现进程与会话 owner

**Adapter 文件：** `src/runtime/omp/{command,process}.ts`、`src/session/session-manager.ts`、tests。

1. 为 resolved executable、argv、cwd、environment allowlist、ready timeout、cancel、exit、stderr redaction、
   process-group cleanup 写红测。
2. 只启动 `omp --mode rpc --no-extensions -e <packaged-lody-extension.mjs>`。一个 ACP connection
   拥有一个 process 和 active prompt；setup mutation 串行化。
3. ACP id 精确绑定已证明的 OMP id；new 用 `new_session`，只有 Task 1R 证据后才能以 `switch_session`
   load/resume；unsafe/not-found 必须失败，不能创建替代 session。
4. 仅在 terminal `agent_end`、idle、delivery drain 后结束 prompt。让
   `pnpm test -- test/runtime test/session` 变绿并提交。

### Task 3：注入 Lody OMP 扩展

1. 为 handshake/redaction 写红测。保留 native `ask` 与所有 active tools，不注册竞争 ask。
2. 仅实现已经证明需要的 AskUserQuestion-form UI、operation usage、compaction、task lifecycle、steering、
   diagnostics 的 versioned frames；第一个 prompt 前注册 listener。
3. required handshake failure 令 setup 失败；optional frame 缺失使 capability absent。让
   `pnpm test -- test/extension` 变绿并提交。

### Task 4：实现标准 ACP 会话行为

1. 为 initialize、new/load/resume、prompt/cancel、images、embedded context、streams、tool lifecycle、
   titles、malformed foreign content 写红测。
2. model 映射为稳定 `provider/id`，只暴露已验证 thinking level，不广告 session history。让
   `pnpm test -- test/acp` 变绿并提交。

### Task 6A：采用已发布 usage scope 并发布 stdio MCP 合约

1. 把 Core 0.1.8 `SessionUsageUpdate._meta.lody.usageScopeId` 及对应 Lody CLI consumer 视为已发布
   accounting 合约；删除已被取代的本地 `accountingId` 提案，不再发布第二个 identity 字段。
2. 只把 `LodyExtensionCapabilities.stdioMcp: { version: 1 }` 迁到 Core 0.1.8。先写 compile-contract
   红测，同时保护 `sessionTitle`、`usageScopeId` 和长度上限，再添加 capability 与 README 合约，
   不重定义 standard ACP。
3. 在 Core 内运行 `npm run typecheck && npm test`，审核、发布/tag，记录 immutable
   version/commit/checksum。`stdioMcp` release 先于 Task 5；Task 6B 可直接使用 Core 0.1.8。

### Task 5：接入 MCP、权限和 elicitation

1. 为 `stdioMcp` 前的 MCP omission、prompt 前 `set_host_tools`、重复名称、非法 transport、rollback、
   timeout、close、cancel、sanitized result 写 MCP 红测。
2. 写 interaction 红测：每个 OMP ask 无损映射到既有 AskUserQuestion form；tool approval 保持 ACP
   permission request 且与 elicitation 使用不同 pending id；answer/decline/cancel 回到 exact OMP id，
   late reply 被忽略。
3. 仅在 Task 1 real host-tool proof 后广告 `stdioMcp` 并接受 `mcpServers`；已广告 session 中 server
   install 失败必须使 setup 失败。
4. 运行 `pnpm test -- test/acp/mcp.test.ts test/acp/elicitation.test.ts test/acp/permissions.test.ts`
   变绿并提交。

### Task 6B：实现 adapter 用量账本

1. 仅在 Task 1R 与 released Core 0.1.8 后开始。为 multi-model、duplicate/incomplete snapshot、cache/
   reasoning、unknown cost、compaction、replay、resume/restart、malformed metadata 写红测。
2. completed stable operation 进入 `SessionUsageAccumulator`，发出非空累计 model usage 与 bounded
   `_meta.lody.usageScopeId`；context meter 仍是 standard ACP usage update。
3. 每个新的 adapter accounting lifetime 使用永不复用的新 scope；同一 lifetime 内 retry 保持 scope。
   usage suite 变绿并提交；不得在此 tag。

### Task 7：接入真实的 Core 生命周期能力

1. 为精确 initialize assertion 写红测；usage、compaction、steering、subagent lifecycle 独立 gate，
   lifecycle 是 list/output 的前置，cancel 保持 absent。
2. steering 必须在后续 output 前具备 native action 和 matching applied evidence；cancel/EOF/turn end
   拒绝 unsettled steer。
3. lifecycle 首帧发非空 title/stable id 的 `tool_call`，后续在同 id 发 `tool_call_update`。`_meta.lody.task`
   必须为 version 1、supported kind、且只用 pending/in_progress/completed/failed；parent 用
   `parentTaskId`/`parentToolCallId`。OMP-only states 保留在 list/output response。
4. proven compaction 映射为 paired activity；负向测试确保不支持 Core capability absent。运行
   `pnpm test -- test/core` 变绿并提交。

### Task 8：打包并独立验证 adapter

1. 构建显式 adapter、manifest、extension asset，不允许依赖 source path。
2. 运行 `pnpm check && pnpm build && pnpm test && pnpm smoke && pnpm pack`，检查 tarball/checksum。
3. 要求 Task 0 macOS/Linux/Windows fake-smoke CI matrix；真实 contract 仅在 provisioned 环境运行。
4. 独立 correctness/security review 无 unresolved finding 后再 tag，记录 commit/version/manifest/checksum；
   这是唯一 adapter tag gate。

### Task 9：注册 builtin OMP 并闭合打包

1. 写 host parser 红测：有效 bounded Core `usageScopeId` 到达 usage side-channel；OMP missing/invalid
   scope 被丢弃，不 fallback；Codex legacy derivation 仍局限 Codex。
2. 解析 Core `stdioMcp`，并在每个 new/load/resume/replacement path 的 built-in 与 external server 前 gate
   `buildMcpServers`；为无 capability 的 omission 写红测。
3. 在所有 Flock/config/Machine-RPC guard 加严格 nonblank `ompPath`，拒绝 unknown key 和任意 argv。OMP
   加入 `BUILTIN_AGENTS` 与 title ownership `none`，不加入 managed runtime。
4. 每个 probe/session path 都先构造相同 effective target environment：configured base、login-shell overlay、
   default PATH entries；再按 `ompPath`、`OMP_PATH`、PATH resolve。保持 environment，仅 overlay resolved
   `OMP_PATH` 给 `omp-acp.js`，保留 Electron `ELECTRON_RUN_AS_NODE`；绝不启动 `omp acp`。
5. resolver 返回 descriptor 与 cacheability。persistent identity 是 adapter version + canonical path + trusted
   version/file identity；mismatch/unsupported version 令 probe 失败，ephemeral result 跳过 refresh 与
   created-session 两个 Flock write。测试 PATH change、same-path replacement、两个 writer。
6. 按 Task 0 isolation decision：workspace adapter 加入 `prepare:acp-adapters`；isolated adapter 用 checksummed
   pack 验证。输出 `omp-acp.js` 后显式复制 manifest/extension 到 dist/dist-dev root entry 旁，由 entry
   resolve 并传给 adapter，扩展 Electron after-pack assertion。测试 dev/prod fake OMP、npm pack、published
   imports、`sync:cli`。
7. 运行 focused host suites、bundle checks、packaged CLI/Electron fake probe、public-boundary，提交。

### Task 10：加入创建时探测、认证指引和 UI

1. 新增独立 versioned machine protocol capability 给 OMP setup；旧 `providerSetup` v1 不足。创建时 gate
   该 capability，unsupported target 直接失败，不能 enqueue row。
2. default OMP PATH 仅在新 capability 存在时走 durable queue。explicit `ompPath` 不进 queue，而走
   config-id foreground refresh；测试 local/remote 中它被 freeze 且绝不静默丢弃。
3. 没有 documented readiness RPC 时，创建只验证 runtime。阻止 OMP startup auth error 变为
   `authRequired`/`awaiting-auth`；首次 model-call auth failure 给 target guidance/retry。
4. 为 setup/dialog/component 加 path precedence、cleared/invalid path、cache、protocol version、auth guidance、
   retry 的 focused tests；运行 package tests/typecheck/i18n/Storybook render 与 fresh desktop interaction，提交。

### Task 11：文档、总验证和 host 贡献准备

1. 仅以已执行 evidence 更新双语 Spec/note，status 遵循 approval rule。
2. 运行 adapter/Lody phase gates、`pnpm check:public-boundary`、`pnpm lint:i18n`、`pnpm check`、
   `pnpm format`、`pnpm run docs check`、`git diff --check` 与 packaged probe。
3. 协议/单测、package、UI、real-provider evidence 分层报告；adapter/host 独立 review 清空 finding 后，
   fresh-context 复审。
4. external fork PR 前向用户取得必需 public conversation link；Core、adapter、host 保持独立可审。

## 延后的 OMP 上游接口

Core 全覆盖仍需要稳定的 rate-limit snapshot、goal control、subagent cancel、只读 history paging、
worktree identity 和 turn-addressable fork。Task 1 门禁失败时，usage、MCP、elicitation、steering
也移入此处。每项提案必须先定义 request/response/event schema、取消与顺序语义、resume 行为
和黑盒契约测试，adapter 才能广告相应 capability。

## 计划批准边界

V1 release-blocking baseline 是 builtin launch、initialize/new/prompt/cancel、exact load/resume、
terminal ordering，以及 restart-safe 的 per-model token/cost accounting。任一门禁失败都停止 V1，
把缺失原语转入 OMP upstream，并重新决定范围。MCP、elicitation、steering、compaction、subagents
保持独立条件能力。若要求 V1 实现 Core 当前全部 capability，项目将扩大为更大的跨仓库 OMP
RPC 工程。
