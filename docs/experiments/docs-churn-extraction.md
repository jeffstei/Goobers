# One-command extraction experiment

Baseline: `1b0655b37dc18a0d9dea9ebd38161bea6fd45d37` (updated upstream main).
This is a bounded experiment, not completion of the componentization epic.

## Selection and predictions (recorded before extraction)

Two candidates were inspected:

| Candidate | Effects and dependencies | Decision |
| --- | --- | --- |
| `docs-churn` | Git subprocesses, current time, watermark persistence, result output; eight existing CLI tests including path validation | Select: one deterministic workflow command with observable JSON and watermark behavior, no remote mutations |
| `select-source` | Provider reads/writes, cross-run journal, claim ledger, annotations, rollback of failed claims, decomposition records | Defer: stronger provider example, but substantially more coupled and riskier for a first experiment |

Move docs-churn execution and its data types into `internal/docchurn`. Keep
argument parsing, environment lookup, instance-path selection, help, and registry
wiring in `cmd/goobers`. Retain existing CLI tests and real Git fixtures.
The shared `firstNonEmpty` helper must remain available to other commands.
Git process construction stays in the command adapter; Git-output parsing belongs
to the library. Do not introduce a framework or change workflows.

Predeclared pass conditions:

- All existing docs-churn CLI assertions pass on both revisions.
- Library behavior can be exercised with a fixed clock and scripted Git answers,
  without environment changes, a demo instance, Git subprocesses, or package-wide
  test setup. Tests must include Git failure and output failure without watermark
  advancement.
- Library production dependency closure is at least 50% smaller than the
  baseline command package, with no executor/runner/provider dependency.
- Warm-cache test-build overhead (`go test -count=1 -run '^$'`) is at least 25%
  lower for the library. This measures build/loading overhead, not equivalent
  behavioral test execution.
- Full command compatibility remains mandatory even if the library is faster.
  Record command-test regressions over 10%, not hide them in the library result.

No CI critical-path improvement, production reliability gain, independent patching,
or fleet-wide applicability is claimed by this small experiment.

## Measurement methodology

`test/componentizationbaseline/measure.ps1` accepts a checkout, package, test
selector, output file, and a new private cache directory. It records the exact
revision, dirty paths, toolchain, dependencies, command, failures, and sample times.
It runs one cold-build-cache sample and three warm-build-cache samples with
`-count=1` (test results are never reused). It never clears the shared Go cache.
The module download cache and OS disk cache are not cold. Runs on this shared
Windows machine are indicative, not controlled CI benchmarks.

Baseline uses a separate detached worktree at the exact revision above, never the
user's primary checkout. Execute measurements sequentially to avoid competition
between the experiment's own builds. Keep full-binary build and race checks
separate from normal test timings.

## Results

Measured with Go 1.26.6, Windows/amd64, `CGO_ENABLED=0`. Raw samples, dependency
inventories, CLI comparison output, and mutation output are in
[`docs-churn-results.json`](docs-churn-results.json).

| Measurement | Main baseline | Extracted command | Library alone |
| --- | ---: | ---: | ---: |
| Production dependency packages, including standard library | 1,178 | 1,179 | 84 |
| Repository packages in that closure, including target | 159 | 160 | 3 |
| Test dependency packages, including generated test packages | 1,292 | 1,293 | 138 |
| Warm no-selected-tests median | 11.388 s | 10.764 s | 3.081 s |
| Warm existing CLI scenario median | 18.821 s | 17.317 s | Not the same workload |
| Cold existing CLI scenario sample | 174.142 s | 170.815 s | Not the same workload |
| Warm new library scenario median | Not available | Not measured here | 3.225 s |
| Cold new library scenario sample | Not available | Not measured here | 17.259 s |

The library dependency reduction is 92.9%, and the no-selected-tests median is
72.9% below the baseline: both numerical thresholds pass. The library's three
repository dependencies are itself, `internal/configboundary`, and
`internal/platform/durability`; there is no executor, runner, instance, or
provider dependency.

The complete command's dependency closure **increases by one**. It still needs
the same product and test setup. Its observed CLI median improves about 8%, but
the after-samples range from 16.979 to 20.039 seconds, overlapping the before
samples. Do not interpret this as a reliable whole-command or CI speedup.
The baseline checkout was on C: and the experiment on Q:, another reason not to
attribute small differences to code. Comparing the extracted command and library
on the same Q: checkout still shows substantially lower library-only overhead.

`-run '^$'` skips individual tests, not `TestMain`. These numbers include Go tool
startup, loading/build/link work, and any test-process setup. They are not pure
compiler benchmarks. The cold samples are one sample each; no distribution or
statistical confidence is claimed. The cold library and CLI workloads have
different tests and cannot be compared as equal behavioral work.

Separate full executable builds, without `embed_portal` or stripping:

| Measurement | Main baseline | Extracted |
| --- | ---: | ---: |
| One warm-cache `go build` sample | 15.993 s | 16.432 s |
| Executable bytes | 147,154,432 | 147,156,992 |

The executable grows 2,560 bytes; there is no demonstrated full-build advantage.
These build samples include compilation and linking, not isolated linker time.

## Behavior and testability

All eight original CLI tests were retained. Their assertions and real-Git
fixtures are unchanged; only moved type/persistence names were updated.
They passed on main and after extraction.

Two full executable builds were also run against the same fixed Git repository
and watermark. Eight additional comparison cases matched exactly on exit status,
stdout, stderr, result file, and retained watermark: stdout digest, file digest,
help, invalid duration, missing workflow, invalid workflow input, corrupt
watermark, and output-file failure. The fixture disables watermark advancement
and uses a future watermark to make the clock-dependent window deterministic;
the existing CLI tests separately exercise real advancement. No shipped YAML,
command registry, help text, schemas, result filename behavior, or exit-code
policy was intentionally changed.

The new package adds 13 scenarios across three top-level tests:

- Exact first-run, overlap, future-clock, empty-window, and disabled-advancement
  behavior.
- Failures at each of the four Git operations, failed stdout/file output, and
  corrupt watermark input, with no unintended watermark advancement.
- Result output occurs before watermark persistence, including a persistence
  failure after successful output.

These tests use per-test scripted Git responses, a fixed clock, and temporary
files. They run in parallel with no environment changes, current-directory
changes, subprocesses, demo instance, or package-wide `TestMain`.
The adapter still runs real Git; existing command fixtures still verify that.
The library's own suite covers 92.9% of its statements. This is not a comparison
against the old package's coverage percentage.

### Deliberate defect experiment

Temporarily replace:

```go
buffer := time.Duration(float64(sinceLast) * multiplier)
```

with division. All eight existing command tests still pass. The new
`TestRunWindowAndDigest/overlap` fails:

```text
got  --before=2026-09-26T07:30:00Z
want --before=2026-09-26T03:00:00Z
```

The defect was reverted and the complete focused suite passed afterward.
This demonstrates a useful new test, not an exclusive capability of packages:
we could inject the clock and add this assertion inside `cmd/goobers` too.
Extraction additionally lets that test compile and run without unrelated
command implementations. Existing pure digest helpers were already directly
testable inside package main; the refactor did not invent unit testing.

## Costs and limits

| Source surface | Before | After |
| --- | ---: | ---: |
| Command production file | 507 lines | 141 lines |
| Library production file | None | 417 lines |
| Total production lines for this family | 507 | 558 |
| Existing CLI test file | 349 lines | 350 lines |
| Added library test file | None | 222 lines |

Costs include 51 additional production lines, an explicit options/dependency
boundary, exported data/persistence names, and one more file boundary to navigate.
The command adapter deliberately retains the current flag/input precedence.
The library retains the stage's integer status-code and output-writer contract;
it is not a fully transport-independent domain API. Filesystem behavior still
uses real temporary files in tests rather than a new filesystem abstraction.

This small docs command is not representative of backlog claims, provider
mutation, or daemon recovery. It does not demonstrate that those families are
equally easy to extract. No whole-product reliability gain, reduced incident
rate, reduced CI critical path/runner minutes, or independently patchable
deployment has been established.

Targeted CLI registry, help golden, flag-order, and provider inventory checks
passed, as did `go vet` for both packages. The complete executable builds passed.
Repository Markdown-link and complexity gates also passed; the complexity gate
reported a pre-existing stale baseline entry outside this extraction, left alone.
The full product test suite, Portal build, and Linux/macOS matrix were not run.
Race validation was attempted but blocked: this environment has cgo disabled
and no `gcc`, `clang`, or `clang-cl` on PATH. No race-safety claim is made.

Follow-up: the [provider-component experiment](provider-component-extraction.md)
installed GCC and successfully ran the complete docs-churn library suite and
the existing docs-churn CLI scenarios with `-race`. This resolves that tooling
blocker without changing the original timing evidence or claiming whole-product
race coverage.

## Reproduce

Use separate worktrees for the baseline revision above and this experiment.
Use fresh private cache paths for cold samples; never clear a shared cache.
Outputs and caches should be outside OneDrive. Example PowerShell commands,
with `$baseline`, `$experiment`, and `$artifacts` set to your own paths:

```powershell
$measure = Join-Path $experiment 'test\componentizationbaseline\measure.ps1'
$cache = Join-Path $env:LOCALAPPDATA 'GoobersExperiments\reproduction'
& $measure -Checkout $baseline -Package './cmd/goobers' `
  -Tests '^TestDocs(Churn|Watermark)' -Cache "$cache-baseline" `
  -Output "$artifacts\baseline-cli.json"
& $measure -Checkout $experiment -Package './cmd/goobers' `
  -Tests '^TestDocs(Churn|Watermark)' -Cache "$cache-extracted" `
  -Output "$artifacts\extracted-cli.json"
& $measure -Checkout $experiment -Package './internal/docchurn' `
  -Tests '.' -Cache "$cache-library" -Output "$artifacts\library-tests.json"

# Reuse the corresponding already-primed private cache for empty-test runs.
& $measure -Checkout $baseline -Package './cmd/goobers' `
  -WarmOnly -Cache "$cache-baseline" -Output "$artifacts\baseline-overhead.json"
& $measure -Checkout $experiment -Package './internal/docchurn' `
  -WarmOnly -Cache "$cache-library" -Output "$artifacts\library-overhead.json"
```

Build each checkout's `./cmd/goobers` into distinct executable paths, then run
`test/componentizationbaseline/compare-cli.ps1 -Before <exe> -After <exe>
-Work <new-fixture-directory> -Output <json-file>`. These scripts retain their
named outputs and private caches so failures remain inspectable; remove only
those specific temporary paths when finished.

## Decision

Keep this as a reviewable, bounded prototype. It demonstrates independently
runnable tests and more precise failure-path coverage at a modest code-size
cost. It does **not** yet justify a broad registry/DI rewrite, migrating every
command family, or changing deployment/version-skew policy.

The next decision should be based on human review of this boundary: is the extra
API easier to work with than the original file? If yes, consider one more
representative family; if not, retain the improved test ideas without assuming
the whole architecture must change.
