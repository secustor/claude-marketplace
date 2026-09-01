# Automerge gates — what must be true before a Renovate PR merges itself

`automerge: true` does one thing on GitHub-like platforms: with the default
`platformAutomerge: true` ("Controls if platform-native auto-merge is used",
default `true` — `get_option_docs` on 44.42.1) Renovate arms the platform's
auto-merge on the PR. Whether the PR then merges without a human is decided by
the repository's settings, not by the config. Renovate neither approves nor
bypasses anything. Say which of the two you mean: "automerge is configured"
versus "automerge actually merges".

Nothing in this file can be verified from `renovate.json`. A finding here is a
hypothesis until the `gh` recipe next to it has been run; report it as such
(`reference/report-format.md`).

## Independent blockers

Each one stops auto-merge on its own. A PR with every check green sits at
`mergeStateStatus: BLOCKED` until all of them are satisfied.

| blocker                                                                                        | symptom on the PR                                                      | note                                                                                                                                                                       |
| ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| repository setting `allow_auto_merge: false`                                                   | `autoMergeRequest: null` although the rule matched                     | an IaC import that lets it default to `false` silently disables every automerge in the repo (reported from an org's Terraform import; Renovate's option docs do not mention it) |
| ruleset or classic rule with `required_approving_review_count >= 1` and empty `bypass_actors`  | `reviewDecision: REVIEW_REQUIRED`, `BLOCKED`                           | a human approval, every time; an author cannot approve their own PR                                                                                                        |
| `require_code_owner_review` + a catch-all `* @org/team` CODEOWNERS line                        | `REVIEW_REQUIRED` with a requested team in `reviewRequests`            | see the carve-out below                                                                                                                                                    |
| `required_signatures`                                                                          | unsigned bot commits rejected                                          | the runner needs `gitPrivateKey`; commits made through the REST Contents API are unsigned                                                                                  |
| a required context no job reports (renamed or deleted job, `renovate/stability-days`, a non-Actions app) | `BLOCKED` with every listed check green                      | blocks every PR in the repo, forever                                                                                                                                       |
| merge queue whose required workflows lack a `merge_group` trigger                              | automerge enqueues; the queue never merges                             | merge-group check runs are separate from PR check runs                                                                                                                     |
| conflict with the base (`mergeable: CONFLICTING`, `mergeStateStatus: DIRTY`)                   | `gh pr checks` shows only external App checks; `gh run list --branch <head>` is empty | GitHub runs no `pull_request` workflow at all when it cannot build the merge commit; rebase, nothing is wrong with the workflows. Frequent where Renovate automerges into a directory an open PR rewrites |
| an org-level ruleset                                                                           | same as above, per-repo settings look permissive                        | classic `branches/<b>/protection` can report `reviews=none checks=0` while an active ruleset requires a code-owner review — query both                                       |

Not blockers (red PRs merge):

- A failing check that is **not** required. Observed: a PR merged with its
  `verify` job red and left `main` red for days; another merged 19 seconds
  after its CI run was created, because the branch had no required checks.
- `renovate/artifacts` = failure and the "Artifact update problem" comment.
  The PR still merges and `main` ends up with `package.json` ahead of the
  lockfile.

## Audit recipes

```bash
gh api repos/O/R --jq '.allow_auto_merge'
gh api repos/O/R/rulesets --jq '.[] | {id, name, enforcement}'
gh api repos/O/R/rulesets/<id> --jq '{rules: [.rules[] | {type, parameters}], bypass_actors}'
gh api repos/O/R/rules/branches/main                       # effective rules, org-level included
gh api repos/O/R/branches/main/protection --jq '{required_status_checks, required_pull_request_reviews, required_signatures}'
gh pr view <n> --json mergeStateStatus,reviewDecision,reviewRequests,autoMergeRequest,statusCheckRollup,mergeable
gh pr checks <n>
gh api repos/O/R/contents/.github/CODEOWNERS --jq .content | base64 -d
gh api repos/O/R/codeowners/errors                         # necessary, not sufficient
gh pr list -R O/R --app <bot-slug> --json number,title,reviewDecision,mergeStateStatus
```

Reading the answers: `autoMergeRequest` non-null means Renovate armed it.
`REVIEW_REQUIRED` plus a requested team is CODEOWNERS. `APPROVED` + `BLOCKED` +
`statusCheckRollup: FAILURE` is CI — no Renovate option fixes it.
`reviewDecision: null` with a pending required team is an org ruleset.

## Required checks that actually gate

- **Pin every required context to its reporting app.** A context stored with
  `app_id: null` (classic protection) is satisfied by any actor with write
  access posting a status of that name. GitHub Actions is `integration_id:
  15368`; reusable-workflow callers report composite names
  `<caller job> / <called job>`, and the required context must be that
  string. Verify: `gh api repos/O/R/branches/main/protection --jq
  '.required_status_checks.checks'` (each entry has `app_id`).
- **Never require `renovate/stability-days`** (Renovate's `minimumReleaseAge`
  status under its legacy name; it exists only on Renovate branches, so it
  blocks every human PR) **or `renovate/artifacts`** (exists only on Renovate
  PRs whose lockfile update failed). Drop other non-Actions contexts (CodeQL,
  security scanners) when codifying required checks from live PRs unless the
  app really reports on every PR.
- **Require a frozen-lockfile install on the PR itself:** `npm ci`,
  `pnpm install --frozen-lockfile`, `cargo ... --locked`. `npm install` hides a
  manifest/lockfile desync because it re-resolves. Where this check is
  missing, `renovate/artifacts` failing is Renovate telling you the lockfile
  did not follow — and the PR merges anyway.
- **The required workflow must invoke the test runner.** "DevDependencies
  with a test suite" is not enough: a test-runner major merged green because
  the required check ran only install, typecheck and synth and never called
  the runner, so the tests had silently stopped running. Verify:
  `grep -nE 'pnpm test|npm test|vitest|jest|ava|cargo test|pytest' .github/workflows/<required>.yml`,
  and run the suite on `main` once after a suspicious green major.
- **One terminal aggregator job per workflow file.** GitHub job results are
  `success | failure | cancelled | skipped`; `skipped` propagates from a
  failed upstream and is not `failure`, so a `!contains(needs.*.result,
  'failure')` gate stays green (observed: a preview job failed, the e2e job
  it fed was skipped, the aggregator passed, a compiler major automerged).
  Jobs in other workflow files can never be `needs:` of the aggregator.

  ```yaml
  ci-result:
    if: always()
    needs: [lint, test, build, docs-preview, test-e2e] # every job in THIS file
    runs-on: ubuntu-latest
    steps:
      - env:
          NEEDS: ${{ toJSON(needs) }}
          ALLOWED_TO_SKIP: '["docs-preview"]' # genuinely conditional jobs; say why each is here
        run: |
          echo "$NEEDS" | jq -e --argjson skip "$ALLOWED_TO_SKIP" '
            to_entries | all(.[];
              .value.result == "success"
              or (.value.result == "skipped" and (.key | IN($skip[]))))'
  ```

  Merge the aggregator first and flip branch protection to it only after it
  has reported once on the default branch; otherwise every open PR waits on
  a context that never reports. Open Renovate PRs pick it up on rebase.

## Triggers: what the gate cannot see

- **Push-only workflows are outside the gate.** A workflow that runs only on
  `push` to the default branch (`deploy.yml`, `publish.yml`) sees an
  automerged change for the first time after the merge, so its failure lands
  on `main`. An org audit of failed post-merge runs found 140 of 177 breakages
  in push-only workflows; 35 were in PR-gated workflows with a stale base or a
  non-required check; the worst case was 37 consecutive failures and 14 days
  red after a security bump. Before enabling automerge, inventory the
  triggers and either add a PR-time equivalent (build, install, plan/synth,
  integration tests) or exclude those dependency classes:

  ```bash
  grep -L -E 'pull_request|merge_group' .github/workflows/*.y*ml     # workflows automerge never waits for
  gh run list --branch main --status failure --json name,conclusion,createdAt
  ```

- **Merge queues:** every workflow named in the required checks needs
  `on: [pull_request, merge_group]`. After renaming or consolidating a
  required check, watch the first queue entries (one queue merged two
  Renovate PRs whose renamed check had just failed; cause unconfirmed).
- **Trust policies keyed on the branch:** a `pull_request` workflow that
  assumes a cloud role whose OIDC trust admits only `refs/heads/main` fails
  100% on `renovate/*` branches while `push` to main passes 100%; compare
  conclusions per event with `gh run list --json conclusion,event,headBranch`.

## CODEOWNERS carve-out

With `require_code_owner_review` on and a catch-all line, every Renovate PR is
`REVIEW_REQUIRED`. CODEOWNERS is last-match-wins and a later pattern with no
owner removes ownership for that path, so the requirement is vacuously
satisfied for PRs touching only those paths. The plain approval count,
required checks and signature rules still apply.

```
* @org/team

# Do not request review for dependency upgrades
package.json
**/package.json
package-lock.json
pnpm-lock.yaml
**/pnpm-lock.yaml
pnpm-workspace.yaml
Cargo.toml
Cargo.lock
.github/workflows/**/*.yml
.github/workflows/**/*.yaml
.github/actions/**/*.yml
.github/actions/**/*.yaml
renovate.json
mise.toml
mise.lock

# Approval and merge bots stay owned (four-eyes note below)
.github/workflows/auto-approve.yml @org/security
```

- Cover every ecosystem Renovate manages in the repo and both `.yml` and
  `.yaml`: a carve-out that covers only npm manifests still blocks the first
  `actions/checkout` bump. Add codegen output rewritten by
  `postUpgradeTasks` and release-please artifacts where used.
- Validate the file literally. A malformed first line (`- @org/team`, or a
  bare `@org/team` without a pattern) leaves the repo with no code owner at
  all; a markdown-mangled glob (`.github/workflows/\*_/_.yml`) silently
  re-adds the review requirement. `GET /repos/O/R/codeowners/errors` returns
  `[]` for both.
- It is evaluated from the PR's **base** branch: editing CODEOWNERS on the
  Renovate branch changes nothing. Land the carve-out on `main` in its own
  PR; the Renovate PR unblocks without a rebase.
- A repo with no CODEOWNERS has a vacuous requirement today; adding one later
  without the carve-outs newly blocks Renovate — ship them in the first
  commit.
- **Four-eyes caveat.** Once `.github/workflows/**` is owner-less, a
  workflow-only PR needs no code-owner review. A same-repo branch can rewrite
  an approval bot's workflow from `on: workflow_run` (runs default-branch
  code) to `on: pull_request` (runs the attacker-controlled head with repo
  secrets), delete its gates and obtain a real `APPROVED` review from the bot
  App — a live test did so in 3 seconds; a repo variable disabling the bot
  did not help because the head branch's copy ignores it. Rules: a bot that
  can approve or merge runs only from default-branch code (`workflow_run`),
  its workflow file is re-owned after the carve-out, and any `.github/**`
  carve-out is weighed against this.
- **Required reviews on bot PRs.** A recorded bot approval (a
  `gh pr review --approve` step with an App token that has `pull-requests:
  write`, org setting `can_approve_pull_request_reviews`) is an audit
  artefact; a ruleset bypass actor for the bot is the absence of a control
  and records nothing. Reserve automerge-with-bypass for repos whose Renovate
  rules and required checks are the real gate. Audit: `gh api
  repos/O/R/rulesets/<id> --jq .bypass_actors`; `gh pr view <n> --json
  reviews` on a merged Renovate PR shows who approved.

## Blast radius

- **Groups multiply it.** One bad member of a weekly automerged group reds
  the default branch for every later group, and root-causing means diffing a
  multi-ecosystem merge commit file by file (observed: 21 PRs across 16 repos
  broke `main`; all four permanently red repos were casualties of the same
  group). Grouping plus automerge needs a required PR-time check that
  exercises every member, or a smaller group, or the group opens for review.
- **Two green PRs break `main` together.** `typescript@7` merged green; a
  week later `@typescript-eslint/eslint-plugin@8.67` merged green in a group;
  only together they violate the plugin's peer range `typescript >=4.8.4
  <6.1.0`. Per-PR CI tests each change against the base at branch time, not
  the combination.
- **Then the lockfile rots.** Once the package manager cannot resolve the
  tree, the artifact update fails on every subsequent Renovate branch,
  Renovate commits `package.json` alone, and the lockfile drifts for
  unrelated packages until someone regenerates it. A frozen-lockfile required
  check (above) is what stops it.

  ```bash
  git log --format=%h -- package.json | while read c; do git show --stat --format= $c | grep -q lock || echo "$c package.json only"; done
  npm ci   # "lock file's X@a does not satisfy X@b"
  ```

- **The package manager's own cooldown** (pnpm `minimumReleaseAge`, minutes)
  is a second gate: an automerged group whose lockfile fails it
  (`ERR_PNPM_MINIMUM_RELEASE_AGE_VIOLATION`) leaves `main` red for every later
  branch. Same fix: the install runs as a required check.
- **IaC monorepos:** a batched push (grouped PR, merge-queue batch) on an
  apply workflow that derives its matrix from `HEAD^ HEAD` skips roots, and
  two runs without per-root `concurrency` interleave so the last writer wins.
  Before automerging: per-root `concurrency` with `cancel-in-progress: false`,
  diff range `${{ github.event.before }}..${{ github.event.after }}`, a lock
  timeout, a scheduled reconcile, and no drift guard that exits 0 whenever
  `modules/**` changed.
- **A change of `extends` governs every open PR on the next run.** Switching
  a repo onto an overlay that automerges more will merge a major PR the repo
  was deliberately holding open as soon as its checks pass. List open
  `renovate/*` PRs first, hold or accept them, and put the behavioural deltas
  (automerge scope, dropped groups, limits, schedule) in the PR body.
- **First-party fast lanes:** exempting the org's own packages from the
  release-age floor and automerging their non-breaking updates means a broken
  first-party patch reaches every consumer within the hour. The consumer
  needs a smoke test as a required check, or the exemption a short buffer.

## Renovate's own statuses

- `renovate/stability-days` — the `minimumReleaseAge` wait: pending while the
  release is inside the window, `pass` afterwards. A PR "waiting" with no
  failing check is usually this. Never a required check.
- `renovate/artifacts` — failure when the lockfile or another artifact could
  not be updated. In `gh pr checks` it renders with an empty workflow column
  (a status, not a job). The build jobs fail downstream because the lockfile
  is stale; read this one first. When the PR body is truncated the
  "Artifact update problem" block moves to a bot comment
  (`gh api repos/O/R/issues/<pr>/comments`). The branch is retried only when
  a package file changes, the branch conflicts, the rebase checkbox is
  ticked, or the title gets a `rebase!` prefix — fixing the cause on `main`
  does not regenerate the open PR.

## Triage order when a PR did not automerge

1. `gh pr view <n> --json mergeStateStatus,reviewDecision,reviewRequests,autoMergeRequest,statusCheckRollup,mergeable`.
   `autoMergeRequest: null` → automerge was never armed: either
   `allow_auto_merge` is off or no rule granted it (`get_provenance
   automerge`, then `simulate` — see step 6).
2. `REVIEW_REQUIRED` → rulesets and CODEOWNERS (recipes above).
3. `BLOCKED` with `statusCheckRollup: FAILURE` → `gh pr checks`;
   `renovate/artifacts` first; then ask whether the same job is red on `main`
   and on unrelated PRs (`gh run list --workflow <file> --limit 8`). A
   Renovate PR failing CI is often a red `main` in disguise, and a permanently
   red required check silently disables automerge for every later PR.
   Renovate branch names help filter history: groups land on
   `renovate/<group slug>`, single dependencies on `renovate/<dep>-<major>.x`.
4. `BLOCKED`, everything green, no review pending → a required context nobody
   reports (compare `rules/branches/main` with `gh pr checks`), a merge queue
   without `merge_group`, or `required_signatures`.
5. `DIRTY` → conflict; rebase.
6. Only now the config. `get_provenance automerge`; `simulate` the update
   type. `simulate` cannot evaluate `matchJsonata` — on 44.42.1 it reports
   the clause `no-match` with `readFields: []` for every update type, minor
   and patch included — so `finalDependencyConfig.automerge: false` from such
   a run is a false negative, not proof. Read the rule, or run
   `LOG_LEVEL=debug renovate --dry-run=lookup` and inspect the update object.
