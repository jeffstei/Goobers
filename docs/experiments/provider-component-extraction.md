# Two provider-facing component extractions

Baseline: `3a1de891` (the committed docs-churn experiment, based on upstream
`1b0655b37dc18a0d9dea9ebd38161bea6fd45d37`). This follow-up tests whether the
benefits extend beyond a small Git/time command. No deployment, version-skew,
workflow YAML, or release changes are in scope.

## Selection and pass conditions, before implementation

- **File-contention ranking:** extracts the existing stable backlog ranking
  algorithm and the provider query used both by backlog selection and by
  implementation-context gathering. Existing unit tests are already simple;
  this is a control against claiming that extraction invented testability.
  The only hidden input in the provider query is the branch namespace.
- **PR-status reporting:** extracts status parsing, publishing, and result
  construction/persistence. The CLI still owns environment/default resolution,
  provider construction, credential checks, and typed stage-error reporting.
  Reuse the existing narrow `providers.PullRequestStatusPublisher` interface
  rather than introduce a second provider abstraction.
- Reviewed but deferred: review-verdict caching (trusted comments, provenance,
  sibling-state and label semantics), and startup memory reporting (small,
  platform-specific probe rather than a representative workflow component).

Pass conditions:

1. Existing assertions move with their implementation or remain as CLI/consumer
   coverage; provider calls, ordering, stdout, error handling and result shape
   remain unchanged.
2. Component tests run independently, and new provider failure/cancellation
   tests use per-test fakes rather than global factory substitution.
3. Both new packages have at least 50% fewer production dependency packages than
   `cmd/goobers`. Report the actual remaining provider dependency cost.
4. Warm-cache no-selected-tests overhead is at least 25% lower for each library.
   This is not a whole-CI performance target.
5. All three extracted libraries pass the Go race detector with the installed
   C compiler. Broader command tests must also be checked, with any failures
   reported rather than silently excluded.

Use a detached baseline checkout on the same Q: drive as the experiment,
separate private Go build caches under LOCALAPPDATA, and sequential measurements.
Record compiler, toolchain, exact commands, cold/warm samples, dependency counts,
source-size costs, and retained/new test scope. Disable test-result caching.
The existing measurement script records wall time, including process startup
and package `TestMain`, not just compiler time. This remains a shared-machine
experiment, not a controlled CI benchmark.

## Results

Both extractions meet the five bounded pass conditions. They make focused tests
cheaper to run, but do not demonstrate a faster full command suite, CI pipeline,
or independently upgradable service. PR-status is a particularly small component:
its packaging benefit is real, while its architectural benefit is modest.

Raw samples, dependency inventories, compiler details, build results, race
output, and final source hashes are in
[`provider-component-results.json`](provider-component-results.json).

### Dependencies and test overhead

Go 1.26.6, Windows/amd64, `CGO_ENABLED=0` for all timings:

| Measurement | Baseline command | Extracted command | Contention alone | PR-status alone |
| --- | ---: | ---: | ---: | ---: |
| Production dependency packages | 1,179 | 1,181 | 425 | 425 |
| Repository packages in that closure | 160 | 162 | 17 | 17 |
| Test dependency packages | 1,293 | 1,295 | 435 | 435 |
| Warm no-selected-tests median | 13.859 s | 13.940 s | 3.303 s | 4.633 s |
| Warm retained CLI scenarios median | 18.512 s | 17.994 s | Different workload | Different workload |
| Warm complete library tests median | Not applicable | Not applicable | 4.218 s | 4.143 s |
| Cold library tests, one sample | Not applicable | Not applicable | 54.485 s | 60.985 s |

Each new library has **64.0% fewer dependency packages** than the baseline
command. No-selected-tests overhead is **76.2% lower for contention** and
**66.6% lower for PR-status**, exceeding the 25% target. Unlike docs-churn's
84-package closure, both import the existing `providers` package and transitively
retain its journal, API, platform, and other dependencies. This is a smaller
boundary, not a lightweight provider-model-only layer.

The complete command gains two dependencies. Its overhead is essentially
unchanged. Retained CLI scenarios range from 17.857 to 20.444 seconds before and
17.766 to 19.603 seconds afterward; those ranges overlap. The apparent 2.8%
median improvement is not evidence of a reliable whole-command speedup.

The same retained CLI selector runs on both revisions. It covers PR-status,
implementation-context, and ADO stage credential behavior; the after version
also strengthens the existing ADO test's stdout/result assertions. The initial
baseline run included the subsequently moved pure tests as well, and has a
23.833-second median. That larger workload is preserved in the raw evidence but
is **not** used as the retained-CLI comparison.

Each timing uses three warm samples and disables test-result caching. The
libraries also have their own initially empty build caches. Baseline command
cache population overlaps the tail of compiler installation; later warm
comparisons and all after/library measurements run after installation completed.
The baseline and experiment are on the same Q: drive. Build-cache isolation does
not isolate the shared module cache, OS cache, disk, antivirus, or other users.
`-run '^$'` includes package loading and command `TestMain`; it is not pure
compiler time. The PR-status full-tests median being slightly below its
no-tests median illustrates the noise rather than a negative execution cost.

### Behavior retained and exercised

`internal/contention` owns the existing stable partition and open-PR file query.
Both backlog selection and implementation-context use it. The branch namespace,
previously read implicitly inside the query, is now passed by each caller.
Backlog still logs failures and falls back to FIFO; implementation-context still
reports a typed stage failure. Their intentionally different policies remain in
their adapters.

All six existing contention tests moved with their original assertions. Four
additional top-level library tests cover exact query fields and call order,
original errors, canceled context, no partial evidence after failure, empty
results, threshold clamping, stable order, duplicate-file counting, and unchanged
caller-owned input. A new command-level fake HTTP test checks that a custom
namespace is honored, unrelated PRs do not trigger file/check requests,
re-sweeps stay at the end, and provider failure preserves FIFO with a warning.
That new consumer test was added after timing and included in race validation.

`internal/prstatus` reuses `providers.PullRequestStatusPublisher` directly.
It parses states, forwards publication requests, constructs the string-valued
result, and writes it. The command still owns defaults, credentials, provider
selection, validation order, stdout, and stage-error classification. Publication
errors retain their identity; file errors remain separate from provider errors.

The existing state-alias test moved unchanged apart from its function name.
Three new top-level tests cover request/context preservation, provider error and
cancellation propagation, exactly one publish call, exact original JSON bytes
including escaping, and write failure without another publish. Existing CLI
rejection, dispatch, ADO credential-plane, and implementation-context tests
remain. The ADO pod-style test now checks exact stdout and the final JSON result,
including the existing command wrapper's `integrity` field.

These are local fakes and existing in-process HTTP fixtures, not live provider
or production-cluster tests. Unlike the first docs-churn experiment, this
follow-up does not claim full-executable before/after output comparison.

### Compiler and race checks

Installed user-scoped WinLibs POSIX/UCRT with:

```powershell
winget install --id BrechtSanders.WinLibs.POSIX.UCRT --exact --scope user `
  --accept-source-agreements --accept-package-agreements --disable-interactivity
```

Installed package: `16.1.0-14.0.0-r4`; compiler: GCC 16.1.0. Race checks explicitly
set `CGO_ENABLED=1`, `CC` to the installed `mingw64\bin\gcc.exe`, and put that bin
directory on the process PATH. No persistent Go configuration was changed.

All three complete library suites pass `go test -race -count=1`, including the
previously compiler-blocked docs-churn suite. Focused command race tests also
pass: both new components' consumers, docs-churn's original CLI scenarios,
branch-namespace resolution, ADO credential boundaries, and flag-order checks.
The raw evidence includes exact selectors. This is not a race check of every
product package.

Independent library statement coverage is 100.0% for contention, 93.3% for
PR-status, and 92.9% for docs-churn. Coverage is not compared to the monolithic
command package's percentage. Vet passes for the command and all three libraries.

### Costs and limits

| Core production source | Before | After |
| --- | ---: | ---: |
| Contention implementation | 168 lines in command | 171 lines in library |
| PR-status command | 145 lines | 96 lines |
| PR-status library | None | 66 lines |
| Combined core production lines | 313 | 333 |

The core costs 20 additional production lines, excluding small consumer import
and type-name changes. It introduces exported package APIs and another file
boundary to navigate. Existing fakes and pure tests were already useful: neither
extraction invented testability, and these new assertions could also have been
written inside the command package. The demonstrated advantage is running them
without compiling and initializing the entire command test package.

Separate full executable builds pass without stripping or embedded Portal:

| Single warm build sample | Baseline | After |
| --- | ---: | ---: |
| Wall time | 24.765 s | 18.515 s |
| Executable bytes | 147,162,112 | 147,156,480 |

One build sample each does not establish a build-speed improvement. The binary
is 5,632 bytes smaller in this run; the deployment unit remains the same binary.
No release, compatibility, dispatcher drain, or version-skew behavior changes.

Markdown-link and complexity gates pass. The complexity gate still reports its
pre-existing stale `internal/authoring/explain.go` baseline entry; it is untouched.
The full product suite, full CI, Portal build, live providers, and Linux/macOS
matrix were not run.

**Decision:** selective extraction remains worthwhile for focused development.
Contention is the stronger example because two command consumers share a real
query/ranking component. PR-status shows that even a small provider-facing
component can be tested independently, but does not justify mechanically giving
every command its own package. These results support bounded extractions, not
the entire proposed refactor at once.

### Reproduce

Use the committed baseline `3a1de891` in a separate worktree and the existing
`test/componentizationbaseline/measure.ps1`. Never clear shared Go caches.

```powershell
$env:CGO_ENABLED = '0'
& .\test\componentizationbaseline\measure.ps1 -Checkout $baseline `
  -Package './cmd/goobers' -Cache "$privateCache-baseline" `
  -Output "$artifacts\baseline-overhead.json"
foreach ($package in @('contention', 'prstatus')) {
  & .\test\componentizationbaseline\measure.ps1 -Checkout $PWD `
    -Package "./internal/$package" -Tests '.' -Cache "$privateCache-$package" `
    -Output "$artifacts\$package-tests.json"
  & .\test\componentizationbaseline\measure.ps1 -Checkout $PWD `
    -Package "./internal/$package" -WarmOnly -Cache "$privateCache-$package" `
    -Output "$artifacts\$package-overhead.json"
}
$env:CGO_ENABLED = '1'
$env:CC = $gccPath
$env:PATH = "$(Split-Path $gccPath);$env:PATH"
go test -race -count=1 ./internal/docchurn ./internal/contention ./internal/prstatus
```
