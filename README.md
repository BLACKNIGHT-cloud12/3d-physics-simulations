# 3D Physics Simulations — Agent Skill

An [Agent Skill](https://agentskills.io) (`SKILL.md`) that teaches AI coding agents (ZCode, Claude Code, Codex, Cursor) to run real physics inside three.js websites: rigid bodies, gravity, collisions, cloth, and GPU particle simulations with Rapier or cannon-es — plus the "fake physics" decision framework that keeps marketing sites at 60fps.

## What it covers

- Engine choice: Rapier vs cannon-es vs GPGPU particles
- Wiring Rapier to three.js (fixed timestep, body→mesh sync)
- The **click-impulse pattern** used on hero sections
- Instancing + body budgets for many objects
- "Fake physics" springs for effects that don't need an engine
- GPGPU ping-pong particle template (`references/gpgpu-particles.md`) — 10k–1M particles on the GPU

## Install

```bash
# All projects (user scope)
cp -r 3d-physics-simulations ~/.agents/skills/

# Or single project
cp -r 3d-physics-simulations .claude/skills/
```

## Use it

Ask your agent naturally: *"add physics to my hero objects and make them react to clicks"*, *"simulate a cloth banner"*, *"make a neural-network particle cloud"* — the skill triggers on physics/simulation/collision requests.

## License

MIT
