# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## What this repository is

`obsidianmd/obsidian-translations` holds the **localization string files for the Obsidian app**.
It is a *data* repository, not an application: there is no source code, no build step, and nothing
to run. Almost every change is an edit to the *values* inside one language's JSON file.

The app consumes these files directly; contributors submit changes as pull requests against `master`.

## Layout

```
en.json                 Source of truth. All other files are derived from it.
en-GB.json              British English variant (89 value differences from en.json)
<code>.json             One file per language, ISO 639-1 code, optionally with a
                        region suffix (pt-BR, zh-TW, en-GB, nan-TW) — 71 files total
<code>.termbase.md      Optional per-language glossary of agreed term translations
fa.guidelines.md        Persian style guide (only language with a full guidelines doc)
README.md               Contributor instructions + table of language codes/status
package.json            Only holds the lint script and its two devDependencies
.eslintrc.json          `extends: plugin:json/recommended`
.github/workflows/main.yml   CI: JSON lint on push/PR to master
```

Termbases currently exist for: `af cs de hu ja ms pt-BR ro ru sa th uk`.

## Structure of a language file

`en.json` has **13 top-level namespaces** and ~2,399 leaf keys:

`setting`, `editor`, `interface`, `commands`, `dialogue`, `menu-items`, `plugins`,
`formulas`, `pdf`, `properties`, `table`, `callout`, `nouns`

Keys are nested and namespaced by where the string appears in the app, e.g.
`plugins.canvas.*`, `plugins.sync.*` (217 keys), `plugins.publish.*` (187 keys),
`setting.appearance.*` (85 keys).

```json
{
	"setting": {
		"editor": {
			"option-spellcheck": "Spellcheck",
			"option-spellcheck-description": "Turn on the spellchecker."
		}
	}
}
```

### Formatting conventions

- **Tabs** for indentation (not spaces). `ga.json` is the one file that uses spaces throughout;
  do not copy that style into other files.
- Key order in every translated file **mirrors `en.json` exactly**. Never reorder or re-sort keys.
- UTF-8, no BOM. `en.json` has no trailing newline; most other files do — leave whatever a file
  already has rather than "fixing" it (it produces noise in the diff).
- English source strings use curly quotes (`“ ”`, 93 occurrences) and three-dot `...` for ellipsis
  (106 occurrences, never `…`). Translations should follow the target language's own punctuation
  conventions — several termbases record explicit decisions about quote marks.

### Placeholders

Interpolation uses `{{name}}` and must be copied verbatim into translations — never translate,
rename, space-adjust, or drop them:

```json
"label-welcome": "Welcome, {{name}}!"
```

Common ones: `{{count}}` (80×), `{{name}}` (21×), `{{time}}`, `{{version}}`, `{{path}}`,
`{{filename}}`. Note that some keys — mostly under `formulas` — use inner spaces
(`{{ key }}`, `{{ formula }}`, `{{ function }}`); preserve the spacing as written.

### Plurals

Countable strings come as a pair, base key plus a `_plural` sibling (36 pairs in `en.json`):

```json
"file-with-count": "{{count}} file",
"file-with-count_plural": "{{count}} files"
```

Languages with more complex plural rules than English still only get these two forms in this
repository — pick wordings that read acceptably for both, and record the decision in the
language's termbase if it is contentious.

## Development workflow

There is no dev server or test suite. The only automated check is a JSON syntax lint.

```bash
npm install          # eslint 7.20.0 + eslint-plugin-json 2.1.2
npm run lint         # eslint . --ext .json
```

CI (`.github/workflows/main.yml`) runs `npm install` then `yarn run lint` on every push and PR
to `master`. The lint only validates **JSON syntax** (trailing commas, duplicate keys, etc.) — it
does *not* check that keys match `en.json` or that translations exist.

**There is no `.gitignore`.** After running `npm install`, `node_modules/` and `package-lock.json`
show up as untracked. Delete them before committing; never add them to a commit.

### Verifying a translated file against the template

Since lint won't catch structural drift, use a script when doing anything beyond a one-string edit:

```bash
python3 - <<'EOF'
import json, sys
def flat(o, p=''):
    r = {}
    for k, v in o.items():
        if isinstance(v, dict):
            r.update(flat(v, p + k + '.'))
        else:
            r[p + k] = v
    return r
en = flat(json.load(open('en.json')))
tr = flat(json.load(open(sys.argv[1] if len(sys.argv) > 1 else 'fr.json')))
print('missing keys :', len(set(en) - set(tr)))
print('stale keys   :', len(set(tr) - set(en)))
print('untranslated :', sum(1 for k in set(en) & set(tr) if en[k] == tr[k]))
EOF
```

A healthy file reports `missing 0`, `stale 0`; "untranslated" counts values still identical to
English, which is the natural to-do list for that language.

## Making changes

1. Work in **one language file per change** — this is how every recent commit in the history looks
   (`bn.json`, then `ar.json`, then `he.json`, …). Mixing languages makes review and merge
   conflicts much worse.
2. Edit **values only**. Adding, removing, or renaming keys is the maintainers' job and happens in
   bulk "Update strings for 1.x.x" commits that touch every language at once.
3. Consult and update the language's `<code>.termbase.md` when you settle a term. Termbases are
   kept in **alphabetical order** and hold the agreed rendering of recurring words (`vault`,
   `plugin`, `canvas`, `property`, `ribbon`, …).
4. Leave English in place rather than guessing. An untranslated string is a visible to-do; a wrong
   or machine-translated one silently degrades the app. Do not bulk machine-translate.
5. Never reformat a whole file (indentation, quote style, key sorting). The diff becomes unreviewable
   and guarantees conflicts with other contributors' in-flight PRs.
6. Adding a language: copy `en.json` to `<language-code>.json` (ISO 639-1), translate, and add a row to the
   README table (code, English name, endonym, status ✅/🚧). The PR description should state the
   language's **endonym**, since that is what the app displays.
7. Commit messages in this repo are plain and descriptive: `Update pl.json (#1209)`,
   `Russian localization fixes (Issue #1205)`, `it: update translation for 1.9.7`.

## Known state of the tree (verified 2026-08-22, at `70e60ed`)

These are pre-existing conditions, not things to fix incidentally — but know they exist so you
don't attribute them to your own change:

- **`he.json` is not valid JSON.** A trailing comma at line 2404 (introduced by #1217) means
  `npm run lint` fails on `master`, and the lines around it use spaces instead of tabs.
  Any lint run you do will report this one error.
- **Stale `formulas.funcs.*` keys** (63 of them) linger in 23 files that were not touched when
  those keys were dropped from `en.json`: `af az bg bn el eo eu fi gl hi ka kn lt ml nn oc se si
  sl sr ta te ur`.
- **Incomplete files** carry real missing keys — worst first: `hr` (713), `nan-TW`/`sa` (590),
  `dv` (479), `or`/`sw`/`tl` (477), `la` (427), `ky` (189).
- **`sk.json`** has 5 keys under older names (`option-own-password` vs `option-encryption-custom`,
  `button-view-snapshots` vs `button-view-history`).
- **The README table is out of date**: `az fi ga hr ka kh kn ky la lt nan-TW nn or se sl sw uz` have
  files but no table row; `fi-fi` and `sv` have rows but no matching file (the Finnish file is
  `fi.json`).

## Testing a translation in the app

From Obsidian's developer console:

```js
selectLanguageFileLocation()          // pick a JSON file; the app reboots using it
localStorage.removeItem('language')   // revert to the bundled language pack
```

## Notes

- The Simplified Chinese translation (`zh.json`) is maintained downstream by Obsidian.zh at
  https://github.com/obsidianzh/obsidian-translations — coordinate there rather than here.
- `package.json` is named `obsidian-community-releases`, a copy-paste leftover from another repo.
  It has no bearing on anything.
