---
name: edit-config
description: Edit, write or migrate a Renovate config (renovate.json/.json5, .renovaterc, package.json's `renovate` key, a shared preset repo, customManagers) and prove the edit with the renovate-config-debugger tools and the pinned renovate-config-validator before handing it over. Use for any request to add, change or remove a packageRule, grouping, automerge, schedule, minimumReleaseAge, a first-party exemption, a pinDigests carve-out, an ignore or version hold, a custom regex manager or `# renovate:` marker, to migrate deprecated options, or to restructure a preset repo, even when the user only pastes a config and says "make it do X". To diagnose an existing config without changing it, use renovate-config-debugger's debug-renovate-config skill instead.
---

# Editing a Renovate config

## Rules

- Edit against the resolved config, not the file: `extends` adds ~1,100
  presets and ~700 `packageRules` before the repo's own rules, which land
  last; options nest in update-type blocks that flatten after the rules;
  deprecated syntax is rewritten silently before validation.
- Prove every edit with a simulation before and after; a diff proves nothing.
- Take option semantics from `get_option_docs` for the pinned version. They
  change between releases, and a shallow read of `renovate-schema.json` misses
  the options its top-level `properties` hides behind `$ref`.

## Tools

- MCP server `rcd` from the `renovate-config-debugger` plugin (a dependency of
  this plugin); the tools are registered as
  `mcp__plugin_renovate-config-debugger_rcd__<tool>`; this skill uses the short
  names. Call `run_config` first; every other tool takes its `runId`.
- Without MCP:
  `npx -y @renovate-config-debugger/cli <validate|digest|provenance|simulate|compare|group|docs> --format json`
  gives the same answers.
- CLI validator (Node 24):
  `npx -y -p renovate@<pinned> renovate-config-validator --strict --no-global <file>`.
  It is a second, narrower check; `reference/validation.md` lists what it
  cannot see.
- `get_option_docs <name>`: the answer names the Renovate version it describes.

## The loop

Find the files (1) -> `run_config` + `get_provenance` on the current text (2)
-> decide where the change lives (3) -> write it (4) -> `run_config` +
`compare_simulations` + validator on the edited text (5) -> hand over with
evidence (6). Write nothing before step 2 has returned `accepted: true` with a
clean `stageStatus.preset`, unless fixing that is the edit.

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

Stop and ask before editing when:

- the file is generated (`.projen/files.json` lists it): edit the source or ask.
- the change belongs in the shared preset or overlay and that repo is not in
  the session.
- `run_config` reports `presetErrors` for a preset you cannot fetch and no
  token is available.
- the pinned Renovate is older than the option needs (`reference/debugging.md`,
  "Gating on the consumer's pinned version").

### 2. Resolve before you edit

`run_config` the current text. Read `accepted`, `stageStatus`, `presetErrors`,
the warnings and the digest. `accepted: true` means nothing while
`stageStatus.preset` is `error` or `presetErrors` is non-empty: the failed
preset expanded into 0 presets and everything downstream describes an empty
config. Supply `RCD_GITHUB_TOKEN` / `GITHUB_TOKEN` for private presets. Fetch
failures and duplicate presets: `reference/debugging.md`, "rcd pitfalls".

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

| you want | write | not |
| --- | --- | --- |
| a per-dependency setting | a `packageRules` entry with a `description` | a top-level option that hits everything |
| lock file maintenance settings | the top-level `lockFileMaintenance` block (merges with `:maintainLockFilesWeekly`) | a rule on `matchUpdateTypes: ["lockFileMaintenance"]` (overrides instead) |
| matchers of a built-in group | `extends: ["monorepo:x"]` (or `packages:x`) in your rule, restating `groupName` and `matchUpdateTypes` | `extends: ["group:x"]` in a rule (nested rules; warns) |
| skip a directory of manifests | `matchFileNames` + `enabled: false` in a rule | fighting a preset's `ignorePaths`; `ignoreFileName` does not exist |
| automerge only non-majors | `matchUpdateTypes: ["!major"]` on the rule | `extends: [":automergeMinor"]` on a rule (scopes only `automerge`); a seven-item list that forgets `replacement` |
| automerge everything non-breaking (0.x included) | `matchJsonata: ["isBreaking = false"]`; `isBreaking = true` for the review lane | `matchUpdateTypes: ["!major"]` plus a hand-written 0.x guard |
| a major that breaks the build | nothing when a required check catches it: the PR stays red until upstream unblocks | `allowedVersions` (only when CI cannot catch it); never `ignoreDeps` |
| all packages except X | `matchPackageNames: ["!X"]` | `["*", "!X"]` (rejected by the validator) |
| a rationale | `description` saying what the rule is **for**; where the link lives and any length cap are the org's style | `comment` (invalid key, config refused) |
| first-party exemption from `minimumReleaseAge` | one rule per identity (`matchSourceUrls`, `matchRegistryUrls`, a namespace), each `minimumReleaseAge: null` | one rule with every matcher (ANDed) |
| exempt first-party GitHub Actions | `matchManagers: ["github-actions"]` + `matchPackageNames: ["Org/**"]` | a URL in `matchPackageNames` (never matches) |
| keep a reusable workflow on a tag ref (OIDC) | `{ matchDepTypes: ["workflow"], pinDigests: false }` on >= 44.43.0 | `matchPackageNames: ["Org/*/.github/workflows/*"]` (names are `owner/repo`) |
| same on < 44.43.0, or for one repo | `matchManagers: ["github-actions"]`, `matchPackageNames: ["Org/repo"]`, `pinDigests: false` | reverting SHAs before the rule merges (stops future pins only) |
| a reminder tied to one dependency's next bump | `prBodyNotes` on a repo-local rule | a README bullet nobody reads at merge time |
| a different cadence for one class of updates | `schedule` inside that rule | a second top-level `schedule:*` preset (`schedule` is not mergeable, it replaces the repo-wide window) |
| a rule for regex-manager deps | `matchManagers: ["custom.regex"]` | `"regex"` (migrated silently; write the current spelling) |
| a version in a Dockerfile, workflow or script | `customManagers:dockerfileVersions` or `:githubActionsVersions`; `# renovate:` marker above the variable | a hand-written regex manager or marker |
| an org-wide setting | the shared preset every repo extends | the same lines copied into each repo |

Before writing any `match*` list with more than one entry, a `*`, a `/…/`
regex or a `!`, read `reference/string-patterns.md` (recognition and
combination rules, a negation over an enumeration, a glob over a regex).
`reference/matchers-and-rules.md` has the semantics of each matcher; read it
before combining matchers or touching an update-type block.

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
- Prefer `["!X"]` to listing everything that is not X, a glob to a regex that
  says the same thing, and an exact name to a scope glob when the scope also
  holds packages that must not join (`@oxlint/**` sweeps in platform
  binaries).
- When inlining a built-in group, copy its whole body: `matchUpdateTypes`
  included, or the rule widens to `pin` updates.
- Delete what is already there: an `extends` entry the chain reaches anyway, a
  repo rule the overlay already sets (`get_provenance` names the writer).
- In a shared preset repo write sibling refs root-anchored, with the extension
  when it is not `.json`: `/presets/x`, `/app.json5`. No `#tag`, `source>` or
  `:subpreset` on a relative ref; the tag is inherited. A repo's own
  `renovate.json`, inherited config and `globalExtends` take only the absolute
  form. Full rules: `reference/preset-repos.md`.

### Facts that change the edit (verified 44.x; `reference/gotchas.md` has the rest)

- Repo `packageRules[0]` lands at merged index ~713 under `config:recommended`;
  later rules win per key.
- Update-type blocks (`minor: {…}`, `major: {…}`) flatten after the rules: a
  preset's `major: { automerge: false }` beats a rule's `automerge: true`.
- `github-actions` names are always `owner/repo`; a URL or workflow path in
  `matchPackageNames` matches nothing.
- `pinDigests: false` stops future pins only; merge the exemption before
  reverting SHAs.
- Regexes are RE2: no lookahead, lookbehind or backreferences; a validator
  that warns `RE2 not usable` proves nothing about regexes.
- Relative preset refs (`/x`, `./x`) resolve only inside a preset fetched from
  a repo-hosted source (runner and validator >= 44.29.0); anywhere else the
  run fails and the validator stays silent.
- Features are gated by the consumer's pinned Renovate, not latest;
  `reference/gotchas.md` lists the thresholds.

### 5. Resolve after, then prove

`run_config` the edited text. Proceed only when `accepted` is true with
`stageStatus.preset` clean and every new warning intended; otherwise return to
step 4. Then:

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
- **A custom manager** is proven by extraction against the real file.
  `extract_deps` cannot run a `customType: regex` manager (it answers `regex is
  not supported in the browser engine`; use it to see which built-in managers
  claim the file). Run the `matchStrings` in node or Python, or
  `LOG_LEVEL=debug renovate --dry-run=extract` in a clone; recipes in
  `reference/custom-managers.md`. A typo in a group name (`datasoure=`)
  extracts nothing and reports nothing.
- **`matchJsonata`** rules cannot be simulated: the clause comes back
  `no-match` with `readFields: []`. Do not report a verdict for them; report
  the limitation and prove on a real PR.
- A `pinDigests: false` exemption is proven with `updateType: "pinDigest"`
  (`manager: "github-actions"`, `packageName: "Owner/repo"`), reading
  `pinDigests` in the result.
- A migration or cleanup must compare `identical`; anything else means the
  edit changed behaviour, so return to step 4.
- Finish with the CLI validator on the pinned version (Tools, above). It is
  necessary, not sufficient: it never fetches top-level `extends`, accepts
  relative refs anywhere, and checks no enum values (`reference/validation.md`).

### 6. Hand it over

In this order:

1. One verdict sentence: "Minor and patch updates of `@types/*` now automerge;
   `@types/node` and every major still open a PR."
2. The diff.
3. Evidence: the Renovate version that resolved it; before/after
   `compare_simulations` verdicts and the dependency fields each simulation
   had; the CLI validator result on the pinned version; any preset node that
   could not be fetched; every claim the tools could not prove (a
   `matchJsonata` rule, a regex manager).
4. Counts in context: "734 rules: 1 yours, 733 from `local>org/renovate-config`".

State what is true for this config; `get_provenance` already answered "if the
preset sets X". Claim only checks you ran.

Before you hand over:

- [ ] `run_config` on the edited text: `accepted: true`, `stageStatus.preset`
      clean, new warnings intended
- [ ] `compare_simulations` on one dependency that must change and one that
      must not, with every field the matchers read set
- [ ] a grouping edit: `simulate_group` run
- [ ] a custom manager: extraction proven against the real file, or declared
      unproven
- [ ] a `matchJsonata` rule: declared unprovable by simulation
- [ ] a migration or cleanup: verdict `identical`
- [ ] CLI validator run on the pinned Renovate version
- [ ] verdict sentence first, counts in context, no check claimed that was
      not run
