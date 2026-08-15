# anycloud-skills

Claude Code / Codex CLI / ChatGPT [Agent Skills](https://code.claude.com/docs/en/skills) for running AI workloads on the cheapest available cloud GPU via [anycloud](https://anycloud.sh).

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
- Use spot instances with automatic checkpoint recovery
- Compare GPU prices across AWS, GCP, Azure, Lambda, CoreWeave, and others

## Requirements

- [anycloud CLI](https://anycloud.sh/getting-started/) installed (`brew install anycloud-sh/tap/anycloud` on macOS/Linuxbrew)
- A cloud credential added (`anycloud credentials new`). The user brings their own AWS / GCP / Azure / Lambda account; anycloud doesn't host compute.

## Scope

This skill is for **AI batch workloads** — training, fine-tuning, evals, batch inference, dataset preprocessing. It does not cover long-running HTTP server deployments.

## License

MIT.
