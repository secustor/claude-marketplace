# Deprecated options and what Renovate rewrites them into

Renovate migrates deprecated options in memory on every run, so an old config
keeps working — but the migrated form is what actually executes, and it is
what you should write when you touch a file. The table was produced by running
configs through Renovate 44.42.1's own migration step (`run_config` followed
by `get_resolved_config`); re-verify against the pinned version the debugger
reports if in doubt, because the rewrite rules change between releases.

| you find                                                                     | Renovate executes                                                                                                  | note                                                                                                                                                                                     |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `extends: ["config:base"]`                                                   | `extends: ["config:recommended"]`                                                                                  | `config:base` is an alias kept only for migration                                                                                                                                        |
| `extends: "config:recommended"` (bare string)                                | `extends: ["config:recommended"]`                                                                                  | the top level is coerced by massage; inside a `packageRules` entry the string is flagged `Config migration necessary` — always write the array                                           |
| `masterIssue: true`                                                          | `dependencyDashboard: true`                                                                                        | same for `masterIssueTitle` → `dependencyDashboardTitle` etc.                                                                                                                            |
| `stabilityDays: 3`                                                           | `minimumReleaseAge: "3 days"`                                                                                      | number of days becomes a duration string                                                                                                                                                 |
| `regexManagers: [...]`                                                       | `customManagers: [{ customType: "regex", ... }]`                                                                   | `customType` is required on every custom manager                                                                                                                                         |
| `matchManagers: ["regex"]`                                                   | `matchManagers: ["custom.regex"]`                                                                                  | migrated silently every run — an old rule keeps matching; write the new spelling                                                                                                         |
| `fileMatch: ["^Dockerfile$"]`                                                | `managerFilePatterns: ["/^Dockerfile$/"]`                                                                          | `fileMatch` no longer exists as an option in 44.x; regexes get `/…/`                                                                                                                     |
| `matchPackagePatterns: ["^@types/"]`                                         | `matchPackageNames: ["/^@types//"]`                                                                                | regex entries are written between slashes                                                                                                                                                |
| `matchPackagePrefixes: ["@octokit/"]`                                        | `matchPackageNames: ["@octokit/{/,}**"]`                                                                           | prefix becomes a glob                                                                                                                                                                    |
| `excludePackageNames: ["@types/node"]`                                       | `matchPackageNames: [..., "!@types/node"]`                                                                         | exclusions become negated entries in the same list                                                                                                                                       |
| `excludePackagePatterns`, `excludePackagePrefixes`                           | negated `/…/` or glob entries in `matchPackageNames`                                                               | same mechanism; a URL prefix under `github-actions` stays inert after migration (names are `owner/repo`)                                                                                 |
| `matchPaths`, `matchFiles`                                                   | `matchFileNames`                                                                                                   |                                                                                                                                                                                          |
| `matchLanguages`                                                             | `matchCategories`                                                                                                  |                                                                                                                                                                                          |
| `matchSourceUrlPrefixes`                                                     | `matchSourceUrls` with glob entries                                                                                |                                                                                                                                                                                          |
| `packageRules[].paths`                                                       | `matchFileNames`                                                                                                   |                                                                                                                                                                                          |
| `packagePatterns`, `packageNames` (rule)                                     | `matchPackageNames`                                                                                                | very old syntax, still migrated                                                                                                                                                          |
| `pinVersions: true` / `false`                                                | `rangeStrategy: "pin"` / `"replace"`                                                                               |                                                                                                                                                                                          |
| `managerBranchPrefix: "deps-"`                                               | `additionalBranchPrefix: "deps-"`                                                                                  | never set `branchName` itself — accepted, but with a deprecation warning                                                                                                                 |
| `transitiveRemediation: true`                                                | _(deleted)_                                                                                                        | the option no longer exists (`get_option_docs` on 44.42.1 knows none); a no-op for a while, and a file carrying it fails `--strict`. No replacement knob                                 |
| `separateMajorReleases`                                                      | **not migrated** — `Invalid configuration option` error                                                            | delete it; `separateMajorMinor` (default `true`) is the current knob                                                                                                                     |
| `hostRules[].platform`                                                       | `hostRules[].hostType`                                                                                             |                                                                                                                                                                                          |
| `hostRules[].baseUrl` / `domainName` / `hostName`                            | `hostRules[].matchHost`                                                                                            | more than one host-matching field is refused                                                                                                                                             |
| `schedule: "every weekday"` as a bare string                                 | `schedule: ["every weekday"]`                                                                                      | schedule is always an array after migration                                                                                                                                              |
| `automergeType: "branch-push"`                                               | `automergeType: "branch"`                                                                                          |                                                                                                                                                                                          |
| `versionScheme`                                                              | `versioning`                                                                                                       |                                                                                                                                                                                          |
| `baseBranch: "main"` (and `baseBranches`)                                    | `baseBranchPatterns: ["main"]`                                                                                     | 44.x renamed `baseBranches` too                                                                                                                                                          |
| `ignoreNodeModules: true`                                                    | `ignorePaths: ["node_modules/"]`                                                                                   |                                                                                                                                                                                          |
| `:unpublishSafe` preset                                                      | `security:minimumReleaseAgeNpm`                                                                                    | preset names get migrated as well                                                                                                                                                        |
| `local>/org/repo`, `github>./org/repo` (source prefix + `/`, `./`, `../`)    | **not migrated** — `Preset is invalid (local>/org/repo)` aborts the run on 44.29+; the validator says `preset "local>./x" is not valid` | resolved on 44.28 because platform APIs normalised the path; drop the slash (`local>org/repo`). Audit old configs when the bot moves past 44.28 |
| bare `/x`, `./x`, `../x` in `extends`                                        | **not migrated** — a relative preset reference since 44.29                                                         | legal only inside a fetched preset; in a repo config, inherited config or `globalExtends` it fails at run time with `Relative preset reference cannot be resolved (<entry>). ...` while the validator passes it |

Dead config worth deleting when touching an old file (defaults, not
migrations): `internalChecksFilter: "strict"`, `separateMajorMinor: true`,
`platformAutomerge: false` next to `automerge: false`.

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
4. Without the debugger, use the module Renovate runs itself. Install the
   pinned version in a scratch directory
   (`npm install --prefix /tmp/rv renovate@<pinned>`), then:

   ```sh
   node -e "const {migrateConfig}=require('/tmp/rv/node_modules/renovate/dist/config/migration.js');
            const r=migrateConfig(require(process.argv[1]));
            console.log(r.isMigrated); console.log(JSON.stringify(r.migratedConfig,null,2))" ./renovate.json
   ```

   Write `migratedConfig` back (keeping formatting), then run it again on the
   edited file: `isMigrated === false` plus a key-order-normalised deep-equal
   against the first output proves the file is a migration no-op, and
   therefore behaviourally identical.
5. The CI gate is
   `npx -y -p renovate@<the bot's pinned version> renovate-config-validator --strict --no-global <file>`.
   `renovate-config-validator` is a bin inside the `renovate` package, not a
   package (`npx renovate-config-validator` is a 404), and bare `-p renovate`
   can run a stale cached major that treats `--no-global` as a file name.
   `--no-global` validates the file as **repo** config; a positional file is
   otherwise validated as the permissive global config and `autodiscover`
   passes. `--strict` adds exactly one thing: the ` WARN: Config migration
   necessary` line, which every validator run prints together with a
   `Config migration diff:`, becomes exit 1 — write exactly what the diff
   shows, never a hand-guessed rewrite. Any other warning already exits 1
   without `--strict`. Renovate 44.x needs Node 24. A green run is a necessary
   gate, not a sufficient one: the validator never fetches top-level `extends`
   (a nonexistent tag or path passes), accepts relative references in any
   file, checks no enum values or array element types
   (`matchUpdateTypes: ["bogusvalue"]` passes; only `$schema` catches it), has
   no preset mode (`isPreset=false`, so bare-selector presets must be wrapped
   in `packageRules`), does resolve `packageRules[].extends` (default branch,
   `GITHUB_COM_TOKEN`, never `RENOVATE_TOKEN`), and may fall back from RE2 to
   JS `RegExp` with a WARN. `reference/validation.md` has the full list.
6. A green `renovate/reconfigure` branch check does not prove current syntax:
   the reconfigure worker migrates before it validates, so a file full of
   deprecated names still reads "Validation Successful". `:configMigration`
   (in `config:best-practices`) opens the "Migrate config renovate.json" PR
   that writes the migrated form back; a Renovate run otherwise logs
   "Config migration necessary" at debug level only.
