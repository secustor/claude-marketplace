# Validating a Renovate config

`renovate-config-validator` is a necessary gate, not a sufficient one. This
file says exactly what it checks, what it cannot see, how to run it in CI for a
preset repository, and which real-run recipes cover the rest. Messages were
read from Renovate 44.48.3 source and checked on 44.56.1.

## The command

```bash
npx -y -p renovate@<the bot's pinned version> renovate-config-validator --strict --no-global <file...>
```

- The bin lives inside the `renovate` package; `npx renovate-config-validator`
  is a 404. Bare `-p renovate` resolves whatever npx has cached -- sessions got
  a 37.x validator that treats the flags as file names
  (`ERROR: File does not exist "file": "--no-global"`). Always give a version;
  make it the version the bot runs (the runner's `package.json` pin), not
  `latest`.
- `--no-global` validates the file as **repo** config
  (`INFO: Validating <file> as repo config`). Without it a positional file is
  validated as **global** config, the permissive superset, and `autodiscover`,
  `allowedCommands` and every other `globalOnly` option pass. Use it for every
  repo config and every shared preset (presets are consumed from repo configs).
  `hostRules`, `npmrc`, `npmrcMerge` and `encrypted` are ordinary repo-level
  options and pass in both modes; they are not the reason for the flag.
- `--strict` adds exactly one thing: `WARN: Config migration necessary` (plus
  a `Config migration diff:`) becomes exit 1. Every other warning already
  exits 1 with or without `--strict`. Fix a strict failure by writing what the
  diff shows (or rcd `get_resolved_config`, mode `keep-internal`), never a
  hand-guessed rewrite.
- Success: `INFO: Config validated successfully against N file(s)`.
- Renovate 44.x declares `engines.node ^24.11.0`. On another Node major the
  native `re2` module may not load; on Node 26 `renovate` itself aborts with
  `Unsupported node environment detected`.
- `WARN: RE2 not usable, falling back to RegExp: regex validation may be inaccurate`
  means the regex checks are not authoritative in that environment: lookaheads,
  lookbehinds and backreferences pass locally and fail at runtime. Fix the
  Node version or the npx cache; CI on Node 24, rcd `run_config` and
  `extract_deps` are authoritative.
- For private GitHub preset repos referenced from `packageRules[].extends`
  export `GITHUB_COM_TOKEN` (or `RENOVATE_GITHUB_COM_TOKEN`); in Actions
  `GITHUB_COM_TOKEN: ${{ secrets.GITHUB_TOKEN }}`. `RENOVATE_TOKEN` is ignored
  by the validator, and a missing token reads exactly like a missing preset.

## What the validator checks

Parse -> migrate -> massage -> `validateConfig`, offline except for one case.

- **`extends` shapes.** Each entry goes through the preset parser only:
  `extends: preset "<x>" is not valid` for anything that does not parse
  (`./x#v1`, `local>./x`, `github>o/r//a:b`). In a repo config
  `extends: ["global:..."]` is the error
  `you cannot extend from "global:" presets in a repository config's "extends"`.
  `packageRules[N].extends: ["group:x"]` is the warning
  `you should not extend "group:" presets` (it is a nested `packageRules`).
  `:timezone(x)` has its zone validated. Inside a rule write `extends` as an
  array; a bare string is flagged `Config migration necessary`. Entries
  containing `{{` skip even the parse check.
- **`packageRules` selectors.** After resolving the rule's own `extends`, a
  rule with none of the `match*`/`exclude*` fields is the error
  `packageRules[N]: Each packageRule must contain at least one match* or exclude* selector. Rule: {...}`;
  a rule with only selectors is the warning
  `Each packageRule must contain at least one non-match* or non-exclude* field`.
  When the rule extends a **relative** preset (44.29.0+) the relative entries
  are stripped before resolution and the selector error is downgraded to the
  warning
  `packageRules[N]: this rule extends a relative preset that cannot be resolved during validation, so its selectors could not be checked. Rule: {...}`
  -- which still exits 1. A rule that has its own selector and a dangling
  `./typo` validates clean. Keep one selector directly on every rule that
  extends a relative preset, and do not conclude "every rule has a selector"
  from a green run when any rule extends one.
- **`packageRules[].extends` is fetched** -- the one network call. It resolves
  against the **default branch** with no repo or tag context, so a preset file
  added in the same PR and used from a rule fails with
  `INFO: Throwing preset error "validationError": "Cannot find preset's package (github>org/repo//packages/x)"`
  until the file is on `main`. Built-in `packages:*` resolve offline.
- **Global-only options** under `--no-global`:
  `The "autodiscover" option is a global option reserved only for Renovate's global configuration and cannot be configured within a repository's config file.`
- **Bare selector presets** (`packages:*` shape: top-level `matchPackageNames`
  and nothing else) are refused in every mode --
  `matchPackageNames should be inside a "packageRule" only` -- because the CLI
  hardcodes `isPreset = false`. A red run on such a file is about repo-config
  shape, not preset validity; validate `{"packageRules":[<file body>]}`
  instead.
- `matchJsonata` expressions are syntax-checked
  (`Invalid JSONata expression for packageRules[0].matchJsonata: ...`).
- Invented keys (`comment`, `ignoreFileName`) are
  `Invalid configuration option: <key>`.

## What it cannot see

- **Top-level `extends` is never fetched.** `github>org/repo#v9.9.9` (no such
  tag), `github>org/repo//does-not-exist`, a typo in a path and
  `local>x/y#{{ env.TYPO }}` all print `Config validated successfully`, also
  with `--strict` and a token. Prove the target separately:
  `gh api repos/<org>/<repo>/contents/<path>.json --jq '.name, .sha'`, rcd
  `run_config` (read `presetErrors`), or a `--dry-run=lookup` with the
  bogus-tag control below.
- **Relative references are accepted in any file.** `{"extends":["./x"]}` in
  a repository's own config, in inherited config or in `globalExtends` passes
  the validator, onboarding validation and the `renovate/reconfigure` check,
  and aborts every run with
  `Relative preset reference cannot be resolved (./x). ...` (full text in
  `preset-repos.md`). rcd `run_config` shows it as `stageStatus.preset: "error"`
  with `accepted: true`.
- **No enum or array-element checks.** `matchUpdateTypes: ["bogusvalue"]` and
  `labels: [1, true]` pass. Only the JSON `$schema` catches them -- keep it on
  every file and use an editor or schema step; check spellings with rcd
  `simulate`.
- Fields a relative preset contributes to a rule are invisible to downstream
  checks (`matchUpdateTypes` + `rangeStrategy` conflict, the
  `baseBranchPatterns` warning for `matchBaseBranches`).
- The RE2 fallback above.

## Messages

| message                                                                                  | means                                                                                                          |
| ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `Validating <file> as global config`                                                     | you forgot `--no-global`                                                                                       |
| `WARN: Config migration necessary` + `Config migration diff:`                            | deprecated syntax; exit 1 only under `--strict`; write the diff                                                |
| `WARN: Found errors in configuration` with `warnings: [...]`                             | any warning exits 1; read the list                                                                             |
| `extends: preset "<x>" is not valid`                                                     | the string does not parse; see the syntax table in `preset-repos.md`                                           |
| `Cannot find preset's package (<x>)` from `Throwing preset error`                        | a `packageRules[].extends` entry could not be fetched: not on the default branch, private without `GITHUB_COM_TOKEN`, or a typo |
| `this rule extends a relative preset that cannot be resolved during validation`          | add a selector directly on the rule                                                                            |
| `matchPackageNames should be inside a "packageRule" only`                                | a bare selector preset; wrap it before validating                                                              |
| `RE2 not usable, falling back to RegExp`                                                 | wrong Node ABI / stale npx cache; regex results unreliable                                                     |
| `Unsupported node environment detected`                                                  | Node outside `engines`; use Node 24 for 44.x                                                                   |

## CI for a preset repository

1. **Discover files dynamically**: walk the tree for `*.json` / `*.json5`
   excluding `.git/`, `.github/`, `node_modules/`, `package.json` and
   lockfiles, including the repo's own `renovate.json`. A hard-coded folder
   list silently skips the next category folder.
2. Validate every preset file with `--strict --no-global`.
3. Validate the repo's **own** config as a repo config too (it is a consumer).
4. Wrap bare-selector presets (`packages/*.json`) in a one-element
   `packageRules` array in a scratch file before validating.
5. Inline same-repo `packageRules[].extends` from the working tree before
   validating: full-slug `github>org/repo//dir/name`, `org/repo:name`,
   `org/repo` and the relative `/x`, `./x`, `../x` forms. This lets a selector
   preset be introduced and used in one change, and turns a dangling internal
   reference (missing target file, a relative ref that escapes the root or
   carries `#`) into a CI failure instead of a silent default-branch lookup.
   Leave entries with an explicit `#ref` to the network. Refuse a target that
   itself declares `packageRules` (only a flat selector can be extended from
   inside a rule).
6. Pin the validator to the bot's version under a marker and let Renovate bump
   it through the same gates as everything else:

   ```yaml
   env:
     # renovate: datasource=npm depName=renovate
     RENOVATE_VERSION: "<the bot's pinned version>"
   ```

   with `customManagers:githubActionsVersions` in the repo's `extends`, and
   `npm install -g renovate@$RENOVATE_VERSION` (or `npx -y -p renovate@...`).
   `renovate@latest` in CI validates against a version production does not run,
   bypasses any release-age floor and fails transiently on registry propagation
   (`npm error notarget`). Bump the pin in the same change that moves the
   runner; relative references put a floor of 44.29.0 under both.
7. When adopting `--strict` on a repo with an unmigrated monolith, carve that
   one file out of `--strict` only until the commit that rewrites it, and drop
   the carve-out in the same PR.
8. Then prove behaviour: resolve a consumer config at the pushed branch
   (`{"extends":["github>org/repo#<branch>"]}` through rcd `run_config`, or the
   dry-run below) and `compare_simulations` against `main`. The validator
   proves syntax; a broken relative reference still fails every consumer at
   runtime.

Expose the workflow through `workflow_call` so consumer repos can validate
their own `renovate.json` with the same pinned version; consumers pin it by
SHA.

## The `renovate/reconfigure` branch

Push a config to a branch named `renovate/reconfigure` and Renovate validates
it on its side: it reads the file on that branch, runs migration, then
validation, then resolves presets (a `github>org/repo#tag` lookup included) and
sets a commit status on the branch; the PR comment listing expected branches
appears only when an open PR exists on it. Because migration runs first, a file
full of deprecated names still gets "Validation Successful" -- it proves parse,
resolve and post-migration validity, not current syntax. Writing the migrated
file back is the separate `configMigration` option (`renovate/migrate-config`
branch, `:configMigration` is in `config:best-practices`).

`renovate/reconfigure` and `renovate/lock-file-maintenance` are the only
`renovate/*` names Renovate does not prune. `renovate/configure` is the
**onboarding** branch: a PR opened there by hand is claimed and retitled
`- abandoned` (see `debugging.md`).

## Dry-run recipes

- **Prove a preset reference resolves** (no repository needed):

  ```js
  // cfg.js -- `export default` where an ESM package.json governs the directory,
  // `module.exports` otherwise
  export default { extends: ["github>org/repo#vX.Y.Z"], repositories: [] };
  ```

  `RENOVATE_TOKEN=$(gh auth token) RENOVATE_CONFIG_FILE=$PWD/cfg.js npx -y renovate@<pinned> --dry-run=lookup`.
  Pass: `WARN: No repositories found - did you want to run with flag --autodiscover?`
  (global-config presets resolve before discovery). Fail:
  `INFO: Throwing preset error` / `ERROR: config-presets-invalid` /
  `Error: config-validation`. Run a bogus tag (`#v9.9.9`) as the control
  every time.
- **Whole-repo extraction** (what Renovate extracts, `skipReason` included):
  shallow-clone, **overwrite** the repo's config with `{}` and commit it (do
  not `rm` it: Renovate still finds the tracked file, logs
  `Null contents when reading config file` and aborts with
  `Repository has changed during renovation - aborting`), then
  `GITHUB_COM_TOKEN=$(gh auth token) RENOVATE_PLATFORM=local RENOVATE_REQUIRE_CONFIG=optional LOG_LEVEL=debug npx -y renovate@<pinned> --dry-run=extract 2>&1 | grep -iE 'depName|skipReason'`.
  The stub is needed because `local>org/preset` cannot resolve under
  `--platform=local`; without `GITHUB_COM_TOKEN` every `github-tags` dep shows
  `skipReason: github-token-required` (a harness artefact). `--dry-run=lookup`
  additionally prints the `branchName`s that would be created.
- **Inject a repo config without committing**:
  `RENOVATE_X_STATIC_REPO_CONFIG_FILE=<json file>` is merged before preset
  resolution; the repo's committed `renovate.json` overrides it.
  `RENOVATE_STATIC_REPO_CONFIG` does not exist and is a silent no-op -- diff
  against a baseline run.
- **A repository dry-run reads the default branch.** `--base-branches=<pr>` /
  `RENOVATE_BASE_BRANCHES` only changes which branches are updated; the config
  still comes from `main`. To test an edit before merge resolve the file text
  directly (rcd `run_config`), push it to `renovate/reconfigure`, or point the
  consumer at a preset branch with a templated `extends`
  (`preset-repos.md`).
- **One-shot real run**: the repository is a positional argument
  (`--repositories=` is `unknown option`). Zero the three limits or
  `config:recommended`'s `prHourlyLimit` (2) holds most PRs back:
  `RENOVATE_TOKEN=... npx -y renovate@<pinned> --platform=github --pr-hourly-limit=0 --pr-concurrent-limit=0 --branch-concurrent-limit=0 owner/repo`.
  The token needs the `workflow` scope to push changes under
  `.github/workflows/`.
- Debug output: `LOG_LEVEL=debug LOG_FILE=<path>` as **environment variables**
  (the logger is built before the config file is read). `debugging.md` has the
  `jq` queries.

## Required checks in repos that consume Renovate PRs

- **Never require `renovate/stability-days` or `renovate/artifacts`.** Both
  are commit statuses Renovate sets on its own branches (the release-age wait
  and artifact failures); they never appear on human PRs, so a ruleset that
  requires them blocks every non-Renovate merge. When codifying required
  checks from live PRs also drop other non-Actions contexts (CodeQL, Socket,
  Danger) unless they report on every PR.
- **A required context no job reports blocks everything.** Required contexts
  are job names; a renamed, deleted or misnamed job leaves every PR
  `mergeStateStatus: BLOCKED` and automerge never fires. Use one aggregate job
  (`pr-success`) with `needs:` on every real job, `if: always()` and a `jq`
  assertion that every `needs.*.result == "success"`, so a skipped or failed
  upstream job reports red instead of vanishing. It exists only on branches
  that carry it: merge it first; open Renovate PRs pick it up on rebase.
  Change the ruleset and the job name in the same step.
- **A frozen-lockfile install must be a required check wherever automerge is
  on** (`npm ci`, `pnpm install --frozen-lockfile`, `cargo ... --locked`).
  `renovate/artifacts` failing means the lockfile did not follow the manifest;
  without a required check the PR still merges and `main` ends up with
  `package.json` ahead of the lockfile, which the next `npm ci` refuses.
  `npm install` in the PR workflow hides the desync because it re-resolves.
- **PR-title and commit lints must not cap header length.** Grouped Renovate
  titles enumerate every dependency and exceed 100 and 120 characters; set
  `header-max-length: [0]` and `body-max-line-length: [0]` (disabled, not
  raised) and keep `type-enum`/`scope-enum`. Renovate's semantic titles satisfy
  `type(scope): subject` when `:semanticCommits` is on, so relax the length
  rather than exempting the bot.
- A Renovate automerge that conflicts with an open PR (a nested `package.json`
  the PR deletes) makes GitHub run **no** `pull_request` workflows for that PR
  (`mergeable: CONFLICTING`, `gh run list --branch <head>` empty). Rebase;
  nothing is wrong with the workflows.
