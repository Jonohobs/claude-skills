# SKILL.md Official Specification

Source: agentskills.io/specification + Anthropic docs

## Frontmatter (YAML between `---` delimiters)

### Required
```yaml
name: kebab-case-name        # Max 64 chars. Lowercase + digits + hyphens.
                              # Must match parent directory name.
                              # Namespaced: "ckm:skill-name" allowed (colon separator).
description: >               # Max 1024 chars. Third person.
  What it does. USE when...   # No XML tags in description.
```

### Optional (Claude Code specific)
```yaml
argument-hint: "[args]"                 # Autocomplete hint for /slash-command
disable-model-invocation: true          # User-only (blocks auto-trigger)
                                        # Known bug: #26251 — may block slash too
allowed-tools:                          # Restrict available tools during skill
  - Read
  - Bash(git:*)                         # Scoped with wildcards
context: fork                           # Run in isolated subagent
agent: Explore                          # Which subagent type (Explore, Plan, custom)
mode: true                              # Marks as mode command (ongoing behavior change)
```

### Optional (Open Standard / cross-platform)
```yaml
license: MIT                            # SPDX identifier
compatibility: "python3, poppler-utils" # Max 500 chars. Env requirements.
metadata:                               # Arbitrary key-value map
  author: "jonathan"
  version: "1.0.0"
  category: "meta"
```

## Body (Markdown after frontmatter)

No structural restrictions. Recommended sections:
1. Purpose / overview (2-4 sentences)
2. Process steps (numbered, imperative, with file references)
3. Reference file index with when-to-load guidance
4. Anti-patterns / gotchas

## Loading Mechanism

1. **Startup:** Claude scans `~/.claude/skills/*/SKILL.md` — reads only `name` + `description`
2. **Activation:** When task matches description, full SKILL.md body loads via internal `load_skill` call
3. **Execution:** Sub-files load on demand when Claude reads them

Total description budget across ALL skills: ~15,000-16,000 characters.
If exceeded, some skills become invisible.

## Cross-Platform Compatibility

The SKILL.md format is an open standard (agentskills.io, December 2025).
Supported by: Claude Code, OpenAI Codex CLI, Cursor, Gemini CLI, GitHub Copilot, OpenCode.

For portability:
- Write tool-agnostic instructions ("Run the project's test command" not "Run npm test")
- Use forward slashes in paths (not backslashes)
- Reference tools by capability ("Read the file" not "Use the Read tool")

## Validation

The official `skill-creator` skill (github.com/anthropics/skills) includes validation.
Known validator bug: incorrectly rejects `license`, `compatibility`, and `metadata` fields.
These ARE valid per the spec.
