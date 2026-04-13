# Research Methodology for Skill Design

## The /recon Pattern

When designing a skill for a new domain, use `/recon` to research from multiple angles before writing.

### Standard Recon (4 agents)
1. **Best Practices** (Sonnet) — established methods, frameworks, docs
2. **Cutting Edge** (Sonnet) — newest tools, AI approaches, trends
3. **Implementation** (Sonnet) — code architecture, libraries, patterns
4. **Gemini** (CLI) — alternative perspective, free tier

### Deep Recon (add more angles)
- Browser-based models (DeepSeek, Manus) for additional perspectives
- Domain-specific searches for the skill's topic area

## Multi-Model Bias Framework

When synthesizing multi-model research, account for structural biases:

| Model Role | Bias | Mechanism |
|-----------|------|-----------|
| Base agent (Claude Code) | Self-preservation | Recommending help = admitting limitation |
| Peripheral models (Gemini, etc.) | Specialist bias | Their role IS being the specialist node |
| Sub-agents (Sonnet spawns) | Specialization confirmation | They only exist for focused tasks |
| Research papers | Least biased | But recency/publication bias exists |

### Weighting Formula
```
paper evidence (3x) > model consensus (2x) > single model opinion (1x)
```

### Red Flag: Suspicious Unanimity
When all models agree, ask: "What would each model lose by disagreeing?"
- If the answer is "nothing" → genuine consensus
- If each model benefits from the consensus → structural bias, weight papers more

## From Research to Skill

### Step 1: Run /recon on the topic
```
/recon "Three.js best practices" --depth deep
```

### Step 2: Identify the knowledge structure
- What's the core that everyone needs? → SKILL.md inline
- What's deep reference? → `references/` files
- What's executable? → `scripts/`

### Step 3: Apply the "Would removing this cause mistakes?" test
For every line in SKILL.md: if removing it wouldn't change Claude's behavior, cut it.

### Step 4: Write pushy description
Include every trigger keyword from the research. Be explicit about when to activate.

### Step 5: Test with 3 prompts
- One obvious trigger ("build a Three.js scene")
- One ambiguous trigger ("make something 3D")
- One negative ("build a React component" — should NOT trigger)
