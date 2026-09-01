# packageRules: how they merge, match and fail

Everything here was verified against Renovate 44.x through the debugger's
`run_config` / `get_provenance` / `simulate`, or read from Renovate's source at
the stated version. Re-verify on the pinned version the tools report before
quoting a number; items marked "reported" were not exercised live.

## Merge order

1. `extends` resolves depth-first. Each preset's children are merged first, in
   `extends` order, then the preset's own body. The extending file's own body is
   applied last, so **the file wins over what it extends** for scalar keys.
2. Options flagged `mergeable: true` **concatenate** across every layer
   (provenance action `concat`): `packageRules`, `customManagers`, `hostRules`,
   `addLabels`, `description`, `onboardingConfig`. Nothing is replaced; a repo's
   `customManagers` array is appended to the preset's, not swapped in.
   `schedule` is **not** mergeable: the last layer wins, so extending two
   `schedule:*` presets at the top level keeps only the second. Check any option
   with `get_option_docs <name>` (`mergeable`). Under `config:recommended` the
   repo's own `packageRules[0]` sits at merged index ~713 of ~714. Renovate's
   own messages cite the merged, 0-based index; the validator cites the file's
   index. Say which one you mean.
3. Preset dedupe compares the exact string per ancestor chain: `github>o/r`,
   `o/r`, `github>o/r#v1` and `github>o/r//default#v1` are four presets, and a
   file reached under two spellings is merged twice — doubling its
   `packageRules`, `addLabels` and every other mergeable array. One spelling per
   preset; read `treeSummary.duplicates` in the digest.
4. At apply time every rule is evaluated in merged order against one
   dependency; each matching rule merges its options over the previous ones.
   **Later rules win per key.** Your rules are last, so they win — but only for
   the keys they set. A rule that matches and sets nothing you care about
   changes nothing.
5. After rules, the update-type block for this update (`major`, `minor`,
   `patch`, `pin`, `pinDigest`, `digest`, `lockFileMaintenance`, `replacement`)
   is merged up, and it wins per key over rule output. Both directions follow:
   top-level `automerge: false` with `minor: { automerge: true }` **does**
   automerge minors; a preset's `major: { automerge: false }` **beats** any
   rule's `automerge: true` for majors (merge trace `automerge: true -> false`).
   Scope automerge with `matchUpdateTypes` on the rule instead of fighting a
   block, and delete rules that are inert for majors and redundant otherwise.
6. A manager block (`npm: {…}`, `nuget: {…}`, `dockerfile: {…}`) is merged over
   the top level for that manager only, and for array options it **replaces**.
   `:ignoreModulesAndTests` uses this: its `nuget.ignorePaths` drops the
   `**/test/**` globs so .NET test projects stay managed.
7. Renovate runs config migration **twice**: on the raw file, and again on the
   resolved config after presets. The second pass flattens `packageRules`
   nested inside a rule (the result of `extends: ["group:x"]` in a rule) and
   normalises `groupName: ["X"]` to `"X"`. Configs that only work because of
   that second pass are working by side effect.

## Matchers

All `match*` keys in one rule are ANDed. "A or B" is two rules that set the
same option: a first-party release-age exemption is one rule per identity
(`matchSourceUrls`, `matchRegistryUrls`, a namespace in `matchPackageNames`,
and `owner/repo` names for the `github-actions` manager), each with
`minimumReleaseAge: null`. Within one list, the pattern rules from
`string-pattern-matching` apply to `matchPackageNames`, `matchDepNames`,
`matchFileNames`, `matchSourceUrls`, `matchRepositories`, `matchBaseBranches`,
`matchUpdateTypes`, `managerFilePatterns`, …:

- plain string = exact match (or a minimatch glob when it contains `*`, `?`,
  `{}`); `/…/` = RE2 regex, `/…/i` for case-insensitive; leading `!` negates.
- At least one positive entry must match **and** every negative entry must
  match. A negation-only list (`["!gradle"]`) means "everything except".
- `*` or `**` next to any other entry is a **validation error** in current
  Renovate. The fix `["*", "!x"]` → `["!x"]` is behaviour-preserving only
  because the remaining entries are all negations; dropping `*` next to a
  positive pattern narrows the rule.
- Write the allowlist when the rule decides what counts as a match (explicit
  names derived from real data), and the plain string when a regex cannot
  distinguish anything the manager emits: `["/^Org/action-x(/|$)/"]` and
  `["Org/action-x"]` are identical for `github-actions`
  (`compare_simulations` verdict `identical`).
- `matchPackageNames` reads `packageName`, which Renovate defaults from
  `depName` before rules run. `matchDepNames` reads the name as written in the
  manifest. Managers where the two diverge: `github-actions` (`packageName` is
  `owner/repo`; `depName` carries a registry-URL prefix on GHE), `mise`
  (`"core:rust" = "1.82"` yields `packageName: core:rust` for tools that
  declare no `packageName`), `dockerfile` with `registryAliases` (`depName`
  stays the alias, `packageName` is the resolved path). `extract_deps` on the
  real file shows both. The singular `matchPackageName` does not exist.
- `matchCurrentValue` reads the raw declared value (`^1.2.0`);
  `matchCurrentVersion` reads the resolved version through the dependency's
  versioning module. Pick by what you are actually comparing.
- `matchSourceUrls` reads the dependency's `sourceUrl` from the datasource.
  Every `monorepo:*` preset is a `matchSourceUrls` list, so a monorepo group
  breaks silently when upstream moves the repo (and is fixed only by a
  Renovate release that updates the bundled data).
- `matchCategories` reads the **manager's** declared categories, never
  anything about the package (`pre-commit` declared `python`, so a Rust hook
  matched a python group). Prefer `matchManagers`.
- `matchFileNames` is valid only inside `packageRules` and matches when
  `packageFile` **or any `lockFiles` entry** passes. A negation such as
  `["!.github/workflows/release.yml"]` fails for every dependency whose manager
  reports that path as a lock file (upstream's github-actions manager declares
  no `lockFiles` for exactly this reason). There is no `ignoreFileName`.
- `matchManagers` for custom managers is `custom.regex` / `custom.jsonata`.
  `"regex"` is migrated to `custom.regex` silently on every run — an old rule
  keeps matching; write the current spelling.
- `matchUpdateTypes` values: `major`, `minor`, `patch`, `pin`, `pinDigest`,
  `digest`, `lockFileMaintenance`, `rollback`, `bump`, `replacement`. It is a
  pattern list: `["!major"]` is every non-major type and beats the seven-item
  list that forgets `replacement` and `lockFileMaintenance`. A misspelled
  value passes the validator and never matches (only `$schema` catches it).
  Built-in `group:*` presets set `matchUpdateTypes` to everything except
  `pin`; a hand-copied group without it starts gating pins too.
- `matchUpdateTypes` cannot share a rule with a pre-lookup option:
  `packageRules[0]: packageRules cannot combine both matchUpdateTypes and
  rangeStrategy. Rule: {...}` is a validation error (`accepted: false`). Put
  `rangeStrategy` in its own rule. The check sees only what the rule states
  directly; the same pair assembled through a preset passes and the option is
  ignored at runtime.
- `matchJsonata` evaluates a JSONata expression over the update.
  `["isBreaking = true"]` is "major, or any change to a `0.x` version" for
  versionings that define `isBreaking` (cargo, composer, github-actions, npm,
  python, semver, semver-coerced, semver-partial; the rest degrade to
  `updateType == "major"`); `isBreaking = false` is the exact inverse, so two
  rules on the pair partition every update with no gap. `isLockfileUpdate =
  true` is what Renovate's own default preset uses. Bad syntax is a validation
  error (`Invalid JSONata expression for packageRules[0].matchJsonata`); a
  runtime evaluation error logs a warning and the rule does not match. **The
  debugger cannot evaluate it**: `simulate` reports the clause as `no-match`
  with `readFields: []`, so an `isBreaking`-based automerge rule cannot be
  proven by simulation — say so in the hand-over and prove it on a real PR.
- `matchDepTypes` values worth knowing: language versions are first-class
  dependencies, enabled by default — pep621 `requires-python`, gomod `golang`
  and `toolchain`, npm `engines`, `rust-toolchain` `toolchain`.
  `{ matchDepTypes: ["requires-python"], enabled: false }` is how to stop one.
  For `github-actions`: `action`, `docker`, `container`, `service`,
  `github-runner`, `uses-with`, and `workflow` from 44.43.0.

### GitHub Actions: the repository is the finest grain

The `github-actions` manager names every `uses:` reference `owner/repo`; the
sub-path (`/.github/workflows/x.yml`, `/init`) survives only in
`replaceString`. Consequences:

- A URL in `matchPackageNames` (`"!https://github.com/actions{/,}**"`, the
  migrated form of `excludePackagePrefixes`) never matches anything, so the
  negation is vacuous and the rule fires for official actions too. Write
  `["actions/**"]`, or `matchSourceUrls: ["https://github.com/actions{/,}**"]`
  for trust decisions. Migration repairs syntax, not intent.
- `matchPackageNames: ["Org/*/.github/workflows/*"]` matches nothing. A
  reusable-workflow call and the same repo's root action are one identity up
  to 44.42.x. From 44.43.0 (renovatebot/renovate#45443) a call whose sub-path
  is exactly `.github/workflows/<file>.ya?ml` is `depType: workflow`, so
  `{ matchDepTypes: ["workflow"], pinDigests: false }` exempts reusable
  workflows; on older versions that matcher does nothing.
- `helpers:pinGitHubActionDigests` and `...ToSemver` are
  `{ matchDepTypes: ["action", "workflow"], pinDigests: true }` from 44.43.0
  (`["action"]` before, which pinned workflow calls anyway because they were
  `action`). No `matchManagers`; only this manager emits those depTypes. A
  copied body with `["action"]` stops covering workflows on >= 44.43.0.
  Re-resolve with `get_preset_node` on the pinned version before quoting.
- `currentValue` comes from the comment after a SHA (`# v4.1.2`); a bare SHA or
  a prose comment is `skipReason: unversioned-reference`, and no rule rescues
  it. `# main` selects `github-digest` (tracks the branch head).
- Prove a `pinDigests: false` exemption with `simulate`
  `{ manager: "github-actions", datasource: "github-tags", packageName:
  "Owner/repo", depType: "action", currentValue: "v1", newValue: "v1",
  updateType: "pinDigest" }` and `keys: ["pinDigests"]`.

### Fail-closed inputs

A matcher whose field the dependency does not carry is a **non-match**, not a
skip. `sourceUrl` is missing for many datasources and for any hand-written
simulation; 446 of the ~714 recommended rules read only `sourceUrl`. A clause
that throws (a `matchCurrentVersion` on an exotic versioning) is also a
non-match. So before concluding "the rule did not fire", look at the clause
evidence: `no-input` on the deciding clause means "no evidence", not "no".

Where `sourceUrl` comes from is per datasource. For crates the index carries no
repository field; the `crate` datasource reads the registry's `config.json` and
only when it has an `api` key fetches `{api}/api/v1/crates/{name}` and sets
`sourceUrl` from `crate.repository`. A private registry without `api` makes
every `matchSourceUrls` rule (monorepo groups, first-party exemptions) miss its
crates. Non-crates.io registries also need the **global** option
`allowCustomCrateRegistries: true`, which a repo config cannot set.

## Rule-scoped versus update-type-scoped

Presets like `:automergeMinor`, `:automergePatch`, `:automergeDigest` set
`automerge` **only inside** `minor` / `patch` / `pin` / `lockFileMaintenance`
blocks. A rule that extends one of them still applies its *other* keys
(`labels`, `assignees`, `autoApprove`, `reviewers`) to every matched update,
majors included. Scope the whole rule with `matchUpdateTypes` instead.

Blocks flatten after rules (Merge order 5), so a rule cannot override what a
block sets for its update type; a team preset's `major: { automerge: false }`
makes a rule's `automerge: true` inert for majors.

Keys that only make sense in update-type blocks are also invisible to a
per-key provenance query; `get_provenance` for `automerge` reports "defaults"
while a rule sets it for minors. `simulate` the dependency to see it.

The `pin` and `pinDigest` blocks both default to `groupName: "Pin
Dependencies"`, so version pins and digest pins across all managers land in one
branch `renovate/pin-dependencies` when `config:best-practices` turns pinning
on. To get one PR per digest pin override **both**:
`pinDigest: { groupName: null, branchTopic: "{{{depNameSanitized}}}-pin-digest" }`
— `groupName: null` alone degenerates to `renovate/<dep>-.x` because the
default `branchTopic` is version-based. Ongoing `digest` refreshes are already
one branch per dependency.

## `group:` presets inside a rule

A `group:x` preset's body **is** a `packageRules` array
(`{ packageRules: [{ extends: ["monorepo:x"], groupName: "x monorepo",
matchUpdateTypes: [...] }] }`). `extends: ["group:x"]` inside a rule therefore
nests rules in a rule. Renovate accepts it with the warning
`packageRules[N].extends: you should not extend "group:" presets`, and it only
works because the second migration pass flattens the nesting. If the config
also extends `config:recommended`, the same group rule is now resolved twice.

Fix: extend the underlying `monorepo:x` (a bare `matchSourceUrls` list) or
`packages:x` in your rule and restate `groupName`, `matchUpdateTypes` and
whatever else the group set. Read the group's body with `get_preset_node`
first; a resolved `group:jacksonMonorepo` is 20 source URLs plus two keys.

`minimumGroupSize` is valid inside a rule and belongs there; at the root it
gates every group in the config.

## Grouping

- The last matching rule's `groupName` wins; `config:recommended` groups
  `react` and `@types/react` as "react monorepo", and an org preset's later
  "linters" rule can silently take a package out of that group.
- To split a package out of an inherited group, a later rule with
  `groupName: null` for that package removes the grouping (verified: the
  merged `groupName` returns to `null`).
- A group rule scoped `matchUpdateTypes: ["minor","patch"]` leaves majors as
  separate PRs. Two members that happen to have different update types in one
  run split as well.
- Whether a group actually forms depends on the repository's pending updates;
  `simulate_group` answers it for the updates you supply and says so.
- In a grouped branch only `upgrades[0]`'s `postUpgradeTasks` run
  (`executionMode: "branch"`): `generate.ts` copies the first upgrade's config
  onto the branch, and `upgrades[0]` is decided by sorting on
  `fileReplacePosition` then `depName` (`@smithy/types` sorts before
  `software.amazon.smithy:*`). A group whose members match different
  `postUpgradeTasks` rules runs one command set — the same one every time.
  Give one rule covering exactly the group's package set the superset command,
  or ungroup. Check: `simulate` each member and compare `postUpgradeTasks`.
- One upstream version declared in two managers (a CLI in `mise.toml` and its
  Maven coordinates in `smithy-build.json`; `cargo:` tools in `mise.toml` and
  crates in `Cargo.toml`) needs a rule whose matcher reaches both spellings
  under one `groupName`, plus a CI check that they agree; otherwise one copy
  drifts and the next automerged bump goes red. `simulate_group` with one
  update per manager must report `wouldForm: true`.
- A grouping preset may be `extends: ["group:aws-cdkMonorepo", …]` at its
  **top level** — the "you should not extend group: presets" warning is about
  `group:` inside a rule. Do not carve `@types/*` into its own group; the
  library's group already carries them.
- A preset that flips an upstream rule (`pinDigests: false` against
  `helpers:pinGitHubActionDigests`) must come **after** the preset it overrides
  in `extends`; rules concatenate in that order and later rules win.

## Disabling and ignoring

- `enabled: false` in a rule yields the runtime `skipReason: disabled`
  (`fetch.ts`, confirmed on 44.48.3); the debugger labels the same outcome
  `package-rules` and its verdict reads "WOULD NOT be raised at all". Quote
  `disabled` when reading Renovate logs. A skipped dependency is also filtered
  out of the dashboard's "Detected dependencies" — absence there is not
  evidence that extraction failed.
- Choose the knob by what should happen. When a required check catches the
  break, add **nothing**: the PR opens, stays red, and merges the day upstream
  unblocks. `allowedVersions: "<7"` with a `description` only when CI cannot
  catch the break (the PR would merge green or carry no information); scope it
  with `matchFileNames` for one workspace package. `enabled: false` with the
  constraints and a revisit condition in `description` when no newer version
  can be adopted until something changes (generated manifests, a runtime the
  artifact cannot load). Never `ignoreDeps` for "cannot upgrade yet".
- A lockstep pair Renovate can bump only half of (a `rust-toolchain.toml`
  nightly that must match a `rev =` in `Cargo.toml`): disable the manageable
  half by file (`matchFileNames: ["rust/dylint/rust-toolchain.toml"]`) and name
  the coupled file in `description`. `matchFileNames` is a literal path; a move
  silently un-disables it unless the rule moves too.
- `ignoreDeps` is a top-level exact-name list; `ignorePaths` is top-level and
  glob-based; both are already populated by `config:recommended`
  (`**/node_modules/**`, `**/bower_components/**`, `**/vendor/**`,
  `**/examples/**`, `**/__tests__/**`, `**/test/**`, `**/tests/**`,
  `**/__fixtures__/**`). "Renovate ignores my example project" is usually this.
- `ignoreDeps: []` inside internal presets is an upstream hack for onboarding
  descriptions, not something to copy (removed in 44.41 in favour of
  `overrideDescription`).

## postUpgradeTasks

- Split by what the update needs: a lockfile-only recipe for the `npm` manager
  (`matchManagers: ["npm"]`, `fileFilters: ["pnpm-lock.yaml"]`) and full codegen
  only for the group that needs it. `executionMode: "branch"` runs once per
  branch; `fileFilters` limits what Renovate commits; the runner allow-lists
  each recipe as an anchored regex in the global `allowedCommands`
  (`^just renovate_lockfiles$`). Recipes must run without a TTY (`pnpm dedupe`
  needs `CI=true`).
- A command that exits non-zero makes Renovate commit nothing and report the
  commit status `renovate/artifacts` = failure plus an `Artifact update
  problem` block in the PR body (a PR comment when the body is truncated). The
  branch is retried only when a package file changes, the branch conflicts,
  the rebase checkbox is ticked, or the title gets a `rebase!` prefix.
- Grouped branches run one member's tasks only — see Grouping.
- Prove: `simulate` a dependency of each manager and read `postUpgradeTasks`
  in the result; run the recipe locally with `CI=true`.

## Options that mislead

- `description` is a mergeable array; presets append theirs. A wrapper preset
  (`{description, extends}` only) loses its own description. Use
  `overrideDescription` (44.41+) to replace the inherited ones. A description
  says what the rule is **for**; whether the justifying link lives there or in
  README / PR body is an org style — shared preset repos may cap length (150
  characters, purpose only, CI-enforced).
- `force` is documented as global-only and warned about in a repo config, yet
  the merge applies it. Do not rely on either direction.
- `hostRules` and `env` are honoured only at the top level; nested copies are
  inert. `matchHost` matches the host or a dotted subdomain of it; the longest
  match wins, and a rule with `hostType` beats an equally specific one without.
  A rule with `hostType` and **no `matchHost`** matches every host of that type
  and ships its credentials to whatever registry a lookup is steered to —
  including via `replacementName` or `registryAliases`, both ordinary
  repo-level options. Always pair credentials with `matchHost`. More than one
  host-matching field is refused (`hostRules cannot contain more than one
  host-matching field - use "matchHost" only.`); legacy `hostName` /
  `domainName` / `baseUrl` migrate to `matchHost`. On a GitHub runner the
  platform token already authenticates `github.com` datasources; extra rules
  for them are noise (`ghcr.io` coverage reported, not verified). npm private
  registries: scope mapping in `npmrc`
  (`@scope:registry=https://npm.pkg.github.com/Org`, `npmrcMerge: true`) and
  the token in one host-level rule (`hostType: "npm"`,
  `matchHost: "https://npm.pkg.github.com/"`).
- Never set `branchName`. It defaults to
  `{{{branchPrefix}}}{{{additionalBranchPrefix}}}{{{branchTopic}}}`; setting it
  is accepted with `Direct editing of branchName is now deprecated. Please edit
  branchPrefix, additionalBranchPrefix, or branchTopic instead`.
  `managerBranchPrefix` migrates to `additionalBranchPrefix`.
- The label template variable is `updateType`
  (`addLabels: ["dependencies", "{{{updateType}}}"]` once at the root; one of
  digest, pin, rollback, patch, minor, major, replacement, pinDigest).
  `{{{upgradeType}}}` is not a field: it renders as an empty string with no
  warning, and every PR gets an empty label. Handlebars in Renovate fails
  silent on unknown fields.
- `semanticCommits` defaults to `auto` under `config:recommended` (provenance
  winner `defaults`): prefixes appear only when Renovate detects conventional
  commits in recent history (per the docs). Where a PR-title lint is required,
  set `semanticCommits: "enabled"` (or `:semanticCommits`) explicitly.
  `semanticCommitType` per rule is the reliable way to make a bot bump cut a
  release under semantic-release; the default is only incidentally `fix`.
- `abandonments:recommended` (inside `config:best-practices`) is four rules,
  the first `{ matchPackageNames: ["*"], abandonmentThreshold: "1 year" }`. It
  shadows any top-level `abandonmentThreshold` because rule output wins over
  top-level values. Override with a **trailing** wildcard rule, and restate the
  community overrides (`@types/*` -> `null`, `eslint-plugin-no-only-tests` ->
  5 years, `lodash` -> 6 years) if wanted. Same mechanism: a runner's global
  top-level options cannot beat repo/preset `packageRules`.
- `schedule` is not mergeable (last layer wins) and evaluates in UTC without a
  top-level `timezone`. Prefer `schedule:*` presets to raw cron (windows move:
  `schedule:quarterly` fires on the 1st). To give one class of updates its own
  cadence, put `schedule` inside that `packageRules` entry, not in a second
  top-level preset that would replace the repo-wide window.
- `prBodyNotes` on a repo-local rule attaches a cleanup reminder to one
  dependency's next bump PR ("Check whether upstream #3572 is in this release;
  if so, remove the `ignore_changes` workaround"). Per-rule, templatable,
  repo-specific — not for the shared preset.
- `versioning` set in a rule to the datasource's default (`rust-version` ->
  `rust-release-channel`, `node-version` -> `node`) is dead config; the default
  is applied before rules run. A different value is an override and should
  say why.
- `registryAliases` and `replacementName` are ordinary repo-level options that
  steer lookups; a self-hosted admin cannot forbid them. There is no
  `replacementDatasource`.
- `ignorePresets` compares the resolved absolute string, tag included
  (`github>org/repo//system/security#v2.0.0`); a relative spelling in a
  consumer changes nothing and raises no error. Once a consumer sets any
  `ignorePresets`, a preset's own `ignorePresets` is inert. The exact string is
  in `get_preset_tree` or the debug line `Ignoring preset <x>`.
- `rangeStrategy: "pin"` is a silent no-op when the current value is already a
  single version under the effective versioning (`regex:` versionings never
  support ranges); Renovate resets it to `replace`. For `cargo`, `auto` means
  `update-lockfile` (or `widen` when the value contains `<`): in-range updates
  touch only `Cargo.lock`, and a dependency without a `Cargo.lock` entry ends as
  `skipReason: invalid-value` (`No currentVersion or lockedVersion found`).
- `constraints: { rust: "1.85" }` in a repo beats what a manager extracted, and
  under `binarySource: install|docker` a single-version constraint is installed
  verbatim (`cargo update` on a `Cargo.lock` `version = 4` needs Rust >= 1.78).
  `constraintsFiltering: strict` keeps a release when the config value matches
  in either direction, so a minimum must be written `<=1.70`;
  `constraintsVersioning` works only for `%`-prefixed constraints, never for
  `rust` / `python` / `node` / `go`. The cargo versioning lacked a `subset`
  override until a fix landed on main (2026-08); on an older pin `strict`
  drops every crate release whose `rust_version` is not `1.70.x` — check
  `dist/modules/versioning/cargo/index.js` for `subset` before enabling it.
- `extractVersion` rewrites the release version *and* the value Renovate writes
  back. Using it to strip a suffix that the registry needs writes a
  non-existent version into the manifest; `versionCompatibility` is the option
  for "registry version carries a suffix the tag does not".
- `minimumReleaseAge` is enforced by Renovate at lookup time (commit status
  `renovate/stability-days`: pending inside the window, pass afterwards; never
  make it a required check) and independently by the package manager at
  install time with **its own** configured age (pnpm: minutes,
  `minimumReleaseAgeExclude`), which can disagree with Renovate's; the failure
  is `ERR_PNPM_MINIMUM_RELEASE_AGE_VIOLATION`. A companion package published
  minutes after the main one can still fail the lockfile step;
  `minimumReleaseAgeBuffer` (44.x, default `10 minutes`) exists for that.
  `vulnerabilityAlerts` defaults to `minimumReleaseAge: null`, so security PRs
  bypass the floor unless you set one there.
- `extends: ["global:x"]` in a repo config is the hard error `you cannot extend
  from "global:" presets in a repository config's "extends"`. Inside a rule
  write `extends` as an array; a bare string is flagged `Config migration
  necessary`.
- Renovate's own warning "Config migration necessary" is logged at debug level
  in a run; the CLI validator prints it at WARN with a `Config migration
  diff:` (exit 1 only under `--strict`). `accepted: true` with no warnings says
  nothing about deprecated syntax; the debugger's digest does ("rewrote N
  deprecated options").
