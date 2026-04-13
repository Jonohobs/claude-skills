# Skills Orchestration Architecture

## The Problem

Claude Code has a **15,000-character budget** for all skill descriptions combined. At ~500 chars/skill:
- 15 skills = 7,500 chars (fine)
- 30 skills = 15,000 chars (cap hit — skills silently dropped)

Tool selection accuracy degrades from 43% (dynamic retrieval) to 13% (all-tools-loaded). Source: RAG-MCP paper, arxiv 2505.03275.

## The Solution: Two-Tier Architecture

### Tier 1: Skills Index (`_index.yaml`)

Lightweight registry. ~20-30 tokens per skill. Zero cost when not queried.

```yaml
categories:
  3d:
    - name: threejs
      description: "Core Three.js — scenes, materials, lighting, loading, animation"
      inputs: [user-request]
      outputs: [three-js-code]
      composes-with: [r3f, webgpu-threejs-tsl, threejs-ecs]
    - name: r3f
      description: "React Three Fiber — declarative 3D in React"
      inputs: [user-request, react-component-context]
      outputs: [r3f-component-code]
      composes-with: [threejs, webgpu-threejs-tsl]
  meta:
    - name: creating-skills
      description: "Skill creation, optimization, orchestration"
      inputs: [skill-topic]
      outputs: [skill-files]
    - name: recon
      description: "Parallel multi-model research fan-out"
      inputs: [research-topic]
      outputs: [synthesis-report]
  design:
    - name: ui-ux-pro-max
      description: "UI/UX design intelligence"
    - name: banner-design
      description: "Social media and web banners"
```

### Tier 2: Skills Expert (`_skills-expert/SKILL.md`)

The coordinator. Its job:

1. **Read `_index.yaml`** to know what's available
2. **Classify the task** — single-skill or multi-skill?
3. **For single-skill:** Route directly (low overhead)
4. **For multi-skill:** Plan a DAG:
   - Which skills needed?
   - What order? (dependencies)
   - What are the handoff formats?
   - Can any run in parallel?
5. **Execute the plan** — activate skills in sequence

### When to Use Each Pattern

| Complexity | Pattern | Example |
|-----------|---------|---------|
| Simple | Native routing | "Add a button component" |
| Medium | Skills-expert routes | "Build a Three.js scene" (→ threejs skill) |
| Complex | Skills-expert plans DAG | "Build a WebGPU particle system in R3F with ECS" (→ threejs + r3f + webgpu-threejs-tsl + threejs-ecs) |

## Composition Recipes

### Recipe: Three.js + React
1. `threejs` — core scene logic, materials, lighting
2. `r3f` — React component patterns, hooks, drei
3. Output: R3F component with proper Three.js patterns

### Recipe: Full Design
1. `ui-ux-pro-max` — design system, colors, typography
2. `design-system` — tokens, component specs
3. `ui-styling` — shadcn/ui implementation
4. Output: Styled, accessible component

### Recipe: Research → Skill
1. `recon` — multi-model research on topic
2. `creating-skills` — structure findings into skill
3. Output: New skill with references

## DAG-Based Orchestration

From AgentSkillOS paper (arxiv 2603.02176): DAG orchestration "substantially outperforms native flat skill invocation even when given the identical skill set."

```
Task: "Build a WebGPU particle system in R3F"
         ┌─────────────┐
         │ skills-expert│ (decomposes)
         └──────┬───────┘
        ┌───────┼───────┐
        ▼       ▼       ▼
   [threejs] [webgpu] [r3f]    (parallel: domain knowledge)
        │       │       │
        └───────┼───────┘
                ▼
         [integration]          (sequential: combine outputs)
                ▼
            [output]
```

## Evidence Summary

| Source | Finding |
|--------|---------|
| RAG-MCP (arxiv 2505.03275) | 13% → 43% accuracy with dynamic retrieval |
| AgentSkillOS (arxiv 2603.02176) | DAG orchestration outperforms flat with identical skills |
| Clinical study (PMC12393657) | Single-agent: 73% → 16% at scale. Multi-agent: 90% → 65% |
| Anthropic guidance | "Use lowest complexity that reliably meets requirements" |
| Gemini consensus | Build as latent capability, mandatory at 50+ |
| Multi-model bias note | All models had structural incentive to recommend orchestration |
