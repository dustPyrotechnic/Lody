# Built-in OMP agent

Status: draft
Translation: current

[中文](builtin-omp.zh.md)

## Scenario

A user who already has Oh My Pi (OMP) installed can choose OMP from Lody's built-in Provider list,
start and resume sessions through Lody, use Lody interaction surfaces, and see correctly attributed
token and cost usage. The user does not enter a custom ACP command, and Lody does not launch OMP's
native `omp acp` server.

## Responsibilities

Lody ships an Apache-2.0 `acp-extension-omp` adapter. Lody communicates with it through ACP and the
provider-neutral `acp-extension-core` contracts. The adapter owns the OMP child process and uses OMP
RPC v1/v2 as follows:

```text
node omp-acp.js
  OMP_PATH=/absolute/path/to/omp
  -> omp --mode rpc --no-extensions -e <lody-extension.mjs>
```

`--no-extensions` disables ambient extension discovery while preserving the explicitly supplied Lody
extension. The adapter must not use OMP's native ACP server as its implementation. V1 uses the user's
installed OMP executable and does not place OMP itself in a Lody managed-runtime archive.

The adapter is a deep module at the ACP seam. OMP-specific RPC frames, session files, events, and
extension UI messages remain internal. Lody consumers branch only on advertised ACP/Core capabilities
and never inspect OMP-private payloads.

## Capability truthfulness

Each optional Core capability is advertised independently. A capability remains unsupported when OMP
does not expose an equivalent stable RPC or extension interface. The adapter must not emulate
control-plane semantics with slash-command text, private-file mutation, or optimistic success.

V1 targets:

- Standard ACP session new/load/resume, prompt/cancel, model and thinking-level configuration, images,
  and tool updates.
- The adapter accepts standard ACP `mcpServers` only after an initialize-time runtime probe proves the
  OMP host-tool bridge. ACP has no `mcpCapabilities.stdio` advertisement field. A versioned Core
  `agentCapabilities._meta.lody.stdioMcp` capability determines whether the host passes built-in and
  external MCP servers; without it, the host passes no MCP servers. Structured elicitation is enabled
  only when the client advertises form support, every OMP ask can be represented without loss by the
  existing Lody AskUserQuestion form, and the runtime typed-UI bridge has been proven.
- Core usage accounting, context occupancy, compaction lifecycle, and same-turn steering are each
  advertised only after their independent gates pass. The adapter maps subagent lifecycle events to
  standard ACP `tool_call`/`tool_call_update` notifications with bounded `_meta.lody.task` metadata.
  Lifecycle is the parent gate for the entire `subagents` capability; list and output add their own
  gates.

Unless contract tests prove an equivalent OMP primitive, V1 does not advertise rate limits, goal
control, subagent cancellation, `sessionHistory`, `worktreeProject`, or `forkAtTurn`. Read-only
history also requires bounded pagination, stable cursors, preservation of unknown records, and proof
that reading history does not restore or mutate the source session.

## Usage accounting

When OMP exposes the required operation identity, accounting derives only from completed OMP
assistant/model operations, never whole-session totals from `get_session_stats`. Each operation keeps
a stable OMP identity, `provider/model` key, disjoint token buckets, and reported cost. One
`SessionUsageAccumulator` spans an ACP accounting lifetime and emits cumulative per-model
`modelUsage`; the host does not add consecutive snapshots.

`get_state.contextUsage` and `get_session_stats.contextUsage` feed only the standard ACP
`usage_update` context gauge and do not replace the per-model ledger. Replay produces no usage. Core
`SessionUsageUpdate._meta.lody.usageScopeId` is a bounded cumulative-accounting scope that the host
validates and uses to derive the accounting side-channel identity. For OMP, a missing or invalid scope
causes the host to drop the accounting notification instead of falling back to the ACP session ID.
Each new adapter accounting lifetime uses a never-reused scope so usage produced after a restart is not
discarded under an earlier cumulative maximum. If the operation contract cannot be proven, V1 remains
blocked on an upstream OMP interface; built-in OMP without Core per-model usage does not satisfy this
Spec.

## Runtime and configuration

OMP is a built-in Provider even though the user installs it. The only permitted stored override is a
strictly validated, non-empty `ompPath`. The executable resolves in order from that setting,
`OMP_PATH` in the target's effective environment, and `omp` on the same effective PATH. One resolver
applies the same login-shell/default-PATH policy to probe and launch and returns a canonical descriptor.
The capability-cache identity includes adapter version, canonical path, and a trustworthy version/file
identity. When the descriptor cannot be verified, probe results remain request-local and never enter a
shared capability cache.

Credentials remain in OMP storage or environment variables and are not copied into Lody configuration.
Creation always validates runtime compatibility. Credential readiness is checked only if OMP exposes a
stable, cost-free, session-side-effect-free primitive with explicit provider/model scope. Otherwise,
setup must not classify an ACP startup error as authenticated or awaiting authentication; the first
model-call authentication error returns target-specific setup guidance and a retry path.

The adapter injects a Lody-owned OMP extension. It preserves native typed `ask` and all other active
tools, translates typed ask UI requests to ACP elicitation, handles tool approval independently, and
forwards only documented lifecycle/control events. If a required bridge cannot be established, session
setup fails instead of starting a partially integrated session.

## Distribution and contribution

The adapter lives in a separate Apache-2.0 repository and depends on the public
`acp-extension-core` package. MPL-2.0 `jiwangyihao/omp-acp` is protocol-research evidence only; its
implementation is not copied. Lody consumes the adapter through a pinned submodule/build entry and
places the JavaScript adapter and explicit extension asset beside the CLI while keeping OMP external.

A community contribution first uses an Issue to confirm the adapter owner, repository URL, submodule
policy, and contribution size. Implementation is then split so the adapter contract can be reviewed
before the focused Lody host integration. The Lody PR follows external-contribution assignment and
conversation-sharing rules.

## Verification

Release evidence includes synthetic protocol tests, controlled real-model OMP 18.2.8 contract probes
for every advertised capability, adapter ACP smoke tests, usage/resume/restart regressions, focused Lody
tests, public-boundary checks, creation-time setup probes, and packaged CLI launch probes for both
development and production builds. Unless the corresponding environment is actually exercised, these
checks do not prove real provider billing accuracy or cross-platform support.

## Evidence

- `packages/acp-extension-core/src/{capabilities,methods,session,usage}.ts`
- `packages/acp-extension-pi/src/connection.ts`
- `apps/cli/src/agent/{setting,managed-agent-runtime}.ts`
- `packages/shared/src/ai.ts`
- OMP 18.2.8 RPC ready/state/stats/model/command/subagent probes run on 2026-09-23
