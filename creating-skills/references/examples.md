# Real-World Skill Examples

## Example 1: Minimal Skill (Instructions Only)

```
fix-linting/
  SKILL.md
```

```yaml
---
name: fix-linting
description: >
  Fix ESLint and Prettier issues in JavaScript/TypeScript files.
  USE when the user mentions linting errors, formatting issues,
  or asks to clean up code style.
---

# Fix Linting

1. Run `npx eslint --fix .` on the target files
2. Run `npx prettier --write .` on the same files
3. If errors remain, read the error output and fix manually
4. Verify: run `npx eslint .` — should return 0 errors
```

## Example 2: Reference-Heavy Skill

```
threejs/
  SKILL.md           (~400 lines: process + gotchas + quick ref)
  references/
    materials.md     (loaded when working with materials)
    lighting.md      (loaded when setting up lights)
    performance.md   (loaded when optimizing)
```

The SKILL.md contains the top 10 gotchas inline (needed every time), with deep domain knowledge in reference files (loaded on demand).

## Example 3: Script-Powered Skill

```
ui-ux-pro-max/
  SKILL.md
  scripts/
    search.py        (searches CSV databases for styles, colors, fonts)
  data/
    styles.csv
    colors.csv
    fonts.csv
```

The skill invokes `python scripts/search.py "minimalist tech"` — script output enters context, but the script source and CSV data do NOT. Massive token savings.

## Example 4: Router Skill

```
design/
  SKILL.md           (routing table only)
  references/
    logo-design.md
    banner-design.md
    slides.md
```

SKILL.md contains a routing table:
```markdown
| Task | Sub-skill | Reference |
|------|-----------|-----------|
| Logo | Logo (built-in) | references/logo-design.md |
| Banner | banner-design | External skill |
| Slides | slides | External skill |
```

## Example 5: Forked Subagent Skill

```yaml
---
name: deep-research
description: Research a topic thoroughly using Explore agent
context: fork
agent: Explore
allowed-tools: [Read, Grep, Glob, WebSearch, WebFetch]
---
```

Runs in an isolated context. Results return to main conversation without polluting it with intermediate searches.

## Example 6: The Recon Pattern (Multi-Agent Fan-Out)

```
recon/
  SKILL.md
```

Spawns 4 parallel agents (3 Sonnet + 1 Gemini CLI), each researching from a different angle. Synthesizes results. No scripts needed — the skill IS the orchestration logic.

## Anti-Pattern Examples

### Bad: Vague Description
```yaml
description: "Helps with documents"
```
Activation rate: ~20%. Claude has no idea when to use this.

### Bad: Body Too Long
A 1,200-line SKILL.md where the critical process steps are buried at line 800. Claude skims and misses them.

### Bad: Nested References
```
SKILL.md → references/advanced.md → references/details/specific.md
```
Claude reads SKILL.md, partially reads advanced.md (head -100), never sees specific.md.

### Bad: All Knowledge Inline
A 2,000-line SKILL.md with every API endpoint documented. Should be: 200-line SKILL.md with process steps + `references/api.md` for lookups.
