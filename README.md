# PlatformSmith — skills for coding agents

Guidance that helps a coding agent drive the [PlatformSmith](https://platformsmith.com) control
plane over MCP instead of guessing at it. Install instructions, and what the skill is for:
**https://docs.platformsmith.com/api/mcp-skill**

## Claude Code

```
/plugin marketplace add platform-smith-labs/claude-plugins
/plugin install platform-smith@platform-smith
```

## Codex

Codex has no marketplace — place the skill directory where Codex looks for it:

```bash
git clone --depth 1 https://github.com/platform-smith-labs/claude-plugins /tmp/ps-plugins
mkdir -p ~/.agents/skills
cp -R /tmp/ps-plugins/plugins/platform-smith/skills/platform-smith ~/.agents/skills/
rm -rf /tmp/ps-plugins
```

For one repository instead, copy that directory to `.agents/skills/` in the repository root.

## Generated

This repository is **generated** from the `ps-mcp` service repo by
`scripts/build-public-skill.sh`. Do not edit the skill here — the change would be overwritten on the
next release, and the tool inventory is produced from the server's own tool specs.
