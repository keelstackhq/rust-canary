# Canary findings: hull template-rust + bosun v0.2.0

Run on 2026-10-25 by a Claude Code cloud session on keelstackhq/rust-canary. Goal: a pure Rust
repo from hull's `template-rust` plus bosun's splice channel (`bosun init --stack rust`), with
green CI and one task walked through bosun as a user would. Tools: splice 0.2.0, hoist 0.2.0 and
bosun 0.2.0 from npm, cargo-nextest and cargo-deny from `cargo install --locked`.

## Result

The whole path works: hull template, bosun setup, a plan task and a feature task, all green, with
risk-check "normal risk" and a posted review-verdict on both bosun PRs. It took 6 PRs: hull render,
hull update, bosun setup, plan, feature and this doc. The weak spots are after the merge: tasks
never reach the merged state when a human merges (F10), and `bosun pr <task>` can act on the
wrong PR (F14).

| Step | PR | Outcome |
| ---- | -- | ------- |
| hull render (earlier session) | #1 | merged; rendered from hull d25ab5e |
| hull update to a7ebb3f (guarded bosun hooks, hull#22) | #4 | merged; `ci-ok` green; applied by hand (F1) |
| `bosun init --stack rust` (R-gov setup PR) | #5 | merged by the owner after `/bosun approve <sha>`; `ci-ok`, `bosun-ok` green |
| plan task `C1/P` (`plans/rust-canary.md`, issue #8) | #10 | merged; risk-check "normal risk"; review-verdict approve 0.92 |
| feature task `C1/T1` (`hello --shout`, issue #9) | #11 | merged; risk-check "normal risk"; review-verdict approve 0.90 |
| this findings doc | #12 | see F12 |

The bosun flow for #10 and #11, as a user runs it:

1. `bosun join` (creates registry issue #6).
2. `bosun plan import plans/rust-canary.md` (epic #7, plan task #8 ready, task #9 planned).
3. `bosun claim 8`, commit the plan, `bosun pr 8`.
4. After #10 merged: `bosun plan import` again (promotes T1), `bosun claim 9`, test first, `bosun pr 9`.
5. Review for each PR: `bosun review-pack <pr>` goes to a fresh reviewer subagent, then
   `bosun verdict comment --from <file>`. verdict.yml applied it as `review-verdict` in about 20 s.

CI times (GitHub ubuntu-latest, warm cache):

| PR | hull `ci` job | bosun gates (wall, `bosun-bin` start to `bosun-ok` end) | jobs |
| -- | ------------- | ------------------------------------------------------- | ---- |
| #5 setup | 26 s | 83 s | hull 2, bosun 8 |
| #10 one `.md` file | 26 s | 69 s | hull 2, bosun 8, plus risk and main-open |
| #11 crate change | 27 s | 80 s | hull 2, bosun 8, plus risk and main-open |

## What worked

- `bosun init --stack rust --owner <login> --no-pr` in a hull repo did the right thing on the
  first try. It detected hull's `ci.yml` (`own_ci = true`, aggregate `bosun-ok`) and did not take
  `CLAUDE.md` (no `claude-md` piece). It excluded the five files other templates own, merged the
  four new required checks into hull's `hoist.toml`, and printed precise owner steps.
- All bosun gates (fmt, lint, test, deps, test-integrity) passed on hull's skeleton with no change.
- The setup PR's `bosun-bin` warns and uses its own pin when the base has none. That bootstrap
  path works.
- hull's guarded hooks (hull#22) are no-ops until `.bosun/bosun.toml` exists, so "hull update
  first, then bosun init" is a safe order. After setup the hooks were live (see F13).
- The owner's one-time path worked as documented: `/bosun approve <sha>` from the GitHub app,
  then a merge by hand.
- After setup, risk-check, main-open and review-verdict posted on every PR without help. A plan
  file and a crate change both came out "normal risk". `verdict comment` from a cloud session,
  applied by verdict.yml, works around the proxy's 403 on commit statuses.
- `bosun plan check` validates a plan offline with clear output. `bosun claim` took the lease
  from the plan's touch paths, and `bosun pr` linked the task and moved it to review.

## Friction points and fixes

Status: rust-canary pins bosun v0.2.0. Each bosun finding below was checked against bosun main at
7303091, which includes #217 (result states) and #218 (reap reads the `task-pr` record). All are
still open there, and #218 does not cover F10, because that record is written only by
`bosun pr --auto-merge`.

F1. **hull: no update path to an untagged template-rust.** hull v0.3.0 is not tagged. `splice
update` targets the latest stable tag, v0.2.2, and fails with "no splice.toml at `template-rust`".
`--to` takes only semver. `--to 0.0.0+ga7ebb3f` is refused as a downgrade from `0.0.0+gd25ab5e`,
because the `+g<sha>` build metadata is compared as text. `splice check` meanwhile reports
"update available: 0.0.0+ga7ebb3f → 0.2.2", so the weekly bot (`splice.yml`) will open a broken
PR. #4 was made by rendering `splice new --ref a7ebb3f` and copying the changed file and the lock
fields by hand.
- Fix (hull): tag v0.3.0 on main (already in hull `docs/PROGRESS.md` Blocked).
- Fix (splice, `src/ops/update.rs`, where "does not downgrade" is raised): accept `--ref
  <sha|branch>` for the main template, as `channel add` does. When the lock is `0.0.0+g…`, compare
  by git ancestry, not semver order. Never offer a tag whose tree lacks the template subdir.

F2. **bosun: `init` hardcodes the branch `bosun/init`** (`src/cli/init.rs` line 29). Claude cloud
sessions can push only to `claude/…` branches, so a user must use `--no-pr`, rename the branch and
open the PR by hand. That also loses the PR body init would write.
- Fix (bosun, `src/cli/init.rs`): add `--branch <name>` (default `bosun/init`). When
  `CLAUDE_CODE_REMOTE_ENVIRONMENT_TYPE` starts with `cloud`, default to `claude/bosun-init`. With
  `--no-pr`, return the PR title and body in `data` so they can be pasted.

F3. **Private repo on a free plan: no rulesets, so no required checks.** `hoist plan --area
github` reports "cannot read rulesets: Upgrade to GitHub Pro or make this repository public".
Without a ruleset, `ci-ok`, `bosun-ok`, `risk-check`, `review-verdict` and `main-open` are only
advisory, and auto-merge (D13) is unusable, so every merge here was manual. Neither hull's nor
bosun's docs say this.
- Fix (bosun, `src/cli/doctor.rs` and `init` owner_steps): detect a private repo without ruleset
  access and say "make the repo public or use a paid plan; the gates are advisory until then".
- Fix (hull README, repo settings; bosun `docs/PROGRESS.md` Blocked 10): one line with the same
  requirement.

F4. **bosun.yml installs Rust 1.97 while the repo pins 1.99.1.** The gate jobs run
`dtolnay/rust-toolchain` with `toolchain: "1.97"`, and then rustup switches to
`rust-toolchain.toml`, so each of the five gate jobs installs two toolchains.
- Fix (bosun, `channel/pieces/core/files/.github/workflows/bosun.yml.jinja` line 90): when
  `rust-toolchain.toml` exists, drop the dtolnay step and let rustup install from the file, as
  hull's `ci.yml` does. The same drift is in
  `channel/pieces/core/files/docs/setup-script.sh.jinja` (`RUST_VERSION=1.97`).

F5. **CI runs twice.** With `own_ci`, hull's `ci` job (fmt, clippy, deny, nextest, doc tests:
about 26 s, one runner) and bosun's gates (fmt, lint, test, deps, test-integrity: 69-83 s wall,
8 jobs) check almost the same things. Doc tests run only in hull's job, and test-integrity only
in bosun's.
- Fix (bosun, `bosun.yml.jinja` `own_ci` variant and spec §3): with `own_ci`, run by default only
  the gates the repo's own CI lacks (test-integrity), with `bosun-ok` still the aggregate, and add
  a `.bosun/bosun.toml` key to run all gates.

F6. **Agent rules outside R-gov.** In a hull repo, `CLAUDE.md` imports `docs/agents/project.md`,
`docs/agents/index.md` and `docs/agents/org-decisions.md`, and each crate has `AGENTS.md` and
`CLAUDE.md`. Of the agent-rule files, bosun's R-gov lists only the root `AGENTS.md`, `CLAUDE.md`
and `REVIEW.md`, so an agent PR can change the instructions every later agent reads, at normal
risk.
- Fix (bosun, `channel/pieces/core/files/.bosun/risk.toml.jinja`): add `docs/agents/**`,
  `**/AGENTS.md` and `**/CLAUDE.md` to R-gov.

F7. **Every hull update PR is R-gov.** The weekly `splice/update` PR always touches `.splice/**`
(often `.claude/**` too), so after bosun the owner must approve it every week. That is by design,
but nothing tells the owner.
- Fix (bosun `docs/PROGRESS.md` Blocked 10 and `init` owner_steps): "each weekly splice update PR
  needs `/bosun approve <sha>`".

F8. **`bosun doctor` reports the Claude GitHub App as missing when it is installed.** Step 2 looks
for an author login that starts with `claude` (`src/cli/doctor.rs` line 74). The app acts as the
owner's login: every PR here (#4, #5, #10, #11) is by the owner.
- Fix (bosun, `src/cli/doctor.rs` step 2): also count items whose `performed_via_github_app.slug`
  starts with `claude`.

F9. **`bosun doctor` reports "required checks: ok" when nothing enforces them.** Step 3 reads only
`hoist.toml` and the workflow files. Here no ruleset exists (F3).
- Fix (bosun, `src/cli/doctor.rs` step 3): read the live rules (`GET
  /repos/{o}/{r}/rules/branches/{default}`). Say "missing" when the required checks are not
  enforced, and "cannot verify" on 403.

F10. **After a manual merge, no bosun command moves the task to merged.** Without a ruleset (F3)
there is no auto-merge, so the orchestrator merged #10 and #11 by hand. Tasks #8 and #9 are closed
but keep the review state label, and their leases stay live until they expire. What each command
did:
- `bosun pr 9` on the task branch saw the merge (`"merged": true, "pr": 11`) but left
  `task_state: review`. `sync_merged` (`src/app.rs`) returns early unless the task is `queued`,
  and only `bosun pr --auto-merge` sets `queued`.
- `bosun reap`, before and after the leases expired, reported `merged: []` and `reaped: []`. It
  lists only queued issues (`src/cli/reap.rs`), so the expired lease on closed #8 was not ended
  either. It did remove this session from the registry (`left`).
- `bosun result 7 --outcome merged --pr 11` posted the result record on epic #7 but ended no lease
  (`lease_ended: false`: the lease is on task #9, not on the epic). Epic #7 stays open with the
  `ready` state, though its only task is merged.
- Fix (bosun, `src/app.rs` `sync_merged`): accept `review` and `fixing` as well as `queued` when
  the linked PR is merged. The state table in spec §4 gets a "review → merged: a human merged
  the PR" row.
- Fix (bosun, `src/cli/reap.rs`): also check review and fixing tasks, and closed issues with a
  live lease, and end those leases with reason `merged`.
- Fix (bosun, `src/cli/result.rs`): with `--outcome merged --pr N`, end this session's leases on
  the tasks that PR N links, and close the epic when all its tasks are merged.

F11. **No docs-only fast path.** #10 changed one Markdown file under `plans/`. bosun still ran 8
jobs (69 s wall), and hull's `ci` ran in full (26 s), because `.github/scripts/docs-only.mjs`
counts only `docs/`, `.claude/` and root `*.md` as docs.
- Fix (bosun, `bosun.yml.jinja`): a `changes` job that skips the gate matrix, with `bosun-ok`
  still passing, when every changed file is under `docs/` or `plans/` or is a root `*.md`.
- Fix (hull, `template-rust/pieces/base/files/.github/scripts/docs-only.mjs` line 13): add `plans/`.

F12. **Work outside the plan has no bosun path.** This doc is not a plan task, so `bosun pr`, which
needs a task issue, cannot open its PR. The only entry is `bosun intake`, then a plan amendment PR,
then a task PR: three PRs for one doc. #12 was opened by hand. risk-check still judged it
("normal risk"), and `verdict comment` works on it like on any PR, so only the task bookkeeping
is missing.
- Fix (bosun, `src/cli/pr.rs` and spec §4): `bosun pr --no-task --kind bookkeeping` for changes
  limited to `docs/**`, with a bookkeeping record instead of a lease.

F13. **The label-lock hook refuses any shell command that mentions a state label.**
`src/core/guard.rs` line 146 refuses every shell segment that contains the state label prefix
and does not start with `bosun`. A heredoc writing this doc was refused, because the doc names
the label, and so was a read-only `git show … | grep` whose pattern held the label prefix. The MCP branch of the same guard already judges only label fields, so prose passes
there.
- Fix (bosun, `src/core/guard.rs` line 146): refuse only label writes, such as `gh issue edit` or
  `gh pr edit` with `--add-label` or `--remove-label`, `gh label`, and `gh api`, `curl` or `http`
  calls to an `/issues/{n}/labels` path, and keep the nested-command check.

F14. **`bosun pr <task>` acts on the current branch's PR, whatever task it links.** `src/cli/pr.rs`
(lines 79-117 on main) looks up the PRs whose head is the current branch. When one is open, it uses
that PR without checking that the PR links the task, and it then moves the task's state.
Reproduction on rust-canary:
1. Task #9 has PR #11 (branch `claude/c1-t1-hello-shout`), merged.
2. `git switch claude/canary-findings` (open PR #12, which links no task).
3. `bosun pr 9` returns `{"pr":12,"created":false,"head_sha":"14fc979…","task_state":"review"}`,
   exit 0. `bosun pr 8` returns the same PR #12.

#9 was already in review, so nothing moved here. A task in `claimed`, `building` or `fixing` would
have moved to `review` on the wrong PR. With `--auto-merge`, bosun would write a `task-pr` record
naming the wrong PR and turn on auto-merge for it.
- Fix (bosun, `src/cli/pr.rs`): when the branch's open PR does not link the task
  (`linked_issues(&p.body)` lacks `n`), refuse with exit 3: "PR #12 on this branch links no task
  #9; switch to the task's branch, or add `Closes #9` to the PR". Also add a test in
  `tests/cli_pr.rs` for "open PR on the branch links another task or none".

F15. **Editing a PR's body cancels its bosun gates and leaves a red `bosun-ok`.** `bosun.yml`
runs on `edited` (for retargeted stacked PRs), and its concurrency group (`bosun-<pr>-<event>`,
`cancel-in-progress` on `pull_request`) cancels the running gate jobs. The cancelled run's
`bosun-ok` (`if: always()`) then reports failure on that head. On #12 the edit came 16 s after a
push. Run 40209956265 was cancelled and posted a red `bosun-ok`, and the run for the edit went
green after it. With the ruleset in place, the newest check decides, but the PR still shows a
failure notice, and the edit costs a full gate run.
- Fix (bosun, `channel/pieces/core/files/.github/workflows/bosun.yml.jinja`): run the jobs on
  `edited` only when the base changed (`github.event.action != 'edited' ||
  github.event.changes.base`). In `bosun-ok`, exit 0 with a notice when a needed job was
  `cancelled` by concurrency and the run's head is no longer current.

F16. **The orchestration prompt asked for superpowers skills that this session did not have.**
They are not in hull's rigging, so the work went ahead without them. This is not a hull or bosun
defect.

## Owner steps that remain (shortest list)

1. Make rust-canary public (Settings → General → Danger Zone → Change visibility), or move the org
   to a paid plan (F3).
2. Then, on your own machine with an admin token, in a checkout of main: `hoist apply --area github
   --yes`. It turns on auto-merge and delete-branch-on-merge, and adds a ruleset that requires
   `ci-ok`, `bosun-ok`, `risk-check`, `review-verdict` and `main-open`.
3. Tag hull v0.3.0 (F1), so the weekly splice update bot stops offering v0.2.2.

Optional:
- `SPLICE_BOT_TOKEN`; without it the weekly update PRs run no CI.
- Paste `docs/setup-script.sh` into the cloud environment.
- Create the three Routines from `bosun doctor`.

The `release` environment is needed only for the first crate release (`docs/agents/project.md`).
The `bosun:halt` label already exists.
