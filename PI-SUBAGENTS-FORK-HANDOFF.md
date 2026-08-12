# Trellis Pi-Subagents Fork Handoff

This branch contains the locally validated Trellis Pi-subagents delivery and is intended for controlled validation in another project folder.

## Release Source

- Fork: `https://github.com/coding-chong/Trellis-win-fixed.git`
- Branch: `feat/pi-subagents-backend`
- Certified handoff release: `1625a77062e1797fb4a63bb594c320f92c5e7bde` (the immutable first-use certification handoff release; contains `b94d8a45969b0d84b151bf0e9e2842e3cfa371db`, `30c3d80fea66f5ca7edc3b049c613dcf70902afd`, `060fbdae5e7751641e5b55a7ecbf8b46c314c516`, and `da241032ec1295158048be85948d6ce12bdc9ffd`). A verifier must checkout this exact release before injection unless a later separately published release record supplies another immutable expected SHA.
- Required migration ancestor: `da241032ec1295158048be85948d6ce12bdc9ffd`

A branch name is not sufficient provenance. Before touching a target project, clone the fork, checkout the certified release SHA, and verify exact equality. For this handoff use `1625a77062e1797fb4a63bb594c320f92c5e7bde`. A later branch head does not replace this release: it requires a separate published release record that gives a new immutable `expectedHead`.

```powershell
git clone --branch feat/pi-subagents-backend --single-branch `
  https://github.com/coding-chong/Trellis-win-fixed.git `
  <TRELLIS_CHECKOUT>

git checkout --detach 1625a77062e1797fb4a63bb594c320f92c5e7bde

$trellis = (Resolve-Path <TRELLIS_CHECKOUT>).Path
$expectedHead = '1625a77062e1797fb4a63bb594c320f92c5e7bde' # Replace only with a separately published immutable later release SHA.
$parent = 'da241032ec1295158048be85948d6ce12bdc9ffd'
$telemetry = '060fbdae5e7751641e5b55a7ecbf8b46c314c516'
$actualOrigin = (git -C $trellis remote get-url origin).Trim()
if ($actualOrigin -ne 'https://github.com/coding-chong/Trellis-win-fixed.git') { throw "Unexpected fork remote: $actualOrigin" }
if (git -C $trellis status --short) { throw 'Trellis checkout is not clean.' }
$actual = (git -C $trellis rev-parse HEAD).Trim()
if ($actual -ne $expectedHead) { throw "Fork HEAD does not equal the recorded immutable release SHA. Expected $expectedHead; got $actual" }
git -C $trellis merge-base --is-ancestor $expectedHead $actual
if ($LASTEXITCODE -ne 0) { throw 'Recorded release SHA is not reachable from this checkout.' }
git -C $trellis merge-base --is-ancestor $parent $actual
if ($LASTEXITCODE -ne 0) { throw 'Required migration commit is absent.' }
git -C $trellis merge-base --is-ancestor $telemetry $actual
if ($LASTEXITCODE -ne 0) { throw 'Required telemetry release commit is absent.' }
```

Stop before injection if the URL, clean-tree check, stable release ancestor, or either delivery ancestor check fails. Record the checked-out SHA in the target-local validation report.

## Build The Verified CLI

Run from the verified Trellis checkout:

```powershell
Set-Location $trellis
pnpm install --frozen-lockfile
pnpm build:core
pnpm --filter @mindfoldhq/trellis build
```

Do not invoke a globally installed Trellis CLI for this validation. The local CLI must be built first because its `dist/cli/index.js` is generated.

## Choose One Target Path

### New Project

Use this only when the target has no `.trellis/` and no Trellis-managed `.pi/` assets:

```powershell
Set-Location <TARGET_PROJECT_ROOT>
node "$trellis/packages/cli/bin/trellis.js" init --pi --yes --no-monorepo -u <DEVELOPER_NAME>
```

The generated canonical Pi contract is portable `bash`. Do not silently replace it with Windows `pwsh` based only on the host OS. A Windows `pwsh` profile requires an explicit, separately validated provider overlay using the home-relative `@4fu/pi-pwsh` provider.

### Existing Project

Use this only when the target already contains `.trellis/.version` and `.trellis/.template-hashes.json`:

```powershell
$script = "$trellis/packages/cli/scripts/migrate-trellis-pi-subagents.ps1"
pwsh -NoProfile -File $script -ProjectRoot <TARGET_PROJECT_ROOT> -WhatIf
```

Review the listed targets. Exit 0 with `would be migrated` is eligible for apply; `No changes required` is already current. Any customization or hash/contract error is a fail-closed stop. Do not force or manually overwrite a managed file.

Apply only after a clean WhatIf:

```powershell
pwsh -NoProfile -File $script -ProjectRoot <TARGET_PROJECT_ROOT>
```

Record the emitted manifest:

```text
<TARGET_PROJECT_ROOT>/.trellis/.migrations/pi-subagents/<timestamp>/manifest.json
```

Rollback preflight:

```powershell
pwsh -NoProfile -File $script -RollbackManifest <ABSOLUTE_MANIFEST_PATH> -WhatIf
```

Use a full rollback only when validation fails or the project owner requests it. Prefer testing full rollback in a disposable copy.

## Provider-Free Checks

After injection, start a new Pi session from the target or use `/reload`. Before launching a model child:

```powershell
Set-Location <TARGET_PROJECT_ROOT>
rg -n 'npm:pi-subagents@0\.46\.0' .pi/settings.json
rg -n 'thinking: medium|defaultContext: fresh|maxSubagentDepth: 0|nestedPiBoundary: unenforced' .pi/agents -g 'trellis-*.md'
rg -n 'TRELLIS_ENABLE_LEGACY_SUBAGENT|PI_SUBAGENT_CHILD' .pi/extensions/trellis/index.ts
Test-Path .pi/extensions/context-telemetry/index.ts
```

Run `/subagents-doctor` without dispatching a child. The default registry must contain `subagent`, `subagent_supervisor`, and `subagent_wait`, but not `trellis_subagent`. Never enable `TRELLIS_ENABLE_LEGACY_SUBAGENT` during normal validation.

For a Git target, record `git status --short` before and after. For a non-Git target, use hashes, file inventory, and validation output; do not claim staged-file or diff evidence.

## First-Use Integration Certification (Mandatory)

The checks above prove that injection and basic registration worked. They are **not** a first-use certification. Before this fork is used in a non-disposable project, a validating AI must complete the following certification in a disposable target project and write a target-local report. A single role canary, static registry output, or package documentation review is not sufficient.

Use four dispositions in the report. Each row must state its disposition, exact command or workflow, expected result, actual result, `PASS` / `FAIL` / `SKIPPED`, evidence paths, workflow/run/session/output/telemetry identifiers where applicable, and residual risk.

| Disposition | Meaning | Certification treatment |
| --- | --- | --- |
| `enabled-and-tested` | Trellis deliberately exposes this capability. | Run the scenario in the disposable target; it is mandatory. |
| `intentionally-disabled-and-rejected` | The package supports it, but the Trellis role contract does not. | Prove absence or expected rejection without changing target settings; it is mandatory. |
| `package-harness-only` | Public pi-subagents functionality outside the Trellis integration contract. | Exercise only through the separate package harness below. Never enable it in the target merely for coverage. |
| `environment-unavailable` | A required provider, UI, or host prerequisite is unavailable. | Record the exact missing prerequisite. This is inconclusive, never a certification PASS. |

### Certification Setup And Guardrails

1. Work only in disposable copies of the target. Create a target-local Trellis certification task and report before live children are launched.
2. Continue to use the generated portable `bash` contract. Do not replace it with `pwsh` unless the target owner separately authorizes and validates the explicit Windows provider overlay.
3. Use `functions.subagent` with `workflowScript`, `async:true`, `mission:false`, `context:"fresh"`, medium thinking, an explicit outer timeout, an absolute task-local output path, and explicit acceptance with `review:false` for every normal role child.
4. Each role task must begin with `Active task: <TARGET_TASK_PATH>`. Package transport may wrap this as exactly one `Task: Active task:` prefix.
5. Do not start a model child until the provider-free checks, the target task pointer, and the default-disabled legacy gate have passed.
6. Do not test control actions, retry, rollback, parallelism, or package-only APIs in a production target. Stop on target drift, unknown migration customization, provider failure, task-identity drift, extension leakage, or telemetry secrecy failure.

### Enabled Trellis Integration Matrix

All rows in this table are mandatory `enabled-and-tested` rows.

| Capability | Disposable scenario | Required evidence |
| --- | --- | --- |
| Contract and registry | Re-run the provider-free checks, `/subagents-doctor`, and inspect resolved role definitions. | Exact package version; `subagent`, `subagent_supervisor`, and `subagent_wait` present in the parent registry; no default `trellis_subagent`; three roles are fresh/medium/depth-zero with explicit telemetry only. |
| Four-stage role sequence | Run `trellis-research -> trellis-check -> trellis-implement -> final trellis-check` as four newly created, awaited fresh children. The implement task may modify only a disposable task-owned marker. | Four workflow child IDs; distinct sessions and absolute outputs; explicit acceptance/evidence and `review:false`; task transport identity; final check validates the marker and reports no unrelated production mutation. |
| Safe parallel read-only workflow | Run a separate `runs.all` workflow with exactly two read-only, no-harm checks writing different absolute reports. Do not run parallel writers. | Parent workflow ID; unique child indexes; distinct run/session/output/telemetry records; ordered aggregate result; before/after target hash or Git status proves no shared write conflict. |
| Async visibility and observation | Keep a harmless disposable child active long enough to inspect FleetView or `status`, then inspect `status` and a bounded transcript. Call `subagent_wait` using a wait/subscription appropriate to the host. | Workflow/run IDs; FleetView/status observation; transcript reference; status/event/output/session paths and terminal completion correlated to the same run. A zero-token JavaScript wrapper is normal. |
| Safe control lifecycle | On a separate harmless async disposable child, obtain a `steer` receipt, then `interrupt` it and record the documented interrupt lifecycle; on another disposable child, use `stop` and record terminal stopped lifecycle. Attempting `resume` on that stopped run must be recorded as the expected package refusal. pi-subagents 0.46 supports interrupt for a running top-level async run; if the host cannot deliver it, record the exact host limitation as `environment-unavailable`, not as a package PASS. | Steer request/delivery receipt; interrupt request and resulting lifecycle; stopped status/process-terminal evidence; resume rejection/error; no role recovery by resume; a later retry, if needed, is a new fresh run with a new explicit acceptance contract. |
| Acceptance, output and telemetry | Inspect each terminal role's resolved acceptance and requested evidence, task-local output, lifecycle artifact, and sibling `context-telemetry.jsonl`. | `review:false`; requested evidence is distinct from generic optional report examples; report JSON parses; telemetry agent/index/run match, monotonic sequence, no absolute session path/prompt/environment/model/provider/credential/cost, and `nestedPiBoundary: unenforced`. |
| Fresh context and extension isolation | Inspect every role launch contract/status/session metadata. | `fresh`, medium, depth zero, no role `subagent` grant, expected tools (`read, write, edit, bash`), telemetry extension plus package runtime only, and no ambient OM, auto-compact, MCP, or parent Trellis extension in children. |
| Injection and rollback | In separate disposable copies, validate fresh init and existing-project migration: `WhatIf`, clean apply, manifest verification, rollback `WhatIf`, and full rollback bytes in the disposable existing-project copy. Also run a deliberately customized managed-file migration to prove fail-closed behavior. | Before/after hashes or Git status; migration manifest; rollback result; no-write proof for custom-file rejection; no source-checkout modification. |

The historical one-child research canary is only the first row of this broader matrix. It cannot satisfy the sequence, parallel, lifecycle, rejection, migration, or package-harness rows by itself.

### Disabled Or Isolated Capability Matrix

These are mandatory `intentionally-disabled-and-rejected` rows. Prove them from role definitions, provider-free registry/preflight, child launch contracts, lifecycle artifacts, and filesystem state. Do not edit settings or broaden tools merely to make a rejection test run.

| Package capability or risk | Required target-project proof |
| --- | --- |
| Nested role fanout | No Trellis role declares `subagent`; `maxSubagentDepth: 0`; no child fanout run exists. Because the role has no callable `subagent` schema, the auditable PASS is provider-free resolved-tool absence plus a static/launch-contract record showing the attempted capability is unavailable; do not broaden the role to manufacture a live rejection. |
| Legacy execution | Default registry has no `trellis_subagent`; `TRELLIS_ENABLE_LEGACY_SUBAGENT` remains unset; no legacy process or warning-gated path is used. |
| Role resume | Trellis never resumes a role. A stopped child is terminal, and a stopped-run resume request is rejected. Failed/stale/interrupted work is restarted with a new fresh run and explicit acceptance. |
| Worktree and durable state | No target worktree, mission state, schedule record, or role-owned durable state is created. The normal workflow sets `mission:false`; no role receives `worktree:true`. |
| Extension-owned dispatch | No Trellis role or project extension uses the process-local RPC, structured delegation, prompt-workflow adapter, or automatic subagent dispatch. Do not claim process-local event-bus APIs are cross-process integration. |
| Coordination bridges | The normal role workflow does not invoke supervisor/intercom coordination. Parent package tools may be present in the registry, but no child bridge invocation is evidence of this Trellis contract. |
| Ambient providers | Children do not load Observational Memory, auto-compaction, MCP, or ambient project extensions. The context telemetry sidecar is passive and does not register a tool or command. |
| Shell boundary | The report states exactly `nestedPiBoundary: unenforced`. `bash` authority is not an OS sandbox and cannot prove prevention of arbitrary shell-launched nested Pi. |

### Separate Pi-Subagents Package Harness

Run the complete pi-subagents package test entry point in a separate disposable checkout, pinned to the exact version installed by the generated target. The authoritative 0.46.0 source identity is the official repository tag/commit `nicobailon/pi-subagents` at `4a2d5284a2ac6a6b0282059e756fc5ee8dbdd58c`. Before running tests, prove the package identity against npm metadata: `npm view pi-subagents@0.46.0 version gitHead dist.integrity dist.tarball --json` must report version `0.46.0`, that gitHead, and the recorded tarball integrity. Also record the target's resolved package path/version and package-lock or pnpm-lock integrity when present. A source checkout with only a matching `package.json` version is insufficient.

```powershell
$package = '<DISPOSABLE_PI_SUBAGENTS_CHECKOUT>'
Set-Location $package
git status --short
git fetch --tags --quiet origin
git checkout --detach 4a2d5284a2ac6a6b0282059e756fc5ee8dbdd58c
git rev-parse HEAD
(Get-Content package.json -Raw | ConvertFrom-Json).version
(Get-FileHash .\src\runs\shared\acceptance.ts -Algorithm SHA256).Hash
npm view pi-subagents@0.46.0 version gitHead dist.integrity dist.tarball --json
npm ci
npm run test:unit
```

The expected npm metadata for this release is `gitHead: 4a2d5284a2ac6a6b0282059e756fc5ee8dbdd58c` and `dist.integrity: sha512-hgldOVlaB05qXkQJRpp8wZCQ+TPZv4Xi+lu0Z2RYKRU3SaQmy3R9sCFM9myoqRSIgsLYKer41784BHbyh4QKIw==`; record the actual registry/tarball URL rather than assuming a registry mirror. Record the resolved source SHA, package version, source hash, exact commands, pass/fail/skip totals, named known platform baseline failures, and every new failure. A package-suite result is `PASS` only when the official suite completes and has no new or unexplained failures relative to that named baseline; known baseline-matched failures are `BASELINE-MATCHED`, not silently counted as clean. An incomplete or unavailable harness is `environment-unavailable` and makes certification inconclusive. Do not modify the installed package, global Pi settings, or target project while running the harness.

Classify package surfaces such as custom worktrees, durable missions/schedules, public process-local RPC, structured delegation, prompt-workflow integration, capability ceilings, background-work providers, and intercom/supervisor bridge behavior as `package-harness-only` unless a future explicit Trellis contract enables them. The package harness may use only its own disposable environment; it must not turn these capabilities on in the target to make coverage look complete.

### Certification Decision

Certification PASS requires every `enabled-and-tested` and `intentionally-disabled-and-rejected` row to pass, plus a package harness with a verified npm/source identity and no new or unexplained package-suite failures. Known, named, baseline-matched package failures are recorded as `BASELINE-MATCHED`; they do not become a clean suite PASS, but they may be accepted only when the target matrix is otherwise fully passed and the report explicitly names the residual risk. Any new package failure, identity mismatch, incomplete harness, or `environment-unavailable` row stops adoption and makes certification inconclusive or failed as appropriate. Any target `FAIL` requires correction in the source fork or rollback in the disposable target.

## Required Report And Boundaries

The target-local report must include the complete disposition matrix in addition to source URL/SHA, injection path, before/after hashes or Git status, WhatIf/apply/rollback results, registry, commands, and residual risks. Include every workflow ID, child run ID, child index, session path, output path, event/status path, telemetry sidecar path, acceptance result, and exact failure or expected rejection.

The generic pi-subagents acceptance prompt may show optional `noStagedFiles` and `diffSummary` examples even when they are not requested. They are non-gating report-format noise; do not treat them as runtime-verified Git evidence and do not add unsupported `scmMode` or `scmEvidence` fields.

Do not modify this source checkout during target validation. Do not access provider secrets. Do not push additional refs, create a PR, publish npm, enable legacy runtime, or claim a hard OS nested-Pi boundary while arbitrary shell authority remains enabled.
