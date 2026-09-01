# Custom managers

Reach for a custom manager only when no built-in manager and no
`customManagers:*` preset covers the file **in the Renovate version the
consumer's runner pins** — not in `latest`. Order of preference:

1. **A built-in manager that exists in the pinned version.** Renovate has ~129;
   `extract_deps` with the file name and contents lists the managers that claim
   it (`matchedManagers`) and what they extract — on the debugger's version,
   which the answer names. Several claim the same name (`pyproject.toml`:
   pep621, pixi, poetry); ~11 have no file pattern and must be enabled
   explicitly (`kubernetes`, `argocd`, `tekton`, …) via `managerFilePatterns`
   in their manager block.

   Confirm on the pin before relying on a manager or deleting a regex manager:

   - The runner's pin is in its `package.json` (or the CI validator's
     `RENOVATE_VERSION`). In that install:
     `ls node_modules/renovate/dist/modules/manager/ | grep <name>`, or
     `renovate-config-validator --strict --no-global` on
     `{"<manager>":{"enabled":true}}` (an unknown manager is an error).
   - To date a fix: `sha=$(gh api repos/renovatebot/renovate/pulls/<N> --jq .merge_commit_sha)`,
     then `gh api "repos/renovatebot/renovate/compare/$sha...<tag>" --jq .status`
     per release tag — `ahead` = the tag contains it, `behind` = not. Renovate
     cuts several releases a day (`44.20.0: behind`, `44.20.1: ahead`).
   - Thresholds that retired hand-written managers: `rust-toolchain` and
     `smithy` since 43.288.0; `cargo` git-sourced dependencies
     (`git = …, tag = …` → `github-tags` / `gitlab-tags` / `git-tags`,
     `rev =` → a `git-refs` digest, `branch =` → `skipReason: git-dependency`)
     since 41.108.0, with `package =` renames only from 44.20.1
     (renovatebot/renovate#45142); `depType: workflow` for reusable-workflow
     calls since 44.43.0.

   Between deleting the regex manager and the runner reaching the floor the
   dependencies are simply undiscovered — bump the runner first, delete second,
   and say in the PR what the built-in now also raises (digest PRs for `rev`
   pins). If the built-in has a gap for the repo's real pattern, fix Renovate
   and gate the deletion on the fix reaching the pin; do not keep the regex
   manager as a permanent workaround.

   `rust-toolchain.toml` is the standing example: the built-in manager reads
   `[toolchain] channel = "…"` (and the legacy one-line file) as
   `depName: rust`, `depType: toolchain`, datasource `rust-version`; delete
   any bespoke marker above `channel`, nothing replaces it. `stable` / `beta` /
   `nightly` never produce a PR unless pinned:
   `{ matchManagers: ["rust-toolchain"], matchDepTypes: ["toolchain"], rangeStrategy: "pin" }`
   (`nightly` → `nightly-2025-10-11`). `registryUrls` and `hostRules` aimed at
   `rust-version` are inert (fixed registry).

2. **`customManagers:dockerfileVersions` / `customManagers:githubActionsVersions`
   / `customManagers:biomeVersions` …** — eleven presets
   (`azurePipelinesVersions`, `biomeVersions`, `bitbucketPipelinesVersions`,
   `dockerfileVersions`, `githubActionsVersions`, `gitlabPipelineVersions`,
   `helmChartYamlAppVersions`, `makefileVersions`, `mavenPropertyVersions`,
   `tfvarsVersions`, `tsconfigNodeVersions`) that read one marker comment on
   the line **above** a variable whose name ends in `_VERSION`:

   ```dockerfile
   # renovate: datasource=github-releases depName=nodejs/node versioning=node
   ARG NODE_VERSION=22.4.1
   ```

   ```yaml
   env:
     # renovate: datasource=crate depName=cargo-index
     CARGO_INDEX_VERSION: "0.2.7"
   ```

   Keys: `datasource=`, `depName=`, optional `packageName=`, `versioning=`,
   `extractVersion=` (`^v(?<version>.+)$` when tags are v-prefixed and the
   value is bare), `registryUrl=`. Any datasource works (`crate`, `npm`,
   `github-releases`). `githubActionsVersions` also covers `.github/actions/**`
   and `action.yml`, so a composite action's `env:` block is read. Never invent
   a marker (`# renovate-automation: rustc version` is recognised by nothing);
   a hard-coded tool version in CI without a marker is a review finding. Prefer
   this over a hand-written regex — the preset tracks Renovate's escaping rules
   and other tooling recognises the convention. None of the eleven covers shell
   scripts (see 4). `githubActionsVersions` does nothing for `uses:` lines; the
   SHA + `# vX.Y.Z` pair there is the `github-actions` manager's own.

3. **Move the version to a file a built-in manager owns.** One place per
   version. Nuances:

   - `mise.toml` is managed only for tools the **pinned** `mise` manager maps
     (`dist/modules/manager/mise/upgradeable-tooling.js`); `extract_deps` on
     the real file must return a row, otherwise add a regex manager for that
     line. Backend-prefixed entries (`"core:rust"`, `asdf:`, `vfox:`) yield a
     prefixed `packageName` for tools that declare none; `cargo:` tools come out
     as `depName: cargo:cargo-dylint`, `packageName: cargo-dylint`, datasource
     `crate`.
   - Rust **channels** belong in `rust-toolchain.toml`, not `mise.toml`. Older
     mise / asdf / proto tables mapped `rust` to `github-tags` /
     `rust-lang/rust`, so `stable` and `nightly-<date>` were
     `skipReason: invalid-value` and only a plain `1.89.1` moved; main now maps
     them to the `rust-version` datasource with `packageName: rust` (rules on
     `matchPackageNames: ["rust-lang/rust"]` stop matching there — use
     `matchDatasources: ["rust-version"]`). Check the pinned
     `upgradeable-tooling.js`.
   - The argument for `mise.toml` over `with:` inputs of `actions/setup-node`,
     `pnpm/action-setup`, `dtolnay/rust-toolchain`, `astral-sh/setup-uv`,
     `jdx/mise-action` is **single-sourcing, not visibility**: those inputs are
     extracted as `depType: uses-with` for actions listed in the manager's
     `community.ts` (an unmapped action yields nothing). Two copies mean two
     PRs, and a major-only `node-version: "24"` moves only on majors.
   - A version deliberately mirrored in two files must be extracted from both
     and grouped under one `groupName` (`reference/matchers-and-rules.md`,
     Grouping); otherwise the copy drifts and a CI equality check turns the
     next automerged bump red.

   Locations no manager reads at all: `run: npx --yes tool@1.2.3` in a workflow
   step, a download URL plus SHA-256 in a shell script, a `with:` input of an
   unmapped action, a same-repo `uses: ./.github/workflows/x.yml` (parsed as
   `kind: local`, by design — usable when a self-call must keep an OIDC
   `job_workflow_ref` on a ref), a `[workspace.metadata.dylint] libraries = […]`
   table in `Cargo.toml`. Make the tool a lockfile-pinned devDependency, or
   hoist the version into a `_VERSION` variable with a marker.

4. **A `customType: regex` manager** when none of the above fits. When the
   value lives in a table no manager owns, a regex manager on a datasource
   that already lists the versions (`datasourceTemplate: "github-releases"`,
   `depNameTemplate: "Org/repo"`) beats publishing to a private registry only
   to give the `crate` datasource something to read.

## Writing a regex manager

Start from the upstream marker regex and change only the assignment tail. The
example tracks a shell variable; the marker sits on the line above, exactly as
the presets expect:

```bash
# renovate: datasource=github-releases depName=jqlang/jq extractVersion=^jq-(?<version>.+)$
JQ_VERSION="1.7.1"
```

```json5
{
  customManagers: [
    {
      customType: "regex",
      description: "Track *_VERSION variables in scripts/ via # renovate: markers",
      managerFilePatterns: ["/^scripts/.+\\.sh$/"],
      matchStrings: [
        // customManagers:githubActionsVersions' regex with its YAML tail
        // `_VERSION\s*:\s*` replaced by the shell tail `_VERSION=`
        "# renovate: datasource=(?<datasource>[a-zA-Z0-9-._]+?) depName=(?<depName>[^\\s]+?)(?: (?:lookupName|packageName)=(?<packageName>[^\\s]+?))?(?: versioning=(?<versioning>[^\\s]+?))?(?: extractVersion=(?<extractVersion>[^\\s]+?))?(?: registryUrl=(?<registryUrl>[^\\s]+?))?\\s+[A-Za-z0-9_]+?_VERSION=[\"']?(?<currentValue>.+?)[\"']?\\s",
      ],
    },
  ],
}
```

Against the file above this extracts `{ datasource: github-releases, depName:
jqlang/jq, extractVersion: ^jq-(?<version>.+)$, currentValue: 1.7.1 }` (RE2
1.26.1, node `RegExp` and Python agree). The upstream regexes live in
`lib/config/presets/internal/custom-managers.preset.ts` (source) or
`node_modules/renovate/dist/config/presets/internal/custom-managers.preset.js`
(an install). Tails per file type: `\s(?:ENV|ARG)\s+[A-Za-z0-9_]+?_VERSION[ =]`
(Dockerfile), `_VERSION\s*:\s*` (YAML), `_VERSION\s*:*\??=\s*` (Makefile).

- `managerFilePatterns` (not `fileMatch`, which no longer exists): globs, or
  regexes written `/…/`. `run_config` migrates the old key and shows the new
  spelling.
- Required capture groups or templates: `depName` (or `packageName`),
  `currentValue`, `datasource`. Optional: `versioning`, `extractVersion`,
  `registryUrl`, `currentDigest`, `depType`, `indentation`. Fixed values can be
  templates instead of groups (`datasourceTemplate`, `depNameTemplate`,
  `versioningTemplate`). A **misspelled group name (`datasoure`) is not an
  error — the manager extracts nothing and logs at trace level.** Re-typing the
  marker regex by hand is where those typos come from.
- `matchStringsStrategy`: `any` (default, each string independent), `recursive`
  (each narrows the previous match), `combination` (all strings contribute
  groups to one dependency).
- Several keys in any order inside one match need lookahead, which RE2 refuses
  (`invalid perl operator: (?=`). The only workaround is one `matchStrings`
  entry per key ordering — a Cargo `package` / `git` / `tag` inline table takes
  six. Record the reason in `description` so nobody retries collapsing them;
  that maintenance cost is the argument for a built-in manager whenever one
  exists.
- Private registry, in this order: declare it where the built-in manager reads
  it (`maven.repositories` in `smithy-build.json`, `.npmrc` scope mapping); in
  a regex manager `registryUrlTemplate` (templates may use `{{repository}}` and
  other fields) or `registryUrl=` in the marker; a `packageRules` entry with
  `registryUrls` last — built-in managers have no `registryUrlTemplate`, and a
  registry set in a rule is invisible to anyone reading the manifest. Custom
  crate registries additionally need the **global** `allowCustomCrateRegistries:
  true` (`crate datasource: allowCustomCrateRegistries=true is required for
  registries other than crates.io, bailing out`); a repo config cannot set it,
  so the repo must run on a self-hosted runner that does.
- `autoReplaceStringTemplate` when the value to rewrite is not just the
  version (`{{{depName}}}:{{{newValue}}}{{#if newDigest}}@{{{newDigest}}}{{/if}}`).
  Auto-replace searches the file for `replaceString` (else `currentValue`); a
  miss logs at INFO and returns the file unchanged, so the run produces a
  **branch / PR with no diff**. An empty PR after a regex-manager change means
  the template or the replace key does not reproduce what is in the file.
- Rules targeting these deps use `matchManagers: ["custom.regex"]`.
- `customManagers` is `mergeable: true`: a repo's array is appended to the
  preset's, not swapped in.

## RE2, not JavaScript

All `matchStrings`, `extractVersion`, `versionCompatibility`, `/…/` matchers
and `managerFilePatterns` regexes compile with RE2. RE2 has **no lookahead,
no lookbehind, no backreferences**; when RE2 loads, an unsupported construct is
a config validation error (`invalid perl operator: (?=`), the config is
refused, nothing degrades. Named groups, `\d`, `\s`, `\xHH`, non-greedy
quantifiers are fine.

The exception is the environment, not the engine: when the `re2` native module
does not match the Node ABI (an npx cache built under another Node major; Node
26 against renovate's `engines ^24.11.0`), `renovate-config-validator` prints
`WARN: RE2 not usable, falling back to RegExp: regex validation may be
inaccurate` and passes lookaheads that fail at run time. Treat that WARN as
"this run proved nothing about regexes": fix the environment (Node 24, a fresh
install) or test with `re2` directly.

Two constructs diverge *silently* between RE2 and a JS regex tester:
`[[:alpha:]]` (POSIX class in RE2, a bracket expression in JS) and `\p{L}`
without the `u` flag. regex101's "Golang" flavour is the closest tester.

A capture group used as a split delimiter is load-bearing; making it
non-capturing drops the separators.

## Proving a custom manager

The debugger's `extract_deps` runs **built-in** managers only. Asked for
`manager: "regex"` or `"custom.regex"` it answers `regex is not supported in
the browser engine` (44.42.1), and it takes no config, so a `customType: regex`
manager cannot be proven with it. Use it to find which built-in managers claim
the file; prove the regex manager with one of these:

1. Run the `matchStrings` in node against the real file, with the `re2` module
   from the pinned install when it resolves (run from inside that install):
   `node -e "const RE2=require('re2');const re=new RE2(process.argv[1],'g');for(const m of require('fs').readFileSync(process.argv[2],'utf8').matchAll(re))console.log(m.groups)" '<matchString>' scripts/setup.sh`.
   Plain `RegExp` is faithful as long as the pattern has no lookarounds or
   backreferences (RE2 forbids them anyway). Check `managerFilePatterns`
   separately against the path.
2. Python: `py = re.sub(r'\(\?<(?!=)', r'(?P<', pat)`, then
   `re.finditer(py, text)`; assert the match count and print `groupdict()`.
3. The real thing: in a throwaway clone, overwrite the repo config with the
   manager under test and commit it (do not `rm` a tracked `renovate.json`;
   Renovate reads the git file list and aborts with `Repository has changed
   during renovation`), then
   `RENOVATE_PLATFORM=local RENOVATE_REQUIRE_CONFIG=optional GITHUB_COM_TOKEN=$(gh auth token) LOG_LEVEL=debug npx -y renovate@<pinned> --dry-run=extract 2>&1 | grep -iE 'depName|skipReason'`.
   `--dry-run=lookup` also prints the branch names it would create.
4. For a marker-based preset, load the regex from
   `node_modules/renovate/dist/config/presets/internal/custom-managers.preset.js`
   and run it as in 1 — no re-typing.

Then check `depName`, `currentValue`, `datasource` and the `versioning` you
expect. No row means the regex or a group name is wrong; do not adjust the
file to fit the regex. Feed the row to `simulate` on the config's `runId` to
see which rules apply (`matchManagers: ["custom.regex"]`, grouping,
automerge). If a lookup step is involved (`extractVersion`,
`versionCompatibility`), remember `extractVersion` rewrites the version
Renovate *writes back*; it cannot strip a suffix the registry needs. A regex
manager can extract correctly and still open an empty PR — see
`autoReplaceStringTemplate` above.
