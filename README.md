# skills

Open-source [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) for Claude.

## Skills

| Skill | What it does |
| --- | --- |
| [adversarial-mr-review](adversarial-mr-review/SKILL.md) | Runs a pull/merge request through repeated rounds of adversarial, architect-level review, fixing every finding immediately, until two consecutive rounds come back clean. |

## Install

Copy a skill folder into your skills directory, for example:

```bash
git clone https://github.com/cbovolo/skills.git
cp -r skills/adversarial-mr-review ~/.claude/skills/
```

## License

MIT
