# CI pipeline

The two E2E consumers executed a balanced workload with all cases passing. The overall speedup remains inconclusive because UI results became the last job. Keep the rebalance and measure the UI path before adding capacity.

This ledger records the 2026-10-12 measurement of [ci.yml](../.github/workflows/ci.yml). Times come from GitHub run, job, and step timestamps and Swift Testing summaries. It distinguishes observed timings from estimates and attributes only the E2E execution change to the rebalance.

## Branch measurements

Both completed runs used the same source suites, Xcode 26.3, `macos-26` runners, 16 UI shards, and matching primary native, package, and agent-tool cache hits. The iOS build used a generic simulator destination in both runs.

| Cohort | Completed run | Measured head SHA | Samples |
| --- | --- | --- | ---: |
| Before | [38323530444](https://github.com/supabitapp/supaterm-legacy/actions/runs/38323530444) | `d4a28c49de1b45183eb5364a7ebb8342f7ea7890` | 1 |
| After | [38324980862](https://github.com/supabitapp/supaterm-legacy/actions/runs/38324980862) | `c733e6dce31bc2b6c5fb945cfeac61778e0b20db` | 1 |

| Measurement | Before median | Before p90 | After median | After p90 | Observed reduction |
| --- | ---: | ---: | ---: | ---: | ---: |
| Workflow creation to last job completion | 12:19 | 12:19 | 12:13 | 12:13 | 0:06 (0.81%) |
| Workflow creation to required check completion | 12:19 | 12:19 | 10:39 | 10:39 | 1:40 (13.53%) |
| Longest E2E consumer job | 5:51 | 5:51 | 5:18 | 5:18 | 0:33 (9.40%) |
| Longest E2E test execution | 283.865 s | 283.865 s | 216.834 s | 216.834 s | 67.031 s (23.61%) |
| Sum of active runner minutes | 92.38 | 92.38 | 91.95 | 91.95 | 0.43 (0.47%) |

With one run per cohort, median and p90 equal the same observation. They do not estimate a stable tail. Workflow creation excludes time before GitHub creates the run, and runner minutes exclude queued time.

The required check evaluates build and test outcomes without waiting for `mac-test-ui-results`. That job merges and inspects UI results and can finish afterward. Both endpoints are reported so an earlier green gate does not hide the remaining workflow time.

### Critical paths

Before: `changes (0:08) -> mac-test-e2e-build (5:53) -> mac-test-agent-e2e (5:51) -> required-checks (0:03)`, plus 0:24 of allocation gaps, totals 12:19.

After, last job: `changes (0:10) -> mac-test-ui-build (4:37) -> mac-test-ui (0) (5:25) -> mac-test-ui-results (1:32)`, plus 0:29 of allocation gaps, totals 12:13.

After, required gate: the same UI build and shard lead to `required-checks (0:04)`, totaling 10:39. The dependency graph has no edge from the required check to UI result inspection.

| Class on the last-job path | Before seconds | After seconds |
| --- | ---: | ---: |
| Gating: change detection, allocation gaps, required gate | 35 | 39 |
| Toolchain: generation, compilation, result inspection | 237 | 232 |
| Setup: checkout, tools, caches, artifacts, runner startup and cleanup | 164 | 185 |
| Execution: test command | 303 | 277 |
| Total | 739 | 733 |

The before test command includes its mock-server preparation and test-runner startup. The finer Swift Testing execution figures appear in the measurement table.

### Step timing and start offsets

Offsets are seconds from workflow creation. These steps cover the baseline E2E build and consumer plus the UI path that finished last afterward.

| Job / step | Class | Before start | Before duration | After start | After duration |
| --- | --- | ---: | ---: | ---: | ---: |
| E2E build / checkout | Setup | 22 | 14 | 23 | 18 |
| E2E build / macOS setup | Setup | 36 | 62 | 41 | 39 |
| E2E build / generation and native preparation | Toolchain | 98 | 81 | 80 | 56 |
| E2E build / build products | Toolchain | 179 | 156 | 136 | 85 |
| General E2E / agent-tool cache | Setup | 397 | 16 | 281 | 30 |
| General E2E / test command | Execution | 421 | 140 | 324 | 243 |
| Agent E2E / test command | Execution | 425 | 303 | 310 | 234 |
| UI build / build-only command | Toolchain | 84 | 185 | 86 | 200 |
| UI shard 0 / test command | Execution | 302 | 259 | 332 | 277 |
| UI results / artifact download | Setup | 619 | 13 | 662 | 22 |
| UI results / result inspection | Toolchain | 637 | 18 | 691 | 32 |

The unchanged UI build starts two seconds later; its slower compilation delays shard execution. The longest UI job grows by 20 seconds and result processing grows by 35 seconds. These timings explain the new last-job path without introducing another dependency.

The E2E build and unit job also finish sooner, by 126 and 118 seconds respectively. Their commands and cache keys did not change. Those differences confound attribution of the required-gate reduction to the rebalance.

## Rebalance and verdict

The change moves `CodexZmxE2ETests` and `ClaudeZmxE2ETests` into the general consumer, with serial execution enabled in both jobs. Complementary exclusion and inclusion lists retain every suite once.

| Swift Testing summary | Before general | Before agent | After general | After agent |
| --- | ---: | ---: | ---: | ---: |
| Passing tests | 52 | 14 | 55 | 11 |
| Passing suites, including the shared parent suite | 13 | 7 | 15 | 5 |
| Execution seconds | 116.635 | 283.865 | 216.834 | 206.712 |

The baseline moved suites total 83.746 seconds. The execution estimate was approximately 200 seconds per consumer; the longest job estimate was 4:30. Unit tests limited the projected full workflow to 11:15, saving 64 seconds. The projected runner-minute change was approximately zero because the same work moved between the same two jobs.

**Verdict: inconclusive for end-to-end speedup.** Observed full-workflow saving: 6 seconds. Projected saving: 64 seconds. Keep the change because the execution rebalance is measured, coverage is intact, and runner capacity is unchanged.

The longest consumer's execution reduction is within 20% of the execution saving estimated from the baseline. Its total job reduction is smaller because overhead rises from about 69 to 101 seconds in the general consumer. Both cache logs report primary-key hits; the agent-tool restore alone grows from 16 to 30 seconds.

Across the two runs, consumers spend 67-101 seconds outside Swift Testing execution. Checkout, tool restoration, artifacts, mock-server preparation, and test-runner startup contribute to that overhead. Both consumer jobs overlap in each run; the lever redistributes an existing parallel workload.

Workflow lint with shell checking, the 47 existing CI-script checks, selector validation, simplify, and both thermos reviews passed before the measurement push. Branch CI confirms the suite counts and successful iOS, unit, UI, E2E, and result-inspection outcomes. No CI-script tests were added.

## Historical context and workflow scope

The exploratory cohort contains 19 successful full PR runs, excluding skipped-only runs and reruns. Mixed layouts, coverage, and cache states make it unsuitable for the before/after comparison.

| Historical full PR cohort | Samples | Median | p90 |
| --- | ---: | ---: | ---: |
| Workflow creation to last job completion | 19 | 12:28 | 25:44 |
| Workflow creation to required check completion | 19 | 11:29 | 25:44 |
| Active runner minutes | 19 | 90.75 | 131.37 |

Run IDs: `38243780808`, `35924653618`, `34157705569`, `34155089468`, `34137078201`, `33962895303`, `33960979247`, `33960917627`, `33957508876`, `33956435746`, `33885900949`, `33805770413`, `33805567507`, `33773958725`, `33760078102`, `33747598778`, `33741413874`, `33692833628`, `33685942220`.

The first diagnostic branch run, [38322336774](https://github.com/supabitapp/supaterm-legacy/actions/runs/38322336774), passed every macOS check but failed the named iOS simulator destination. It is excluded from successful cohorts. The build-only iOS check uses a generic simulator destination, which passed in both compared runs.

The Apple workflow is reused by stable and tip releases. Web, compatibility, and documentation checks have separate path-filtered workflows. This branch changes the Apple workflow and leaves release, deployment, change-detection, concurrency, artifact, and coverage safeguards intact. Release and deployment after-measurements are outside this PR's branch evidence.

The separate PR workflows were surveyed with `gh api 'repos/supabitapp/supaterm-legacy/actions/workflows/<workflow>/runs?status=success&event=pull_request&per_page=20'`. Their historical layouts are context only; no improvement is claimed for them.

| Separate PR workflow | Samples | Median seconds | p90 seconds | Newest / oldest run |
| --- | ---: | ---: | ---: | --- |
| `deploy-web.yml` | 20 | 39.5 | 47 | [33957508890](https://github.com/supabitapp/supaterm-legacy/actions/runs/33957508890) / [33524228831](https://github.com/supabitapp/supaterm-legacy/actions/runs/33524228831) |
| `deploy-docs.yml` | 20 | 54.5 | 61 | [34157705528](https://github.com/supabitapp/supaterm-legacy/actions/runs/34157705528) / [33530082090](https://github.com/supabitapp/supaterm-legacy/actions/runs/33530082090) |
| `deploy-web-compat.yml` | 1 | 11 | 11 | [38311666504](https://github.com/supabitapp/supaterm-legacy/actions/runs/38311666504) |

Reproduce timings with `gh api repos/supabitapp/supaterm-legacy/actions/runs/<run-id>` and `gh api 'repos/supabitapp/supaterm-legacy/actions/runs/<run-id>/jobs?per_page=100'`. Use workflow `created_at` as the start and the latest active job `completed_at` as the end. Sum job execution durations for runner minutes; median uses the middle value and p90 uses nearest rank.

## Trend and remaining levers

The count includes test and support Swift files in `supatermTests`, `supatermE2E`, and `supatermUITests`. At the measured head, `git ls-tree -r --name-only HEAD apps/mac` lists 240, 25, and 35 respectively. Snapshot sources are outside this count.

| Week | Test and support files | Before median / p90 | After median / p90 | After minutes per 100 files |
| --- | ---: | ---: | ---: | ---: |
| 2026-10-12 | 300 | 12:19 / 12:19 | 12:13 / 12:13 | 4:04 |

File growth and a stable timing trend cannot be inferred from one comparable pair. Re-measure after 20 comparable warm-cache PR runs, at 330 files, or when the median full workflow exceeds 13 minutes, whichever comes first.

- Profile the longest UI suites and rebalance the existing 16 shards using actual suite durations. More shards add setup and require evidence before spending on capacity.
- Measure lightweight setup for UI result inspection and its artifact transfer. Preserve result merging and inspection; they are now on the last-job path.
- Investigate unit compilation when it becomes the longest job. Keep its separate test bundle from delaying E2E consumers until a shared-build experiment proves a shorter chain.
- Re-measure web and documentation paths independently when those paths become a bottleneck. The Apple comparison does not establish their performance gains.
