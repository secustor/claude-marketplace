---
name: edit-config
description: Edit, write or migrate a Renovate configuration — renovate.json / renovate.json5 / .renovaterc, the `renovate` key of package.json, a shared preset repository, customManagers — and prove the edit with the renovate-config-debugger tools and the pinned renovate-config-validator before handing it over. Use when asked to add, change or remove a packageRule, grouping, automerge, schedule, minimumReleaseAge, a first-party exemption, a pinDigests carve-out for reusable workflows, an ignore or version hold, a custom (regex) manager or `# renovate:` marker, to migrate deprecated options, or to restructure a preset repo. For diagnosing an existing config without changing it, use renovate-config-debugger's debug-renovate-config skill instead.
---

# Editing a Renovate config

A Renovate config is not what its file says. `extends` pulls in ~1,100 presets
and ~700 `packageRules` before the first line the user wrote takes effect; the
user's own rules land **last** in that merged array; options nest inside
update-type blocks that flatten *after* the rules; deprecated syntax is
rewritten silently before validation. Every edit is therefore made against the
*resolved* config, and every edit is proven with a simulation, not argued from
the diff.

Tools: the MCP server `rcd` from the `renovate-config-debugger` plugin (this
plugin depends on it). `run_config` first, everything else by `runId`. No MCP?
`npx -y @renovate-config-debugger/cli <validate|digest|provenance|simulate|compare|group|docs>`
gives the same answers with `--format json`. The CLI validator is a second,
narrower check:
`npx -y -p renovate@<the bot's pinned version> renovate-config-validator --strict --no-global <file>`
(Node 24; `reference/validation.md` lists what it cannot see). Option
semantics: `get_option_docs`, never memory and never a shallow read of
`renovate-schema.json` (its top-level `properties` hide options behind `$ref`)
— they change every release, and the pinned version is in the answer.

## The loop

### 1. Find every file that is in play

Renovate reads the first hit from `renovate.json`, `renovate.json5`,
`.github/renovate.json[5]`, `.gitlab/renovate.json[5]`, `.renovaterc`,
`.renovaterc.json[5]`, and the `renovate` key of `package.json`. Then:

- Is the file generated? If `.projen/files.json` lists it, the source is
  `.projenrc.ts` and an edit to the rendered file is overwritten on the next
  `projen` run; a `.projen/` directory that does not list it means hand edits
  stick. Look for the same in other generators before editing.
- Read its `extends`. A repo usually extends a team or product overlay that in
  turn extends the org preset (`local>org/renovate-config`, `github>org/repo`);
  most of the effective config lives *there*, and the change may belong in the
  overlay, in the org preset, or in the repo. A rule every repo of the org
  needs goes into the shared preset, not into one consumer; a consumer's own
  file should end up as `$schema` + `extends`, with local `packageRules` only
  for a repo-specific reason.
- Find the runner's pinned Renovate version (its `package.json`, or the CI
  validator's pin). It decides which options and managers exist.

### 2. Resolve before you edit

`run_config` the current text. Read `accepted`, `stageStatus`, `presetErrors`,
the warnings and the digest. `accepted: true` means nothing while
`stageStatus.preset` is `error` or `presetErrors` is non-empty: the failed
preset expanded into 0 presets and everything downstream describes an empty
config. Supply `RCD_GITHUB_TOKEN` / `GITHUB_TOKEN` for private presets, or read
the file with
`gh api "repos/<org>/<repo>/contents/<path>.json?ref=<tag>" --jq .content | base64 -d`.
A `treeSummary.duplicates` above zero means one preset is reached under two
spellings and its rules and labels are doubled.

Then `get_provenance` with the key you are about to change: it tells you who
sets it today (a preset, the repo, a default) and whether it concatenates
(`packageRules`, `customManagers`, `hostRules`, `description`, `addLabels`) or
overrides (scalars, `schedule`). A value that is "not set" may be set inside an
update-type block or a manager block — `get_provenance` cannot see those;
`simulate` a dependency to find out.

Fix `accepted: false` and preset errors first. A rejected config runs nothing
(Renovate raises the "Action Required: Fix Renovate Configuration" issue and
opens no branches); any reasoning about behaviour on top of it is void.
`reference/debugging.md` maps the error texts to their causes.

### 3. Put the change where it composes

| you want                                                             | write                                                                                                                                                                                                                                | not                                                                                                                                                                       |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| a per-dependency setting                                             | a `packageRules` entry with a `description`                                                                                                                                                                                          | a top-level option that hits everything                                                                                                                                   |
| lock file maintenance settings                                       | the top-level `lockFileMaintenance` block — it merges with `:maintainLockFilesWeekly`                                                                                                                                                | a rule on `matchUpdateTypes: ["lockFileMaintenance"]` (overrides instead)                                                                                                 |
| matchers of a built-in group                                         | `extends: ["monorepo:x"]` (or `packages:x`) in your rule, restating `groupName` and `matchUpdateTypes`                                                                                                                               | `extends: ["group:x"]` in a rule — Renovate warns, it is a nested `packageRules`                                                                                          |
| skip a directory of manifests                                        | `matchFileNames` + `enabled: false` in a rule                                                                                                                                                                                        | fighting a preset's `ignorePaths`; `ignoreFileName` does not exist                                                                                                        |
| automerge only non-majors                                            | `matchUpdateTypes: ["!major"]` on the rule                                                                                                                                                                                           | `extends: [":automergeMinor"]` on a rule — it scopes only `automerge`, sibling keys hit majors too; a seven-item list that forgets `replacement`                          |
| automerge everything non-breaking (0.x bumps included)               | `matchJsonata: ["isBreaking = false"]`; its inverse `isBreaking = true` for the review lane. The debugger cannot simulate either — say so                                                                                             | `matchUpdateTypes: ["!major"]` plus a hand-written 0.x guard                                                                                                              |
| a dependency whose next major breaks the build                       | nothing, when a required check catches it: the PR opens, stays red, and merges when upstream unblocks                                                                                                                                 | `allowedVersions` (only when CI cannot catch the break, and then with a `description`); `ignoreDeps` (never)                                                              |
| all packages except X                                                | `matchPackageNames: ["!X"]`                                                                                                                                                                                                          | `["*", "!X"]` — rejected by current validators                                                                                                                            |
| a rationale                                                          | `description` saying what the rule is **for**; where the justifying link lives (description, README, PR body) is the org's style, and shared preset repos may cap descriptions (purpose only, 150 chars)                             | `comment` — invalid key, config refused                                                                                                                                   |
| exempt first-party packages from `minimumReleaseAge`                 | one rule per identity, each `minimumReleaseAge: null`: `matchSourceUrls`, `matchRegistryUrls`, a namespace in `matchPackageNames`, and `matchManagers: ["github-actions"]` + `matchPackageNames: ["Org/**"]`                          | one rule with all matchers (ANDed); a URL in `matchPackageNames` (never matches)                                                                                          |
| keep a reusable workflow on a tag ref (OIDC `job_workflow_ref` trust) | `{ matchManagers: ["github-actions"], matchPackageNames: ["Org/repo"], pinDigests: false }` — or `matchDepTypes: ["workflow"]` on >= 44.43.0 — merged **before** existing pins are reverted                                          | `matchPackageNames: ["Org/*/.github/workflows/*"]` — the name is `owner/repo`, this matches nothing                                                                       |
| a reminder tied to one dependency's next bump                        | `prBodyNotes` on a repo-local rule                                                                                                                                                                                                   | a README bullet nobody reads at merge time                                                                                                                                |
| a different cadence for one class of updates                         | `schedule` inside that rule                                                                                                                                                                                                          | a second top-level `schedule:*` preset — `schedule` is not mergeable, it replaces the repo-wide window                                                                     |
| a rule for regex-manager deps                                        | `matchManagers: ["custom.regex"]`                                                                                                                                                                                                    | `"regex"` (migrated silently; write the current spelling)                                                                                                                 |
| track a version in a Dockerfile / workflow / script                  | `customManagers:dockerfileVersions` / `customManagers:githubActionsVersions` + a `# renovate:` marker on the line **above** the `_VERSION` variable                                                                                    | a hand-written regex manager; an invented marker format                                                                                                                   |
| an org-wide setting                                                  | the shared preset every repo extends                                                                                                                                                                                                 | the same lines copied into each repo                                                                                                                                      |

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
- One rule per intent, each with a `description` that states its purpose.
  Where the justification and link live is the org's style; follow the preset
  repo's convention and length cap.
- Exact package names beat scope globs when the scope also holds packages that
  must not join (`@oxlint/**` sweeps in platform binaries). A plain string
  beats a regex that cannot distinguish anything the manager emits.
- When inlining a built-in group, copy its whole body: `matchUpdateTypes`
  included, or the rule widens to `pin` updates.
- Delete what is already there: an `extends` entry the chain reaches anyway, a
  repo rule the overlay already sets (`get_provenance` names the writer).
- For a shared preset repo, reference sibling files root-anchored
  (`/presets/x`; `./x` and `../x` are legal but break when the referencing
  file moves) and keep the extension when the file is not `.json`
  (`/app.json5`). No `#tag`, `source>` prefix or `:subpreset` on a relative
  ref — the tag is inherited. The repo's own `renovate.json`, inherited config
  and `globalExtends` must use the absolute form. See `reference/preset-repos.md`.

### 5. Resolve after, then prove

`run_config` the edited text. `accepted` must be true with `stageStatus.preset`
clean; new warnings must be intended. Then:

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
- **A custom manager** is proven by extraction — but not with `extract_deps`
  when it is a `customType: regex` manager (`regex is not supported in the
  browser engine`; `extract_deps` finds which *built-in* managers claim a
  file). Run the `matchStrings` in node or Python against the real file, or
  `LOG_LEVEL=debug renovate --dry-run=extract` in a clone; recipes in
  `reference/custom-managers.md`. A typo in a group name (`datasoure=`)
  extracts nothing and says nothing.
- **`matchJsonata`** rules cannot be simulated: the clause comes back
  `no-match` with `readFields: []`. Do not report a verdict for them; report
  the limitation and prove on a real PR.
- A `pinDigests: false` exemption is proven with `updateType: "pinDigest"`
  (`manager: "github-actions"`, `packageName: "Owner/repo"`), reading
  `pinDigests` in the result.
- A migration or cleanup must compare `identical`.
- Finish with the CLI validator on the pinned version (Tools, above). It is
  necessary, not sufficient: it never fetches top-level `extends`, accepts
  relative refs anywhere, and checks no enum values — `reference/validation.md`.

### 6. Hand it over

One verdict sentence first ("Minor and patch updates of `@types/*` now
automerge; `@types/node` and every major still open a PR"), then the diff,
then the evidence: the Renovate version that resolved it, before/after
verdicts, which dependency fields the simulation had, what a preset fetch
could not reach if any node failed, and which claims (a `matchJsonata` rule, a
regex manager) the tools could not prove. Frame counts ("734 rules: 1 yours,
733 from `local>org/renovate-config`"). Do not hedge in conditionals about the
config in hand; state what is true for *this* config. Do not claim a check you
did not run.

## Things that bite (verified against Renovate 44.x, re-verify per pin)

- Repo `packageRules[0]` is merged index ~713 under `config:recommended`; later
  rules win per key. A rule you add after a `devDependencies: enabled: false`
  rule cannot re-enable what that rule disabled unless it sets `enabled: true`.
- `matchSourceUrls`-based rules (`monorepo:*`, 446 of the recommended rules)
  fire on `sourceUrl`, not on the name. When upstream moves its repo the group
  silently stops; a private crate registry without `api` in `config.json`
  never supplies `sourceUrl` at all.
- `matchCategories` matches the *manager's* declared categories, not the
  package.
- `packageName` defaults to `depName`; `matchCurrentValue` reads the raw range,
  `matchCurrentVersion` the resolved version. For `github-actions` the name is
  always `owner/repo` — a URL or a workflow path in `matchPackageNames` matches
  nothing.
- Update-type blocks (`minor: {…}`, `major: {…}`, `lockFileMaintenance: {…}`)
  flatten **after** the rules, for updates of that type only: a preset's
  `major: { automerge: false }` beats a rule's `automerge: true`, and
  rule-level keys next to a block do not merge up.
- A manager block (`nuget: { ignorePaths: [...] }`) *replaces* the array at top
  level for that manager. `:ignoreModulesAndTests` uses this to keep nuget's
  `test/` projects — "best-practices ignores test dirs" is false for nuget.
- `force` in a repo config triggers a "global only" warning and is applied
  anyway.
- Regexes are RE2: no lookahead (`invalid perl operator: (?=`), no lookbehind,
  no backreferences; an unsupported construct is a config validation error
  when RE2 loads. The CLI validator falls back to JS `RegExp` on a Node/ABI
  mismatch with `WARN: RE2 not usable, falling back to RegExp: regex
  validation may be inaccurate`, and then passes lookaheads that fail at run
  time.
- Relative preset refs (`/x`, `./x`) resolve only inside a preset fetched from
  a repo-hosted source (Renovate >= 44.29.0, runner *and* validator). Anywhere
  else — a repo's own config, inherited config, `globalExtends`, or escaping
  the repo — the run fails with `Relative preset reference cannot be resolved
  (<entry>). Relative presets can only be used within presets from a supported
  source, must stay inside their repository, and cannot be templated or used
  outside of a preset (for example in the repository config, inherited
  config, or globalExtends)`. The validator will not tell you: it accepts
  relative refs in any file. The real cause is the WARN `Could not resolve
  relative preset reference` in the preset author's run; a consumer
  neutralises the broken entry with `ignorePresets: ["<entry exactly as written>"]`.
- `postUpgradeTasks` that exit non-zero make Renovate commit nothing and report
  the commit status `renovate/artifacts` = failure plus an `Artifact update
  problem` block that moves to a PR comment when the body is truncated
  (`> This PR body was truncated due to platform limits.`). The branch is
  retried only on a package-file change, a conflict, the rebase checkbox, or a
  `rebase!` title prefix — fixing the cause on `main` does not regenerate the
  open PR. In a grouped branch only `upgrades[0]`'s tasks run.
- `renovate/*` is the bot's prefix. A commit by an author not in
  `gitIgnoredAuthors` stops Renovate rebasing that branch (push a fix and merge
  as-is; letting Renovate recreate the branch discards it). A branch under the
  prefix that the run did not produce is pruned at the end of every run:
  unmodified -> autoclosed and deleted; modified by a human -> retitled
  `<title> - abandoned` with an "Autoclosing Skipped" comment. Exempt:
  `renovate/lock-file-maintenance` and `renovate/reconfigure` (push a config
  there to have Renovate validate it). `renovate/configure` is the onboarding
  branch and is always claimed. Use another prefix for manual PRs.
- `pinDigests: false` stops **future** pins only; existing SHAs stay until
  reverted by hand, and Renovate re-pins within hours if the exemption is not
  merged first.
- `uses: owner/repo@<sha>` with no version comment — or a prose comment — is
  `skipReason: unversioned-reference`: disabled, absent from the dashboard,
  unrescuable by `packageRules`. `# v1.2.3` (the exact release) makes it
  release-tracking, `# main` digest-tracking; they are modes, not spellings.
- A repository `--dry-run` reads the config from the default branch;
  `--base-branches=<pr-branch>` does not test a config edit. Resolve the edited
  text with `run_config`, or push it to `renovate/reconfigure`.
- Features are gated by the consumer's **pinned** Renovate, not latest:
  `workflow` depType and `helpers:pinGitHubActionDigests*` covering workflows
  44.43.0; relative refs 44.29.0; cargo git-tag deps with `package =` renames
  44.20.1; `rust-toolchain` / `smithy` managers 43.288.0; `overrideDescription`
  44.41. `reference/custom-managers.md` has the recipe to date a fix. 44.0.0
  was an accidental major: 43.x -> 44.x carries no breaking changes.
