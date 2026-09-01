# GitHub Actions — pins, comments and what Renovate reads

Every extraction fact below was checked with `extract_deps` on a workflow file
(Renovate 44.42.1, 2026-09-01) unless marked otherwise. Re-run it on the
consumer's pinned version before quoting a `depType`: the `workflow` depType
and the widened `helpers:pinGitHubActionDigests*` bodies exist only from
44.43.0 (renovatebot/renovate#45443).

## The line to write

```yaml
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
```

Full 40-character SHA plus the **exact released version** that SHA resolves
to. Resolve it with `gh api repos/O/R/git/ref/tags/v4.1.1` (an annotated tag
points at a tag object — dereference it via `repos/O/R/git/tags/<sha>`) or
`git ls-remote --tags`. Never copy a `# v7` from an upstream README; never
`@latest` (nothing to track). This is exactly the shape Renovate's own pin
writes (`autoReplaceStringTemplate: {{depName}}@{{newDigest}} # {{newValue}}`),
so a bare SHA in a repo is hand- or tool-written and will not fix itself.

Exceptions:

- A first-party action with no release yet: `@main` plus a
  `TODO: SHA-pin after the first release`; pin in a follow-up PR the moment
  the first tag exists.
- Generated workflows (projen) cannot carry the trailing comment; keep SHA
  and version in the generator source and accept that Renovate does not
  manage those lines. projen also fills `ignoreDeps` with every action it
  renders, and `ignoreDeps` is repo-wide, so hand-written workflows using the
  same action stop updating too.
- Reusable workflows behind an OIDC trust policy: tag ref, no comment (below).

## What the comment does

| `uses:` line                                        | `currentValue` | datasource                           | what Renovate does                                                                                                                                                                                                                   |
| --------------------------------------------------- | -------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `o/r@<sha>`                                         | none           | —                                    | `enabled: false`, `skipReason: unversioned-reference`: never looked up, no PR, filtered out of the dashboard's "Detected dependencies". No `packageRules` entry can re-enable it (skipped deps are dropped before rules run). Edit the line. |
| `o/r@<sha> # v4.1.1`                                | `v4.1.1`       | `github-tags`                        | release tracking: version PRs move SHA and comment together; a `digest` update follows a moved tag                                                                                                                                  |
| `o/r@<sha> # main`                                  | `main`         | `github-digest` (`versioning: exact`) | branch tracking: one digest PR per upstream push. Only when following the branch is the intent; name the action's real default branch (`master` is common)                                                                            |
| `o/r@<sha> # v4`                                    | `v4`           | `github-tags`                        | extracted, but at major granularity: movement inside the major is compared against `v4`. Rewriting the comment to the exact release of the same SHA is a behaviour-preserving edit (consequence reasoned from the value, not observed on a PR) |
| `o/r@<sha> # updates checkout to v4 (Node 20)`      | none           | —                                    | prose is no comment: `unversioned-reference`, exactly like a bare SHA                                                                                                                                                                 |
| `o/r@v4.1.1`                                        | `v4.1.1`       | `github-tags`                        | tracked, unpinned; `pinDigests: true` turns it into the second row                                                                                                                                                                    |
| `./.github/actions/x`, `./.github/workflows/x.yml`  | —              | —                                    | no dependency at all (same-repo reference)                                                                                                                                                                                           |
| `$/.github/actions/x`                               | —              | —                                    | no dependency at all (self-repository syntax, see below)                                                                                                                                                                             |

Two comment shapes are accepted: a version token at the **start** of the
comment, optionally prefixed `renovate: `, `pin `, `tag=`,
`ratchet:owner/repo` or `@`; or a single bare token and nothing else (the
branch hint). `ratchet:exclude` is honoured. Anything else is a bare SHA to
Renovate. The comment is load-bearing, not decoration.

Diagnostic: an action present in the workflows but missing from the
dashboard is almost always a bare SHA. `renovate --platform=local
--dry-run=extract` on a clone lists every dependency with its `skipReason`.

Remediation order for an estate of bare SHAs: (1) the exact release whose
commit is the SHA; (2) otherwise the next newer release — exclude alias tags
(`v6`, `v6.0`) or an alias wins the tie; (3) `# main` only when the SHA is
ahead of the newest release, and when the commits in between touch no runtime
files (`dist/*`, `src/*`) downgrade to that release instead. Call out
separately any move that crosses a major.

## Pin, digest and version updates

- `pinDigest` freezes: it appends the digest the current tag resolves to
  right now — a no-change hardening PR. `digest` repoints to the newest
  digest of the same tag or branch — an actual upstream change. A version
  update moves tag and SHA together.
- `minimumReleaseAge` applies to `digest` and `pinDigest` updates through a
  dedicated code path (they do not go through the normal internal-checks
  filter).
- Every `pin` and `pinDigest` across all managers lands in one branch,
  `renovate/pin-dependencies` ("Pin Dependencies"). One PR per digest pin:
  `pinDigest: { groupName: null, branchTopic: "{{{depNameSanitized}}}-pin-digest" }`
  (`groupName: null` alone degenerates the branch name, because the default
  `branchTopic` is version-based).
- Two workflow files pinning different SHAs of one action converge through
  Renovate's digest updates on the next run; do not hand-align them.

## The presets

- `helpers:pinGitHubActionDigests` (inside `config:best-practices`): one rule
  `{ matchDepTypes: [...], pinDigests: true }` with **no** `matchManagers`
  (only the github-actions manager emits these depTypes). Body on 44.42.1:
  `["action"]`; from 44.43.0: `["action", "workflow"]`. Either way it pins
  reusable-workflow calls: `uses: owner/repo/.github/workflows/x.yml@v1.2.0`
  is `depType: action` before 44.43.0 and `workflow` after.
- `helpers:pinGitHubActionDigestsToSemver` extends it and adds, on the same
  depTypes, `extractVersion: "^(?<version>v?\\d+\\.\\d+\\.\\d+)$"` and a regex
  `versioning` with optional minor and patch. Effect: every digest update
  rewrites the comment to the exact release. The optional minor/patch tolerate
  actions that publish only major tags; they are not licence to write `# v4`
  when `v4.1.2` exists. A user rule copied from the older body with
  `matchDepTypes: ["action"]` stops covering reusable workflows on >= 44.43.0.
  Re-resolve with `get_preset_node` on the pinned version.
- The repository is the finest matchable grain: `depName` and `packageName`
  are `owner/repo`; the sub-path survives only in `replaceString` and the
  replace template. `matchPackageNames: ["Org/*/.github/workflows/*"]`
  matches nothing; below 44.43.0 a reusable workflow and its repo's root
  action are one identity; `matchFileNames` (the **calling** workflow file)
  is the only other lever. A URL in `matchPackageNames`
  (`"!https://github.com/actions{/,}**"`) never matches and excludes
  nothing — write `actions/**` or `matchSourceUrls`. `["Org/repo"]` and
  `["/^Org/repo(/|$)/"]` are identical (`compare_simulations`); write the
  string.
- `customManagers:githubActionsVersions` is unrelated to `uses:` lines (below).

## Reusable workflows and OIDC

- The `job_workflow_ref` claim carries the literal ref the caller wrote:
  `.../x.yml@refs/tags/v1.2.3`, `@refs/heads/main`, or `@<sha>`. A trust
  policy that admits only `refs/heads/main` / `refs/tags/v*` (AWS
  `sts:AssumeRoleWithWebIdentity`, workload-identity federations) rejects a
  SHA-pinned caller: `Not authorized to perform sts:AssumeRoleWithWebIdentity`,
  right after a `chore(deps): pin <owner>/<repo> action to <sha>` PR merged.
  Often only the `pull_request` leg fails while `push` keeps passing.
- Exempt the hosting repositories:
  `{ "matchManagers": ["github-actions"], "matchPackageNames": ["Owner/repo"], "pinDigests": false }`
  (enumerate repos, not `Owner/**`); on >= 44.43.0
  `{ "matchManagers": ["github-actions"], "matchDepTypes": ["workflow"], "pinDigests": false }`
  covers every reusable-workflow call and leaves root actions pinned. Place
  the rule after `config:best-practices` in `extends` so it wins.
- Callers write `@vX.Y.Z` with no comment (the tag is what Renovate tracks)
  and a comment naming the deviation from the SHA rule. Compensating control:
  a tag ruleset (restrict create, update, delete; the release App as sole
  bypass actor) plus release immutability — created **before** widening the
  trust policy to `refs/tags/v*`. Reusable workflows that do no OIDC exchange
  can stay SHA-pinned.
- Rollout order: (1) exemption rule merged; (2) revert `@<sha> # v1.2.0` to
  `@v1.2.0` by hand — `pinDigests: false` stops future pins only, existing
  digests stay; (3) close any re-proposed pin PR. Reversed, Renovate re-pins
  within hours.
- Prove it: `simulate`
  `{ manager: "github-actions", depName: "Owner/repo", packageName: "Owner/repo", depType: "action", datasource: "github-tags", currentValue: "v1", newValue: "v1", updateType: "pinDigest" }`
  with `keys: ["pinDigests"]` → `false` (use `depType: "workflow"` on >=
  44.43.0); `compare_simulations` against a third-party action that must stay
  `true`.

## Self-references

- `uses: ./...` is a local reference: no `depName`, no dependency. Renovate
  can never pin or bump it. Use it on purpose for a self-call that must keep a
  ref-based `job_workflow_ref`. Inside a reusable workflow the checked-out
  tree is the **caller's**, so `./` does not reach the workflow's own action;
  the working pattern there is `uses: Owner/repo@<sha> # vX.Y.Z` with a
  comment that Renovate keeps the pin current. A self-pin by tag never
  converges: tag `vN` always embeds a pin to `v(N-1)`.
- `uses: $/path` (GitHub's self-repository syntax; runner >= 2.336.0, not on
  GHES, forbids an `@ref` suffix) resolves at the running commit: nothing for
  Renovate to bump, and it removes a self-referential SHA pin structurally.
  Reported, not reproduced: the runner rejects `$/` inside composite-action
  steps, and a `$/` call resolves `job_workflow_ref` to the commit, which a
  `main`/`v*`-only trust policy rejects like a SHA.

## `with:` inputs (`uses-with`)

Versions passed to well-known setup actions are extracted natively as
`depType: uses-with`: `actions/setup-node` `node-version: 22` → depName
`node`, packageName `actions/node-versions` (`github-releases`, versioning
`node`); `pnpm/action-setup` `version:` → `pnpm` on npm;
`dtolnay/rust-toolchain` `toolchain:` → `rust-version`; `astral-sh/setup-uv`
and `jdx/mise-action` `version:`. Not every action is mapped — `extract_deps`
shows a `uses-with` row or nothing. No marker needed. They are a **second
copy** of a `mise.toml` value: two PRs, and a major-only `"24"` moves only on
majors. Prefer one source (`mise.toml` installed by `jdx/mise-action`) or a
shared `groupName`.

## `customManagers:githubActionsVersions`

A regex manager for `# renovate:` markers. File patterns:
`.github|.gitea|.forgejo/(workflows|actions)/**/*.ya?ml`, `workflow-templates/`,
any `action.ya?ml` — a composite action's `env:` block counts. The marker sits
on the line **above** a key ending in `_VERSION` (the regex is the marker,
whitespace, then `[A-Za-z0-9_]+?_VERSION\s*:`):

```yaml
env:
  # renovate: datasource=npm depName=renovate
  RENOVATE_VERSION: "44.42.1"
```

Optional keys: `packageName=`, `versioning=`, `extractVersion=`,
`registryUrl=`. Any datasource works (`datasource=crate depName=cargo-index`
for a `cargo install cargo-index@$CARGO_INDEX_VERSION --locked` step). Every
hard-coded tool version in a workflow or composite action without a marker is
a review finding. It **never touches `uses:` lines**: the SHA + `# vX.Y.Z`
pair is the github-actions manager's own replace template. `extract_deps`
cannot run regex managers — prove a marker by running the preset's
`matchStrings` against the file, or `renovate --dry-run=extract`.

## `actions.lock`

`.github/workflows/actions.lock` (`gh actions-lock`, experimental; upstream
main as of 2026-08-25 — check the pinned version) marks the workflows it lists
as `digestManagedExternally`: Renovate stops inline pinning there (the
extension would strip inline SHAs back out) and instead regenerates the lock
for upgrades of the `action` and `docker` depTypes (`workflow` from 44.43.0).
`docker://` images in `uses:` are not lock-managed and stay Renovate's to pin.
Repos on `actions.lock` need no `pinDigests` for those workflows.

## Freshness

Judge an action by its last release and last human commit
(`gh api repos/O/R/releases`, `gh api repos/O/R/commits/<default-branch>`),
not by `pushed_at`: JS actions execute a bundled `dist/index.js` compiled
from the lockfile at release time, so consumers run dependencies as old as
the last release regardless of bot merges on the default branch. A repo whose
only activity is Renovate or Dependabot merges is maintenance-by-bot.

## Tooling installed at run time

Never `npx <pkg>` (nor `pnpm dlx`, `yarn dlx`) in a CI step: it resolves the
package and its whole transitive tree from the registry at run time, outside
the lockfile, outside Renovate's release-age gate and outside the package
manager's cooldown. Preference: a native GitHub Actions feature > an
org-internal action > a devDependency in `package.json` with an exact pin, a
committed lockfile and `npm ci` / `pnpm exec`. Declare plugins Renovate must
track as direct dependencies even when a parent pulls them in. Do not commit
`node_modules` and do not restore it from a CI cache (`npm ci` verifies every
package against the lockfile's integrity hashes on each run). The one
sanctioned `npx` is a literal, marker-managed version:
`npx -y -p renovate@<pinned> renovate-config-validator --strict --no-global`.

## Platform enforcement

When the org's Actions settings leave `sha_pinning_required` off
(`gh api orgs/<org>/actions/permissions`), GitHub accepts floating `@v1` refs
for any allow-listed action, and `helpers:pinGitHubActionDigests` plus review
convention **is** the SHA-pin control: do not weaken it per repo. When
retiring an allow-list entry an internal action still needs, wait for that
action's release **and** Renovate's SHA-bump PR in every consumer — removing
the entry first fails closed with "action not allowed".
