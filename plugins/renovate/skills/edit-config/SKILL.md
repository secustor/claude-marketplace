---
name: edit-config
description: Edit, write or migrate a Renovate configuration — renovate.json / renovate.json5 / .renovaterc, the `renovate` key of package.json, a shared preset repository, customManagers — and prove the edit with the renovate-config-debugger tools before handing it over. Use when asked to add, change or remove a packageRule, grouping, automerge, schedule, minimumReleaseAge, an ignore, a custom (regex) manager, to migrate deprecated options, or to restructure a preset repo. For diagnosing an existing config without changing it, use renovate-config-debugger's debug-renovate-config skill instead.
---

# Editing a Renovate config

A Renovate config is not what its file says. `extends` pulls in ~1,100 presets
and ~700 `packageRules` before the first line the user wrote takes effect; the
user's own rules land **last** in that merged array; options nest inside
update-type blocks that only apply when the update's type matches; deprecated
syntax is rewritten silently before validation. Every edit is therefore made
against the *resolved* config, and every edit is proven with a simulation, not
argued from the diff.

Tools: the MCP server `rcd` from the `renovate-config-debugger` plugin (this
plugin depends on it). `run_config` first, everything else by `runId`. No MCP?
`npx -y @renovate-config-debugger/cli <validate|digest|provenance|simulate|compare|group|docs>`
gives the same answers with `--format json`. Option semantics: `get_option_docs`,
never memory — they change every release, and the pinned version is in the
answer.

## The loop

### 1. Find every file that is in play

Renovate reads the first hit from `renovate.json`, `renovate.json5`,
`.github/renovate.json[5]`, `.gitlab/renovate.json[5]`, `.renovaterc`,
`.renovaterc.json[5]`, and the `renovate` key of `package.json`. Then read its
`extends`: if the repo extends an org preset (`local>org/renovate-config`,
`github>org/repo`), most of the effective config lives *there*. Decide which
file the change belongs in before touching anything — a rule every repo of
the org needs goes into the shared preset, not into one consumer.

### 2. Resolve before you edit

`run_config` the current text. Read `accepted`, the warnings, and the digest.
Then `get_provenance` with the key you are about to change: it tells you who
sets it today (a preset, the repo, a default) and whether it concatenates
(`packageRules`, `description`, `ignorePaths`) or overrides (scalars). A value
that is "not set" may be set inside an update-type block or a manager block
— `get_provenance` cannot see those; `simulate` a dependency to find out.

Fix `accepted: false` first. A rejected config runs nothing; any reasoning
about behaviour on top of it is void.

### 3. Put the change where it composes

| you want                                    | write                                                                                           | not                                                                    |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| a per-dependency setting                    | a `packageRules` entry with a `description`                                                     | a top-level option that hits everything                                |
| lock file maintenance settings              | the top-level `lockFileMaintenance` block — it merges with `:maintainLockFilesWeekly`             | a rule on `matchUpdateTypes: ["lockFileMaintenance"]` (overrides instead) |
| matchers of a built-in group                | `extends: ["monorepo:x"]` (or `packages:x`) in your rule, restating `groupName` and `matchUpdateTypes` | `extends: ["group:x"]` in a rule — Renovate warns, it is a nested `packageRules` |
| skip a directory of manifests               | `matchFileNames` + `enabled: false` in a rule                                                   | fighting a preset's `ignorePaths`; `ignoreFileName` does not exist     |
| automerge only minor/patch                  | `matchUpdateTypes: ["minor","patch"]` on the rule                                               | `extends: [":automergeMinor"]` on a rule — it scopes only `automerge`, sibling keys hit majors too |
| all packages except X                       | `matchPackageNames: ["!X"]`                                                                     | `["*", "!X"]` — rejected by current validators                         |
| a rationale                                 | `description` (string or array, links welcome)                                                  | `comment` — invalid key, config refused                                |
| a rule for regex-manager deps               | `matchManagers: ["custom.regex"]`                                                               | `"regex"`                                                              |
| track a version in a Dockerfile / workflow  | `customManagers:dockerfileVersions` / `customManagers:githubActionsVersions` + a `# renovate:` marker comment | a hand-written regex manager                                  |
| an org-wide setting                         | the shared preset every repo extends                                                            | the same lines copied into each repo                                   |

`reference/matchers-and-rules.md` has the semantics behind each row; read it
before writing a rule with more than one matcher, a negation, a glob, or an
update-type block.

### 4. Write the edit

- Keep the file's format, comments and key order. JSON5 comments are fine in a
  `.json5` file; a `.json` file gets `description` instead.
- Write current syntax. If the file still carries deprecated options, the
  `run_config` digest says so ("rewrote N deprecated options"); rewrite them
  with `reference/migrations.md` while you are there, or leave them alone and say
  so — do not half-migrate.
- One rule per intent, each with a `description`. Put the issue or discussion
  link in it; the next reader will not re-derive why `express` is held below v5.
- Exact package names beat scope globs when the scope also holds packages that
  must not join (`@oxlint/**` sweeps in platform binaries).
- When inlining a built-in group, copy its whole body: `matchUpdateTypes`
  included, or the rule widens to `pin` updates.
- For a shared preset repo, use relative references (`./presets/x`) between its
  own files and keep the extension when the file is not `.json`
  (`./app.json5`). The repo's own `renovate.json` must not use them. See
  `reference/preset-repos.md`.

### 5. Resolve after, then prove

`run_config` the edited text. `accepted` must be true; new warnings must be
intended. Then:

- **`compare_simulations`** (`runId` before, `runIdB` after) with a dependency
  that *should* change, and a second one that *must not*. Set the fields the
  matchers read (`depName`, `packageName`, `datasource`, `manager`, `depType`,
  `currentValue`/`newValue` or `updateType`, `sourceUrl` for `matchSourceUrls`,
  `packageFile` for `matchFileNames`). A field left unset makes that clause
  `no-input`, which counts as no match — a "rule did not fire" answer built on
  a missing field is not an answer.
- Read the two axes separately: `verdict: identical | differs | documentation-only`
  is behaviour; `identity` is selector text. Selector churn is not a
  behaviour change, and `noChange` on a cleanup is the point.
- **`simulate_group`** when the edit is a grouping: it tallies whether the
  group forms for the deps you give it. Its `wouldForm: false` means "not with
  these updates alone".
- **`extract_deps`** when the edit is a custom manager: a typo in a named group
  (`datasoure=`) extracts nothing and says nothing. Prove the manager by
  extracting from the real file.
- A migration or cleanup must compare `identical`.

### 6. Hand it over

One verdict sentence first ("Minor and patch updates of `@types/*` now
automerge; `@types/node` and every major still open a PR"), then the diff,
then the evidence: the Renovate version that resolved it, before/after
verdicts, which dependency fields the simulation had, and what a preset fetch
could not reach if any node failed. Frame counts ("734 rules: 1 yours, 733
from `local>org/renovate-config`"). Do not hedge in conditionals about the
config in hand; state what is true for *this* config. Do not claim a check you
did not run.

## Things that bite (verified against Renovate 44.x, re-verify per pin)

- Repo `packageRules[0]` is merged index ~713 under `config:recommended`; later
  rules win per key. A rule you add after a `devDependencies: enabled: false`
  rule cannot re-enable what that rule disabled unless it sets `enabled: true`.
- `matchSourceUrls`-based rules (`monorepo:*`, 446 of the recommended rules)
  fire on `sourceUrl`, not on the name. When upstream moves its repo the group
  silently stops.
- `matchCategories` matches the *manager's* declared categories, not the
  package.
- `packageName` defaults to `depName`; `matchCurrentValue` reads the raw range,
  `matchCurrentVersion` the resolved version.
- Update-type blocks (`minor: {…}`, `patch: {…}`, `lockFileMaintenance: {…}`)
  merge up only for updates of that type; rule-level keys next to them do not.
- A manager block (`nuget: { ignorePaths: [...] }`) *replaces* the array at top
  level for that manager. `:ignoreModulesAndTests` uses this to keep nuget's
  `test/` projects — "best-practices ignores test dirs" is false for nuget.
- `force` in a repo config triggers a "global only" warning and is applied
  anyway.
- Regexes are RE2: no lookahead, no lookbehind, no backreferences; an
  unsupported construct is a config validation error, not a fallback.
- Relative preset refs need a repo-hosted parent preset and cannot escape the
  repo (`PRESET_RELATIVE_NO_PARENT`, `PRESET_RELATIVE_OUTSIDE_REPO`).
- `postUpgradeTasks` that exit non-zero make Renovate commit nothing and report
  an artifact problem on every PR.
- Commits pushed onto a `renovate/*` branch stop Renovate from rebasing it.
