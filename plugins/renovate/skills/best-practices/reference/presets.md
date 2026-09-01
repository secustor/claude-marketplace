# What the recommended presets actually contain

Resolved with Renovate 44.42.1 through the debugger (`run_config` +
`get_preset_node`). Re-resolve before quoting numbers or bodies — internal
presets change between releases, and a consumer's pinned Renovate may be older.

## `config:best-practices`

Pure router. It sets nothing itself and extends, in this order:

| preset                            | what it contributes                                                                                       |
| --------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `config:recommended`              | the baseline: dependency dashboard, `ignorePaths` for vendored dirs, ~700 monorepo/grouping rules, semantic commits, `:configMigration` is **not** in here |
| `docker:pinDigests`               | `pinDigests: true` for docker-type dependencies                                                           |
| `helpers:pinGitHubActionDigests`  | `pinDigests: true` for `depType: action` (GitHub Actions by SHA)                                          |
| `:configMigration`                | `configMigration: true` — Renovate opens a PR rewriting deprecated options in the repo's own config file |
| `:pinDevDependencies`             | `rangeStrategy: pin` for `devDependencies`                                                                |
| `abandonments:recommended`        | flags abandoned packages (4 rules)                                                                        |
| `security:minimumReleaseAgeNpm`   | `minimumReleaseAge` for npm packages (5 rules); this is where the "3 days" a user never wrote comes from   |
| `:maintainLockFilesWeekly`        | `lockFileMaintenance: { enabled: true, schedule: ["before 4am on monday"] }`                              |

So `config:best-practices` is `config:recommended` plus supply-chain hardening:
digest pinning for docker and actions, a release-age gate for npm, pinned dev
deps, weekly lock file refresh and config self-migration. Recommend it when the
repo can tolerate PRs that rewrite `uses:` lines to SHAs; otherwise start from
`config:recommended` and add the pieces individually.

## `helpers:pinGitHubActionDigestsToSemver`

Extends `helpers:pinGitHubActionDigests` and adds a rule that makes Renovate keep
the `# vX.Y.Z` comment behind the SHA in sync:

```json
{
  "matchDepTypes": ["action"],
  "extractVersion": "^(?<version>v?\\d+\\.\\d+\\.\\d+)$",
  "versioning": "regex:^v?(?<major>\\d+)(\\.(?<minor>\\d+)\\.(?<patch>\\d+))?$"
}
```

Prefer it over the plain `pinGitHubActionDigests` preset: the SHA is what runs,
the semver comment is what humans read, and this keeps both truthful. Actions
that only publish major tags (`v4`) still update because the regex makes minor
and patch optional.

## `customManagers:dockerfileVersions` and `customManagers:githubActionsVersions`

Both are regex custom managers driven by a marker comment. The Dockerfile one
matches `**/[Dd]ockerfile*`, `**/[Cc]ontainerfile*` and `*.Dockerfile`; the
Actions one matches `.github/workflows/*.y(a)ml`, `.github/actions/**` and
`action.yml`. The line they need directly above an `ENV`/`ARG` (or a YAML
`KEY: value`) whose name ends in `_VERSION`:

```dockerfile
# renovate: datasource=github-releases depName=nodejs/node versioning=node
ARG NODE_VERSION=22.4.1
```

Optional keys in the comment: `packageName=` (registry lookup name when it
differs from `depName`), `versioning=`, `extractVersion=`, `registryUrl=`.
Extending these presets beats hand-writing the same regex: the preset tracks
Renovate's own escaping rules for `managerFilePatterns`, and the marker comment
is the convention other tools (and other people's presets) already understand.

## `config:recommended` facts that surprise people

- It enables `dependencyDashboard`, so a repo "suddenly" gets a dashboard issue.
- It carries ~720 `packageRules`. Any rule the user writes lands **after**
  them, which is why a user's rule appears as index 700-something in provenance.
  Their rules still win for the keys they set, because later rules override
  earlier ones for the same key.
- Its monorepo groups (`group:monorepos`) are why `@types/react` and `react`
  travel in one "react monorepo" PR. To split them, a later rule with
  `groupName: null` for that package is the documented escape hatch.
- `:ignoreModulesAndTests` inside it sets `ignorePaths` for `node_modules`,
  `bower_components`, `vendor`, `examples`, `__tests__`, `test(s)`, `__fixtures__`
  — a "Renovate ignores my example project" report is usually this.
- It does **not** pin digests, does **not** set `minimumReleaseAge`, and does
  **not** enable lock file maintenance; those are `config:best-practices`.
