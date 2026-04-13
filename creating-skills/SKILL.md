---
name: creating-skills
description: >
  Create, structure, and optimize Claude Code skills (SKILL.md plugins).
  USE when building new skills, improving existing ones, organizing skill libraries,
  planning skill composition, or debugging skill activation issues.
  Also use when the user says "make a skill", "create a skill", "skill for X",
  or asks about SKILL.md format, skill best practices, or skill orchestration.
argument-hint: "[skill-topic] [--eval] [--improve]"
metadata:
  author: community
  version: "1.0.0"
---

# Creating Skills — The Skill Forge

Build excellent Claude Code skills. Based on Anthropic's official guidance, the agentskills.io open standard, and extensive multi-model research.

## Core Philosophy

> "Skills are generalizations, not scripts. The best skills encode the WHY behind instructions so the model can handle novel variants — not just the WHAT for the cases you tested." — Anthropic skill-creator source

A great skill is a **state machine**, not a prompt. It has defined triggers, inputs, steps, outputs, and validation.

## Quick Start — Create a New Skill

### Step 1: Scaffold
```bash
mkdir -p ~/.claude/skills/my-skill-name/references
```

### Step 2: Write SKILL.md
```yaml
---
name: my-skill-name
description: >
  Does X for Y. USE when the user asks about Z, mentions A,
  or is working with B files. Even if they don't explicitly ask.
metadata:
  version: "1.0.0"
---

# My Skill Name

## When to Use
[Redundant with description but helps execution context]

## Process
1. [Step with explicit file references]
2. [Step with validation]
3. [Step with output spec]

## Reference Files
- See `references/domain-knowledge.md` for [specific lookup needs]
```

### Step 3: Test
Complete the task WITHOUT the skill first. Then WITH. Compare.

## The Rules

### Description Is Everything
The description field is the **ONLY trigger mechanism**. "When to Use" sections in the body are invisible at selection time.

**Bad:** `"Helps with Three.js"`
**Good:** `"Core Three.js skill for building 3D web experiences. USE when creating scenes, loading GLTF models, setting up WebGL renderers, adding post-processing, writing GLSL shaders, or debugging Three.js performance. Even if the user just says 'make something 3D'."`

Formula: **Capability statement + explicit trigger conditions + pushy "even if" clause**

Write in third person. Max 1024 chars. Aim for 150-300 in practice.

### Progressive Disclosure (3 Tiers)

| Tier | What | Tokens | When |
|------|------|--------|------|
| 0 | name + description | ~100/skill | Always (startup) |
| 1 | Full SKILL.md body | <5,000 | On skill activation |
| 2 | Reference files | Variable | On-demand during execution |

This means: **a massive `references/` directory costs zero tokens unless needed.**

### SKILL.md Size Limits
- **Hard ceiling: 500 lines**
- **Soft target: <300 lines** for most skills
- Move domain knowledge to `references/`
- Keep SKILL.md focused on PROCESS (steps), not CONTEXT (knowledge)

### One Level Rule
Never reference files from within reference files. Claude uses `head -100` on nested files and misses content. All reference links must originate from SKILL.md directly.

### Explain WHY, Not Just WHAT
Anthropic's own guidance: "If you find yourself writing ALWAYS or NEVER in caps, that's a yellow flag — reframe and explain the reasoning. That's more humane, powerful, and effective."

Bad: `"ALWAYS use SRGBColorSpace for color textures."`
Good: `"Use SRGBColorSpace for color textures because the renderer does linear workflow — it converts sRGB inputs to linear for lighting math, then back to sRGB for display. Without this, colors look washed out."`

## Frontmatter Spec

### Required Fields
| Field | Constraints |
|-------|------------|
| `name` | Max 64 chars. Lowercase + hyphens + digits. Must match directory name. |
| `description` | Max 1024 chars. Third person. What + When + pushy triggers. |

### Optional Fields
| Field | Purpose |
|-------|---------|
| `argument-hint` | Autocomplete hint for `/skill-name`. E.g., `"[topic] [--depth deep]"` |
| `disable-model-invocation` | User-only invocation (blocks auto-trigger). Use for side-effect skills. |
| `allowed-tools` | Restrict which tools Claude may use. E.g., `[Read, Grep, Glob]` |
| `context: fork` | Run skill in isolated subagent context. |
| `agent` | Which subagent type handles it (Explore, Plan, or custom). |
| `license` | SPDX identifier. E.g., `MIT` |
| `metadata` | Key-value map: `author`, `version`, `category`, etc. |
| `compatibility` | Max 500 chars. Special env requirements. |

### Reserved
- Never use `anthropic-*` or `claude-*` in names.

## File Organization

```
my-skill/
  SKILL.md              ← Process steps (<500 lines)
  references/           ← Domain knowledge (loaded on demand)
    api-reference.md
    patterns.md
    gotchas.md
  scripts/              ← Python tools (stdlib-only, zero pip)
    validate.py
    generate.py
  templates/            ← Output templates
  examples/             ← Sample inputs/outputs for eval
```

### Reference File Best Practices
- One level deep from SKILL.md only
- Files over 100 lines: include a table of contents at top
- Name descriptively: `form_validation_rules.md` not `doc2.md`
- YAML > JSON for data — LLMs parse it more reliably with fewer tokens
- Shared reference files across skills: put in `~/.claude/skills/_shared/`

### Scripts Best Practices
- **stdlib-only Python** — zero pip installs, works everywhere
- Scripts execute without source entering context — only stdout consumes tokens
- Error handling belongs IN the script, not as a Claude inference problem
- Use `{baseDir}` in paths for portability
- No magic numbers — document every constant

## Naming Conventions
- Preferred: gerund form — `creating-skills`, `processing-pdfs`
- Also fine: `skill-forge`, `pdf-processor`
- Namespace large domains: `three-core`, `three-r3f`, `three-webgpu`
- Avoid: `helper`, `utils`, `tools`, `data` (too generic)

## The Dev Loop (Official Anthropic Method)

1. **Do the task WITHOUT the skill** — capture what context you naturally provided
2. **Draft the skill** from that captured context
3. **Write 3 realistic test prompts** — messy, like real users write
4. **Run with-skill AND baseline in parallel**
5. **Review outputs** qualitatively + quantitatively
6. **Improve based on observed behavior**, not assumptions
7. **Repeat** until delta shrinks
8. **Optimize description** — test 20 trigger/no-trigger prompts

Key insight: "If all test cases result in Claude writing the same helper script, that's a signal the skill should bundle that script."

## Composition & Orchestration

### When Skills Need to Work Together
- Reference other skills by name in description: `"Composes with three-r3f and three-webgpu skills"`
- Define input/output contracts: what a skill produces → what the next skill consumes
- For sequencing, be explicit: `"After generating the component, run /test-skill"`

### The Skills Index Pattern (15+ skills)
Create `~/.claude/skills/_index.yaml`:
```yaml
skills:
  - name: creating-skills
    category: meta
    triggers: "skill creation, SKILL.md, skill best practices"
    composes-with: []
  - name: threejs
    category: 3d
    triggers: "Three.js, 3D, WebGL, scene, mesh"
    composes-with: [r3f, webgpu-threejs-tsl]
```

### The Skills Expert Pattern (30+ skills)
Build `_skills-expert/SKILL.md` — a coordinator that:
- Reads `_index.yaml` (compact: ~20-30 tokens per skill)
- Decomposes multi-domain tasks into skill sequences
- Loads only needed skills (avoids 15,000-char description budget cap)
- Optional for simple tasks, mandatory at 30+ skills

See `references/orchestration-architecture.md` for full pattern.

### Scale Thresholds
| Skills | Strategy |
|--------|----------|
| 1-15 | Flat routing (native). Keep descriptions sharp. |
| 15-30 | Add `_index.yaml`. Consider skills-expert. |
| 30+ | Skills-expert mandatory. Categorical routing. |

## Anti-Patterns

1. **Dumping everything into SKILL.md** — process steps get lost in walls of text
2. **Vague descriptions** — "Helps with documents" = zero discovery
3. **"When to Use" only in body** — invisible at trigger time. Put in description.
4. **Nested reference chains** — SKILL.md → a.md → b.md. Claude misses b.md.
5. **Too many choices** — "Use pypdf or pdfplumber or PyMuPDF..." Pick one.
6. **ALWAYS/NEVER in caps** — explain WHY instead
7. **Scripts that punt to Claude** — error handling goes in the script
8. **Hardcoded absolute paths** — use `{baseDir}/` for portability
9. **Creating objects in render loops** — wait, that's Three.js. Stay focused.
10. **Not testing** — 3 eval prompts minimum before shipping

## Multi-Model Research Bias

When using `/recon` or multi-model queries to inform skill design, account for structural biases:
- **Base agent** has self-preservation bias (says "I can handle it")
- **Peripheral models** have specialist bias (recommend specialist nodes)
- **Papers** are least biased but have publication/recency bias
- Weight: paper evidence > model consensus > single model opinion

See `references/research-methodology.md` for the full bias framework.

## Reference Files

| File | When to Load |
|------|-------------|
| `references/orchestration-architecture.md` | Planning skill coordination or building a skills-expert |
| `references/research-methodology.md` | Running /recon or multi-model research for skill design |
| `references/official-spec.md` | Need exact frontmatter field constraints |
| `references/examples.md` | Need to see real-world skill patterns |

## Resources

- Official Anthropic skills: `github.com/anthropics/skills`
- Skill-creator plugin: `claude.com/plugins/skill-creator`
- Open standard spec: `agentskills.io/specification`
- Community skills (5.2k stars): `github.com/alirezarezvani/claude-skills`
- Skill factory toolkit: `github.com/alirezarezvani/claude-code-skill-factory`
- Awesome list: `github.com/travisvn/awesome-claude-skills`
- Memory reference: your local memory files (e.g., `skills-engineering.md`)
