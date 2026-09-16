# anycloud-skills

Claude Code / Codex CLI / ChatGPT [Agent Skills](https://code.claude.com/docs/en/skills) for running Jobs, Services, and VMs and inspecting Kubernetes Deployments via [anycloud](https://anycloud.sh).

## Install

### Claude Code

```
/plugin marketplace add anycloud-sh/anycloud-skills
/plugin install anycloud@anycloud-skills
```

Or, once listed in the community marketplace:

```
/plugin marketplace add anthropics/claude-plugins-community
/plugin install anycloud@claude-community
```

### Codex CLI / Cursor / Other agents

The skill follows the open [Agent Skills](https://github.com/anthropics/skills) spec — point your agent at `skills/anycloud/SKILL.md`.

## What it does

Teaches the agent how to:

- Train, fine-tune, or evaluate AI models on remote cloud GPUs (H100, A100, B200, etc.)
- Run batch inference and hyperparameter sweeps
- Preprocess large datasets that don't fit on a laptop
- Submit containerized batch jobs to multi-cloud BYOC infrastructure
- Deploy long-running HTTP Services and create persistent VMs
- List workloads by type and filter Deployments by Cluster or Jobs by Deployment
- Use spot instances with automatic checkpoint recovery
- Compare GPU prices across AWS, GCP, Azure, Lambda, CoreWeave, and others

## Requirements

- [anycloud CLI](https://anycloud.sh/getting-started/) installed (`brew install anycloud-sh/tap/anycloud` on macOS/Linuxbrew)
- A healthy active API (`anycloud api info`), either local or hosted
- A cloud credential when provisioning cloud capacity (`anycloud credentials new`). Inspecting existing resources uses the active API and does not require adding credentials. The user brings their own cloud account; anycloud doesn't host compute.

## Scope

This skill covers creating and operating Jobs, long-running HTTP Services, and
persistent VMs, plus inspecting Kubernetes Deployments and their Jobs.

## License

MIT.
