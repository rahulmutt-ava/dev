# Slice: Repo conventions, citable rules & verification

Cross-cutting. Use this to (a) name the exact rule a change violates and (b)
tell the author what to run. Much of this is also caught by CI lint — list those
under "Lint/CI will catch", not as headline findings — but naming the rule makes
a review actionable.

## Citable lint / forbidden-pattern rules (`.golangci.yml`, `scripts/lint.sh`)
- **License header** on every `.go` (except generated `*.pb.go`, `*.connect.go`,
  `mock_*.go`, `mocks_*.go`, `*mock/*.go`, `*.canoto.go`, `*.bindings.go`):
  `// Copyright (C) 2019, Ava Labs, Inc. All rights reserved.` /
  `// See the file LICENSE for licensing terms.`
- **depguard** denied imports → replacement: `container/list` → `utils/linked`;
  `github.com/golang/mock/gomock` → `go.uber.org/mock/gomock`; `io/ioutil` →
  `io`/`os`.
- **importas** required aliases: `utils/math` → `safemath`; `utils/json` →
  `avajson`.
- **forbidigo**: `require/assert.Error`/`ErrorContains` → `ErrorIs`;
  `EqualValues` → `Equal`; `NotEqualValues` → `NotEqual`; `require.Fail/FailNow`
  → `t.Fatal`; `sort.Slice`/`sort.Strings` → the `slices` package.
- **revive `string-format`**: `fmt.Errorf` with no `%` verb → `errors.New`;
  `fmt.Sprintf`/`Printf`/`t.Logf`/etc. with no directive → the non-`f` form.
- **usetesting** (tests): `t.TempDir()`/`t.Setenv()`/`t.Context()` not `os.*` /
  `context.Background()`.
- Custom `lint.sh` checks: `single_import` (no parenthesized single import);
  `interface_compliance_nil` (`var _ Iface = (*T)(nil)`, not `&T{}`);
  `require_no_error_inline_func`; `import_testing_only_in_tests`;
  `warn_testify_assert` (prefer `require`).
- Many linters on: `errcheck`, `bodyclose`, `gosec`, `gocritic`, `staticcheck`,
  `spancheck` (spans must `.End()`), `prealloc`, `unparam`, `unused`, `nilerr`.

## Two repo-specific gotchas
- **G115 / lint-saevm**: gosec G115 (integer-conversion overflow) is excluded
  repo-wide but **enabled for `vms/saevm/...`** via the `lint-saevm` task. Hold
  saevm conversions to that bar; bare `//#nosec G115` is a red flag everywhere.
- **tausecondslint**: a CI grep fails on `\.Add([^)]*TauSeconds`. Never write
  `someTime.Add(...TauSeconds)` — use a `time.Duration` (`params.Tau`).

## saevm invariants
The source of truth is `vms/saevm/docs/invariants.md`. For any `vms/saevm/`
change, check it against that doc (summarized in `vm-gas-economics.md`).

## PR norms (`CONTRIBUTING.md`, `.github/`)
- PRs target `master`; CI must be green to merge; cosmetic-only patches are
  rejected. Security issues go through the bug-bounty process privately, never
  public issues/PRs — flag any diff that publicly references an exploit.
- PR description template headings: *Why this should be merged* / *How this
  works* / *How this was tested* / *Need to be documented in RELEASES.md?*
- `CODEOWNERS`: `/vms/saevm` → `@ARR4N` `@StephenButtolph`; `/vms/saevm/cchain`
  → `@StephenButtolph`; `graft/*` & `RELEASES.md` → `@ava-labs/platform-evm`;
  everything else → `@ava-labs/platform-avalanchego`.
- Regenerate & **commit** generated artifacts when inputs change (mocks, protobuf,
  canoto, load-contract bindings) — CI fails on a dirty tree.

## Verification commands (put the relevant ones in the report)
All via the Task runner; `-tags test` is required for tests.
- Build: `./scripts/run_task.sh build` (or `build-race`).
- Unit (canonical, shuffle+race): `./scripts/run_task.sh test-unit`; fast:
  `test-unit-fast`. Single: `go test -tags test -race -run TestName ./path/to/pkg`.
- Graft module: `cd graft/coreth && ../../scripts/run_task.sh build-test`.
- Lint: `./scripts/run_task.sh lint`; **`lint-saevm`** (G115/gosec on saevm);
  `lint-all` (everything CI's lint-all-ci runs).
- Deps changed: `go-mod-tidy`. Bazel files touched: `bazel-check-metadata`.
- Codegen freshness: `generate-mocks` / `generate-protobuf` / `generate-canoto`
  / `generate-load-contract-bindings`.
- macOS: `lint.sh` needs GNU grep + find (`brew install grep findutils`).
