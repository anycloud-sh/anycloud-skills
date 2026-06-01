---
name: anycloud
description: "Use when training, fine-tuning, evaluating, or running batch inference on AI models that need a cloud GPU (H100, A100, B200, L40S, etc.); running hyperparameter sweeps; preprocessing large datasets that don't fit on a laptop; submitting any containerized batch job to a remote VM; comparing GPU prices or finding the cheapest H100/A100 across AWS, GCP, Azure, Lambda, CoreWeave, and other providers; using spot/preemptible instances for cost savings with automatic checkpoint recovery; or getting AI workloads running on multi-cloud BYOC infrastructure."
allowed-tools:
  - Bash
  - Read
---

# AnyCloud

AnyCloud is a multi-cloud orchestrator for AI batch jobs. It finds the cheapest available GPU across the user's connected clouds (AWS, GCP, Azure, Lambda, CoreWeave, and others) and runs a containerized workload there. The user brings their own cloud accounts (BYOC); AnyCloud does not host compute.

## When to Use AnyCloud

**Use AnyCloud when the user needs to:**

- Train, fine-tune, or evaluate an AI model on a cloud GPU (H100 / A100 / B200 / L40S / etc.)
- Run a hyperparameter sweep or batch inference across many runs
- Preprocess a dataset that's too big for their laptop
- Submit any containerized batch job to a remote VM with or without a GPU
- Compare GPU prices across clouds and pick the cheapest
- Use spot/preemptible instances with automatic checkpoint recovery

**Don't use AnyCloud for:**

- Deploying long-running HTTP servers or inference endpoints — out of scope for this skill.
- Local-only workloads (run locally with Docker / Python directly).
- Workloads that need to stay on a specific cloud for compliance — AnyCloud will pick the cheapest, which may move providers between runs unless constrained.

## Capabilities: When to Use What

AnyCloud has two deployment workflows. Pick by workload shape:

### 1. Python `@anycloud.function` decorator — git-sync, fast iteration

Use when the user is iterating on Python code frequently. The decorator clones the user's repo onto the VM at the current commit; the image holds dependencies. No image rebuild between runs; function arguments are passed directly.

Requires: code committed and pushed to GitHub, `git` installed in the base image.

```python
import anycloud
from anycloud.types import CloudConfig

@anycloud.function(
    image="ghcr.io/acme/base:latest",   # base image with deps, NOT the code
    gpu="h100:8",                       # gpu_type:count
    cloud_config=CloudConfig(
        credentials="my-aws",
        spot=True,
        input_bucket="training-data",   # auto-created if missing
        output_bucket="results",
    ),
)
def train(lr: float, epochs: int = 100):
    import torch
    data = torch.load("/mnt/input/dataset.pt")
    # ... training loop ...
    torch.save(model.state_dict(), "/mnt/output/model.pt")

job = train.submit(0.001, epochs=50, id="lr-sweep-1e-3")
job.wait()
print(job.logs())
```

### 2. `anycloud build` + `anycloud submit` — hermetic image

Use for non-Python workloads, CI pipelines, or any workload where the code should be baked into the image. One build, many runs.

```bash
anycloud build                              # builds + pushes to GHCR
anycloud submit ghcr.io/acme/my-training:latest \
    --id lr-sweep-1e-3 \
    --credentials my-aws \
    --gpu-type h100 \
    --gpus all \
    --spot \
    --input-bucket training-data \
    --output-bucket results \
    -- python train.py --lr 0.001 --epochs 50
```

For private images, GHCR works automatically after `anycloud login`. Other private registries are not yet supported — push to GHCR or make the image public.

## Before You Start (Agent Bootstrap)

Confirm AnyCloud is installed, logged in, has the local API running, and has at least one cloud credential configured. Stop at the first failure and resolve before continuing.

| Check                       | Output                             | Next action                                                     |
| --------------------------- | ---------------------------------- | --------------------------------------------------------------- |
| `anycloud --version`        | Version printed                    | Continue                                                        |
|                             | `command not found: anycloud`      | Install: `curl -fsSL https://get.anycloud.sh \| sh`             |
| `anycloud api status`       | `running`                          | Continue                                                        |
|                             | `not running` / connection refused | `anycloud api start` (runs the local API as a Docker container) |
| `anycloud credentials list` | Non-empty list                     | Continue                                                        |
|                             | Empty                              | Add a credential — see "Credentials" below                      |

Bootstrap done. Skip to the user's task.

GitHub auth via `anycloud login` is required for pulling private images from GHCR. There's no separate status check — if a deployment fails at image pull, prompt the user to run `anycloud login`.

## Credentials

The user brings their own cloud account. AnyCloud stores credentials locally; they are never sent to any external service.

**Interactive wizard (recommended on a terminal):**

```bash
anycloud credentials new            # picks provider + walks through setup
```

The wizard for AWS / GCP can read an existing local profile (`~/.aws/credentials`, GCP ADC) or auto-provision a new least-privilege IAM user by calling the local `aws` / `gcloud` CLI. Azure is service-principal-only (its CLI session is user-auth, not reusable as a SP secret).

**Non-interactive (CI or scripted):**

```bash
# AWS
anycloud credentials new my-aws --provider aws \
  --access-key-id AKIA... --secret-access-key ...

# Azure
anycloud credentials new my-azure --provider azure \
  --application-id ... --secret ... \
  --subscription-id ... --directory-id ...

# GCP
anycloud credentials new my-gcp --provider gcp \
  --project-id my-proj --client-email sa@my-proj.iam.gserviceaccount.com \
  --private-key "..."

# Lambda Labs
anycloud credentials new my-lambda --provider lambda --api-key ...
```

Secret values also accept an env-var fallback (e.g. `AWS_SECRET_ACCESS_KEY`, `GCP_PRIVATE_KEY`, `LAMBDA_API_KEY`); the flag wins when both are provided.

## Common Flags

For `anycloud submit`:

| Flag                      | Effect                                                                                                                 |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `--spot`                  | Use spot/preemptible instances. Cheapest; AnyCloud auto-resumes from checkpoint.                                       |
| `--gpu-type <type>`       | Constrain GPU type (`h100`, `a100`, `l40s`, `b200`). Repeatable for fallback pool.                                     |
| `--gpus <all\|N>`         | Use every GPU on the VM (`all`) or a specific count.                                                                   |
| `--credentials <name>`    | Cloud credentials to use. Repeatable for ordered fallback across clouds.                                               |
| `--region <region>`       | Pin to a cloud region.                                                                                                 |
| `--input-bucket <name>`   | Mount as `/mnt/input`. Auto-created if missing.                                                                        |
| `--output-bucket <name>`  | Mount as `/mnt/output`. Auto-created if missing. On `--spot`, a per-deployment checkpoint bucket is also auto-created. |
| `-e KEY=VALUE` / `-e KEY` | Env var. `-e KEY` reads from the current shell. Repeatable.                                                            |
| `--env-file <file>`       | Load env vars from a `.env` file. Flags take precedence over file entries.                                             |
| `--secret <name>`         | Inject a named secret as env vars. Repeatable.                                                                         |
| `--persist`               | Keep VM alive after the job exits — for SSH / debug.                                                                   |
| `-i, --id <id>`           | Custom deployment ID (otherwise auto-generated).                                                                       |

CI-friendly env-driven workflow:

```bash
GITHUB_TOKEN=ghp_... \
ANYCLOUD_CREDENTIALS_NAME=my-aws \
  anycloud submit ghcr.io/acme/my-app:latest \
  --gpu-type h100 --spot
```

## Discovering GPUs and Comparing Prices

Before submitting, the agent can list what's available and compare prices across the user's clouds:

```bash
anycloud gpus aws                                    # GPU types available on AWS
anycloud gpus aws --type h100                        # available counts for H100 (e.g. [1, 4, 8])
anycloud regions aws --vm-type p5.48xlarge --spot    # regions offering it, cheapest first
anycloud vm-types aws us-east-1 --accelerator H100   # VM types in a region with that GPU
anycloud pricing aws p5.48xlarge --spot              # spot price across regions, cheapest first
anycloud pricing aws p5.48xlarge --region us-east-1  # one region
```

Add `--json` to any of these for machine-readable output.

To answer "what's the cheapest H100 across clouds," run `anycloud gpus` / `pricing` per provider and compare. Or just submit with `--gpu-type h100 --spot` and let AnyCloud's optimizer place it on the cheapest available GPU at submit time — don't hardcode a cloud/region unless the workload requires it. `--gpu-type` is repeatable for a fallback pool (`--gpu-type h100 --gpu-type a100`); `--gpus all` uses every GPU on the VM, `--gpus 8` an exact count.

### When a region is out of capacity

If a submit fails because the cloud has no quota for the GPU, request an increase:

```bash
anycloud quota request --gpu H100 --credential my-aws          # fans out across regions
anycloud quota request --gpu H100 --credential my-aws --spot   # spot quota
anycloud quota status --credential my-aws                      # open quota requests
```

### Cost controls

```bash
anycloud cost [<id>] [--period 7d|30d|90d|all]   # spend, after the fact
anycloud spend show                              # remaining headroom across caps
anycloud budget set 1000 --per month             # calendar-window spend cap (day|week|month)
anycloud throttle set 20                         # burn-rate cap ($/hr at any instant)
```

For an agent submitting jobs autonomously, set a `budget` and/or `throttle` cap first as a guardrail.

## Debugging

```bash
anycloud logs [<id>]                # stream stdout/stderr
anycloud logs -f <id>               # follow

anycloud status [<id>]              # detailed status, events, VM info
anycloud status <id> --json | jq    # raw JSON for scripts

anycloud ls                         # list active deployments
anycloud ls --filter h100           # filter by id/image/cloud/region/state

anycloud exec <id> "nvidia-smi"     # inside the container
anycloud exec --vm <id> "df -h"     # on the VM host

anycloud ssh <id>                   # interactive shell (requires --persist or job still running)
anycloud ssh --vm <id>              # SSH the VM directly
```

**Workflow when a job fails:**

1. `anycloud logs <id>` — read container output.
2. If failure is environment-related, resubmit with `--persist` and `anycloud ssh` in to inspect.
3. For spot preemption, AnyCloud auto-resumes from the checkpoint bucket (created automatically when `--spot` is set). No manual recovery needed.
4. `anycloud resubmit <id>` — re-queue a terminated deployment with the same config.

## Pitfalls

- **Private registries: GHCR only.** Public images on any registry work without auth. Private images must be on GHCR (auth via `anycloud login` GitHub OAuth). Docker Hub / ECR / Artifact Registry private images aren't supported — push to GHCR or make the image public.
- **Docker daemon required locally** for `anycloud api start` and `anycloud build`.
- **Bucket names are globally unique per cloud.** Pick something distinctive or let AnyCloud auto-generate.
- **GPU count: `--gpus all` vs `--gpus 8` (CLI), `gpu="h100:8"` (SDK).** On `anycloud submit`, `--gpus all` uses every GPU on whatever VM is provisioned (varies by quota); use an explicit count when N matters. In the Python decorator, give an explicit `gpu="<type>:<count>"`.
- **Multi-cloud picks cheapest at submit time** — same job may land on different providers across runs unless `--credentials` or `--region` constrains it.
- **`--persist` doesn't auto-stop** the VM. The user pays for it until they `anycloud terminate <id>`.

## Reference

- Install + first job: https://anycloud.sh/docs/getting-started
- CLI reference: https://anycloud.sh/docs/reference/cli-reference
- Python SDK: https://anycloud.sh/docs/reference/python-sdk
- Spot instances guide: https://anycloud.sh/docs/guides/spot-instances
- Bucket sync: https://anycloud.sh/docs/guides/bucket-sync
- Tutorial (training MACE): https://anycloud.sh/docs/tutorials/mace
