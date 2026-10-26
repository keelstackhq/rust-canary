# Rust workspace

Generated from hull-rust 0.0.0+ga7ebb3f: a Cargo workspace, no Node toolchain.
Crates live in `crates/*`; versions, lints and profiles are set in the root `Cargo.toml`.

## Commands

The CI steps, in order (run them all before saying a change is done):

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets --locked -- -D warnings
cargo deny --locked check bans licenses sources
cargo nextest run --workspace --locked --profile ci
cargo test --workspace --doc --locked
```

Also: `cargo fmt --all` (format), `cargo nextest run -p <crate>` (one crate),
`node scripts/agents-index.mjs` (regenerate `docs/agents/index.md`; CI checks it).

## Rules

- Test first: write the failing test, watch it fail, implement, watch it pass.
- Verify with real output before saying a task is done or fixed.
- Keep files small and focused; one task per change; no drive-by refactors.
- Dependencies are exact pins (`=x.y.z`); `--locked` everywhere; no `unsafe` (workspace lint).
- Files owned by the template (configs, workflows, `.claude/`) are updated by `splice update`;
  put local changes in seed files, `docs/agents/project.md` or a crate's `AGENTS.md`.
- Never print or commit secrets; never read `.env*`.
- Releases are made by the release workflow (hoist) after a human merges the release PR; never
  publish, tag or push tags yourself.
- At most 2-3 parallel agents, on files that barely overlap.
- Before a long task write the next steps to `docs/agents/project.md`; start fresh rather than
  compacting.
- A hook `ask` becomes a denial in `-p`/unattended runs: do the change by hand or run interactively.

## Crates

- `crates/common`: shared library code (seed, unpublished).
- `crates/*` (rust-cli): distributable CLI skeleton with one JSON object on `--json`, fixed exit
  codes and cargo-deny; see `crates/<name>/AGENTS.md`.

## Where to look

- `docs/agents/project.md`: repo-specific notes; read first.
- `docs/agents/index.md`: generated map of commands, crates and docs.
- `<dir>/AGENTS.md`: notes for one crate.

<!-- splice:bosun:begin — managed by `splice update --channel bosun`; edit the source -->

## bosun: agents in this repo

Agents write all the code here. bosun owns the process: you call a `bosun` command and act on
the one JSON object it prints (`ok`, `exit_code`, `data`, `error.fix`, `next`). Exit codes:
0 ok, 1 error, 2 usage, 3 conflict, 4 halted, 5 needs-human, 6 limit. Always do what
`error.fix` says; never work around a refusal.

### The loop for one task

1. `bosun status` — see sessions, leases, halt and main-red.
2. `bosun claim <issue>` — exit 3 means another session holds an overlapping lease; pick
   another task (`bosun wave plan <epic>`).
3. Work in your own workspace (`berth up <task>`) with `BOSUN_SESSION=<session>/<task>` set, so
   your lease is told apart from your orchestrator's and from other wave agents'. Run `bosun heartbeat` at least every
   20 minutes while you work.
4. Test first. Before every push run what CI runs, locally:
   `for g in fmt lint test deps test-integrity; do bosun gate "$g" || break; done`.
   CI minutes are scarce: push once per PR, when it is done and green locally; do review
   rounds locally (step 6) and push their fixes together. Do not edit the PR body or title
   just to retry CI.
5. `bosun pr <issue>` — opens or reuses the PR and moves the task to review.
6. Review: a fresh subagent reads `bosun review-pack <pr>` and returns the verdict JSON; post it
   with `bosun verdict post --from -` (in a cloud session, which cannot write statuses:
   `bosun verdict comment --from -`). Fix request-changes findings, then review again.
7. `bosun risk-check <pr>` runs in CI. A high-risk PR waits for an owner; ask with
   `bosun ask <issue> "<question>"`.
8. `bosun wait --pr <n>` until the checks are green, then `bosun pr <issue> --auto-merge`.
   GitHub merges it. You never merge, tag or force-push (the hooks refuse it).
9. `bosun release <issue>` if you stop before the task is done.

### Sessions

- An orchestrator starts a child only with `bosun spawn <epic>`, makes the `create_session`
  call from `next`, then runs `bosun spawn <epic> --record --from -` with the result.
- `bosun wait --child <session>` waits for a child's result; `bosun msg` returns a
  `send_message` call for steering.
- After a container restart, read your leases with `bosun status` and continue.

### Rules

- Change `bosun:state/*` labels only through bosun commands.
- Never append to CHANGELOG.md or the Done list of docs/PROGRESS.md in a PR: add a one-line
  fragment `changelog.d/<name>.md` or `docs/progress.d/<name>.md` instead (name: the branch with
  `/` as `-`), so merging main never conflicts. Before a release, a docs PR to main runs
  `.bosun/fold-fragments.sh`; the release PR then moves Unreleased under the version.
- Never skip, delete or weaken a test to get green; test-integrity refuses it.
- Lessons are not in this file. A review pack carries the lessons that match your files.
- bosun's own rules and their reasons: https://github.com/keelstackhq/bosun/blob/main/docs/spec.md.

<!-- splice:bosun:end -->
