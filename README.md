[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

# Claude Code Skills

A collection of open-source skills for [Claude Code](https://claude.ai/claude-code).

## Included Skills

| Skill | Description |
|-------|-------------|
| **creating-skills** | Create, structure, and optimize Claude Code skills (SKILL.md plugins). Covers the full skill dev loop: scaffolding, frontmatter spec, progressive disclosure, composition, and orchestration. |
| **threejs** | Core Three.js skill for building 3D web experiences. Scene setup, GLTF loading, materials, lighting, shaders, post-processing, animation, physics, instancing, and performance debugging. |

## Installation

Copy any skill folder into your Claude Code skills directory:

```bash
# Clone the repo
git clone https://github.com/Jonohobs/claude-skills.git

# Copy the skill(s) you want
cp -r claude-skills/creating-skills ~/.claude/skills/
cp -r claude-skills/threejs ~/.claude/skills/
```

Skills are picked up automatically by Claude Code on the next session.

## How Skills Work

Each skill is a directory containing a `SKILL.md` file with YAML frontmatter (name, description, metadata) and markdown body (process steps, reference tables). Optional `references/` subdirectories hold domain knowledge loaded on demand.

See the **creating-skills** skill itself for the full specification and best practices.

## Contributing

PRs welcome. Each skill should follow the structure documented in `creating-skills/SKILL.md`.

## License

MIT -- see [LICENSE](LICENSE).

---

Created by Jonathan Hobman with Claude Code.
