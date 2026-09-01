# What the recommended presets actually contain

Rule one: re-resolve on the **consumer's pinned** Renovate before quoting a
body or a number — `run_config` `{"extends": ["config:best-practices"]}` then
`get_preset_node <name>` with `body: fetched`. Internal presets change between
releases (the `helpers:pinGitHubActionDigests*` bodies changed in 44.43.0),
and the consumer's pin is usually older than the debugger's. The facts below
were resolved with Renovate 44.42.1 on 2026-09-01 and are worded so the reader
knows what to look for, not what to paste.

## `config:best-practices`

Pure router. It sets nothing itself and extends, in this order:

| preset                            | what it contributes                                                                                                                                                                                                                                                                                                                                              | check                                                |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| `config:recommended`              | the baseline: dependency dashboard, `ignorePaths` for vendored dirs, ~700 monorepo/grouping rules, semantic commits (`auto`); `:configMigration` is **not** in here, and it sets no PR rate limits (`prHourlyLimit: 2` is Renovate's own default)                                                                                                                | `get_preset_tree`                                    |
| `docker:pinDigests`               | `pinDigests: true` for docker-type dependencies                                                                                                                                                                                                                                                                                                                  |                                                      |
| `helpers:pinGitHubActionDigests`  | one rule `{ matchDepTypes: [...], pinDigests: true }` with **no** `matchManagers`: `["action"]` up to 44.42.x, `["action", "workflow"]` from 44.43.0 (renovatebot/renovate#45443). Either way it SHA-pins reusable-workflow calls; hosts of OIDC-gated reusable workflows need an explicit `pinDigests: false` rule placed after this preset (`reference/github-actions.md`) | `get_preset_node … body: fetched`                    |
| `:configMigration`                | `configMigration: true` — Renovate opens a PR rewriting deprecated options in the repo's own config file                                                                                                                                                                                                                                                          |                                                      |
| `:pinDevDependencies`             | `rangeStrategy: pin` for `devDependencies`                                                                                                                                                                                                                                                                                                                       |                                                      |
| `abandonments:recommended`        | four rules: `{ matchPackageNames: ["*"], abandonmentThreshold: "1 year" }`, then `null` for npm `@types/*`, `5 years` for `eslint-plugin-no-only-tests`, `6 years` for `lodash`. The wildcard rule **shadows** any top-level `abandonmentThreshold` (a top-level `"3 years"` simulates to `1 year`); override with your own `matchPackageNames: ["*"]` rule placed **last**, restating the three community overrides if wanted | `simulate` any dep, `keys: ["abandonmentThreshold"]` |
| `security:minimumReleaseAgeNpm`   | `minimumReleaseAge: "3 days"` for npm (5 rules) — where the "3 days" a user never wrote comes from. It does not touch `vulnerabilityAlerts`, whose default `minimumReleaseAge: null` lets vulnerability PRs bypass it                                                                                                                                             | `get_provenance minimumReleaseAge`                   |
| `:maintainLockFilesWeekly`        | `lockFileMaintenance: { enabled: true, schedule: ["before 4am on monday"] }`                                                                                                                                                                                                                                                                                     |                                                      |

So `config:best-practices` is `config:recommended` plus supply-chain
hardening. Two consequences of turning it on: every `pin` and `pinDigest`
across all managers lands in **one** branch, `renovate/pin-dependencies`
("Pin Dependencies"), and reusable-workflow calls get SHA-pinned. Recommend it
when the repo can tolerate PRs that rewrite `uses:` lines to SHAs **and** no
`uses:` targets a reusable workflow whose OIDC trust is keyed on
`job_workflow_ref`; otherwise add the `pinDigests: false` carve-out in the same
change, or start from `config:recommended` and add the pieces individually.

## `helpers:pinGitHubActionDigestsToSemver`

Extends `helpers:pinGitHubActionDigests` and adds one rule on the same
depTypes: `extractVersion: "^(?<version>v?\\d+\\.\\d+\\.\\d+)$"` plus a regex
`versioning` whose minor and patch are optional. Effect: on every digest
update Renovate rewrites the `# vX.Y.Z` comment to the exact release, so the
SHA (what runs) and the comment (what Renovate reads and humans see) stay
truthful. Prefer it over the plain preset.

- Extending it next to `config:best-practices` resolves
  `helpers:pinGitHubActionDigests` twice (`treeSummary.duplicates: 1`) —
  harmless, the rule is idempotent.
- The optional minor/patch tolerate actions that publish only major tags. They
  are not licence to write `# v4` when `v4.1.2` exists: the exact release is
  the form, and an unreleased first-party action may sit on `@main` with a
  TODO until its first tag.
- A SHA pin with no version comment, or with a prose comment, is
  `skipReason: unversioned-reference`; no preset can help it. `# main` is
  branch tracking (`github-digest`), a different mode. The full table is in
  `reference/github-actions.md`.
- A user rule copied from the pre-44.43.0 body with `matchDepTypes: ["action"]`
  stops covering reusable workflows on >= 44.43.0.

## `customManagers:dockerfileVersions` and `customManagers:githubActionsVersions`

Regex custom managers driven by a marker comment on the line **above** the
value (the regex is `# renovate: …`, whitespace, then an `ENV`/`ARG` or YAML
key ending in `_VERSION`). The Dockerfile one matches `**/[Dd]ockerfile*`,
`**/[Cc]ontainerfile*` and `*.Dockerfile`; the Actions one matches
`.github|.gitea|.forgejo/(workflows|actions)/**/*.ya?ml`,
`workflow-templates/` and any `action.ya?ml` — a composite action's `env:`
block counts:

```yaml
env:
  # renovate: datasource=npm depName=renovate
  RENOVATE_VERSION: "44.42.1"
```

Optional keys: `packageName=` (registry lookup name when it differs from
`depName`), `versioning=`, `extractVersion=`, `registryUrl=`. Any datasource
works (`datasource=crate depName=cargo-index`). Extending these presets beats
hand-writing the same regex: the preset tracks Renovate's own escaping rules,
and the marker is the convention other tools already understand.

Scope: `_VERSION` variables **only**. `githubActionsVersions` never touches
`uses:` lines — the SHA + `# vX.Y.Z` pair there is the `github-actions`
manager's own replace template, kept semver by
`helpers:pinGitHubActionDigestsToSemver` — and `with:` inputs of well-known
setup actions (`node-version`, pnpm `version`) are extracted natively as
`depType: uses-with` without any marker. `extract_deps` cannot run regex
managers; prove a marker by running the preset's `matchStrings` against the
file.

## `config:recommended` facts that surprise people

- It enables `dependencyDashboard`, so a repo "suddenly" gets a dashboard issue.
- It carries ~720 `packageRules`. Any rule the user writes lands **after**
  them, which is why a user's rule appears as index 700-something in
  provenance. Their rules still win for the keys they set, because later
  rules override earlier ones for the same key.
- Its monorepo groups (`group:monorepos`) are why `@types/react` and `react`
  travel in one "react monorepo" PR. To split them, a later rule with
  `groupName: null` for that package is the documented escape hatch.
- `:ignoreModulesAndTests` inside it sets `ignorePaths` for `node_modules`,
  `bower_components`, `vendor`, `examples`, `__tests__`, `test(s)`,
  `__fixtures__` — a "Renovate ignores my example project" report is usually
  this.
- `semanticCommits` stays at its default `auto` (`get_provenance` shows
  `defaults`): prefixes appear only when Renovate detects conventional commits
  in recent history, so a PR-title lint can fail on a fresh repo — extend
  `:semanticCommits` where the lint is mandatory.
- It sets no PR rate limits. `prHourlyLimit: 2` is Renovate's own default
  (`get_provenance prHourlyLimit` → `winner: defaults`, `isDefaultOnly: true`),
  which is why a freshly onboarded repo's updates trickle in — and why a
  repo's `prHourlyLimit: 0` is a change from the default, not a preset
  override.
- It does **not** pin digests, does **not** set `minimumReleaseAge`, and does
  **not** enable lock file maintenance; those are `config:best-practices`.
