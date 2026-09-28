# Goobers componentization evidence

> **Status:** draft — supporting evidence
> **Spec:** Reproducible componentization measurements and observations
> **Authors:** Jeff Steinbok, GitHub Copilot
> **Owner:** @jeffstei
> **Area:** architecture, CI performance, binary composition
> **Updated:** 2026-09-25

Supports
[`goobers-componentization-analysis.md`](goobers-componentization-analysis.md).

This document records the measurements behind the componentization analysis.
It is intentionally separate from the recommendation so later remeasurement
does not require rewriting the decision narrative.

**Follow-up, 2026-09-28:** the measurements below remain the 2026-09-25 baseline.
See the frozen [docs-churn experiment](../experiments/docs-churn-extraction.md)
and [contention/PR-status follow-up](../experiments/provider-component-extraction.md)
for separately measured focused-test boundaries and their limitations. These
reports do not establish full-CI savings or production reliability gains, and
their prototype code is not included in this documentation branch. The
[revised recommendation](goobers-componentization-analysis.md#37-bounded-extraction-evidence-2026-09-28)
distinguishes those observations from hypotheses; no baseline values below
have been silently refreshed.

## 1. Measurement context

| Item | Value |
| --- | --- |
| Source revision | `1ccc0848d627c0931ce76816fcc9aa0fd989d5a4` |
| Go module | `github.com/goobers/goobers` |
| Go version declared by the module | `1.26.6` |
| Repository Go packages | 222 |
| Go source files | 3,590 |
| `cmd/goobers` dependency closure | 1,174 packages |
| Repository packages in that closure | 159 |
| Direct imports from `cmd/goobers` | 202 |
| Direct repository imports from `cmd/goobers` | 129 |
| Generated `goobers` man pages | 173 |

The measurements below used a warm developer build cache. Build elapsed time
is therefore evidence about relative shape on this machine, not a CI benchmark.
Binary and source sizes are deterministic enough to support the conclusions.

## 2. Executable measurements

The binaries were built with:

```powershell
go build -trimpath -ldflags '-s -w' `
  -o $output\goobers-core.exe ./cmd/goobers

npm --prefix portal ci --no-audit --no-fund
npm --prefix portal run build
go build -tags embed_portal -trimpath -ldflags '-s -w' `
  -o $output\goobers-full.exe ./cmd/goobers
```

| Artifact | Size | Observed warm build |
| --- | ---: | ---: |
| `goobers` without `embed_portal` | 101.31 MiB | 78.85 s |
| `goobers` with production Portal | 103.37 MiB | 25.57 s |
| Portal contribution to linked binary | 2.06 MiB | — |
| Existing `cmd/config-sync` binary | 35.39 MiB | 41.73 s |
| Existing `cmd/operator` binary | 31.64 MiB | 5.44 s |

The compiled Portal directory contained nine files and 2.32 MiB of raw data.
The main observation is not that Portal embedding is large: it is about two
percent of the linked executable. Most executable weight is code and dependency
closure, not static UI files.

Existing smaller commands also show why splitting can increase aggregate
release size. `config-sync` and `operator` are individually much smaller than
`goobers`, but shipping all three duplicates runtime and shared dependencies.
A deploy-time split must earn its operational cost through validation or patch
isolation; aggregate bytes will not necessarily fall.

## 3. `cmd/goobers` concentration

| Measure | Value |
| --- | ---: |
| Production files | 402 |
| Production source | 4.57 MiB |
| Test files | 644 |
| Test source | 6.80 MiB |
| Total files in package | 1,046 |
| Recorded top-level tests in the split table | 3,520 |
| Race timing weight | 1,520.058 s |

The package is both the composition root and the largest test ownership
boundary. It imports 129 repository packages directly, so a change to many
subsystems requires recompiling or retesting the same package even when the
public command is unrelated.

A filename-based estimate of production command-family concentration found:

| Family | Files | Source |
| --- | ---: | ---: |
| Core lifecycle and service/update paths | 45 | 0.71 MiB |
| Validation and authoring paths | 30 | 0.30 MiB |
| Portal/read/operations paths | 30 | 0.28 MiB |
| Deterministic workflow-stage commands | 134 | 1.83 MiB |

The categories overlap in responsibility and are not proposed package lists.
They demonstrate that deterministic workflow-stage commands are a substantial,
identifiable body inside the composition root.

## 4. Test and CI measurements

`.github/unit-shard-weights.json` records 67 packages with a combined
race-mode serial weight of 5,758.4 seconds.

The largest weights are:

| Package | Recorded seconds |
| --- | ---: |
| `cmd/goobers` | 1,520.058 |
| `release` | 499.080 |
| `internal/telemetry/rollup` | 416.540 |
| `internal/readmodel` | 373.286 |
| `internal/authoring` | 326.687 |
| `internal/journal` | 243.920 |
| `test/scale` | 210.701 |
| `internal/runner` | 197.296 |
| `internal/readservice` | 157.746 |
| `test/configvalidate` | 151.118 |

The repository has already exhausted ordinary package-level sharding for its
largest package. `.github/unit-shard-splits.json` divides:

- `cmd/goobers` into three test pieces;
- `release` into two test pieces.

The CI workflow runs five Linux race shards and describes the split as reducing
the modeled critical path from about 23 minutes to about 10 minutes. Commit
`fa0e8b2f` records why: package-level LPT could not place one package on more
than one runner, and `cmd/goobers` had roughly 3,500 serial tests. Commit
`98d730f0` then changed the weights to use race-mode measurements after
non-race timings proved misleading.

This history matters to the recommendation:

1. More runners cannot fully repair an oversized package boundary.
2. Extracting independently testable command packages can improve CI before
   any runtime split exists.
3. Runtime componentization without source/test ownership changes would not
   remove the current long pole.

The whole-tree coverage baseline is 676 seconds with a 900-second advisory
budget (`.github/test-timing-budgets.json`).

## 5. Candidate dependency closures

`go list -deps` produced:

| Target | All dependency packages | Repository packages |
| --- | ---: | ---: |
| `api/validate` | 383 | 33 |
| `internal/instance` | 470 | 54 |
| `internal/workflow` | 288 | 16 |
| `internal/runner` | 725 | 86 |
| `internal/engine` | 1,092 | 97 |
| `internal/httpapi` | 704 | 80 |
| `internal/readmodel` | 466 | 32 |
| `internal/readservice` | 697 | 73 |
| `internal/telemetry` | 618 | 27 |
| `internal/agentkit` | 386 | 35 |
| `internal/portalextension` | 102 | 1 |
| `internal/selfupdate` | 295 | 17 |
| `cmd/config-sync` | 903 | 43 |
| `cmd/operator` | 825 | 9 |
| `cmd/goobers` | 1,174 | 159 |

`internal/engine` is nearly as broad as the executable and is a poor first
deploy-time boundary. `internal/portalextension` is extremely narrow.
Validation and workflow compilation are materially narrower than the full
binary, but they are recovery-critical and widely imported, which favors
source-level extraction before optional delegation.

The most widely imported repository packages are:

| Package | Repository importers |
| --- | ---: |
| `api/v1alpha1` | 48 |
| `internal/journal` | 43 |
| `providers` | 23 |
| `internal/platform/durability` | 20 |
| `internal/instance` | 19 |
| `internal/workflow` | 14 |
| `internal/telemetry` | 13 |
| `internal/capability` | 13 |
| `api/validate` | 10 |

These packages are compatibility or infrastructure hubs. Turning them into
process boundaries first would create a large serialization and versioning
surface rather than isolate a coherent optional capability.

## 6. Embedded payload measurements

The exact `go:embed` selections reachable from the main executable account for
about 5.10 MiB of raw source/build data:

| Payload | Files | Raw size | Primary consumers |
| --- | ---: | ---: | --- |
| JSON Schemas and field purposes | 37 | 0.372 MiB | validation, schema/explain, workflow checks |
| Instance starter/demo/quickstart templates | 20 | 0.017 MiB | `init` |
| Scaffold templates | 5 | 0.004 MiB | `scaffold` |
| Canonical workflow examples | 12 | 0.070 MiB | `examples`, guided authoring |
| Tutorial sample | 12 | 0.029 MiB | onboarding |
| Agent toolkit source | 160 | 1.484 MiB | `agent-kit` |
| Portal extension source | 22 | 0.795 MiB | `portal-extension` |
| Production Portal build | 9 | 2.324 MiB | `dashboard`, getting started |
| systemd/launchd definitions | 2 | 0.006 MiB | `service install` |

At least 0.442 MiB is deliberately duplicated:

- schema files appear in `api/schemas.FS` and again in
  `AgentToolkitAssets`;
- the selected canonical `acme-web` examples appear in
  `configexamples.Files` and again through the complete `config-examples`
  tree in `AgentToolkitAssets`.

This duplication is a maintainability and size opportunity, but removing all
embedding would save only a small fraction of a 103 MiB executable.

## 7. Existing seams relevant to compatibility

### CLI registry

`cmd/goobers/runtime_capabilities.go` is the authoritative command registry for
dispatch, capability parity, and help. Generated man pages confirm a broad
surface. The entry point can remain stable while handlers are moved behind
private package or process adapters.

### Workflow-stage subprocess contract

Shipped and example workflows contain 163 textual invocations of `goobers`.
The shell executor already:

- recognizes `command[0] == "goobers"`;
- substitutes the active self binary;
- injects run, gaggle, workflow, repository, credential, config-generation,
  and journal-plane environment;
- supplies an executor-owned typed-error file;
- applies implicit result-file names from `internal/providerstage`;
- captures bounded stdout/stderr and exit status;
- lifts result files into typed outputs and artifacts.

This is an existing private process protocol even though the implementation
currently dispatches inside the same executable. A front executable can
delegate a command to a sibling helper while preserving the protocol.

Agentic harnesses also receive `GOOBERS_BIN`, so nested invocations continue to
hit the stable front executable rather than discovering helpers directly.

### Portal filesystem seam

`dashboardAssetFS` already accepts `--dev-assets=<dir>` and returns `os.DirFS`
instead of `portalassets.FS`. This proves the Portal server is already coded
against `fs.FS`; production externalization needs integrity and activation
policy, not a handler redesign.

The release pipeline already emits a separately checksummed
`goobers_portal_<version>.tar.gz`, even though the normal executable embeds the
same Portal. That artifact is useful prior art for a resource pack.

### Bundle-builder seams

`internal/agentkit.Build` and `internal/portalextension.Build` both accept an
`fs.FS`. Their command wiring currently passes embedded filesystems, but the
builders do not intrinsically require embedding.

### Update seam

Self-update currently:

1. downloads one platform archive and `SHA256SUMS`;
2. verifies the archive digest;
3. extracts only `goobers` or `goobers.exe`;
4. smoke-checks `version`, `validate`, and `config diff`;
5. copies the current executable to `updates/previous`;
6. atomically writes the candidate to `updates/current`;
7. starts it and observes clean heartbeats;
8. restores the retained previous executable on failure.

Component packs would have to extend this binary-only transaction. Updating a
helper independently without an atomic compatible set would weaken an existing
rollback guarantee.

### Service and container layouts

- systemd and launchd execute the absolute `goobers` path with
  `__service-supervise`;
- Windows service installation likewise resolves the current executable;
- Linux images place `goobers` and `goobers-operator` in `/usr/local/bin`;
- Windows images copy `goobers.exe` and `goobers-operator.exe`;
- both container images use `goobers` as the entry point.

Sibling helpers can fit these layouts, but discovery must be relative to a
verified component root, never ambient `PATH`.

## 8. Go plugin evidence

`go doc plugin` on the declared toolchain states:

- plugins are supported only on Linux, FreeBSD, and macOS;
- the race detector poorly supports them;
- plugins cannot be closed;
- the application and plugin are likely to crash unless toolchain, build tags,
  flags, environment, and common dependency source are identical;
- the standard library recommends considering IPC instead.

That directly conflicts with Goobers' required Windows support, race-test
posture, and independently patchable build goal.

## 9. Reproduction snippets

Counts and dependency closures:

```powershell
@(go list ./...).Count
@(go list -deps ./cmd/goobers).Count
go list -json ./cmd/goobers |
  ConvertFrom-Json |
  Select-Object ImportPath, Imports, GoFiles, TestGoFiles
```

Test-weight summary:

```powershell
$timing = Get-Content .github\unit-shard-weights.json -Raw |
  ConvertFrom-Json
$rows = $timing.packages.PSObject.Properties | ForEach-Object {
  [pscustomobject]@{ Package = $_.Name; Seconds = [double]$_.Value }
}
$rows | Sort-Object Seconds -Descending | Select-Object -First 15
($rows | Measure-Object Seconds -Sum).Sum
```

Exact source-tree sizes should be remeasured from the `go:embed` declarations,
not inferred from release archives, because several payloads intentionally
select subsets or duplicate files in another bundle.
