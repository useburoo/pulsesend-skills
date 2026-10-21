# PulseSend skills for AI agents

Agent skills that teach coding assistants (Claude Code, and any tool that reads `SKILL.md` folders) how to integrate [PulseSend](https://pulse-send.jiordiviera.me) correctly.

| Skill           | Covers                                                                                 |
| --------------- | -------------------------------------------------------------------------------------- |
| `pulsesend`     | Sending with the API, test and live keys, idempotency, error codes, limits, webhook signatures |
| `pulsesend-cli` | The `pulsesend` command: sending from a terminal or CI, `--json` scripting, debugging deliveries |

## Install

Claude Code, for every project:

```bash
git clone https://github.com/jiordiviera/pulsesend-skills.git ~/.local/share/pulsesend-skills
ln -s ~/.local/share/pulsesend-skills/pulsesend ~/.claude/skills/pulsesend
ln -s ~/.local/share/pulsesend-skills/pulsesend-cli ~/.claude/skills/pulsesend-cli
```

For one project only, copy the folders into `.claude/skills/` at its root. `git pull` in the clone updates the linked skills.

Other agents: point them at the `SKILL.md` files; each one is self-contained, with details in `references/`.

## Contents

```
pulsesend/
  SKILL.md
  references/examples.md   # Node, Laravel, Python, PHP, Go
  references/errors.md     # every error code and the retry policy
  references/limits.md
  references/webhooks.md   # signature verification in four languages
pulsesend-cli/
  SKILL.md
```

Documentation: https://docs.pulse-send.jiordiviera.me

This repository mirrors the `skills/` folder of the PulseSend monorepo.

## License

MIT
