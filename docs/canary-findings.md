# Canary findings: hull template-rust + bosun v0.2.0

Run on 2026-10-25 by a Claude Code cloud session on keelstackhq/rust-canary. Goal: a pure Rust
repo from hull's `template-rust` plus bosun's splice channel (`bosun init --stack rust`), with
green CI and one task walked through bosun as a user would. Tools: splice 0.2.0, hoist 0.2.0 and
bosun 0.2.0 from npm, cargo-nextest and cargo-deny from `cargo install --locked`.

## Result

The whole path works: hull template, bosun setup, a plan task and a feature task, all green, with
risk-check "normal risk" and a posted review-verdict on both bosun PRs. It took 6 PRs: hull render,
hull update, bosun setup, plan, feature and this doc.

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

F10. **A task keeps its review state label after a manual merge.** #10 was merged by hand, with no
auto-merge (F3). Plan task #8 is closed but still has the review state label, and `bosun reap`
reports nothing, because it moves only queued tasks to merged.
- Fix (bosun, `src/cli/reap.rs`): also move review and fixing tasks whose linked PR merged to
  merged, and end their leases with reason `merged`.

F11. **No docs-only fast path.** #10 changed one Markdown file under `plans/`. bosun still ran 8
jobs (69 s wall), and hull's `ci` ran in full (26 s), because `.github/scripts/docs-only.mjs`
counts only `docs/`, `.claude/` and root `*.md` as docs.
- Fix (bosun, `bosun.yml.jinja`): a `changes` job that skips the gate matrix, with `bosun-ok`
  still passing, when every changed file is under `docs/` or `plans/` or is a root `*.md`.
- Fix (hull, `template-rust/pieces/base/files/.github/scripts/docs-only.mjs` line 13): add `plans/`.

F12. **Work outside the plan has no bosun path.** This doc is not a plan task, so `bosun pr`, which
needs a task issue, cannot open its PR. The only entry is `bosun intake`, then a plan amendment PR,
then a task PR: three PRs for one doc. #12 was opened by hand. risk-check still judged it
("normal risk") and review-verdict still applied, so only the task bookkeeping is missing.
- Fix (bosun, `src/cli/pr.rs` and spec §4): `bosun pr --no-task --kind bookkeeping` for changes
  limited to `docs/**`, with a bookkeeping record instead of a lease.

F13. **The label-lock hook refuses any shell command that mentions a state label.**
`src/core/guard.rs` line 146 refuses every shell segment that contains the state label prefix
and does not start with `bosun`. A heredoc writing this doc was refused, because the doc names
the label. The MCP branch of the same guard already judges only label fields, so prose passes
there.
- Fix (bosun, `src/core/guard.rs` line 146): refuse only label writes, such as `gh issue edit` or
  `gh pr edit` with `--add-label` or `--remove-label`, `gh label`, and `gh api`, `curl` or `http`
  calls to an `/issues/{n}/labels` path, and keep the nested-command check.

F14. **The orchestration prompt asked for superpowers skills that this session did not have.**
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
