---
name: agent-training
description: >
  Train and improve AI agents and Claude Code skills by absorbing external codebases,
  GitHub repos, and documentation. USE when the user wants to "train" a skill, "absorb"
  a codebase, improve an agent's knowledge, calibrate agent behavior, evaluate skill
  quality, or says "learn from this repo". Also use for skill composition, agent
  orchestration patterns, and quality metrics.
metadata:
  version: "1.0.0"
  license: MIT
  author: community
---

# Agent Training — Skill Forge Companion

Train AI agents and skills by absorbing external knowledge sources.

## Process

### Phase 1: Source Discovery
1. Identify the source (GitHub user/org, repo, docs site, API reference)
2. Map the repo landscape:
   - `WebFetch` the GitHub profile → list all repos
   - Sort by: stars, recency, language relevance
   - Identify 5-10 highest-signal repos for the target domain
3. Filter for quality signals: stars, commit frequency, README completeness, test coverage

### Phase 2: Pattern Extraction
1. For each selected repo, fetch and analyze:
   - README for architecture overview
   - Source structure (languages, file organization)
   - Key techniques used (from code analysis)
2. Categorize patterns found:
   - **Architecture patterns** — how code is organized
   - **Technique patterns** — specific algorithms, APIs, or approaches
   - **Anti-patterns** — things explicitly avoided and why
   - **Integration patterns** — how components connect
3. Deduplicate across repos — merge similar patterns, note variations

### Phase 3: Knowledge Structuring
1. Organize extracted patterns into reference file format:
   - Table of contents at top
   - Code-heavy sections (snippets > paragraphs)
   - Decision trees for choosing between alternatives
   - Performance/tradeoff notes
2. Size targets:
   - Reference file: <300 lines per file
   - Split large domains into multiple reference files
   - Each file = one coherent topic

### Phase 4: Skill Integration
1. Determine target skill to enhance (existing or new)
2. For existing skills:
   - Read current SKILL.md and all references
   - Identify gaps the new knowledge fills
   - Add new reference files — don't bloat SKILL.md
   - Update SKILL.md reference table to point to new files
3. For new skills:
   - Scaffold: mkdir -p ~/.claude/skills/skill-name/references
   - Write SKILL.md (<300 lines, process-focused)
   - Write reference files (knowledge-focused)
4. Update description if new triggers are relevant

### Phase 5: Validation
1. **Baseline test**: Do the target task WITHOUT the updated skill
2. **Enhanced test**: Do the same task WITH the skill
3. **Compare**: Did the skill improve output quality, speed, or correctness?
4. **Edge cases**: Test 3 messy, realistic prompts
5. Quality metrics:
   - Trigger accuracy: does it activate when it should? (test 10 prompts)
   - Output quality: are responses better with the skill? (A/B compare)
   - Token efficiency: does it load too much context unnecessarily?
   - Composition: does it play well with related skills?

## Absorption Patterns

### GitHub User Mining
```
1. Fetch profile → list repos (sort by stars/updated)
2. Filter: language match + topic relevance + quality signals
3. Deep-dive top 5-10 repos (README + structure + key files)
4. Extract patterns → deduplicate → structure → write references
```

### Documentation Absorption
```
1. WebFetch docs site → map structure (nav, sections)
2. Prioritize: getting started, API reference, advanced guides
3. Extract: API patterns, gotchas, migration notes
4. Cross-reference with existing skill knowledge
```

### Codebase Reverse Engineering
```
1. Clone or fetch repo structure
2. Identify entry points (main, index, app)
3. Trace execution flow → document architecture
4. Extract reusable patterns with code snippets
```

## Agent Calibration

### Behavior Tuning
- **Trigger sensitivity**: Broaden description for under-triggering, narrow for over-triggering
- **Output format**: Add explicit output specs if responses are inconsistent
- **Depth control**: Use "progressive disclosure" — summary first, detail on request
- **Tool use**: Restrict with `allowed-tools` if agent uses wrong tools

### Composition Patterns
- **Sequential**: Skill A produces artifact → Skill B consumes it
- **Parallel**: Multiple skills provide context for the same task
- **Hierarchical**: Skills-expert routes to domain skills
- **Feedback loop**: Validation skill checks output of creation skill

## Anti-Patterns
1. **Absorbing without filtering** — 99 repos doesn't mean 99 useful repos
2. **Dumping raw content** — extracted patterns must be structured and deduped
3. **Ignoring existing knowledge** — always read current skill state first
4. **Testing only happy path** — messy prompts reveal real skill quality
5. **Over-training** — more knowledge ≠ better skill. Focus on gaps.

## Quality Metrics Checklist
- [ ] Trigger accuracy ≥ 80% on 10-prompt test
- [ ] A/B comparison shows measurable improvement
- [ ] Token load < 5000 on activation (excluding on-demand references)
- [ ] No duplicate knowledge with existing references
- [ ] Works with related skills (no conflicts)
