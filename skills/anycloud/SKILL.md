---
name: anycloud
description: "Use when training, fine-tuning, evaluating, or running batch inference on AI models that need a cloud GPU (H100, A100, B200, L40S, etc.); running hyperparameter sweeps; preprocessing large datasets that don't fit on a laptop; submitting any containerized batch job to a remote VM; comparing GPU prices or finding the cheapest H100/A100 across AWS, GCP, Azure, Lambda, CoreWeave, and other providers; using spot/preemptible instances for cost savings with automatic checkpoint recovery; getting AI workloads running on multi-cloud BYOC infrastructure; or monitoring, debugging, and inspecting AnyCloud jobs already submitted — checking deployment status and logs, tracking spend, or querying deployment state and events directly with read-only SQL when no dedicated command exposes what you need."
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
- Monitor, debug, or inspect deployments already submitted — status, logs, spend, events, or ad-hoc read-only SQL queries on deployment state

**Don't use AnyCloud for:**

- Deploying long-running HTTP servers or inference endpoints — out of scope for this skill.
- Local-only workloads (run locally with Docker / Python directly).
- Workloads that need to stay on a specific cloud for compliance — AnyCloud will pick the cheapest, which may move providers between runs unless constrained.

## Capabilities: When to Use What

Two ways to get code onto the VM — pick by whether you need a custom image:

- **No build:** the `@anycloud.function` decorator git-syncs your code onto a stock public image (e.g. `pytorch/pytorch:*cuda*`) at run time — use when an off-the-shelf image already has your deps + `git`. (Workflow 1 below.)
- **Build your own image:** bake code + deps into a hermetic image, push to GHCR, then `anycloud submit` — for non-Python, CI, or when no public image fits. (Workflow 2 + "Building and pushing your image" below.)

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
        input_bucket="training-data",   # read-only; create + upload before run
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

### 2. Bring your own image + `anycloud submit` — hermetic image

Use for non-Python workloads, CI pipelines, or any workload where the code should be baked into the image. Build and push the image yourself (laptop or CI), then submit the reference. One build, many runs.

```bash
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

`anycloud login` logs your local Docker CLI into GHCR, so private GHCR images pull automatically. Add `--bake` when you'll run the same image digest repeatedly. The first run pulls and snapshots a baked VM image; subsequent runs reuse it only once that bake has finished — a still-baking image is invisible, so a sweep fired all at once won't share it. Warm the cache with one run, then fan out. (Submits reuse an _available_ baked image automatically; `--bake` only creates one.) Clean up with `anycloud images prune`.

## Building and pushing your image

Building and pushing is plain Docker — the only AnyCloud command in this step is `anycloud login`, which logs your local Docker CLI into GHCR so pushes (and later private pulls) just work. AnyCloud runs a prebuilt image; it does not build one for you.

**Build only for deps no off-the-shelf image provides** — otherwise run a public image directly or git-sync via the decorator (the two paths above).

A minimal `Dockerfile`:

```dockerfile
FROM python:3.11                      # for GPU, start FROM nvidia/cuda:* or pytorch/pytorch:*cuda*
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "train.py"]
```

Log in once, then build and push to GHCR:

```bash
anycloud login                        # logs the local Docker CLI into GHCR (skips if Docker absent)
docker buildx build \
    --platform linux/amd64 \
    -t ghcr.io/<your-gh-username>/my-training:latest \
    --push .
```

Then submit it with the `anycloud submit` flags shown above.

- **`--platform linux/amd64` is mandatory.** A plain `docker build` on an Apple Silicon Mac may publish an `arm64` image that pulls fine but can't run on the VM.
- **GPU images must be Linux-tested.** Start `FROM nvidia/cuda:*`, `pytorch/pytorch:*cuda*`, or an NVIDIA image you've run on Linux — a Mac build won't validate GPU access.
- **Push rejected (`denied` / `401`)?** Re-run `anycloud login` — the stored Docker credential is only as fresh as your GitHub token, which Docker never refreshes on its own.
- **GHCR image names must be lowercase.**
- **For repeatable, commit-pinned builds, use CI.** GitHub Actions gives a `packages: write` `GITHUB_TOKEN` (no `anycloud login` needed). Full workflow: https://anycloud.sh/concepts/docker#build-in-ci-github-actions

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
# AWS — other providers: --provider azure|gcp|lambda (see `anycloud credentials new --help`)
anycloud credentials new my-aws --provider aws \
  --access-key-id AKIA... --secret-access-key ...
```

Secret values also accept an env-var fallback (e.g. `AWS_SECRET_ACCESS_KEY`, `GCP_PRIVATE_KEY`, `LAMBDA_API_KEY`); the flag wins when both are provided.

## Secrets

Create a named secret bundle first, then inject it with `--secret <name>` (values are write-only — never returned, unlike `-e`):

```bash
anycloud secrets new hf HF_TOKEN=hf_xxx     # create (repeatable KEY=VALUE)
anycloud secrets list                       # names only, no values
anycloud submit ghcr.io/acme/app:latest --secret hf -- python train.py
```

## Common Flags

For `anycloud submit`:

| Flag                      | Effect                                                                                                                 |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `--spot`                  | Use spot/preemptible instances. Cheapest; restores `/mnt/checkpoint` on preemption (your code must write it).          |
| `--gpu-type <type>`       | Constrain GPU type (`h100`, `a100`, `l40s`, `b200`). Repeatable for fallback pool.                                     |
| `--gpus <all\|N>`         | Use every GPU on the VM (`all`) or a specific count.                                                                   |
| `--shm-size <size>`       | Shared memory (e.g. `8g`). Bump for PyTorch DataLoader / NCCL, else multi-GPU can hang.                                |
| `--credentials <name>`    | Cloud credentials to use. Repeatable for an ordered fallback list.                                                     |
| `--region <region>`       | Pin to a cloud region.                                                                                                 |
| `--input-bucket <name>`   | Read-only mount at `/mnt/input`. **Must exist + be populated before submit** — see Moving data.                        |
| `--output-bucket <name>`  | Mount as `/mnt/output`. Auto-created if missing. On `--spot`, a per-deployment checkpoint bucket is also auto-created. |
| `-e KEY=VALUE` / `-e KEY` | Env var. `-e KEY` reads from the current shell. Repeatable.                                                            |
| `--env-file <file>`       | Load env vars from a `.env` file. Flags take precedence over file entries.                                             |
| `--secret <name>`         | Inject a named secret as env vars (create with `anycloud secrets new`). Repeatable.                                    |
| `--persist`               | Keep VM alive after the job exits — for exec / debug.                                                                  |
| `--bake`                  | Snapshot a baked VM image after the pull so later same-digest, same-region runs skip it. Pin `--region`.               |
| `-i, --id <id>`           | Custom deployment ID (otherwise auto-generated).                                                                       |

Other Docker-runtime / targeting flags: `--memory`, `--cpus`, `--ipc`, `--runtime`, `--disk-size`, `--vm-type` (repeatable, explicit instance types), `--zone`, `--persist-bucket` — see the CLI reference.

CI-friendly env-driven workflow:

```bash
GITHUB_TOKEN=ghp_... \
ANYCLOUD_CREDENTIALS_NAME=my-aws \
  anycloud submit ghcr.io/acme/my-app:latest \
  --gpu-type h100 --spot
```

## Moving data (buckets)

Three mount points, synced automatically — request them with `--input-bucket` / `--output-bucket`:

- `/mnt/input` — **read-only, and the bucket must exist + be populated before you submit.** Create and fill it first: `anycloud bucket create <name> --credentials <cred>`, then `anycloud bucket upload <name> <local> <remote> --credentials <cred>`.
- `/mnt/output` — read-write, **auto-created**; uploads to the cloud every ~60s. Fetch results after with `anycloud bucket download <name> <remote> <local> --credentials <cred>`.
- `/mnt/checkpoint` — auto-created per deployment on `--spot`; downloaded on startup, uploaded ~60s. **Your code must read it on startup and write to it** to actually resume after preemption.

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

## Cost & spend controls

Two independent caps gate **new dispatches** — neither kills running jobs:

- **throttle** — live burn-rate cap (`$/hr` summed across running VMs). Clears as VMs finish.
- **budget** — calendar-window total-spend cap (day / week / month). Clears at the UTC window rollover.

```bash
anycloud throttle set 20                          # $20/hr at any instant
anycloud budget set 1000 --per month              # window: day | week | month
anycloud budget set 50 --per day --agent-session  # scope a cap to THIS agent run only
anycloud spend show                               # remaining headroom across all caps
anycloud cost [<id>] [--period 7d|30d|90d|all]    # actual spend, after the fact
```

**A hit cap doesn't fail `submit`** — it returns an id, but the deployment stays `Queued` with a `blocked by throttle|budget …` reason in `anycloud status` / `ls`, then dispatches automatically once the cap clears (a VM ends, the window rolls over, or you raise the cap). Don't mistake a spend-blocked job for a stuck one — check `status`.

**Scopes:** account-wide (default — counts human submits too) or `--agent-session` (only the current agent run). For an agent submitting autonomously, set an `--agent-session` `budget` and/or `throttle` cap first as your guardrail.

## Debugging

```bash
anycloud status [<id>]              # status, events, VM info, error details
anycloud status <id> --verbose      # include detailed logs
anycloud status <id> --json | jq    # raw JSON for scripts

anycloud ls                         # list active deployments
anycloud ls --status failed         # filter by exact state

anycloud exec <id> "nvidia-smi"     # run a command in the job execution environment
anycloud exec <id> "tail -n 100 train.log"
```

**Workflow when a job fails:**

1. `anycloud status <id> --verbose` — read events, error details, and logs.
2. If environment-related, resubmit with `--persist` and `anycloud exec <id> "<command>"` to inspect the live environment.
3. For spot preemption, AnyCloud re-provisions and restores `/mnt/checkpoint` automatically — but it only resumes work if your code reads/writes checkpoints there (see Moving data); otherwise it restarts from scratch.
4. `anycloud resubmit <id>` — re-queue a terminated deployment with the same config.
5. Need a detail `status` / `ls` don't surface (events, timing, cross-deployment aggregates)? Query it read-only with `anycloud db query` (see below).

## Inspecting state directly (read-only SQL)

When no dedicated command exposes the field or aggregate you need, query the local API database directly — this is the agent escape-hatch that goes beyond `status` / `ls`. It is read-only (writes are refused at the SQLite engine level), so exploring is safe. Discover the structure first, then query:

```bash
anycloud db schema --json                    # tables, columns, foreign keys, indexes (use this first)
anycloud db schema deployments               # narrow to one table
anycloud db query "SELECT id, state FROM deployments ORDER BY started_at DESC LIMIT 10"
anycloud db query "SELECT * FROM deployment_events WHERE deployment_id = '<id>'" --json
```

Only `SELECT` / `WITH` / `EXPLAIN` / `PRAGMA` run; results cap at 10,000 rows (`--json` sets `truncated: true` when the cap fires — add `LIMIT`). Don't hardcode columns — run `anycloud db schema --json` to introspect, since the schema can change between releases. Mutate state with the regular commands (`submit` / `terminate` / `resubmit`), never SQL.

## Pitfalls

- **Private registries: GHCR only.** Public images on any registry work without auth. Private images must be on GHCR (auth via `anycloud login` GitHub OAuth). Docker Hub / ECR / Artifact Registry private images aren't supported — push to GHCR or make the image public.
- **Docker daemon required locally** for `anycloud api start` and for building/pushing your own image. Image validation runs server-side, so submitting a prebuilt image doesn't need local Docker.
- **Bucket names are globally unique per cloud.** Pick something distinctive or let AnyCloud auto-generate.
- **GPU count: `--gpus all` vs `--gpus 8` (CLI), `gpu="h100:8"` (SDK).** On `anycloud submit`, `--gpus all` uses every GPU on whatever VM is provisioned (varies by quota); use an explicit count when N matters. In the Python decorator, give an explicit `gpu="<type>:<count>"`.
- **Multi-cloud picks cheapest at submit time** — same job may land on different providers across runs unless `--credentials` or `--region` constrains it.
- **`--persist` doesn't auto-stop** the VM. The user pays for it until they `anycloud terminate <id>`.
- **Agent runs are session-scoped.** Invoked non-interactively (as you are), `anycloud ls` / `status` list only the current agent session's deployments — an empty list doesn't mean no jobs exist. Pass `--session <id>` or `--agent <name>` to widen.

## Reference

- Install + first job: https://anycloud.sh/getting-started
- CLI reference: https://anycloud.sh/reference/cli-reference
- Python SDK: https://anycloud.sh/reference/python-sdk
- Build & push images (Docker/GHCR): https://anycloud.sh/concepts/docker
- Spot instances guide: https://anycloud.sh/guides/spot-instances
- Spend controls (budget/throttle): https://anycloud.sh/guides/spend-controls
- Bucket sync: https://anycloud.sh/guides/bucket-sync
- Tutorial (training MACE): https://anycloud.sh/tutorials/mace
