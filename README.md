# skills

Open-source [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) for Claude.

## Skills

| Skill | What it does |
| --- | --- |
| [adversarial-mr-review](adversarial-mr-review/SKILL.md) | Runs a pull/merge request through repeated adversarial, architect-level reviews, using a new independent reviewer sub-agent every round. Fixes every finding immediately and continues until two immediately consecutive fresh reviewers report zero findings. |

## Install

Copy a skill folder into your skills directory, for example:

```bash
git clone https://github.com/cbovolo/skills.git
cp -r skills/adversarial-mr-review ~/.claude/skills/
```

## License

MIT
