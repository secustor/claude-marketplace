# Custom managers

Reach for a custom manager only when no built-in manager and no
`customManagers:*` preset covers the file. Order of preference:

1. **A built-in manager.** Renovate has ~129; `extract_deps` with the file name
   and contents tells you which managers claim the file and what they extract.
   Several claim the same name (`pyproject.toml`: pep621, pixi, poetry); ~11
   have no file pattern and must be enabled explicitly (`kubernetes`, `argocd`,
   `tekton`, …) via `managerFilePatterns` in their manager block.
2. **`customManagers:dockerfileVersions` / `customManagers:githubActionsVersions`
   / `customManagers:biomeVersions` …** — presets that read a marker comment:

   ```dockerfile
   # renovate: datasource=github-releases depName=nodejs/node versioning=node
   ARG NODE_VERSION=22.4.1
   ```

   ```yaml
   env:
     # renovate: datasource=npm depName=pnpm
     PNPM_VERSION: 10.4.1
   ```

   The variable name must end in `_VERSION`. Optional keys: `packageName=`,
   `versioning=`, `extractVersion=`, `registryUrl=`. Prefer this over a
   hand-written regex — the preset tracks Renovate's escaping rules and other
   tooling recognises the convention.
3. **Move the version to a file a built-in manager owns.** A `mise.toml` entry
   is managed by the `mise` manager with no extra config; a Dockerfile `ARG`
   duplicating it is a second copy that drifts. One place per version.
4. **A `customType: regex` manager** when none of the above fits.

## Writing a regex manager

```json5
{
  customManagers: [
    {
      customType: "regex",
      description: "Track the jq release used by scripts/setup.sh",
      managerFilePatterns: ["/^scripts/setup\\.sh$/"],
      matchStrings: [
        "JQ_VERSION=\"(?<currentValue>[^\"]+)\"\\s*# renovate: datasource=(?<datasource>[^ ]+) depName=(?<depName>[^\\s]+)"
      ],
      // fixed values can also be given as templates instead of capture groups:
      // datasourceTemplate: "github-releases", depNameTemplate: "jqlang/jq",
      versioningTemplate: "semver",
      extractVersionTemplate: "^jq-(?<version>.*)$"
    }
  ]
}
```

- `managerFilePatterns` (not `fileMatch`, which no longer exists): globs, or
  regexes written `/…/`. `run_config` migrates the old key and shows the new
  spelling.
- Required capture groups or templates: `depName` (or `packageName`),
  `currentValue`, `datasource`. Optional: `versioning`, `extractVersion`,
  `registryUrl`, `currentDigest`, `depType`, `indentation`. A **misspelled
  group name (`datasoure`) is not an error — the manager extracts nothing and
  logs at trace level.** Prove every manager with `extract_deps`.
- `matchStringsStrategy`: `any` (default, each string independent), `recursive`
  (each narrows the previous match), `combination` (all strings contribute
  groups to one dependency).
- `autoReplaceStringTemplate` when the value to rewrite is not just the
  version (`{{{depName}}}:{{{newValue}}}{{#if newDigest}}@{{{newDigest}}}{{/if}}`).
- Rules targeting these deps use `matchManagers: ["custom.regex"]`.

## RE2, not JavaScript

All `matchStrings`, `extractVersion`, `versionCompatibility`, `/…/` matchers
and `managerFilePatterns` regexes compile with RE2. RE2 has **no lookahead,
no lookbehind, no backreferences**; an unsupported construct is a config
validation error, the config is refused, nothing degrades gracefully. Named
groups, `\d`, `\s`, `\xHH`, non-greedy quantifiers are fine.

Two constructs diverge *silently* between RE2 and a JS regex tester:
`[[:alpha:]]` (POSIX class in RE2, a bracket expression in JS) and `\p{L}`
without the `u` flag. regex101's "Golang" flavour is the closest tester;
`extract_deps` is the real one.

A capture group used as a split delimiter is load-bearing; making it
non-capturing drops the separators.

## Proving a custom manager

1. `extract_deps` with the real file: check `depName`, `currentValue`,
   `datasource` and the `versioning` you expect. No row means the regex or a
   group name is wrong; do not adjust the file to fit the regex.
2. Feed the extracted row to `simulate` on the config's `runId` to see which
   rules would apply (`matchManagers: ["custom.regex"]`, grouping, automerge).
3. If a lookup step is involved (`extractVersion`, `versionCompatibility`),
   remember `extractVersion` rewrites the version Renovate *writes back*; it
   cannot strip a suffix the registry needs.
