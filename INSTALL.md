# Install

## Claude Code

```
/plugin marketplace add anycloud-sh/anycloud-skills
/plugin install anycloud@anycloud-skills
```

Once installed, run `/reload-plugins` to activate.

## Community marketplace (once listed)

```
/plugin marketplace add anthropics/claude-plugins-community
/plugin install anycloud@claude-community
```

## Codex CLI / Cursor / Other agents

The skill follows the open Agent Skills spec. Tell your agent:

> Fetch and follow https://github.com/anycloud-sh/anycloud-skills/blob/main/skills/anycloud/SKILL.md to install the anycloud skill.

## After install

The skill bootstraps the anycloud CLI on first use. The user will need:

1. anycloud installed: `brew install anycloud-sh/tap/anycloud` (or follow the [manual install guide](https://anycloud.sh/getting-started/))
2. Logged in: `anycloud login` (GitHub OAuth)
3. Active API healthy and compatible: `anycloud api info`. Start a local target with `anycloud api start`; a hosted target does not require a local server.
4. A cloud credential when provisioning cloud capacity: `anycloud credentials new`

The skill checks the setup needed for the task. Inspecting existing resources
does not require adding cloud credentials.
