# CLAUDE.md

Guidance for AI coding agents working in the **avalanchego** repository (the Go
implementation of an Avalanche node). Read this before making changes; it
captures how to build, test, lint, and conform to repo conventions so your work
passes CI on the first try.

## Repo at a glance

- **Module:** `github.com/ava-labs/avalanchego`
- **Go version:** `1.25.10` (pinned in `go.mod`; CI sources it from there). CGO is
  required (`CGO_ENABLED=1`).
- **Multi-module workspace** (`go.work`): the root module plus three grafted
  modules under `graft/` — `graft/coreth`, `graft/evm`, `graft/subnet-evm`.
  `graft/` is a staging area for repos being migrated into the monorepo. A
  fourth module, `tools/external/go.mod`, holds `go tool` CLIs and is
  intentionally **not** in the workspace (keeps tool deps out of the main module).
- **Build/dev tooling:** [Task](https://taskfile.dev) (`Taskfile.yml`), Nix
  (pinned toolchain), and Bazel (`bazelisk`) as an alternate build/test path.

### Active work on this branch (`rahulmutt/saevm-todos`)

The focus is **`vms/saevm/`** — the reference implementation of **SAE
(Streaming Asynchronous Execution, [ACP-194](https://github.com/avalanche-foundation/ACPs/tree/main/ACPs/194-streaming-asynchronous-execution))**.
Under active development, **no API-stability guarantees**. Key subpackages:

- `sae/` — core SAE VM (block builder, consensus, p2p, recovery, RPC, health)
- `cchain/` — the C-Chain VM built atop `sae.VM` (`state`, `tx`, `txpool`)
- `blocks/` — SAE block definitions; `saexec/` — execution; `saedb/` — storage
- `adaptor/` — exposes SAE as a Snowman `block.ChainVM`; `hook/` — block lifecycle hooks
- `gastime/`, `gasprice/`, `proxytime/` — gas-as-time accounting & pricing
- `txgossip/`, `intmath/`, `cmputils/`, `types/`, `params/`, `saetest/`, `docs/`

> **Note:** `vms/saevm/` has a **stricter dedicated lint pass** (`lint-saevm`,
> gosec G115). Hold this code to a higher bar. See `vms/saevm/docs/invariants.md`.

## Directory map (top level)

| Dir | Purpose |
|-----|---------|
| `snow/` | Avalanche/Snowman consensus engines; snow context & VM interfaces |
| `vms/` | VM implementations (platformvm, avm, evm, **saevm**, proposervm, …) |
| `chains/` | Chain manager — wires VMs into consensus |
| `network/` | P2P networking (peers, handshakes, routing) |
| `node/`, `app/`, `main/` | Node assembly and process entrypoint |
| `message/` | Wire message definitions & (de)serialization |
| `codec/` | Serialization framework (linearcodec, reflectcodec) |
| `database/` | K/V DB abstraction & backends (leveldb, pebbledb, memdb) |
| `x/` | Next-gen storage: `merkledb`, `blockdb`, `archivedb` |
| `wallet/` | Go SDK for building/signing/issuing transactions |
| `api/`, `indexer/` | HTTP/Connect API servers & clients |
| `genesis/` | Network genesis configs |
| `utils/` | Shared utilities (crypto, math, set, timer, logging, …) |
| `ids/`, `staking/`, `version/`, `upgrade/`, `subnets/` | Core supporting types/subsystems |
| `tests/` | Integration / e2e / load suites & fixtures |
| `simplex/` | Simplex consensus protocol integration |
| `proto/`, `connectproto/` | Protobuf / Connect RPC definitions |
| `graft/` | Staging for migrating external repos (coreth, evm, subnet-evm) |
| `scripts/` | Build/CI/dev helper scripts |

## Running tasks

Everything goes through the Task runner. You do **not** need `task` installed —
use the bootstrap wrapper (it uses `task` from PATH or runs it via `go tool`):

```sh
./scripts/run_task.sh <task-name>     # primary entrypoint
./scripts/run_task.sh --list          # list available tasks
```

Tasks are wrapped in the Nix dev-shell (`scripts/nix_run.sh`) when Nix is
available, which guarantees correct tool versions. CI does **not** call
`go test` directly — it always goes through these tasks.

## Build

```sh
./scripts/run_task.sh build           # -> ./build/avalanchego  (scripts/build.sh)
./scripts/run_task.sh build-race      # with -race
```

Run it: `./build/avalanchego` (mainnet) or `--network-id=fuji`.

Bazel path: `bazel-build`, `bazel-build-race`, `bazel-build-opt`.

## Test

```sh
./scripts/run_task.sh test-unit       # canonical: shuffle ON + race ON (scripts/build_test.sh)
./scripts/run_task.sh test-unit-fast  # NO_SHUFFLE + NO_RACE — fast local iteration
```

The canonical run is:
```sh
go test -tags test -shuffle=on -race -timeout=120s -coverprofile=coverage.out -covermode=atomic <targets>
```
**`-tags test` is required.** Env knobs: `NO_SHUFFLE`, `NO_RACE`, `TIMEOUT`
(default `120s`). Excludes mocks, proto, e2e, load/c, upgrade, reexecute.

Single package / single test (run directly):
```sh
go test -tags test ./vms/saevm/...
go test -tags test -race -run TestName ./path/to/pkg
```

Graft modules test from their own dir: `cd graft/coreth && ../../scripts/run_task.sh build-test`.

Other suites: `test-e2e` / `test-e2e-ci`, `test-upgrade`, `test-load`,
`test-fuzz` (smoke) / `test-fuzz-long`, `bazel-test`. Most accept extra args
after `--`, e.g. `test-load -- --load-timeout=30s`.

## Lint & format

```sh
./scripts/run_task.sh lint            # golangci-lint + license headers + custom checks
./scripts/run_task.sh lint-fix        # golangci-lint --fix
./scripts/run_task.sh lint-saevm      # stricter gosec pass on vms/saevm/...
./scripts/run_task.sh lint-all        # everything CI's lint-all-ci runs
```

`scripts/lint.sh` runs `golangci-lint run --config .golangci.yml` plus custom
checks (license headers, single-import, interface-compliance-nil, etc.). There
is **no separate `gofmt` task** — formatting is handled by golangci-lint
(gofmt + gofumpt).

> **macOS:** `lint.sh` needs GNU grep + GNU find on PATH:
> `brew install grep findutils`. BSD defaults will fail.

## Code generation

Regenerate and **commit** the output when you change the relevant inputs — CI
fails if the tree is dirty after regenerating.

| What | Command | When |
|------|---------|------|
| Mocks | `./scripts/run_task.sh generate-mocks` | changed a mocked interface (`go.uber.org/mock`) |
| Protobuf | `./scripts/run_task.sh generate-protobuf` | changed `.proto` files (buf 1.59.0, protoc-gen-go v1.36.10, protoc-gen-go-grpc 1.5.1) |
| Canoto | `./scripts/run_task.sh generate-canoto` | changed canoto-annotated types |
| Load contract bindings | `./scripts/run_task.sh generate-load-contract-bindings` | changed load-test contracts |

## Before you push — pass CI

CI runs through `lint-all-ci`, runs unit tests with race+shuffle, and runs
"check" jobs that regenerate artifacts then fail on a dirty tree. Run locally:

1. `./scripts/run_task.sh lint-all` (covers lint, lint-saevm, shellcheck,
   actionlint, yamlfmt, go-mod-tidy, go-version, require-directives).
2. `./scripts/run_task.sh test-unit` (root) and `build-test` in any touched
   `graft/*` module.
3. If you changed mocks/proto/canoto/contracts, run the matching `generate-*`
   task and commit results.
4. `./scripts/run_task.sh go-mod-tidy` if you changed dependencies (tidies root,
   `tools/external`, and all `graft/*`, then syncs go.work + bazel).
5. If you touched Bazel-relevant files: `./scripts/run_task.sh bazel-check-metadata`.
6. **Never** write `time.Add(...TauSeconds)` — a CI grep check (`tausecondslint`)
   fails on it. Use a `time.Duration` (e.g. `params.Tau`).

## Go coding conventions (enforced by `.golangci.yml`)

- **License header** required on every `.go` file (from `header.yml`):
  ```go
  // Copyright (C) 2019, Ava Labs, Inc. All rights reserved.
  // See the file LICENSE for licensing terms.
  ```
- **Tabs** for Go indentation (`.editorconfig`); 2-space for other files. LF
  endings, trailing whitespace trimmed, final newline.
- **Import grouping** (`gci`, strict order): standard → third-party → blank →
  `prefix(github.com/ava-labs/avalanchego)` → alias → dot.
- **Required import aliases** (`importas`): `utils/math` → `safemath`;
  `utils/json` → `avajson`.
- **Use `errors.New` not `fmt.Errorf`** when there is no `%` verb (and
  `fmt.Sprint`/`t.Log`/etc. over their `...f` forms with no directive).
- Many linters on: `errcheck`, `bodyclose`, `gosec`, `gocritic`, `staticcheck`,
  `spancheck` (must `.End()` spans), `prealloc`, `unparam`, `unused`, `revive`, …
- In tests, `usetesting` requires `t.TempDir()`, `t.Setenv()`, `t.Context()`
  instead of the `os.*` / `context.Background()` equivalents.

### Forbidden patterns

depguard-denied imports (use the replacement):
- `container/list` → `utils/linked`
- `github.com/golang/mock/gomock` → `go.uber.org/mock/gomock`
- `io/ioutil` → `io` / `os`

forbidigo-denied identifiers:
- `require.Error` / `assert.Error` / `*.ErrorContains` → `ErrorIs`
- `*.EqualValues` → `Equal`; `*.NotEqualValues` → `NotEqual`
- `require.Fail` / `require.FailNow` → `t.Fatal`
- `sort.Slice` / `sort.Strings` → the `slices` package

## Testing conventions

- **Use `github.com/stretchr/testify/require`** (not `assert` — a separate warn
  pass discourages `assert` and raw `t.Fatal`/`t.Error`). Common idiom:
  ```go
  require := require.New(t)
  require.NoError(err)
  require.Equal(want, got)
  ```
- **`testifylint`** runs near enable-all — prefer `require.NoError(err)` over
  `require.Nil(err)`, `require.Len`, etc.
- **Table tests** are standard, using `[]struct{name string; ...}` or
  `map[string]struct{...}`, iterated with `t.Run(name, ...)`.
- **Mocks:** generated via `go.uber.org/mock/mockgen` from `//go:generate`
  directives; the project is reducing mock usage and prefers narrow, local
  (unexported) mocks. Regenerate with `generate-mocks`.
- `gosec`/`prealloc` are skipped on test files.

## PR conventions (`CONTRIBUTING.md`)

- PRs open against `master`. Describe problem + solution; link the issue. Use
  draft PRs for early feedback; PRs won't merge unless CI is green.
- Don't start a large feature PR without prior maintainer feedback. Cosmetic /
  whitespace-only patches are generally rejected.
- Security issues go through `SECURITY.md`, never public issues.
- No mandated commit-message format; history uses concise summary lines, often
  scope-prefixed (`sae:`, `refactor:`, `ACP-236 (1):`) with the PR number in parens.

## Key files

`Taskfile.yml` · `scripts/run_task.sh` · `scripts/build.sh` ·
`scripts/build_test.sh` · `scripts/lint.sh` · `.golangci.yml` ·
`.gosec-golangci.yml` · `header.yml` · `go.work` · `CONTRIBUTING.md` ·
`.github/workflows/ci.yml`
