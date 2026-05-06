# Agent Skills

Personal Codex/agent skills synced through a private GitHub repository.

Each skill lives in its own top-level directory:

```text
agent-skills/
  pr-review-loop/
    SKILL.md
    agents/openai.yaml
```

To install a skill on a device, symlink or copy the skill directory into the local skills directory, for example:

```bash
ln -s ~/.agents/agent-skills/pr-review-loop ~/.agents/skills/pr-review-loop
```
