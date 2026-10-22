# estate-ci

The estate CI contract, gen-4: one `workflow_call` reusable workflow that every estate
repo opts into via a parameter block. The merge-blocking answer is exactly one required
check named **`gates`**. Shape generalized from the two proven cuts (hermes-workflows
PR#502 shard fan-in + fail-closed rollup; hermes-beads PR#20 floor/tip parallel legs).

**Spec:** `/home/hermes/.hermes/work/ci-nextgen/NEXTGEN-CI.md` — section (d) is the
parameter surface implemented here, section (g) dispositions every red-team patch. This
repo implements (d): the seven knobs, fail-closed `gates` rollup, admission teeth,
preflight, and the canary. `BREAK-GLASS.md` in this repo is the committed bypass spec.

## How a repo opts in — the `ci-contract:` pin line convention

A consumer's `.github/workflows/ci.yml` shrinks to a parameter block that calls this
reusable, and the file carries one pin comment line — the version this repo is opted in
at. A contract bump is a **visible per-repo PR** that moves only that line (plus any
knob renames it forces):

```yaml
# ci-contract: estate-ci@v1
name: ci
on:
  pull_request:
  push:
    branches: [main]
jobs:
  gates:
    uses: jacobhausler/estate-ci/.github/workflows/estate-ci.yml@v1
    with:
      legs: '[{"label":"tip","python":"3.12","hermes_pin":"tip"}]'
      components: '["py","pkg","doc"]'
      shards: auto
      lane_cap_s: 150
      budget: tests/budget.json
      paths: default
      artifact-gate: '{"provenance_dir":"tests/runs","artifact_name":"ledger","retention_days":90}'
```

Preflight (inside the run) enforces the convention: the pin line must exist, must parse,
must agree with the `@ref` in `uses:`, the ref must be a tag, the `with:` block may
carry no key outside the seven knobs, and no caller job may be named `gates` without
delegating to the contract (≤1 emitter per required name, or `DUPLICATE-EMITTER` red).

## The seven knobs

| # | knob | meaning |
|---|---|---|
| 1 | `legs` | JSON pin arms `[{label, python, hermes_pin, consumes?}]`; floor/tip lives here; default single tip arm |
| 2 | `components` | JSON list over `py, node, docker, go, doc, receipt, pkg`; the pins × components cross is the matrix — **D11: pins multiply only over components that consume them** (`consumes` per arm) |
| 3 | `shards` | computed, never hand-set: sticky LPT over registry+manifest, `ceil(gate_suite_s / lane_cap_s)`, estate ceiling 24; any literal number reds preflight |
| 4 | `lane_cap_s` | suite-time cap per leg, default 150 |
| 5 | `budget` | pointer to `tests/budget.json` (`pr_gate_wall_s`, `runner_min_per_pr`, `runner_min_wk`) |
| 6 | `paths` | admission specs, fail-closed five ways (below); default allow-spec is anchored `docs/**.md`, root `*.md`, `LICENSE*`, docs images |
| 7 | `artifact-gate` | JSON `{provenance_dir, artifact_name, retention_days}`, provenance dir default `tests/runs`, retention default 90 d |

Non-optional, not knobs: the single required name `gates`; the ledger grammar
`LEG <id> OK|SKIP|FAIL cases=N wall=S`; artifact names embedding run_id + `-a<run_attempt>`;
provenance path `tests/runs/<repo>/<sha>/<run_id>/attempt-N/`; `SKIP` markers with
`reason` + `digest`; per-case timeout = manifest wall ×3 (hook receives
`ESTATE_CASE_TIMEOUT_S`); hermetic full provisioning per lane (the hook's duty —
`--require-hashes` installs, cache input bytes only).

## Fail-closed law (what makes `gates` mean something)

- **No green without every marker, including SKIP markers.** The rollup (`gates` job)
  recomputes the expected marker set from the rendered matrix and demands one marker per
  row from the *current attempt*; a leg that did not speak is red, and a rerun of a
  subset reds `STALE_ATTEMPT` (artifact names carry `-a<run_attempt>`; stitched
  cross-attempt marker sets are detected).
- **Admission, five ways** (all teeth fall to FULL, never to a guess): empty/unparseable
  diff → FULL; deny-wins (`.github/**`, `tests/**`, source/lock/manifest, timing/budget/
  registry json, and `tests/md_inputs.txt`-listed consumers — this estate ships .md as
  runtime behavior) → FULL; non-PR events → FULL; machine shape (`graphify-out/**`-only)
  runs the output-shape validator plus edge-delta-size teeth; and the rollup recomputes
  admission **independently via the compare API pinned to the event's base…head shas**
  (S6) — event-sha ≠ live head yields **pending, not red** (run cancelled + commit-status
  pending; the amend re-run re-litigates).
- **Budget is an economic flag, not a contract violation (S5):** over-budget at the gate
  emits a ledger `BUDGET FLAG` + PR comment; red fires only at the weekly rollup on
  ledger-attributed overrun. `tests/budget.json` per repo at the pointer.
- **Preflight is needs-free and inside the required signal:** a caller typo (bad YAML,
  missing/conflicting pin line, unknown knob, hand-set shards, empty matrix, duplicate
  emitter) is a named `PREFLIGHT FAIL <reason>` red marker on `gates`, never a hang. On
  `.github/**` diffs this is the only signal the diff can get, and deny-wins routes those
  PRs to FULL where preflight runs.
- **Leg hooks:** a lane runs `tools/ci/component/<component>.sh` in the consumer repo
  (env: `ESTATE_LEG_ID`, `ESTATE_LANE`, `ESTATE_LANES`, `ESTATE_COMPONENT`,
  `ESTATE_CASE_TIMEOUT_S`); the hook reports `CASES=<n> WALL=<s>` in
  `.estate-leg-report`. No hook at the path → `SKIP reason=no-hook`. An `OK` marker with
  `cases=0`, or Σcases ≠ plan when `tests/timing.json` provides the plan, reds `FORGERY`.

## Canary

`.github/workflows/canary.yml` renders **all seven consumer parameter blocks** (one per
estate repo, mirroring the spec (b) target rows — including hb's D11 node-arm-once
shape) on every push to main and every pull request of this repo. A contract typo dies
there, not in hw. Each consumer block uses a unique `artifact-gate.artifact_name`
prefix — preflight reds duplicate prefixes within one caller file.

## Version law

`v1` = this contract. Tags are immutable pins; bumps cut a new tag and each consumer
moves its pin line in its own visible PR. `main` moves freely — pinning to `main` is not
a contract, and preflight refuses non-tag remote refs.

## P2 — honesty notes

- **This repo has no ruleset yet: no protection = honest.** The `gates` required-check
  ruleset for consumers (and for this repo) lands with the P2 cutover PRs; until then
  nothing here is merge-blocking and nothing here claims otherwise.
- Weekly full matrix (Mon 06:00 UTC), freshness receipts, orphan sweeper, and the weekly
  budget rollup are per-repo P2 wiring on the consumer side, not part of this reusable.
- The sticky-LPT packer writes consumer-side lane plans (P2 hw lane); the contract
  computes shard *count* today, per-lane case assignment lands with the packer.
- Machine-shape edge-delta teeth are FLAG-only in v1 (validator hook reds if missing).
