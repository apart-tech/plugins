# Evaluating Dependencies — Design Spec

**Date:** 2026-03-30
**Status:** Approved
**Plugin:** apart-tech/plugins
**Skill:** evaluating-dependencies

---

## Purpose

A Claude Code skill that evaluates npm packages for build-vs-buy decisions. Helps both beginners (who lack judgment about dependency trade-offs) and seniors (who lack time to do the research) make informed decisions before adding dependencies.

## Trigger Conditions

- **User-invoked:** `/apart:evaluating-dependencies <package> [function]`
- **Agent-invoked:** Auto-triggers when Claude is about to run `npm install <new-package>` or add a new dependency to package.json

## Skill Identity

```yaml
name: evaluating-dependencies
description: "Evaluate npm packages for build-vs-buy decisions. Use when adding a new dependency, reviewing package.json changes, or deciding whether to use a library or implement functionality yourself."
argument-hint: <package-name> [specific-function]
allowed-tools: Bash, Read, Grep, Glob, WebFetch
```

## Data Gathering (6 Steps)

All data gathered via shell commands — no external tools or scripts required.

### Step 1 — Package Metadata (parallel)
- `npm info <pkg> --json` — version, description, dependencies, license, maintainers, last publish date
- `npm pack <pkg> --dry-run 2>&1` — files that would be installed, total size

### Step 2 — Dependency Tree
- `npm info <pkg> dependencies --json` — direct deps
- `npm view <pkg> dist.unpackedSize` — raw size

### Step 3 — Security
- `npm audit --json` (if installed) or check advisories from npm info output

### Step 4 — Source Analysis (key differentiator)
- Fetch package entry point from unpkg to identify exports
- If user specified a function, fetch its source to assess complexity (LOC, internal deps, edge cases)
- Count total exports vs. what user needs → usage ratio

### Step 5 — Maintenance Signals
- From npm info: last publish date, maintainer count, weekly downloads
- If GitHub repo linked: `gh api` for stars, open issues, last commit

### Step 6 — Alternatives
- `npm search <keywords>` for smaller/focused alternatives

## Decision Matrix

| Signal | Weight | Strong USE | Neutral | Strong BUILD |
|--------|--------|-----------|---------|-------------|
| Source complexity | 1 (highest) | >200 LOC or edge cases (tz, unicode, crypto) | 50-200 LOC | <50 LOC, straightforward |
| Usage ratio | 2 | >20% of exports | 5-20% | <5% of exports |
| Bundle size (tree-shaken) | 3 | <10KB | 10-50KB | >50KB for what you need |
| Transitive deps | 4 | 0 | 1-5 | >5 |
| Security | 5 | Clean, no CVEs | Minor/resolved | Active CVEs |
| Maintenance | 6 | Active (<6mo, 2+ maintainers) | Moderate | Stale (>12mo, solo maintainer) |
| License | 7 | MIT/Apache/ISC | BSD variants | GPL/AGPL in proprietary |
| Alternatives | 8 | No smaller option | Comparable | Focused package does exactly what you need |

## Verdicts

Three possible outcomes:

- **USE** — Install the package. The complexity justifies the dependency.
- **EXTRACT** — Copy the specific function you need (with attribution). Package too big but implementation non-trivial.
- **BUILD** — Write it yourself. Simple enough to avoid the dependency.

## Output Formats

### User-invoked (full report)

```
┌─ Dependency Evaluation: <package> ──────────────┐
│ You need: <function/purpose>                     │
│                                                  │
│ Source complexity:  47 LOC, 3 internal deps    → USE
│ Usage ratio:       1/200 exports (0.5%)       → BUILD
│ Bundle size:       5.2KB tree-shaken           → USE
│ Transitive deps:   0                           → USE
│ Security:          Clean                       → USE
│ Maintenance:       Active, 15 contributors     → USE
│ License:           MIT                         → USE
│ Alternatives:      None smaller                → USE
│                                                  │
│ ➤ VERDICT: USE                                   │
│   Tree-shakes well, zero deps, active. The edge  │
│   cases (locales, DST) make building risky.       │
│                                                  │
│ ➤ IF EXTRACTING: Function at src/formatDistance   │
│   47 LOC, depends on locale + constants only.     │
└──────────────────────────────────────────────────┘
```

### Agent-invoked (condensed, 1 line)

```
dep-eval: date-fns → USE (47 LOC, 0 deps, tree-shakes to 5.2KB, MIT, active)
dep-eval: left-pad → BUILD (1 LOC, trivial implementation)
dep-eval: moment → EXTRACT (need formatRelative only, 284KB non-tree-shakeable, consider date-fns instead)
```

## Scope

- **v1:** npm only
- **v1:** No config file — sensible defaults baked in
- **v1:** Pure skill prompt — no companion scripts, no MCP server

## Future (v2+)

- `.dep-eval.json` project config for team thresholds
- PyPI, Go modules, Rust crates support
- Companion script for parallel data gathering
- Integration with CI pipelines
