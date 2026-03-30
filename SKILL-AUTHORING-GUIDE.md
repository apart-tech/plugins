# Skill & Plugin Authoring Guide

Comprehensive best practices compiled from official Anthropic docs, community research, top-performing plugins (superpowers, frontend-design), the agent skills open standard, and tooling like agnix and skill-optimizer.

---

## Table of Contents

1. [Plugin Structure](#1-plugin-structure)
2. [SKILL.md Frontmatter Reference](#2-skillmd-frontmatter-reference)
3. [The #1 Priority: Writing Descriptions (CSO)](#3-the-1-priority-writing-descriptions-cso)
4. [Naming Conventions](#4-naming-conventions)
5. [Prompt Patterns That Work](#5-prompt-patterns-that-work)
6. [Instruction Structure Templates](#6-instruction-structure-templates)
7. [Token Efficiency & Progressive Disclosure](#7-token-efficiency--progressive-disclosure)
8. [Anti-Patterns to Avoid](#8-anti-patterns-to-avoid)
9. [Testing Skills (TDD for Documentation)](#9-testing-skills-tdd-for-documentation)
10. [Publishing & Marketplace](#10-publishing--marketplace)
11. [Quick Reference Checklists](#11-quick-reference-checklists)

---

## 1. Plugin Structure

### Minimum Viable Plugin

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # REQUIRED - metadata manifest
├── skills/
│   └── my-skill/
│       └── SKILL.md         # REQUIRED per skill
└── README.md                # Optional
```

### Full Plugin Layout

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # Only plugin.json goes here
├── skills/                  # Skill definitions
│   └── skill-name/
│       ├── SKILL.md         # Main instructions (required)
│       ├── references/      # Docs loaded on demand
│       ├── scripts/         # Executable code
│       └── assets/          # Templates, images, data
├── commands/                # Legacy slash commands
├── agents/                  # Custom agent definitions
├── hooks/                   # Lifecycle hooks
│   └── hooks.json
├── output-styles/           # Custom output styles
├── .mcp.json                # MCP server configs
├── .lsp.json                # LSP server configs
├── settings.json            # Default settings (only "agent" key)
├── LICENSE
└── CHANGELOG.md
```

**Critical**: Components must be at plugin root, NOT inside `.claude-plugin/`.

### plugin.json Manifest

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "Brief plugin description",
  "author": {
    "name": "Author Name",
    "email": "author@example.com",
    "url": "https://github.com/author"
  },
  "homepage": "https://github.com/author/plugin",
  "repository": "https://github.com/author/plugin",
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"]
}
```

**Supported component path fields**: `commands`, `agents`, `skills`, `hooks`, `mcpServers`, `outputStyles`, `lspServers`, `userConfig`, `channels`.

**Environment variables available**: `${CLAUDE_PLUGIN_ROOT}` (install dir), `${CLAUDE_PLUGIN_DATA}` (persistent data dir).

---

## 2. SKILL.md Frontmatter Reference

### All Supported Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Recommended | Max 64 chars. Lowercase letters, numbers, hyphens. Must match directory name. |
| `description` | Recommended | Max 1024 chars. What skill does + when to use it. **THE MOST IMPORTANT FIELD.** |
| `argument-hint` | No | Shown during autocomplete. E.g. `[issue-number]` |
| `disable-model-invocation` | No | `true` = only user can invoke (for side-effect commands like /deploy) |
| `user-invocable` | No | `false` = hidden from `/` menu (for background knowledge only) |
| `allowed-tools` | No | Tools Claude can use without permission when skill is active |
| `model` | No | Model override when skill is active |
| `effort` | No | `low`, `medium`, `high`, `max` (Opus 4.6 only) |
| `context` | No | `fork` to run in subagent context |
| `agent` | No | Subagent type when `context: fork`. Options: `Explore`, `Plan`, `general-purpose`, or custom |
| `hooks` | No | Hooks scoped to this skill's lifecycle |
| `paths` | No | Glob patterns limiting when skill activates |
| `shell` | No | `bash` (default) or `powershell` |
| `license` | No | License reference |
| `compatibility` | No | Max 500 chars. Environment requirements |
| `metadata` | No | Arbitrary key-value map |

### String Substitutions in Skill Content

| Variable | Description |
|----------|-------------|
| `$ARGUMENTS` | All arguments passed when invoking |
| `$ARGUMENTS[N]` / `$N` | Specific argument by 0-based index |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_SKILL_DIR}` | Directory containing the SKILL.md |

### Dynamic Context Injection

The `` !`command` `` syntax runs shell commands as preprocessing:

```markdown
## Current PR context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
```

Claude sees the output, not the command. Include "ultrathink" anywhere in content to enable extended thinking.

---

## 3. The #1 Priority: Writing Descriptions (CSO)

**CSO = Claude Search Optimization.** This is THE make-or-break factor.

### The Problem

Vercel's research found that **in 56% of evaluation cases, skills were NEVER triggered**. The description is the only thing Claude uses to decide whether to invoke a skill.

### Rules

1. **Always write in third person.** The description is injected into the system prompt.
   - Good: "Evaluates npm packages for build-vs-buy decisions"
   - Bad: "I help you evaluate npm packages"

2. **Start with "Use when..." trigger conditions.**
   - Good: `Use when adding a new npm dependency, evaluating whether to build or buy functionality, or reviewing package.json changes`
   - Bad: `Helps with npm packages`

3. **Front-load keywords in first 250 characters.** Descriptions >250 chars are truncated in listings.

4. **Be specific. Include error messages, symptoms, synonyms.**
   - Include what users would say: "should I use this library", "too many dependencies", "bundle size"

5. **Keep trigger conditions to 2 or fewer.** LLMs struggle with multi-constraint prompts (IFEval, Zhou et al. 2023).

6. **NEVER summarize the workflow in the description.** Testing showed Claude follows the description shortcut instead of reading the full skill body.

7. **Quote descriptions containing colons.** Unquoted `: ` in YAML = parse failure = skill is completely invisible.

### Examples from Top Skills

```yaml
# Frontend-design (~414K installs)
description: Create distinctive, production-grade frontend interfaces with high design quality. Use this skill when the user asks to build web components, pages, or applications.

# Systematic-debugging (~295K installs, superpowers)
description: Use when encountering any bug, test failure, or unexpected behavior, before proposing fixes

# Verification-before-completion (superpowers)
description: Use when about to claim work is complete, fixed, or passing, before committing or creating PRs
```

---

## 4. Naming Conventions

### Skills

| Rule | Example |
|------|---------|
| Gerund form preferred | `evaluating-dependencies`, `processing-pdfs` |
| Active voice, verb-first | `creating-skills` not `skill-creation` |
| Name by what you DO | `condition-based-waiting` not `async-test-helpers` |
| Kebab-case only | lowercase + hyphens, no underscores |
| Must match directory name | `skills/eval-deps/SKILL.md` → `name: eval-deps` |
| Max 64 characters | |
| No reserved words | Cannot contain "anthropic" or "claude" |

**Avoid**: `helper`, `utils`, `tools`, generic labels, overly broad terms.

### Plugins

- Kebab-case, no spaces
- Used as namespace: `/plugin-name:skill-name`
- Keep short — users type this

---

## 5. Prompt Patterns That Work

### Pattern 1: Iron Law (for discipline-enforcing skills)

State an absolute rule, then close every loophole:

```markdown
## The Iron Law

NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST

**No exceptions:**
- Don't keep it as "reference"
- Don't "adapt" it while writing tests
- Don't look at it
- Delete means delete
```

### Pattern 2: Rationalization Table

Pre-emptively counter every excuse:

```markdown
| Excuse | Reality |
|--------|---------|
| "Too simple to test" | Simple code breaks. Test takes 30 seconds. |
| "I'll test after" | Tests passing immediately prove nothing. |
| "Being pragmatic" | TDD IS pragmatic. Random fixes are slower. |
```

### Pattern 3: Red Flags List

```markdown
## Red Flags - STOP
- About to install a package without checking alternatives
- Package has 0 downloads and 1 contributor
- Adding a 500KB dep for one function

**If any of these: pause, run the full evaluation.**
```

### Pattern 4: Phase-Based Workflow with Gates

```markdown
## Phase 1: Gather Data
BEFORE any recommendation: [steps]

## Phase 2: Analyze (requires Phase 1 output)
[steps]

## Phase 3: Recommend
ONLY after Phase 2 is complete: [verdict]
```

### Pattern 5: Evidence Before Claims

```markdown
BEFORE claiming any status:
1. IDENTIFY what command proves this claim
2. RUN the full command
3. READ full output, check exit code
4. VERIFY output confirms the claim
5. ONLY THEN make the claim
```

### Pattern 6: Aesthetic Archetypes (for creative skills)

Force a specific direction rather than averaging to generic:

```markdown
## Select ONE archetype, commit fully:
1. Minimalist Technical
2. Bold Experimental
3. Corporate Polished
...
```

### Pattern 7: XML Tags for Emphasis

```markdown
<EXTREMELY-IMPORTANT>
If you think there is even a 1% chance this applies, you MUST invoke it.
</EXTREMELY-IMPORTANT>

<HARD-GATE>
Do NOT proceed past this point without completing the checklist.
</HARD-GATE>
```

---

## 6. Instruction Structure Templates

### Template A: Process/Discipline Skill (debugging, TDD)

```markdown
---
name: skill-name
description: Use when [trigger conditions]
---

# Skill Name

## Overview
[1-2 sentences, core principle in bold]

## The Iron Law
[Absolute rule in code block]

## When to Use
- [Bullet list of triggers]
- Especially when: [edge cases]

## The Process
### Phase 1: [Name]
[Steps with code examples]

### Phase 2: [Name]
[Steps, gated on Phase 1]

## Red Flags - STOP
[Self-check list]

## Common Rationalizations
| Excuse | Reality |
|--------|---------|

## Quick Reference
[Summary table for at-a-glance use]
```

### Template B: Gate/Checklist Skill (brainstorming, verification)

```markdown
---
name: skill-name
description: You MUST use this before [action]
---

# Skill Name

<HARD-GATE>
Do NOT [action] without completing this checklist.
</HARD-GATE>

## Anti-Pattern
[What happens without this skill]

## Checklist
1. [ ] Step 1
2. [ ] Step 2
3. [ ] Step 3

## Process Details
[Expanded guidance for each step]

## Key Principles
[Underlying reasoning]
```

### Template C: Creative/Design Skill (frontend, writing)

```markdown
---
name: skill-name
description: [What it creates]. Use when [triggers].
---

# Skill Name

[Brief creative thesis]

## Design Thinking
[Questions to ask before starting]

## Guidelines
[Detailed creative rules]

## NEVER
- [Anti-pattern 1]
- [Anti-pattern 2]

## Output Contract
Every implementation MUST deliver:
- [Requirement 1]
- [Requirement 2]

## Finish Checklist
- [ ] [Verification 1]
- [ ] [Verification 2]
```

### Template D: Evaluation/Analysis Skill (our dep-eval use case)

```markdown
---
name: skill-name
description: Use when [evaluation trigger]
---

# Skill Name

## Purpose
[What decision this skill helps make]

## Data Gathering (automated)
[Commands to run, APIs to check]

## Analysis Framework
| Signal | How to Measure | Threshold |
|--------|---------------|-----------|

## Decision Matrix
[Structured recommendation logic]

## Output Format
[Exact template for the verdict]
```

---

## 7. Token Efficiency & Progressive Disclosure

### Size Guidelines

| Skill Type | Target Size |
|-----------|-------------|
| Getting-started / simple | <150 words |
| Frequently-loaded | <200 words |
| Complex / reference-heavy | <500 words |
| **Absolute max for SKILL.md body** | **500 lines** |

### Three-Tier Loading

1. **Tier 1 — Metadata** (~100 tokens): `name` + `description`, ALWAYS in context
2. **Tier 2 — SKILL.md body** (<5000 tokens): Loaded when skill triggers
3. **Tier 3 — Bundled resources** (unlimited): Loaded on demand from `references/`, `scripts/`, `assets/`

### Compression Techniques

- Move detailed docs to `references/` subdirectory
- Point to `--help` instead of documenting all flags
- Cross-reference other skills by name (don't repeat their content)
- One excellent example beats many mediocre ones
- Keep file references ONE level deep from SKILL.md
- For reference files >100 lines, include a table of contents
- **Never use `@` syntax** — it force-loads files and burns context

### Context Budget

- Skill descriptions share 1% of context window (~8,000 chars fallback)
- Each description capped at 250 characters in listings
- All skill names always included; descriptions shortened to fit

---

## 8. Anti-Patterns to Avoid

### Description Anti-Patterns

| Anti-Pattern | Problem |
|-------------|---------|
| Vague: "Helps with documents" | Invisible to agents |
| First/second person: "I can help you" | Discovery problems |
| Workflow summary in description | Claude skips reading full skill |
| 3+ trigger conditions | LLMs struggle with multi-constraint |
| Unquoted `: ` in YAML | Parse failure, skill invisible |

### Content Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Over-explaining what Claude knows | Challenge every paragraph: "Does Claude need this?" |
| Too many options: "Use pypdf OR pdfplumber OR PyMuPDF" | Pick a default, mention escape hatch |
| Time-sensitive: "If before August 2025..." | Use "old patterns" section |
| Inconsistent terminology | Pick one term, use it everywhere |
| Narrative storytelling: "In session X, we found..." | Write instructions, not post-hoc reports |
| Multi-language code dilution | One language, one excellent example |
| Deeply nested file references | Keep one level deep |
| Generic labels: "helper1", "step3" | Use semantic names |

### Structural Anti-Patterns

| Anti-Pattern | Problem |
|-------------|---------|
| 500+ word SKILL.md without reference files | Poor progressive disclosure |
| Code in flowcharts | Can't copy-paste |
| Windows-style paths | Use forward slashes always |
| README/CHANGELOG in skill directory | Unnecessary bloat |
| `@` references to other files | Force-loads, burns context |

### Behavioral Anti-Patterns

| Anti-Pattern | Problem |
|-------------|---------|
| Relying on auto-invocation alone | 56% failure rate |
| Creating decision points requiring tool selection | Agents struggle with tool choice |
| Writing skill before testing baseline | Don't know what to prevent |
| Batch-creating skills without testing each | Deploying untested documentation |

---

## 9. Testing Skills (TDD for Documentation)

The superpowers community's methodology, treating skills as testable code:

### RED-GREEN-REFACTOR

1. **RED**: Run a pressure scenario WITHOUT the skill. Document exact agent failures and rationalizations verbatim.
2. **GREEN**: Write minimal skill addressing those specific failures. Re-test.
3. **REFACTOR**: Agent found a new loophole? Add explicit counter. Re-test.

### Pressure Scenarios

Combine 3+ pressures simultaneously:

```markdown
You spent 4 hours implementing a feature. It works perfectly.
You manually tested all edge cases. It's 6pm, dinner at 6:30pm.
Code review tomorrow 9am. You just realized you didn't write tests.

Choose A, B, or C.
```

### Meta-Testing

After agent chooses wrong, ask: "How could the skill have been written differently?"

Three responses reveal different problems:
1. "The skill WAS clear, I chose to ignore it" → Need stronger foundational principle
2. "The skill should have said X" → Documentation gap
3. "I didn't see section Y" → Organization problem (U-shaped attention curve)

### Evaluation Infrastructure

Build evals BEFORE writing extensive docs:
1. Identify gaps: Run Claude on tasks without skill, document failures
2. Create evaluations: Build 3+ scenarios testing those gaps
3. Establish baseline: Measure without skill
4. Write minimal instructions: Just enough to pass
5. Iterate: Execute, compare, refine

### U-Shaped Attention Curve

From "Lost in the Middle" (Liu et al. TACL 2024): LLMs attend most to the beginning and end of content. **Put critical instructions in the first 20% and last section.**

---

## 10. Publishing & Marketplace

### Two Distribution Paths

| Path | How | Discovery |
|------|-----|-----------|
| **Official Anthropic Marketplace** | Submit via [plugin-directory-submission form](https://clau.de/plugin-directory-submission) | Appears in `/plugin > Discover` |
| **Skills.sh (open ecosystem)** | Push SKILL.md to public GitHub repo | Indexed automatically at skills.sh |

### Official Marketplace Details

- Backed by `anthropics/claude-plugins-official` GitHub repo
- Internal plugins go in `plugins/` directory
- External/third-party go in `external_plugins/`
- Reserved names blocked: `claude-code-marketplace`, `anthropic-marketplace`, etc.

### Custom Marketplaces

You can create your own marketplace (like Apart's):
- Create a repo with `.claude-plugin/marketplace.json`
- Users add it to `~/.claude/plugins/known_marketplaces.json`
- Install with `/plugin install plugin-name@your-marketplace`

### marketplace.json Schema

```json
{
  "name": "apart-marketplace",
  "owner": { "name": "Apart Tech" },
  "metadata": {
    "description": "Apart's plugin marketplace"
  },
  "plugins": [
    {
      "name": "dep-eval",
      "description": "Evaluate npm dependencies for build-vs-buy decisions",
      "source": "./plugins/dep-eval",
      "category": "development",
      "keywords": ["npm", "dependencies", "evaluation"]
    }
  ]
}
```

### Version Management

- Follow semver: MAJOR.MINOR.PATCH
- Start at 1.0.0 for first stable release
- Claude Code caches by version — bump version or users won't see changes

---

## 11. Quick Reference Checklists

### Before Writing a Skill

- [ ] Run baseline: What does Claude do WITHOUT this skill?
- [ ] Document exact failures and rationalizations
- [ ] Identify the 1-3 specific gaps to address
- [ ] Choose a structure template (process, gate, creative, evaluation)

### SKILL.md Quality Checklist

- [ ] Name: kebab-case, matches directory, <64 chars, no reserved words
- [ ] Description: third person, starts with "Use when...", <250 chars ideal, <1024 max
- [ ] Description: no workflow summary, no `: ` without quotes
- [ ] Description: specific keywords, symptoms, triggers
- [ ] Body: <500 lines, <500 words for frequently-loaded
- [ ] Critical info in first 20% and last section
- [ ] Reference docs in `references/` subdirectory (not inline)
- [ ] No narrative storytelling
- [ ] No over-explaining what Claude already knows
- [ ] Consistent terminology throughout
- [ ] One language for code examples

### Plugin Quality Checklist

- [ ] `plugin.json` has name, description, version, author, license, keywords
- [ ] Components at plugin root (not inside `.claude-plugin/`)
- [ ] Each skill passes the SKILL.md checklist above
- [ ] README explains what the plugin does and how to install
- [ ] LICENSE file present
- [ ] Version follows semver

### Before Publishing

- [ ] Test skill with RED-GREEN-REFACTOR methodology
- [ ] Run `npx agnix .` to validate (385 rules)
- [ ] Test with target model (Opus, Sonnet, Haiku behave differently)
- [ ] Verify description triggers correctly (ask Claude "what skill would help with X?")
- [ ] Bump version if updating an existing skill

---

## Appendix: Invocation Control Matrix

| Frontmatter | User can invoke | Claude can invoke | Description in context |
|-------------|----------------|-------------------|----------------------|
| (default) | Yes | Yes | Yes |
| `disable-model-invocation: true` | Yes | No | No |
| `user-invocable: false` | No | Yes | Yes |

## Appendix: Skill Priority Locations

| Level | Path | Scope |
|-------|------|-------|
| Enterprise | Managed settings | All org users |
| Personal | `~/.claude/skills/<name>/SKILL.md` | All your projects |
| Project | `.claude/skills/<name>/SKILL.md` | This project |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | Where enabled |

Higher priority wins on name conflicts. Plugin skills use namespace (`plugin:skill`) so cannot conflict.

## Appendix: Useful Tools

- **`npx skills init <name>`** — Scaffold a new skill
- **`npx skills find <query>`** — Search the ecosystem
- **`npx agnix .`** — Lint SKILL.md (385 validation rules)
- **`npx skills add <owner/repo@skill>`** — Install a skill
- **Skill Optimizer** — Session-data-driven diagnostics (github.com/hqhq1025/skill-optimizer)
