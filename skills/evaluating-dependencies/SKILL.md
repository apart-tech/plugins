---
name: evaluating-dependencies
description: "Evaluate npm or Python packages for build-vs-buy decisions. Use when adding a new dependency, reviewing package.json or requirements.txt changes, pip install, or deciding whether to use a library or implement functionality yourself."
argument-hint: "[package-name] [specific-function]"
allowed-tools: Bash(npm:*), Bash(gh:*), Bash(npx:*), Bash(pip:*), Bash(pip-audit:*), Bash(python:*), Bash(curl:*), Read, Grep, Glob, WebFetch
---

# Evaluating Dependencies

Evaluate whether to **USE**, **EXTRACT**, or **BUILD** before adding any dependency.

## The Rule

**No new dependency without an evaluation.** If you are about to install a new package or add one to package.json / requirements.txt / pyproject.toml, run this evaluation first.

## Phase 0: Detect Ecosystem

Determine the ecosystem from context:

- **npm** — user mentions npm, package.json, node_modules, or `npm install`
- **Python** — user mentions pip, PyPI, requirements.txt, pyproject.toml, or `pip install`
- **Ambiguous** — check if the package exists on npm (`npm info <pkg>`) and/or PyPI (`curl -s https://pypi.org/pypi/<pkg>/json`). If both, ask the user which ecosystem they mean.

Then follow the ecosystem-specific gathering steps below.

## When Invoked by User

User runs `/evaluating-dependencies <package> [function]`. The package name is available as `$ARGUMENTS[0]` and the optional function as `$ARGUMENTS[1]`. Produce the **full report** format.

## When Auto-Triggered

You are about to add a dependency. Run the evaluation silently and produce the **condensed 1-line** format. If the verdict is BUILD or EXTRACT, pause and present the finding before proceeding.

## Phase 1: Gather Data

Run these commands. Parallelize where possible.

### npm Ecosystem

#### 1A — Package metadata (parallel)
```bash
npm info <pkg> --json
npm pack <pkg> --dry-run 2>&1
```

#### 1B — Dependency tree
```bash
npm info <pkg> dependencies --json
npm view <pkg> dist.unpackedSize
```

#### 1C — Security
```bash
# If package is in node_modules:
npm audit --json
# Otherwise, check advisories from npm info output
```

#### 1D — Source analysis
Fetch the package entry point to identify its public API:
```bash
# Get the main/exports field from npm info output
# Then fetch source to count exports and assess complexity
```

If the user specified a specific function, find that function's source code:
- Check unpkg.com or the GitHub repo linked in npm info
- Measure: lines of code, internal dependencies, edge case handling (unicode, timezones, locales, browser quirks, crypto)

Count total public exports. Calculate usage ratio: functions needed / total exports.

#### 1E — Maintenance signals
Extract from the npm info JSON output:
- `time.modified` — last publish date
- `maintainers` — count
- `dist-tags` — release activity

If a GitHub repository is linked:
```bash
gh api repos/<owner>/<repo> --jq '{stars: .stargazers_count, open_issues: .open_issues_count, pushed_at: .pushed_at}'
```

#### 1F — Alternatives
```bash
npm search <keywords-describing-the-functionality> --json | head -20
```

### Python Ecosystem

#### 1A — Package metadata
```bash
curl -s https://pypi.org/pypi/<pkg>/json
```

Extract from the JSON response: `info.version`, `info.license`, `info.summary`, `info.author`, `info.requires_dist` (dependencies), `info.project_urls` (GitHub link), `urls` (release files and sizes).

#### 1B — Dependency tree
From the PyPI JSON `info.requires_dist` field — list direct dependencies. For transitive depth:
```bash
pip install --dry-run <pkg> 2>&1
```
Count packages that would be installed.

#### 1C — Security
```bash
pip-audit --desc <pkg> 2>&1
# Or check PyPI advisories from the JSON response
# Or: curl -s https://pypi.org/pypi/<pkg>/json | look for yanked releases
```

Also check for `postinstall` equivalent: `.pth` files, `setup.py` with arbitrary code execution, or suspicious `__init__.py` imports (relevant to supply chain attacks like litellm).

#### 1D — Source analysis
From PyPI JSON `info.project_urls`, find the GitHub repo. Then:
- Fetch the module's `__init__.py` or main source file to count public exports (`__all__` or top-level functions/classes)
- If user specified a function, find its source on GitHub and assess complexity

Count total public API surface. Calculate usage ratio.

#### 1E — Maintenance signals
From the PyPI JSON response:
- Latest release date from `urls[0].upload_time`
- Release frequency from `releases` object (count recent versions)
- Maintainer count from `info.maintainer` and `info.author`

If GitHub repo is linked:
```bash
gh api repos/<owner>/<repo> --jq '{stars: .stargazers_count, open_issues: .open_issues_count, pushed_at: .pushed_at}'
```

#### 1F — Alternatives
```bash
# PyPI has no search API — use GitHub or web search
```
Search GitHub or the web for `<functionality> python library` to find focused alternatives.

Focus on packages that are smaller or more focused than the one being evaluated.

## Phase 2: Score Each Signal

Evaluate each signal as leaning toward USE, EXTRACT, or BUILD:

| Signal | → USE | → Neutral | → BUILD |
|--------|-------|-----------|---------|
| **Source complexity** | >200 LOC or handles edge cases (timezones, unicode, crypto, browser quirks) | 50-200 LOC, moderate complexity | <50 LOC, straightforward logic |
| **Usage ratio** | Need >20% of exports | 5-20% | <5% — paying for a lot you won't use |
| **Bundle size** | <10KB tree-shaken | 10-50KB | >50KB for what you need |
| **Transitive deps** | 0 dependencies | 1-5 | >5 — hidden risk |
| **Security** | Clean audit, no CVEs | Minor or resolved advisories | Active CVEs or unpatched vulnerabilities |
| **Maintenance** | Published <6 months ago, 2+ maintainers | Published 6-12 months ago | >12 months stale, solo maintainer |
| **License** | MIT, Apache-2.0, ISC | BSD variants | GPL/AGPL in a proprietary project (blocker) |
| **Alternatives** | No smaller package exists | Comparable alternatives available | A focused package does exactly what you need |

**Weight priority** (highest first): Source complexity > Usage ratio > Bundle size > Transitive deps > Security > Maintenance > License > Alternatives.

## Phase 3: Determine Verdict

Count the signal leans. Apply weights. Determine one of three verdicts:

- **USE** — Install it. The complexity, edge cases, or ongoing maintenance justify the dependency. Majority of weighted signals lean USE or neutral.
- **EXTRACT** — Copy the specific function or code you need, with attribution. The package is too large for your needs but the implementation handles non-trivial edge cases. Usage ratio leans BUILD but source complexity leans USE.
- **BUILD** — Write it yourself. The function is simple, well-understood, and doesn't handle tricky edge cases. Majority of weighted signals lean BUILD.

**License is a hard gate:** GPL/AGPL in a proprietary project = BUILD or EXTRACT regardless of other signals.

**Active CVEs are a hard gate:** Unpatched vulnerabilities = BUILD or find an alternative regardless of other signals.

## Phase 4: Format Output

### Full Report (user-invoked)

Use a markdown table with no box-drawing characters. Keep each signal value to ONE line — no wrapping.

```markdown
## dep-eval: <package>

**You need:** `<function>` — <brief purpose>

| Signal | Value | Lean |
|--------|-------|------|
| Source complexity | <n> LOC, <brief note> | BUILD / USE |
| Usage ratio | <n>/<total> exports (<n>%) | BUILD / USE |
| Bundle size | <size>, <tree-shake status> | BUILD / USE |
| Transitive deps | <count> | BUILD / USE |
| Security | <status> | BUILD / USE |
| Maintenance | <last publish>, <n> maintainers | BUILD / USE |
| License | <license> | BUILD / USE |
| Alternatives | <name or "none"> | BUILD / USE |

### VERDICT: <USE / EXTRACT / BUILD>

<2-3 sentence justification>

### If extracting

<file list, LOC, internal deps>

### If building

<code snippet or description of what to implement>
```

**Rules for the table:**
- Every signal value MUST fit on a single line. Abbreviate if needed.
- The "Lean" column is ONLY "USE" or "BUILD" — never a sentence.
- Source complexity details (edge cases, what it handles) go in the justification, not the table.
- Include "If extracting" when verdict is USE or EXTRACT.
- Include "If building" when verdict is BUILD or EXTRACT.

### Condensed (agent-invoked)

One line, no markdown:

```
dep-eval: <package> → <VERDICT> (<key reasons, comma-separated>)
```

Examples:
```
dep-eval: date-fns → USE (47 LOC, 0 deps, tree-shakes to 5.2KB, MIT, active)
dep-eval: left-pad → BUILD (1 LOC, trivial implementation)
dep-eval: moment → EXTRACT (need formatRelative only, 284KB non-tree-shakeable, consider date-fns instead)
```

## Red Flags — STOP

If you notice any of these, flag them prominently in the report:

**Both ecosystems:**
- Package has 0 weekly downloads
- Solo maintainer with no activity in 12+ months
- Package name is a typosquat of a popular package
- License field is missing or "UNLICENSED"/"UNKNOWN"

**npm-specific:**
- `install` or `postinstall` scripts that execute arbitrary code
- Dependencies that pull in native/binary modules unnecessarily

**Python-specific:**
- `.pth` files in the package (code execution on import — litellm attack vector)
- `setup.py` with network calls or base64-encoded payloads
- Yanked releases on PyPI (check `releases` object for `yanked: true`)
- Package imports that trigger side effects in `__init__.py`
- Mismatched package name vs. import name (typosquatting signal)

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Everyone uses this package" | Popularity does not mean it's right for YOUR use case |
| "It's only one extra dependency" | Check the transitive tree — it might be 50 |
| "I don't have time to evaluate" | You don't have time to debug a bad dependency later |
| "Tree-shaking will handle it" | Only if the package uses ESM exports. Check. |
| "The package is well-maintained" | Maintained does not mean small or appropriate |
| "Building it would be reinventing the wheel" | 10 lines of code is not a wheel |
