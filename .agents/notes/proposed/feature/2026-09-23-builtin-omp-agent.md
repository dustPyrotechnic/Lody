# Built-in OMP agent implementation plan

Status: proposed
Translation: current

[中文](2026-09-23-builtin-omp-agent.zh.md)

## Abstract

Lody cannot obtain a truthful built-in OMP integration by launching OMP's native ACP server or by
registering a custom command: both bypass Lody's Core extension behavior and accounting contract.
This plan creates a separate Apache-2.0 adapter that owns `omp --mode rpc`, then adds a focused
built-in host registration. Every advertised ACP/Core capability has a black-box OMP contract test;
features without an equivalent primitive remain absent.

## Goal, architecture, and constraints

**Goal:** ship OMP beside Pi and Codex as a Lody built-in Provider with reliable session behavior,
Lody-native interaction surfaces, and per-model token/cost accounting.

**Architecture:** Lody speaks ACP/Core to `acp-extension-omp`; the adapter speaks OMP RPC v1/v2 and
injects one explicit Lody OMP extension. The adapter is the only seam that knows OMP payloads. OMP
remains externally installed. Resolution precedence is the validated `ompPath` override, inherited
`OMP_PATH`, then a resolved `omp` on the target machine's PATH.

**Tech stack:** Node.js 22, TypeScript, `@agentclientprotocol/sdk`, `acp-extension-core`, Zod,
Vitest, OMP JSONL RPC, and the Lody pnpm workspace.

Implementation uses test-driven development. Completion uses `verification-before-completion`.
Do not copy MPL-2.0 implementation from `jiwangyihao/omp-acp`; black-box protocol facts and
independently written tests are allowed.

The plan lives in the owning Agent Note instead of `docs/plans`: Lody reserves cross-module
decisions for `.agents/notes` and prohibits a second root documentation tree.

## Progress snapshot (2026-09-26)

- Task 0 is waiting for maintainer confirmation of the adapter owner, repository URL, submodule
  policy, isolation model, and assignment. Lody Issue #930 is open; the planned
  `LodyAI/acp-extension-omp` repository does not yet exist.
- Candidate repository `dustPyrotechnic/acp-extension-omp` has a `feat/bootstrap` branch at
  `a678bb4`: Node/pnpm metadata and lockfile, verified Apache-2.0 license, fake child-process ready-frame
  red/green coverage, fail-closed real-contract entrypoint, package smoke, and a three-OS fake-smoke
  CI matrix. CI run `36367716998` passed on macOS, Linux, and Windows. This is a personal candidate
  repository, not an accepted official source.
- A credential-free isolated OMP 18.2.8 probe confirmed the canonical executable
  `/opt/homebrew/Cellar/omp/18.2.8/bin/omp` and a ready frame advertising protocol 1, supported
  versions `[1,2]`, a 1 MiB frame limit, and a 64 MiB reassembly limit. It also proved correlated
  state/stats/models/commands responses, clean stdin-EOF exit, SIGTERM exit 143, and unsolicited
  built-in extension events despite `--no-extensions`. It did not run a model or prove Task 1R.
- The accounting part of Task 6A is complete upstream: Core 0.1.8 publishes bounded
  `_meta.lody.usageScopeId`, and Lody PR #970 consumes it with restart-safe accounting. The old
  local `accountingId` proposal is superseded.
- The remaining Task 6A `stdioMcp` contract has a local Core 0.1.8 red/green implementation with
  typecheck and all five Core tests passing. It still requires review, release/tag, and immutable
  version/commit/checksum evidence.
- Task 1 has partial synthetic implementation evidence: bounded UTF-8 JSONL framing, ready negotiation
  with adapter-owned 1 MiB/64 MiB ceilings, and deterministic request/response correlation for the four
  credential-free queries. The final suite has 39 passing tests and independent Spec/quality approval;
  red-first failures were reported during the implementation run but were not retained as repository
  artifacts. Session/prompt/cancel/resume/terminal and usage contracts remain unimplemented.
- Tasks 2–5 and 6B–11 have no implementation evidence yet. In particular, no credentialed real OMP
  contract run, release-ready adapter package, Lody host wiring, Lody packaged smoke, or UI verification
  has passed.

## Delivery gates and structure

1. Confirm the contribution route and adapter ownership before host implementation.
2. Use the released Core 0.1.8 usage scopes for restart-safe accounting. Publish the remaining
   provider-neutral `stdioMcp` capability (Task 6A) before Task 5 consumes it.
3. Build the adapter against that released Core version, independently review it, and tag it.
4. Update Lody's Core/adapter submodule pointers, host parser, and persistence together, then close
   both development and production packaging paths.
5. Propose OMP upstream interfaces only for capabilities absent from the stable RPC contract.

The adapter is reviewed and tagged before the host PR. A capability named below as conditional must
not appear in initialize output until its gate passes.
### Phase test gates

Every phase starts with the named failing test and records its red result before implementation. A
passing synthetic fixture proves only the synthetic contract; it never substitutes for a required
real-runtime gate.

| Phase | Red/green proof | Advance only when |
| --- | --- | --- |
| 0 — ownership/bootstrap | fake ready-frame test, then adapter-root `pnpm test` and `pnpm build` | maintainer accepts owner, repository URL, submodule source, and in-workspace/isolation model; CI defines macOS/Linux/Windows fake-smoke matrix |
| 1 — RPC contract | `pnpm test -- test/contract/omp-rpc` | synthetic wire parser is green; this does **not** unblock V1 |
| 1R — real baseline | fail-closed `pnpm contract:real` against pinned OMP 18.2.8 | recorded run proves new/prompt/cancel, exact load/resume, terminal delivery order, and per-model operation identity/usage; missing binary or credentials fails the job and blocks V1 |
| 6A — Core release | Core 0.1.8 usage tests plus focused red/green `stdioMcp` contract test, then `npm run typecheck && npm test` | released `usageScopeId` remains intact and a reviewed/tagged release contains `stdioMcp`; only Task 5 waits for that release, while Task 6B may use Core 0.1.8 |
| 2–5, 6B–7 — adapter | focused runtime/session, extension, ACP, MCP/elicitation/permission, usage, and Core suites | each named suite green; optional feature remains absent unless its real probe passed |
| 8 — adapter package | `pnpm check && pnpm build && pnpm test && pnpm smoke && pnpm pack`, CI fake-smoke matrix | manifest/tarball and all CI matrix jobs green; this is the sole adapter tag gate |
| 9–10 — Lody host/UI | focused Vitest, fake-OMP dev/prod bundle smoke, provider setup and component suites | cache, setup, packaged asset, and UI behavior pass without a live-provider claim |
| 11 — integration | root public-boundary, check, format, i18n, docs, packaged probes | separate evidence exists for unit/protocol, packages, UI, and any intentionally run real-provider test |


### Task 0: Confirm ownership and bootstrap the adapter repository

**Files in the future adapter repository:** `package.json`, `pnpm-lock.yaml`, `tsconfig.json`,
`vitest.config.ts`, `src/index.ts`, `test/fixtures/omp-rpc-process.mjs`, `README.md`, `LICENSE`, and
`.github/workflows/check.yml`.

**Steps:**

1. Open a Lody Issue with the architecture, capability matrix, expected host diff, proposed adapter
   owner/URL, submodule route, isolation model, tag/checksum policy, and the fact that OMP remains
   external. If the host change exceeds the community limit, obtain maintainer assignment before code.
2. Confirm LodyAI accepts the named repository and pinned submodule source, and record whether the
   adapter is a normal workspace member or an isolated package. Do not build the host against an
   unaccepted personal repository location.
3. Initialize Node 22, package manager, ACP SDK/Core/Zod/Vitest versions, TypeScript, lockfile, and
   `check`, `build`, `test`, `smoke`, `contract:real`, and `pack` scripts.
4. Write a failing fake-runtime ready-frame test, then make adapter-root `pnpm test` and `pnpm build`
   green. Create CI that runs fake smoke on `ubuntu-latest`, `macos-latest`, and `windows-latest`; the
   real contract job is opt-in and fail-closed.
5. Commit: `chore: bootstrap OMP ACP adapter`.

### Task 1: Freeze OMP RPC and extension contracts

**Adapter files:** `src/runtime/omp/{frames,schemas}.ts`, `test/contract/omp-rpc/*.test.ts`, and
`test/contract/real-omp/` fixtures.

1. Write failing parser tests for ready/version negotiation, correlation, frame limits, malformed JSON,
   unknown events, process exit, and forward-compatible fields; run
   `pnpm test -- test/contract/omp-rpc` and record red.
2. Freeze query/session/prompt frames, including state/stats/models/commands/new/switch/prompt/steer/
   follow-up/abort and terminal ordering. Cover exact identity, unsafe switch rejection, replay, prompt
   admission, abort acknowledgement versus terminal end, idle, and delivery drain.
3. Freeze host-tool, typed-UI, subagent subscription/list/output, and extension-frame contracts with
   stable ids, bounded payloads, cancellation, disconnect, and sanitization. Declare an OMP ask bridge
   viable only if every native kind has a lossless existing AskUserQuestion-form representation.
4. Make the synthetic suite green with `pnpm test -- test/contract/omp-rpc`; commit
   `test: freeze OMP RPC contract`. This proves only wire behavior.
5. Implement fail-closed `contract:real` against pinned OMP 18.2.8 and intentionally provisioned
   credentials. It must prove new/prompt/cancel, exact load/resume, terminal delivery order, and
   completed-operation identity/provider/model/disjoint tokens/cost. Missing binary or credentials
   exits nonzero. Record the green run before any V1 baseline implementation; fixtures never substitute.
6. Probe host-tool, AskUserQuestion-form typed UI, steering acknowledgement, compaction, and subagents
   separately. Baseline failure stops V1; optional failure removes only that advertised capability.

### Task 2: Build the process and session owner

**Adapter files:** `src/runtime/omp/{command,process}.ts`, `src/session/session-manager.ts`, tests.

1. Write failing tests for resolved executable, argv, cwd, environment allowlist, ready timeout,
   cancellation, exit, stderr redaction, and process-group cleanup; run the focused suite red.
2. Launch only `omp --mode rpc --no-extensions -e <packaged-lody-extension.mjs>`. One ACP connection
   owns one process and active prompt; setup mutations serialize.
3. Bind ACP ids exactly to proven OMP ids. Use `new_session` for new and `switch_session` only after
   Task 1R evidence; unsafe/not-found load/resume fails without creating a replacement session.
4. End a prompt only after terminal `agent_end`, idle, and delivery drain. Make
   `pnpm test -- test/runtime test/session` green; commit `feat: own OMP RPC process lifecycle`.

### Task 3: Inject the Lody OMP extension

**Adapter files:** `src/extension/lody-extension.mjs`, `src/extension/protocol.ts`, tests.

1. Write failing handshake and redaction tests. Preserve native `ask` and all active tools; never add a
   competing ask tool.
2. Add only versioned frames proven necessary for AskUserQuestion-form UI, operation usage, compaction,
   task lifecycle, steering application, and diagnostics. Register listeners before the first prompt.
3. Required handshake failure fails setup; optional-frame absence removes its capability. Make
   `pnpm test -- test/extension` green; commit `feat: add Lody bridge for OMP`.

### Task 4: Implement standard ACP session behavior

**Adapter files:** `src/acp/{server,session,content,tools}.ts`, ACP tests.

1. Write red tests for initialize, new/load/resume, prompt/cancel, images, embedded context, streams,
   tool lifecycle, titles, and malformed foreign content.
2. Map models to stable `provider/id` values and expose only verified thinking levels. Do not advertise
   session history. Make `pnpm test -- test/acp` green; commit `feat: bridge OMP sessions to ACP`.

### Task 6A: Adopt released usage scopes and publish the stdio MCP contract

**Core files:** capability/usage types, tests, README, version, release notes.

1. Treat Core 0.1.8 `SessionUsageUpdate._meta.lody.usageScopeId` and the corresponding Lody CLI
   consumer as the released accounting contract. Remove the superseded local `accountingId`
   proposal rather than publishing a second identity field.
2. Rebase only `LodyExtensionCapabilities.stdioMcp: { version: 1 }` on Core 0.1.8. First add a failing
   compile-contract test that also preserves `sessionTitle`, `usageScopeId`, and its maximum-length
   bound; then add the capability and README contract without redefining standard ACP.
3. Run `npm run typecheck && npm test`, obtain review, publish/tag, and record immutable
   version/commit/checksum. The `stdioMcp` release precedes Task 5; Task 6B may use Core 0.1.8.

### Task 5: Add MCP, permissions, and elicitation

**Adapter files:** `src/acp/{mcp,elicitation,permissions}.ts`, tests; dependency: released Task 6A.

1. Write red MCP tests for omission before `stdioMcp`, `set_host_tools` install before prompt, duplicate
   names, invalid transport, rollback, timeout, close, cancellation, and sanitized results.
2. Write red interaction tests: every OMP ask maps losslessly to the existing AskUserQuestion form;
   tool approval remains an ACP permission request with a distinct pending id; answer/decline/cancel
   return to the exact OMP id and late replies are ignored.
3. Advertise `stdioMcp` and accept `mcpServers` only after Task 1 real host-tool proof. Once advertised,
   a configured server installation failure fails session setup.
4. Run `pnpm test -- test/acp/mcp.test.ts test/acp/elicitation.test.ts test/acp/permissions.test.ts`
   green; commit `feat: bridge OMP interactions to Lody`.

### Task 6B: Implement adapter usage accounting

**Adapter files:** `src/usage/{omp-operation-usage,session-usage}.ts`, usage/resume tests.

1. Start only after Task 1R and released Core 0.1.8. Write red tests for multi-model operations, duplicate/
   incomplete snapshots, cache/reasoning buckets, unknown cost, compaction, replay, resume/restart, and
   malformed metadata.
2. Feed completed stable operations to `SessionUsageAccumulator`; emit nonempty cumulative model usage
   with bounded `_meta.lody.usageScopeId`. Context meter remains standard ACP usage update.
3. Give each new adapter accounting lifetime a never-reused scope; retries inside that lifetime retain
   the scope. Make usage tests green and commit `feat: account for OMP usage by model`. Do not tag here.

### Task 7: Add truthful Core lifecycle capabilities

**Adapter files:** `src/core/{capabilities,steering,subagents,compaction}.ts`, focused tests.

1. Write exact initialize assertions. Gate usage, compaction, steering, and subagent lifecycle
   independently; lifecycle is required before list/output and cancel remains absent.
2. Require native steering plus matching applied evidence before later output; reject unsettled steer on
   cancel, EOF, or turn end.
3. For lifecycle, first emit `tool_call` with nonempty title/stable id, then `tool_call_update` on it.
   `_meta.lody.task` has version 1, supported kind, and only pending/in_progress/completed/failed;
   parent linkage uses `parentTaskId`/`parentToolCallId`. OMP-only states stay in list/output responses.
4. Map proven compaction to paired activity calls. Negative tests ensure unsupported Core capabilities
   are absent. Make `pnpm test -- test/core` green; commit `feat: expose verified OMP Core capabilities`.

### Task 8: Package and independently validate the adapter

1. Build explicit adapter, manifest, and extension assets; resolution never depends on source paths.
2. Run `pnpm check && pnpm build && pnpm test && pnpm smoke && pnpm pack`; inspect tarball/checksum.
3. Require Task 0's macOS/Linux/Windows fake-smoke CI matrix; run real contract only where provisioned.
4. Obtain independent correctness/security review without open findings, then tag and record the
   adapter commit/version/manifest/checksum. This is the only adapter tag gate.

### Task 9: Register OMP as a Lody built-in Provider and close packaging

**Files:** pinned Core/adapter source, `apps/cli/src/{omp-acp-entry.ts,agent/{setting,acp-runner,
acp-capabilities,agent-client,lody-acp-extension}.ts,session/session-execution-service.ts}`, CLI build
scripts/config, Electron asset assertions, shared AI/cache schemas, and focused tests.

1. Write red host parser tests: a valid bounded Core `usageScopeId` reaches the usage side-channel;
   OMP missing/invalid scope is dropped rather than falling back; Codex legacy derivation remains scoped.
2. Add `stdioMcp` Core parsing and gate `buildMcpServers` before both built-in and external servers for
   every new/load/resume/replacement path. Write red tests for omission without the capability.
3. Add strictly nonblank `ompPath` at every Flock/config/Machine-RPC guard, rejecting unknown keys and
   arbitrary argv. Add OMP to `BUILTIN_AGENTS` and title ownership `none`, not managed runtimes.
4. Build the shared effective target environment before resolver use in every probe/session path:
   configured base, login-shell overlay, then default PATH entries. Resolve `ompPath`, `OMP_PATH`, then
   PATH from that environment; preserve it and overlay only resolved `OMP_PATH` for `omp-acp.js` so
   Electron keeps `ELECTRON_RUN_AS_NODE`. Never start `omp acp`.
5. Return descriptor plus cacheability. Persistent identity is adapter version + canonical path + trusted
   version/file identity; mismatch/unsupported version fails probe, and ephemeral results skip both
   refresh and created-session Flock writes. Test PATH change, same-path replacement, and both writers.
6. After Task 0's isolation decision: workspace adapter joins `prepare:acp-adapters`; isolated adapter
   is proved from its checksummed pack. Emit `omp-acp.js`, then explicitly copy manifest/extension beside
   the dist/dist-dev root entry, resolve that path in `omp-acp-entry.ts`, pass it to the adapter, and
   extend Electron after-pack assertions. Test dev/prod fake OMP, npm pack, published imports, and
   `sync:cli`.
7. Run focused host suites, bundle checks, packaged CLI/Electron fake probe, and public-boundary check;
   commit `feat: register built-in OMP agent`.

### Task 10: Add creation-time setup, authentication guidance, and UI

1. Add a distinct versioned machine protocol capability for OMP setup; old `providerSetup` v1 is not
   sufficient. Gate OMP configuration creation on that capability, failing unsupported targets rather
   than enqueuing a row.
2. Default OMP PATH may use the durable queue only when the new capability exists. Explicit `ompPath`
   stays off the queue and uses a config-id foreground refresh; tests prove it is frozen and never
   silently dropped on local or remote targets.
3. Without documented readiness RPC, creation checks runtime only. Prevent OMP startup auth errors from
   becoming `authRequired`/`awaiting-auth`; first model-call auth failure provides target guidance/retry.
4. Add focused setup/dialog/component tests for path precedence, cleared/invalid path, cache states,
   protocol versions, auth guidance, and retry. Run package tests/typecheck/i18n/Storybook render and
   fresh desktop interactions; commit `feat: configure built-in OMP provider`.

### Task 11: Document, verify, and prepare the host contribution

1. Update this bilingual Spec/note with executed evidence only; status changes follow approval rules.
2. Run adapter and Lody phase gates, `pnpm check:public-boundary`, `pnpm lint:i18n`, `pnpm check`,
   `pnpm format`, `pnpm run docs check`, `git diff --check`, and packaged probes.
3. Report protocol/unit, package, UI, and real-provider evidence separately. Obtain independent adapter
   and host reviews, resolve findings, then run a fresh no-findings review.
4. Immediately before an external fork PR, request the required public conversation link. Keep Core,
   adapter, and host changes independently reviewable.

## Deferred OMP upstream interface work

Full Core coverage requires stable OMP primitives for rate-limit snapshots, goal control, subagent
cancellation, read-only history paging, worktree project identity, and turn-addressable fork. Usage,
MCP, elicitation, and steering also move here if their Task 1 gates fail. Each proposal defines
request/response/event schemas, cancellation and ordering, resume behavior, and a black-box contract
test before the adapter advertises it.

## Plan acceptance

The V1 release-blocking baseline is built-in launch, initialize/new/prompt/cancel, exact load/resume,
terminal ordering, and restart-safe per-model token/cost accounting. Failure of any baseline gate
stops V1, moves the missing primitive to OMP upstream work, and requires a new scope decision. MCP,
elicitation, steering, compaction, and subagents remain independently conditional. Requiring every
current Core capability in V1 is a larger cross-repository OMP RPC project.
