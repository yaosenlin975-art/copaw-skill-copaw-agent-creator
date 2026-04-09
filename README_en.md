# CoPaw Skill: Agent Creator (with Skill Auto-Assembly)

English | [中文](README.md)

This repository provides a custom **CoPaw (AgentScope)** skill: **copaw_agent_creator**.  
It creates a new CoPaw agent (workspace), registers it so it becomes visible in the CoPaw Console UI, and follows the required “skill auto-assembly” workflow:

1. Search the local skill pool (`~/.copaw/skill_pool`) and import matching skills into the new agent  
2. Search online skill marketplaces (prefer `npx clawhub search`) and import matching skills (dedupe vs step 1 / similar ones)  
3. If nothing is imported from 1+2, generate a minimal new skill for the agent (Skill-Creator style)

## IMPORTANT: Write-safety protocol

This skill creates directories and updates JSON configs, so it is **dry-run by default**.  
It will only write when you explicitly allow it AND you run with `--write`.

For every write, the agent must:

1) Ask for your permission every time  
2) Before overwriting any file:
- validate format (JSON must be parseable)
- create a same-directory backup: `<file>.bak.<timestamp>`
- prefer temp-file + atomic replace

The script enforces a hard guard: without `--write`, it does not write anything.

## Quick start

Dry-run:

```bash
python scripts/create_agent.py --spec-md ./agent_spec.md --dry-run
```

Write (only after explicit approval):

```bash
python scripts/create_agent.py --spec-md ./agent_spec.md --write
```

## Contents

- `SKILL.md`: Skill handbook for CoPaw agents (workflow, boundaries, permission prompts)
- `scripts/create_agent.py`: The main implementation (workspace + registry + skill assembly)

## License

Apache-2.0. See [LICENSE](LICENSE).

