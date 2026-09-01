# Deprecated options and what Renovate rewrites them into

Renovate migrates deprecated options in memory on every run, so an old config
keeps working — but the migrated form is what actually executes, and it is
what you should write when you touch a file. The table was produced by running
a config through Renovate 44.42.1's own migration step (`run_config` followed
by `get_resolved_config`); re-verify against the pinned version the debugger
reports if in doubt, because the rewrite rules change between releases.

| you find                                   | Renovate executes                                                | note                                                                  |
| ------------------------------------------ | ---------------------------------------------------------------- | --------------------------------------------------------------------- |
| `extends: ["config:base"]`                 | `extends: ["config:recommended"]`                                | `config:base` is an alias kept only for migration                     |
| `masterIssue: true`                        | `dependencyDashboard: true`                                      | same for `masterIssueTitle` → `dependencyDashboardTitle` etc.         |
| `stabilityDays: 3`                         | `minimumReleaseAge: "3 days"`                                    | number of days becomes a duration string                              |
| `regexManagers: [...]`                     | `customManagers: [{ customType: "regex", ... }]`                 | `customType` is required on every custom manager                      |
| `fileMatch: ["^Dockerfile$"]`              | `managerFilePatterns: ["/^Dockerfile$/"]`                        | `fileMatch` no longer exists as an option in 44.x; regexes get `/…/` |
| `matchPackagePatterns: ["^@types/"]`       | `matchPackageNames: ["/^@types//"]`                              | regex entries are written between slashes                             |
| `matchPackagePrefixes: ["@octokit/"]`      | `matchPackageNames: ["@octokit/{/,}**"]`                         | prefix becomes a glob                                                 |
| `excludePackageNames: ["@types/node"]`     | `matchPackageNames: [..., "!@types/node"]`                       | exclusions become negated entries in the same list                    |
| `excludePackagePatterns`, `excludePackagePrefixes` | negated `/…/` or glob entries in `matchPackageNames`     | same mechanism                                                        |
| `matchPaths`, `matchFiles`                 | `matchFileNames`                                                 |                                                                       |
| `matchLanguages`                           | `matchCategories`                                                |                                                                       |
| `matchSourceUrlPrefixes`                   | `matchSourceUrls` with glob entries                              |                                                                       |
| `packageRules[].paths`                     | `matchFileNames`                                                 |                                                                       |
| `packagePatterns`, `packageNames` (rule)   | `matchPackageNames`                                              | very old syntax, still migrated                                       |
| `pinVersions: true` / `false`              | `rangeStrategy: "pin"` / `"replace"`                             |                                                                       |
| `separateMajorReleases`                    | **not migrated** — `Invalid configuration option` error          | delete it; `separateMajorMinor` (default `true`) is the current knob  |
| `hostRules[].platform`                     | `hostRules[].hostType`                                           |                                                                       |
| `hostRules[].baseUrl` / `domainName` / `hostName` | `hostRules[].matchHost`                                   |                                                                       |
| `schedule: "every weekday"` as a bare string | `schedule: ["every weekday"]`                                  | schedule is always an array after migration                           |
| `automergeType: "branch-push"`             | `automergeType: "branch"`                                        |                                                                       |
| `versionScheme`                            | `versioning`                                                     |                                                                       |
| `baseBranch: "main"` (and `baseBranches`)  | `baseBranchPatterns: ["main"]`                                   | 44.x renamed `baseBranches` too                                       |
| `ignoreNodeModules: true`                  | `ignorePaths: ["node_modules/"]`                                 |                                                                       |
| `:unpublishSafe` preset                    | `security:minimumReleaseAgeNpm`                                  | preset names get migrated as well                                     |

How to use this table:

1. Do not hand-migrate from memory. Run the file through `run_config` and read
   `get_resolved_config` (mode `keep-internal`): the `config` it returns is the
   migrated form of the user's own file with `extends` still in place. That is
   the text to write back, keeping the user's comments and key order.
2. The digest names how many options were rewritten (`It rewrote 7 deprecated
   options`). If the count is zero, do not present a "migration" — there is
   nothing to migrate.
3. Migrating is behaviorally inert by definition. Prove it: `compare_simulations`
   before/after must report `identical` for a representative dependency. If it
   does not, you changed more than the syntax.
4. `npx renovate-config-validator --strict` fails on any option that still needs
   migration; that is the check a CI job or a Claude Code hook should run.
