# Install

## Claude Code

```
/plugin marketplace add anycloud-sh/anycloud-skills
/plugin install anycloud@anycloud-skills
```

Once installed, run `/reload-plugins` to activate.

## Community marketplace

Anycloud is not listed in the community catalog as of October 8, 2026. Use the
direct installation above. Directory review and publication are separate from
installing this plugin; check the [community catalog](https://github.com/anthropics/claude-plugins-community/blob/main/.claude-plugin/marketplace.json)
before relying on that distribution path.

## Codex CLI

Install the skill in Codex's native user skill directory:

```bash
mkdir -p ~/.agents/skills/anycloud
curl -fsSL https://anycloud.sh/SKILL.md -o ~/.agents/skills/anycloud/SKILL.md
```

Start a new Codex session after installation. Codex can select the skill from
its description when a task matches; installing it does not itself provision
compute. See [Codex skills](https://developers.openai.com/codex/skills) for
project-scoped installation and skill discovery.

## Cursor / Other agents

The skill follows the open Agent Skills spec. Tell your agent:

> set up https://anycloud.sh/SKILL.md

## After install

The skill bootstraps the anycloud CLI on first use. The user will need:

1. anycloud installed: `curl -fsSL https://get.anycloud.sh | sh` (no sudo) or `brew install anycloud-sh/tap/anycloud`
2. Logged in: `anycloud login` (GitHub OAuth)
3. Active API healthy and compatible: `anycloud api status`. Start a local target with `anycloud api start`; a hosted target does not require a local server.
4. A cloud credential when provisioning cloud capacity: `anycloud credentials new`

The skill checks the setup needed for the task. Inspecting existing resources
does not require adding cloud credentials.
