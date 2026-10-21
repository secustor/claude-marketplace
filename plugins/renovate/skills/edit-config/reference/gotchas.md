# Things that bite (verified against Renovate 44.x, re-verify per pin)

Moved here from SKILL.md; the facts most likely to make an edit wrong are
repeated there under "Facts that change the edit". Longer treatments:
relative refs in `preset-repos.md` and `debugging.md`; `renovate/*` branches,
version gating and dry-run in `debugging.md`; `postUpgradeTasks`, update-type
and manager blocks, `matchSourceUrls` and `matchCategories` in
`matchers-and-rules.md`; RE2 and the validator fallback in `custom-managers.md`.

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
