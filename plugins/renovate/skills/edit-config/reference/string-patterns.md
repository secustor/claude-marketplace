# String pattern matching: exact, glob or regex

Source: docs.renovatebot.com/string-pattern-matching (docs at Renovate
44.176.0) and the `packageRules` / `managerFilePatterns` option docs;
behaviour verified with renovate-config-debugger at 44.164.10. A line marked
"(unverified)" comes from neither. Every entry in a pattern list is one of
two things: a minimatch glob (an exact string is a glob with no
metacharacters) or a regex. Read `matchers-and-rules.md` for what each
`match*` key compares against; this sheet is only about the strings.

## Decision ladder

Stop at the first rung that expresses the intent.

1. Exact string: `["angular"]`. The names are known and finite.
2. Glob: `["@angular/**"]`, `["eslint**"]`. A namespace, prefix or path.
3. Negation: `["!@angular/abc"]`. "Everything except"; add a positive glob
   (`["@angular/**", "!@angular/abc"]`) only to carve a hole in a prefix.
4. Regex: `["/^angular/"]`. Only when a glob cannot say it: a case-sensitive
   match, a literal leading `!`, character classes or repetition
   (`/^v[0-9]+$/`), or a match that must cross a `/` where a single `*`
   stops.

## The three forms

| Form  | Detected when                        | Case                       | Example       | Use for                                       |
| ----- | ------------------------------------ | -------------------------- | ------------- | --------------------------------------------- |
| exact | glob with no `*`, `?`, `{}`          | insensitive, cannot change | `angular`     | a known name; `package.json` = root file only |
| glob  | any string not shaped like a regex   | insensitive, cannot change | `@angular/**` | a prefix, namespace or path at any depth      |
| regex | starts `/` or `!/`, ends `/` or `/i` | sensitive; `/i` to ignore  | `/^angular/`  | what a glob cannot express                    |

Docs examples, all case-insensitive: `abc123` matches `abc123` or `AbC123`;
`abc*` matches `abc`, `abc123`, `ABCabc`, but not `abc/def`; `abc**/*`
matches `abc/def` but not `abc`, `abcd` or `abc/def/ghi`; `abc**/**` matches
`abc/def` and `abc/def/ghi` but not `abc` or `abcd`; `abc{/,}**` matches
`abc`, `abcd`, `abc/def` and `abc/def/ghi`. Regex: `/^abc/` matches strings
starting with lower-case `abc`; `/^abc/i` any case; `!/^a/` strings not
starting with lower-case `a`.

## How a list combines

- A pattern is negative when it starts with `!` (`!/^a/`, `!b*`), positive
  otherwise. If the list has positives, at least one must match; if it has
  negatives, none may match. A negative hit wins over a positive hit.
  `["/^abc/", "!/^abcd/", "!/abce/"]` matches `abc` and `abcf`, not `foo`,
  `abcd`, `abce` or `abcdef`.
- Exact strings, globs and regexes mix freely in one list; the rule applies
  when either matches (`["neutrino", "@neutrino/**"]`).
- `*` alone means "match everything". `*` or `**` next to any other entry,
  positive or negative, is a validation error (`["*", "ABC"]`,
  `["*", "!ABC"]`; the `**` case verified with `run_config` on 44.x).
  A negation-only list already means "everything except": `["!ABC"]` is
  `["*", "!ABC"]` with the `*` omitted. Dropping `*` next to a positive
  pattern is not the same fix; it narrows the rule.
- Inside a rule, each `match*` key is one list and the keys are ANDed. "A or
  B" across keys is two rules, or `$matchRegexOrGlob(input, patterns)` in a
  `matchJsonata` expression (returns `false` when `input` is missing).

## Prefer a negation over a long positive list

"Everything except X" is `["!X"]`. An enumeration of what is allowed is
correct only on the day it is written: a package added next month silently
falls outside it, while the negation keeps covering it. The exception: an
enumeration is right when the rule itself defines membership (the exact
package set of a group, a lockstep pair); see `matchers-and-rules.md`,
Matchers.

Intent: a release-age floor on every third-party package.

Before (misses every package not listed):

```json
{
  "matchPackageNames": ["react", "react-dom", "lodash", "express"],
  "minimumReleaseAge": "3 days"
}
```

After (first-party namespace exempt, everything else covered):

```json
{
  "matchPackageNames": ["!@mycorp/**"],
  "minimumReleaseAge": "3 days"
}
```

Same for `matchUpdateTypes`: `["!major"]` beats the seven-item list that
forgets `replacement` and `lockFileMaintenance`.

## Prefer a glob over a regex

The docs note users find glob syntax easier to read than regex and prefer
it. The same intent, "every package starting with eslint":

```json
{ "matchPackageNames": ["eslint**"], "groupName": "eslint packages" }
```

```json
{ "matchPackageNames": ["/^eslint/i"], "groupName": "eslint packages" }
```

Pick the glob. The regex needs `/i` to match what the glob matches, because
globs are always case-insensitive and regexes are case-sensitive by default.
Keep the regex only when the glob cannot express the match (ladder rung 4).
Regexes are RE2: no lookahead, lookbehind or backreferences (`gotchas.md`
and `custom-managers.md` "RE2, not JavaScript" have the error texts).

## Gotchas

- `*` in a glob does not cross `/`; only `**` spans path separators. For
  `matchFileNames`, `package.json` matches the root file only and
  `**/package.json` matches at any depth.
- `abc**/*` and `abc**/**` need a `/` after `abc`; `abc{/,}**` is the form
  that takes `abc` itself, `abcd` and any depth below.
- Globs ignore case and you cannot change it; a case-sensitive match is a
  regex. A `*` segment matches a leading `.` (`*/workflows/ci.yml` matches
  `.github/workflows/ci.yml`; verified at 44.164.10).
- A leading `!` in a glob is always negation. To positively match a string
  that begins with `!`, write a regex: `["/^!abc$/"]` matches `"!abc"`.
- Backslashes double in JSON: `\n\s` becomes `\\n\\s`, `\.` becomes `\\.`
  (`"managerFilePatterns": ["/^ACTUALLY_A_DOCKERFILE\\.template$/"]`).
  Test on regex101 with the JavaScript flavour, then escape.
- Multi-valued inputs: the matcher is true if any value matches, and a
  negation passes if any one value avoids it. `"matchCategories":
  ["!docker"]` still matches a manager with categories `java` and `docker`.
  `matchFileNames` has the same shape (package file or any lock file; see
  `matchers-and-rules.md`).
- `managerFilePatterns` (formerly regex-only `fileMatch`, migrated
  automatically) takes glob or regex, is added to the manager defaults
  (which cannot be removed), and must sit inside a manager object.
- Not every option uses this syntax. Options whose docs link to the page
  (44.176.0): `matchPackageNames`, `matchFileNames`, `matchSourceUrls`,
  `matchUpdateTypes`, `matchDepTypes`, `matchNewValue`, `matchJsonata` (via
  `$matchRegexOrGlob`), `bumpVersions.filePatterns`, `gitIgnoredAuthors`,
  `cloneSubmodulesFilter` (unverified: `get_option_docs` at 44.164.10 shows
  no `patternMatch` flag for it). `matchDepNames`, `managerFilePatterns`,
  `matchRepositories` and `matchBaseBranches` are pattern options by
  `get_option_docs` (`patternMatch: true`) although their doc sections do
  not link. `changelogUrl` is a templated string, not a pattern list; its
  section links only because its example uses `matchPackageNames`. Older
  options may be inconsistent; check `get_option_docs <name>` for
  `patternMatch` before assuming.
- Interactive testers: Digital Ocean's glob tool for globs, regex101
  (JavaScript flavour) for regexes.

## How to prove it

`run_config` the edited text, then `simulate` one dependency that must match
and one that must not, with `keys` set to the option the rule changes. A
`*` next to any other entry comes back `accepted: false`; fix the list, do
not argue with the validator. For a glob -> regex or regex -> glob rewrite,
two preconditions: the two configs must differ only in the pattern list
(same `groupName` and every other key, or `compare_simulations` reports
`differs` for the unrelated key), and `compare_simulations` must report
`identical` for more than one dependency: at least one name with different
casing (`@Types/Node`) and one that crosses a `/` boundary. One `identical`
on one dependency proves nothing; `differs` with unrelated keys in
`configDelta` means the test, not the pattern, is wrong; `differs` on the
pattern itself means the forms differ (usually case or a `/` boundary) and
the rewrite is not a pure readability change.
