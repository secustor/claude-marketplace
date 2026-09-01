# packageRules: how they merge, match and fail

Everything here was verified against Renovate 44.x through the debugger's
`run_config` / `get_provenance` / `simulate`. Re-verify on the pinned version
the tools report before quoting a number.

## Merge order

1. `extends` resolves depth-first. Each preset's children are merged first, in
   `extends` order, then the preset's own body. The extending file's own body is
   applied last, so **the file wins over what it extends** for scalar keys.
2. `packageRules` **concatenate** across every layer (provenance action
   `concat`). Nothing is replaced. Under `config:recommended` the repo's own
   `packageRules[0]` sits at merged index ~713 of ~714. Renovate's own messages
   cite the merged, 0-based index; the validator cites the file's index. Say
   which one you mean.
3. At apply time every rule is evaluated in merged order against one
   dependency; each matching rule merges its options over the previous ones.
   **Later rules win per key.** Your rules are last, so they win — but only for
   the keys they set. A rule that matches and sets nothing you care about
   changes nothing.
4. After rules, the update-type block for this update (`major`, `minor`,
   `patch`, `pin`, `digest`, `lockFileMaintenance`, `replacement`) is merged up.
   Top-level `automerge: false` with `minor: { automerge: true }` **does**
   automerge minors.
5. A manager block (`npm: {…}`, `nuget: {…}`, `dockerfile: {…}`) is merged over
   the top level for that manager only, and for array options it **replaces**.
   `:ignoreModulesAndTests` uses this: its `nuget.ignorePaths` drops the
   `**/test/**` globs so .NET test projects stay managed.
6. Renovate runs config migration **twice**: on the raw file, and again on the
   resolved config after presets. The second pass flattens `packageRules`
   nested inside a rule (the result of `extends: ["group:x"]` in a rule) and
   normalises `groupName: ["X"]` to `"X"`. Configs that only work because of
   that second pass are working by side effect.

## Matchers

All `match*` keys in one rule are ANDed. Within one list, the pattern rules
from `string-pattern-matching` apply to `matchPackageNames`, `matchDepNames`,
`matchFileNames`, `matchSourceUrls`, `matchRepositories`, `matchBaseBranches`,
`managerFilePatterns`, …:

- plain string = exact match (or a minimatch glob when it contains `*`, `?`,
  `{}`); `/…/` = RE2 regex, `/…/i` for case-insensitive; leading `!` negates.
- At least one positive entry must match **and** every negative entry must
  match. A negation-only list (`["!gradle"]`) means "everything except".
- `*` or `**` next to any other entry is a **validation error** in current
  Renovate. The fix `["*", "!x"]` → `["!x"]` is behaviour-preserving only
  because the remaining entries are all negations; dropping `*` next to a
  positive pattern narrows the rule.
- `matchPackageNames` reads `packageName`, which Renovate defaults from
  `depName` before rules run. `matchDepNames` reads the name as written in the
  manifest.
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
- `matchFileNames` is valid only inside `packageRules` and reads
  `packageFile`. There is no `ignoreFileName`.
- `matchManagers` for custom managers is `custom.regex` / `custom.jsonata`.
- `matchUpdateTypes` values: `major`, `minor`, `patch`, `pin`, `pinDigest`,
  `digest`, `lockFileMaintenance`, `rollback`, `bump`, `replacement`. Built-in
  `group:*` presets set `matchUpdateTypes` to everything except `pin`; a
  hand-copied group without it starts gating pins too.

### Fail-closed inputs

A matcher whose field the dependency does not carry is a **non-match**, not a
skip. `sourceUrl` is missing for many datasources and for any hand-written
simulation; 446 of the ~714 recommended rules read only `sourceUrl`. A clause
that throws (a `matchCurrentVersion` on an exotic versioning) is also a
non-match. So before concluding "the rule did not fire", look at the clause
evidence: `no-input` on the deciding clause means "no evidence", not "no".

## Rule-scoped versus update-type-scoped

Presets like `:automergeMinor`, `:automergePatch`, `:automergeDigest` set
`automerge` **only inside** `minor` / `patch` / `pin` / `lockFileMaintenance`
blocks. A rule that extends one of them still applies its *other* keys
(`labels`, `assignees`, `autoApprove`, `reviewers`) to every matched update,
majors included. Scope the whole rule with `matchUpdateTypes` instead.

Keys that only make sense in update-type blocks are also invisible to a
per-key provenance query; `get_provenance` for `automerge` reports "defaults"
while a rule sets it for minors. `simulate` the dependency to see it.

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

## Disabling and ignoring

- `enabled: false` in a rule sets `skipReason: package-rules`; the simulation
  verdict reads "WOULD NOT be raised at all".
- `ignoreDeps` is a top-level exact-name list; `ignorePaths` is top-level and
  glob-based; both are already populated by `config:recommended`
  (`**/node_modules/**`, `**/bower_components/**`, `**/vendor/**`,
  `**/examples/**`, `**/__tests__/**`, `**/test/**`, `**/tests/**`,
  `**/__fixtures__/**`). "Renovate ignores my example project" is usually this.
- `ignoreDeps: []` inside internal presets is an upstream hack for onboarding
  descriptions, not something to copy (removed in 44.41 in favour of
  `overrideDescription`).

## Options that mislead

- `description` is a mergeable array; presets append theirs. A wrapper preset
  (`{description, extends}` only) loses its own description. Use
  `overrideDescription` (44.41+) to replace the inherited ones.
- `force` is documented as global-only and warned about in a repo config, yet
  the merge applies it. Do not rely on either direction.
- `hostRules` and `env` are honoured only at the top level; nested copies are
  inert. `matchHost` matches the host or a dotted subdomain of it; the longest
  match wins, and a rule with `hostType` beats an equally specific one without.
- `rangeStrategy: "pin"` is a silent no-op when the current value is already a
  single version under the effective versioning (`regex:` versionings never
  support ranges); Renovate resets it to `replace`.
- `extractVersion` rewrites the release version *and* the value Renovate writes
  back. Using it to strip a suffix that the registry needs writes a
  non-existent version into the manifest; `versionCompatibility` is the option
  for "registry version carries a suffix the tag does not".
- `minimumReleaseAge` is enforced by Renovate at lookup time and independently
  by npm (`--before`), pnpm and poetry at install time; a companion package
  published minutes after the main one can still fail the lockfile step.
  `minimumReleaseAgeBuffer` (new in 44.x, default `10 minutes`) exists for that.
- `semanticCommitType` per rule is the reliable way to make a bot bump cut a
  release under semantic-release; the default is only incidentally `fix`.
- Renovate's own warning "Config migration necessary" is logged at debug level
  only. `accepted: true` with no warnings says nothing about deprecated syntax;
  the debugger's digest does ("rewrote N deprecated options").
