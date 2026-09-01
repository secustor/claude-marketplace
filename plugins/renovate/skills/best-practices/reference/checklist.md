# Checklist — practices, their reasons, and the anti-patterns

Each item states the practice, why, and how to verify it on the resolved
config. Severity is a default; the repo's context can move it.

## Baseline

- **`$schema` on every config and preset file.** Editors validate keys and
  values against it; it catches `comment`, `ignoreFileName` and other invented
  keys before Renovate does. (low)
- **`extends: ["config:recommended"]` at minimum; `config:best-practices`
  where digest-pinning PRs are acceptable.** `config:recommended` is the
  dashboard, sensible ignores, semantic commits and ~700 grouping rules;
  `config:best-practices` adds digest pinning for docker images and GitHub
  Actions, `minimumReleaseAge` for npm, pinned devDependencies, weekly lock
  file maintenance, abandonment detection and config self-migration
  (`reference/presets.md`). `config:base` is a migration alias only. (high)
  Verify: `get_preset_tree` shows the entry; the digest names the count.
- **One shared preset per org, every repo extends it, repo files stay small.**
  Duplicated fragments drift; a fix in the preset reaches every repo on its
  next run without a release. (medium) Verify: a repo-level option that the
  preset also sets is a finding against the repo.
- **No deprecated syntax.** Renovate migrates silently and logs it at debug
  level; `renovate-config-validator --strict` and the debugger's digest are the
  only places it shows. Rewrite per `edit-config/reference/migrations.md`;
  `:configMigration` (in `config:best-practices`) makes Renovate open that PR
  itself. (medium)
- **The config is accepted with no warnings.** A warning is a config that
  works by accident: `group:` presets in a rule, global-only options in a repo
  file (`force` is applied anyway), `*` mixed with other patterns. (high)

## Supply chain

- **Pin GitHub Action digests, with the semver comment kept in sync:
  `helpers:pinGitHubActionDigestsToSemver`.** The SHA is what runs; the
  `# v4.1.2` comment is what people read; the `digest` update is what tracks a
  moved tag. Plain `pinGitHubActionDigests` (inside `config:best-practices`)
  pins but leaves the comment to rot. (high)
- **Pin docker image digests (`docker:pinDigests`)** for images built or
  deployed from the repo. Fixture and example files should not be named like
  a manifest at all (`Dockerfile.fixture` still matches `*Dockerfile*`;
  `container.fixture` does not). (medium)
- **`minimumReleaseAge` for npm (and any registry with publish-then-yank
  history), with an exception for packages the org publishes itself.** The
  user's preset: `3 days` for npm, `matchSourceUrls: ["!https://github.com/org/**"]`
  excluded; `security:minimumReleaseAgeNpm` is the built-in equivalent. Note the
  package manager enforces the same age at install time independently, so a
  companion package published minutes later can fail the lockfile step;
  `minimumReleaseAgeBuffer` (44.x) addresses that. (high)
- **Lock file maintenance on, weekly, automerged when CI is the real gate.**
  `lockFileMaintenance: { enabled: true, automerge: true }` at the top level so
  it composes with `:maintainLockFilesWeekly`'s schedule. It heals in-range
  transitive advisories; exact pins in a parent do not move. (medium)
- **Vulnerability alerts enabled** (default on GitHub; needs the Dependabot
  alerts permission) for the reactive gap between weekly refreshes. (medium)
- **No `ignoreDeps` for "we cannot upgrade yet".** It also hides security
  updates. Hold the major with `allowedVersions: "<5.0.0"` and a `description`
  linking the blocking issue; delete the rule when the issue closes. (medium)
- **No exact-version `overrides`/`resolutions` as a CVE fix when the parent's
  range already admits the patch.** They block every future in-range fix.
  Prefer lock file maintenance; if an override is needed, retire it in the
  same commit as the bump that makes it redundant. (medium)

## Automerge

- **Automerge only what CI actually tests.** DevDependencies with a test
  suite, GitHub Actions, package managers (`npm`, `pnpm`, `yarn`) on
  patch/minor, Node patch/minor via `group:nodeJs` — all with a `description`
  saying why. (medium)
- **Scope automerge with `matchUpdateTypes` on the rule, not with
  `:automergeMinor` inside a rule.** The preset scopes only `automerge`; every
  other key in that rule (`labels`, `assignees`, `autoApprove`) still applies to
  majors. Verify: `simulate` a major of a matched package. (high)
- **Never automerge majors by accident.** Any rule with `automerge: true` and
  no `matchUpdateTypes` on a broad matcher is a finding unless the
  description says majors are intended. (high)
- **`platformAutomerge` / required checks.** Automerge without any status check
  merges untested code; check `ignoreTests` is not set, and that the
  repository has required checks. (high, cannot be verified from config alone
  — say so)

## Grouping and noise

- **Group by upstream, not by name.** `matchSourceUrls: ["https://github.com/backstage/backstage"]`
  catches every package published from the monorepo; templated
  `groupName` (`Backstage plugin {{ lookup (split packageName '-') 1 }}`)
  keeps plugin families together. (low)
- **Do not fight the built-in groups blindly.** `config:recommended` already
  groups ~700 monorepos; a later `groupName` on the same packages silently
  takes over. Check with `simulate` which rule wins. (medium)
- **Rate limits match the review capacity.** The user's preset:
  `prConcurrentLimit: 20`, `prHourlyLimit: 0`. Low limits plus a dashboard
  hide updates in "Pending Approval"; unlimited plus no automerge floods. (low)
- **A schedule only where CI cost or on-call matters**; with automerge and
  limits it is usually unnecessary, and an unscheduled `lockFileMaintenance` is
  already weekly. Always with `timezone`. (low)
- **Dependency dashboard on** (default via `config:recommended`); it is where
  pending, rate-limited and errored updates become visible. (low)

## Rules hygiene

- **Every rule has a `description`** with the reason and a link; `comment` is
  not a key. (low)
- **Every rule has at least one matcher**; a matcher-less rule applies to
  everything. (high)
- **Matchers read fields the dependency has.** A rule whose only matcher is
  `matchSourceUrls` or `matchCategories` on a datasource that supplies no
  `sourceUrl`, or on a manager with the wrong category, never fires. Verify
  with `simulate` and look at `missingInputs`. (medium)
- **Rules are ordered general → specific**, since later rules win, and no
  rule is fully shadowed by a later one setting the same keys on the same
  matches. (medium)
- **Custom managers are proven by extraction** (`extract_deps`), use
  `managerFilePatterns`, and target deps with `matchManagers: ["custom.regex"]`.
  Prefer `customManagers:*` presets and marker comments; prefer moving the
  version into a file a built-in manager owns (`mise.toml`) over any custom
  manager. (medium)
- **`postUpgradeTasks` tolerate drift.** A command that fails on an unrelated
  snapshot turns every bump into a "failed to update artifact" PR. (medium)
- **`semanticCommitType` is pinned on rules whose bump must cut a release.**
  (low)
- **No manual commits on `renovate/*` branches**; Renovate stops rebasing
  them. (informational)

## Preset repositories

- Relative references between the repo's own files; absolute in its own
  `renovate.json`; `#branch` testing then resolves the whole chain
  (`edit-config/reference/preset-repos.md`).
- `overrideDescription` on collection presets whose summary must survive.
- No `ignoreDeps: []` sentinel copied from upstream.
- A consumer config resolved before and after each change, compared on a
  representative dependency.

## Anti-patterns seen in the wild (each verified against Renovate 44.x)

| pattern                                                     | what actually happens                                                       | fix                                                                  |
| ----------------------------------------------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `matchPackageNames: ["*", "!gradle"]`                       | validation error on current Renovate                                        | `["!gradle"]`                                                        |
| `extends: ["group:jackson"]` inside a rule                  | warning; nested `packageRules`; works via migration side effect; duplicate  | `extends: ["monorepo:jackson"]` + `groupName` + `matchUpdateTypes`   |
| `extends: [":automergeMinor"]` + `labels` in one rule       | labels on majors too                                                        | `matchUpdateTypes: ["minor","patch"]` on the rule                    |
| `matchUpdateTypes: ["lockFileMaintenance"]` rule            | overrides `:maintainLockFilesWeekly` instead of composing                   | top-level `lockFileMaintenance: {…}`                                 |
| `matchCategories: ["python"]` to group python tools         | matches every `pre-commit` hook (manager category)                          | `matchManagers`                                                      |
| `ignorePaths` in a repo to skip `test/` for .NET            | nuget's manager block replaces the array; `test/` stays managed             | rule with `matchFileNames: ["test/**"]`, `enabled: false`            |
| `comment: "…"` / `ignoreFileName`                           | config refused / option does not exist                                      | `description` / `matchFileNames`                                     |
| `matchManagers: ["regex"]`                                  | never matches                                                               | `custom.regex`                                                       |
| `fileMatch`, `regexManagers`, `matchPackagePatterns`, `stabilityDays`, `masterIssue`, `config:base` | migrated silently every run                       | write the migrated form                                              |
| `extractVersion` to drop a `-alpha` suffix for changelogs   | writes a version that does not exist into the manifest                      | `versionCompatibility`                                               |
| `rangeStrategy: "pin"` under a `regex:` versioning          | silent no-op, reset to `replace`                                            | drop it                                                              |
| `groupName: ["NodeJS"]`                                     | migrated to a string, with a warning                                        | `"NodeJS"`                                                           |
| version duplicated in `Dockerfile ARG` and `mise.toml`      | the copy drifts                                                             | one source, managed by its manager                                   |
| `@scope/**` glob for a monorepo group                       | sweeps in `@scope/darwin-arm64` platform binaries                           | explicit names or `matchSourceUrls`                                  |
| `minimumGroupSize` at the root                              | gates every group                                                           | put it on the rule                                                   |
| `force` in a repo config                                    | warning says ignored; merge applies it                                      | remove it                                                            |
