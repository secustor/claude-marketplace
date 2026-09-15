# Debugging a Renovate run

Order of work when Renovate "does nothing" or does the wrong thing: read the
`skipReason`, then the preset error text, then the PR's own statuses, then the
config. Source references are Renovate 44.48.3; re-verify per pin.

## "Renovate ignores my dependency": read the `skipReason` first

The Dependency Dashboard is **not an inventory**. A dependency extracted with a
`skipReason` stays in the package-file results (counted in `depCount`, printed
in the debug log with its reason) but never reaches a datasource, never gets a
branch and is filtered out of "Detected dependencies". A manager that returns
`null` for a file drops the whole file, which then appears nowhere.

| signal                                                | meaning                                                                  |
| ----------------------------------------------------- | ------------------------------------------------------------------------ |
| absent from the dashboard, in the debug log with a reason | extraction worked, the value is unsupported                          |
| absent from the debug log                             | the file did not match a manager, or the extractor returned `null`       |
| `unversioned-reference`                               | bare-SHA action ref with no `# vX.Y.Z` comment; unrescuable by rules     |
| `unspecified-version`, `invalid-version`, `invalid-value` | the extracted value cannot be looked up (e.g. cargo `auto` -> `update-lockfile` with no `Cargo.lock` entry, debug line `No currentVersion or lockedVersion found for <pkg>`) |
| `unsupported-datasource`, `contains-variable`, `path-dependency` | the manager knows the dep but cannot update it                   |
| `github-token-required`                               | a `github-tags` lookup without `GITHUB_COM_TOKEN` (a local-harness artefact) |
| `ignored`, `disabled`, `internal-package`             | set at lookup: `ignoreDeps`, a rule with `enabled: false` (rcd labels the same outcome `package-rules`), a workspace package |

The full list is `lib/types/skip-reason.ts`. Recipes: rcd `extract_deps` with
the file name and contents for one file (built-in managers only; a
`customType: regex` manager is not supported there -- run its `matchStrings`
in node against the file instead); `--dry-run=extract` on a throwaway clone for
the whole repository (`validation.md`, "Whole-repo extraction");
`gh issue view <dashboard> --json body` to read the dashboard as Renovate wrote
it.

## Debug-log recipes

`LOG_LEVEL=debug LOG_FILE=<path>` as environment variables (not config keys:
the logger is built before the config file is read). One JSON record per line.

```bash
jq 'select(.msg == "Resolved shallow config, without merging internal presets") | .visitedPresets' run.log
```

That record carries `renovateVersion`, `config` (the merged repo config with
`extends` consumed) and `visitedPresets.merged` / `.unmerged` (internal presets
such as `config:recommended` land in `unmerged`; relative references appear
only in their canonical `source>repo//path#tag` form -- the strings
`ignorePresets` must match).

| log line                                                | means                                                                                               |
| ------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `Could not resolve relative preset reference` (WARN)    | `{preset, parentPreset, err}`; the entry was left raw and will fail in every consumer                |
| `Throwing preset error` (INFO)                          | carries the `validationError` text the user sees; names the exact preset string                      |
| `Preset fetch error` (debug)                            | the underlying fetch error before it is rewrapped                                                    |
| `Ignoring preset <x>`                                   | an `ignorePresets` entry matched; the string shown is the resolved form                              |
| `Already seen preset <x>`                               | ancestor-chain cycle guard; a preset back-referencing its own chain resolves once                    |
| `Failed to parse preset`                                | the `renovate-config` manager could not parse an `extends` entry (`skipReason: invalid-value`)       |
| `Repository has invalid config`                         | CONFIG_VALIDATION: branch list dropped, config-warning issue raised, zero updates                     |
| `Config migration necessary`                            | debug-only in a run (WARN in the validator)                                                          |

## Preset error messages

Every fetch failure is a CONFIG_VALIDATION error with one of these
`validationError` texts. When thrown below the repo's own config the suffix
`. Note: this is a *nested* preset so please contact the preset author if you are unable to fix it yourself.`
is appended. Where it surfaces: the repository worker logs
`Repository has invalid config`, raises the issue "Action Required: Fix
Renovate Configuration" and creates nothing -- look there and at the dashboard
first, then in the log for `Throwing preset error`.

| message                                                                                                                                          | cause                                                                                                                                                       |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Cannot find preset's package (X)`                                                                                                               | repo **or file** not found, fetch failed (a private repo without a token reads the same), malformed http URL, unknown local repo; on a runner < 44.29.0 also every relative reference |
| `Preset name not found within published preset config (X)`                                                                                       | the `:key` or sub-key is missing in the file                                                                                                                |
| `Preset is invalid (X)`                                                                                                                          | the string does not parse: `#tag`/`:sub`/source prefix on a relative ref, a `.`/`..` segment, `local>/org/x`, unbalanced parentheses, `.//base` from a templated path |
| `Sub-presets cannot be combined with a custom path (X)`                                                                                          | `//path` together with `:sub`                                                                                                                               |
| `Preset is invalid JSON (X)`                                                                                                                     | the file does not parse                                                                                                                                     |
| `Preset package is missing a renovate-config entry (X)`                                                                                          | npm preset without the key                                                                                                                                  |
| `Relative preset reference cannot be resolved (X). Relative presets can only be used within presets from a supported source, must stay inside their repository, and cannot be templated or used outside of a preset (for example in the repository config, inherited config, or globalExtends)` | a relative ref reached resolution raw: written in a repo config / inherited config / `globalExtends`, escaped the repo, templated path, or npm/http parent. The sentence lists all causes; the real one is in the author's WARN line |
| `Preset caused unexpected error (X)`                                                                                                             | anything else (a `TypeError` in the parser)                                                                                                                 |

rcd `explain_message` takes these by position from a `run_config`. Parsing
facts that explain a failed fetch: `github>o/r//x#v1(p1, p2)` is
`presetName x, tag v1, params [p1, p2]`; `github>o/r//a/b/c#feature/x` is
`presetPath a/b, presetName c, tag feature/x` (slashes in tags are fine); the
fetcher appends `.json` unless the name ends in `.json`, `.json5` or `.jsonc`.

## Relative-reference failure signature

- Author's run: WARN `Could not resolve relative preset reference`; consumer
  runs: the long message above naming the **entry** (`../oops`), not the
  parent. The consumer's escape hatch is `ignorePresets: ["../oops"]` -- the
  raw string, because that is what the entry still is.
- A templated path (`./{{ env.X }}`) is returned unchanged **silently** and
  fails with the same message even inside a preset; if the variable is unset it
  compiles to `.//base` and fails as `Preset is invalid (.//base)`.
- Runner older than 44.29.0: every `./x` is parsed as a local repo named `./x`
  -> `Cannot find preset's package (./x)`. Validator older than 44.29.0:
  `preset "./x" is not valid`. The fix is the runner pin, not the preset.
- Forms that were once misresolved and are hard errors from 44.29.0:
  `./a/..` (fetched `..json`), `./group(eslint)#v2` (tag discarded), `./x()`
  (leaked `{{arg0}}` literally). All three now `Preset is invalid`.
- In a Renovate checkout the parser can be executed directly:
  `NODE_ENV=test node --experimental-strip-types -e "const { parsePreset } = await import('./lib/config/presets/parse.ts'); console.log(JSON.stringify(parsePreset('github>some/repo//x#v1(p1, p2)')))"`.

## Preset caching

External presets are cached under `preset:<raw preset string>` (tag and params
included) in the per-run memory cache; with the global-only
`presetCachePersistence: true` they go to the package cache for a hard 15
minutes, shared across repositories. Internal presets are never cached.
`github>o/r`, `github>o/r#v2` and `github>o/r(x)` are separate entries, so a
tagged pin has its own copy while an untagged reference can serve a stale
default-branch copy. "Pushed a preset change, the next run did not see it" on
a persistent-cache runner is this; a runner without the option refetches every
run and a stale preset there is not a cache problem.

## rcd pitfalls

- **`accepted: true` means nothing while `stageStatus.preset` is `"error"` or
  `presetErrors` is non-empty.** `run_config` on
  `{"extends": ["github>org/private-repo#v1"]}` without `RCD_GITHUB_TOKEN`
  returns `accepted: true`, the digest "expanded into 0 presets. 1 preset could
  not be fetched", and every `get_provenance` / `simulate` answer describes an
  empty config (checked on 44.42.1). Read `stageStatus`, `errors[].topic` and
  `presetErrors` first.
- Without a token, read the preset directly:
  `gh api "repos/<org>/<repo>/contents/<file>.json?ref=<tag>" --jq .content | base64 -d`
  (a `base64: error decoding` on the 404 body is the "file does not exist"
  signal); `gh search code --repo <org>/<repo> <preset>` finds where a preset
  lives.
- `simulate` cannot evaluate `matchJsonata` (returns `no-match` with
  `readFields: []`), so `isBreaking`-based automerge rules cannot be proven by
  simulation; say so instead of reporting them as non-matching.
- `extract_deps` finds which **built-in** managers claim a file and cannot run
  a `customType: regex` manager.
- Option semantics come from `get_option_docs`, not from a shallow read of
  `renovate-schema.json` (top-level `properties` hides `$ref`'d definitions).

## Commit statuses Renovate sets

Two contexts, configured under `statusCheckNames`:

- `renovate/stability-days` -- the `minimumReleaseAge` wait: pending while the
  release is inside the window, pass afterwards. A PR "waiting" with no failing
  check is usually this.
- `renovate/artifacts` -- fail on a PR whose lockfile or artifact update
  errored. In `gh pr checks` it shows with an empty workflow column (a status,
  not an Actions job). Read it before blaming CI: the build jobs fail
  downstream because the lockfile is stale.

Neither exists on human PRs; never require them (`validation.md`).

The `### Artifact update problem` block sits in the PR body, **unless** the body
was truncated (`> This PR body was truncated due to platform limits.`); then it
is posted as a separate bot comment -- grep
`gh api repos/<o>/<r>/issues/<pr>/comments` for it first. The block states the
retry rule: Renovate retries the branch, including artifacts, only when
_any of the package files in this branch needs updating_, _the branch becomes
conflicted_, _you click the rebase/retry checkbox_, or _you rename this PR's
title to start with "rebase!"_. Fixing the root cause on the default branch
does not regenerate the lockfile in the open PR.

## When a PR did not automerge

```bash
gh pr view <n> -R <org>/<repo> --json autoMergeRequest,statusCheckRollup,reviewDecision,mergeStateStatus
```

1. `autoMergeRequest: null` -- Renovate never armed platform auto-merge: the
   rule did not fire (rcd `simulate` the update type), or `allow_auto_merge`
   is off on the repo. With `platformAutomerge: false` the field is always
   null; on a runner at 44.73.0 or later with a merge queue, Renovate
   enqueues the PR itself (log line `PR added to the merge queue`, GraphQL
   `pullRequest.mergeQueueEntry` non-null) and needs no `allow_auto_merge`.
2. `reviewDecision: REVIEW_REQUIRED` -- required reviews or
   `require_code_owner_review` with a catch-all CODEOWNERS line; no Renovate
   option fixes it. CODEOWNERS is evaluated from the PR's **base** branch:
   editing it on the Renovate branch head changes nothing; land the carve-out
   on `main` and the PR unblocks without a rebase. Commits created through the
   REST Contents API are unsigned and fail `required_signatures`.
3. `mergeStateStatus: BLOCKED` with `statusCheckRollup: FAILURE` -- CI. Check
   whether the same job is red on every other branch
   (`gh run list --workflow <file> --limit 8`, `gh pr checks` on an unrelated
   PR): a red Renovate PR is often a red `main` in disguise, and a permanently
   red required check silently disables automerge for every later PR.
   `BLOCKED` with every visible check green means a required context no job
   reports (`gh api repos/<o>/<r>/rules/branches/main` lists them).
4. Compare conclusions per event: a `pull_request` workflow that assumes a
   cloud role whose trust policy admits only `refs/heads/main` fails 100% on
   `renovate/*` branches while `push` passes 100%
   (`gh run list --json conclusion,event,headBranch`).
5. Non-required failing checks are **not** blockers: a red PR merges. `needs:`
   aggregators that go green on `skipped`, push-only workflows and merge
   queues without `merge_group` are the usual ways a check is not actually
   gating.
6. `automergeType: branch` behind a merge queue or merge train (44.73.0+):
   Renovate tries the push, and when the queue refuses it logs
   `automergeType=branch is not possible because the base branch only accepts
   changes through its merge queue - falling back to creating a PR` and opens
   a PR carrying the same hint. Fix in the ruleset (Renovate as a bypass
   actor with `bypass_mode: always`) or in the config (`automergeType: pr`).
   Behind a queue `rebaseWhen: auto` also resolves to `conflicted`, so a PR
   left behind the base is expected. The rest is in
   `best-practices/reference/automerge-gates.md`, "Merge queues and merge
   trains".

Branch names let you filter history: groups land on `renovate/<groupSlug>`
(`renovate/lock-file-maintenance`), single deps on `renovate/<depName>-<major>.x`;
`gh run list --json headBranch,event,conclusion` on the prefix shows whether
Renovate PRs pass CI at all. Run logs expire (`HTTP 410`) but
`gh api repos/<r>/actions/runs/<id>/jobs` conclusions survive.

## `renovate/*` branches

`pruneStaleBranches` treats every branch under `branchPrefix` (`renovate/`)
that the current run did not produce as stale:

- Open PR, all commits by Renovate or an author in `gitIgnoredAuthors`:
  retitled `- autoclosed`, closed, branch deleted.
- Open PR with a commit by any other author (a human, another bot): retitled
  `<title> - abandoned` and commented `Autoclosing Skipped -- This PR has been
  flagged for autoclosing. However, it is being skipped due to the branch being
  already modified.`
- No PR: `Deleting orphan branch`.

Exempt: `renovate/lock-file-maintenance` and `renovate/reconfigure`.
`renovate/configure` is the onboarding branch and, after onboarding, is never
in the branch list -- a hand-made PR there is claimed within a run. Renaming a
head branch through the API closes the PR. Rollout scripts must not open PRs
on `renovate/*`.

The other consequence of the same mechanism: a commit by an author not in
`gitIgnoredAuthors` marks a Renovate branch modified and Renovate stops
rebasing it. To fix a Renovate PR that cannot go green on its own, push the fix
onto its branch and merge as-is; letting Renovate recreate the branch discards
the fix.

## Self-hosted runner notes

- **`gitAuthor`.** A runner authenticating with a PAT and no `gitAuthor` logs
  `WARN: Using the default gitAuthor email address, renovate@whitesourcesoftware.com, is not recommended on GitHub.com ...`
  and commits as a Mend-owned identity with unsigned-commit flags; set
  `gitAuthor` (and `gitPrivateKey` when commits must be signed). A GitHub App
  token needs neither: Renovate derives
  `<name> <id>+<login>@users.noreply.github.com` from the App's bot user.
- **`postUpgradeTasks` design.** One rule per manager (`matchManagers: ["npm"]`
  -> lockfile-only refresh; codegen only for the group that needs it, selected
  through the shared `packages/` selector so every member triggers it),
  `executionMode: "branch"` (once per branch), narrow `fileFilters` (only what
  the command may commit), and an anchored regex per command in the global
  `allowedCommands` (`^pnpm install --lockfile-only$`). Commands run without a
  TTY (`pnpm dedupe` needs `CI=true`). A command that fails on an unrelated
  snapshot turns every bump into a `renovate/artifacts` failure. In a
  **grouped** branch only `upgrades[0]`'s tasks run (the first upgrade's config
  becomes the branch config; sorted by `fileReplacePosition`, then `depName`),
  so members matching different `postUpgradeTasks` rules run one command set,
  deterministically -- give every member the same rule or ungroup.
- **mise lock files.** `lockFileMaintenance` on `mise.lock` runs
  `mise lock --bump`, which can execute repository-defined scripts, so it is
  gated: add `"mise"` to the global `allowedUnsafeExecutions`, or a committed
  `mise.lock` is never refreshed.
- **cargo.** `rangeStrategy: auto` is `update-lockfile` (or `widen` when the
  value contains `<`); a dep with no `Cargo.lock` entry ends as
  `skipReason: invalid-value` (`No currentVersion or lockedVersion found`).
- **npm.** `postUpdateOptions: ["npmInstallTwice"]` re-runs a **successful**
  install to work around npm writing an invalid lock in one pass; Renovate
  throws on the first non-zero exit, so it cannot rescue `EOVERRIDE`,
  `ERESOLVE` or `EUSAGE`. Renovate also guts the upgraded packages'
  `node_modules/*` entries from the committed lock before installing, so a
  local repro must do the same. When a failure does not reproduce locally the
  variable is the npm inside the runner image (`binarySource: "global"`);
  `constraints: { npm: "<version>" }` or a `packageManager` field pins it
  (reported, not verified).
- **Preset caching** as above: `presetCachePersistence: false` while iterating.

## Gating on the consumer's pinned version

Before using an option, deleting a custom manager because upstream went
native, or converting a catalog to relative references, confirm the feature
exists in the Renovate version the **consumer's runner pins** (its
`package.json`) and in the CI validator pin -- not in `latest`. Thresholds
that matter: relative references 44.29.0; cargo git-tag deps 44.20.1 (44.20.0
lacks the `package =` fix); `rust-toolchain` and `smithy` managers 43.288.0;
`overrideDescription` 44.41; depType `workflow` and
`helpers:pinGitHubActionDigests*` = `["action", "workflow"]` 44.43.0
(renovatebot/renovate#45443). Between the runner bump and the catalog change
the affected dependencies are simply undiscovered.

To date a fix: `sha=$(gh api repos/renovatebot/renovate/pulls/<N> --jq .merge_commit_sha)`,
then per candidate tag
`gh api "repos/renovatebot/renovate/compare/$sha...$tag" --jq .status` --
`ahead` means the tag contains it, `behind` means not
(`gh api repos/renovatebot/renovate/releases --jq '.[0:6][].tag_name'` lists
recent tags). Renovate cuts several releases a day, so "merged today" can
still be behind the latest patch. The runner's own bump is subject to any
`minimumReleaseAge` floor its repo applies.
