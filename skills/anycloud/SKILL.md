---
name: anycloud
description: 'Use when training, fine-tuning, evaluating, or running batch inference on cloud GPUs; running sweeps or large preprocessing jobs; creating a persistent cloud VM with container and host access; deploying a long-running Anycloud Service; comparing GPU prices across AWS, GCP, Azure, Lambda, and other providers; using spot instances with checkpoint recovery; or monitoring, debugging, operating, and tracking spend for Anycloud Jobs, Services, and VMs.'
allowed-tools:
  - Bash
  - Read
---

# AnyCloud

AnyCloud is a multi-cloud orchestrator for containerized Jobs, Services, and
persistent VMs. It finds suitable compute across the user's
connected clouds (AWS, GCP, Azure, Lambda, and others) and provisions it in the
user's own account (BYOC); AnyCloud does not host compute.

## When to Use AnyCloud

**Use AnyCloud when the user needs to:**

- Train, fine-tune, or evaluate an AI model on a cloud GPU (H100 / A100 / B200 / L40S / etc.)
- Run a hyperparameter sweep or batch inference across many runs
- Preprocess a dataset that's too big for their laptop
- Submit any containerized batch job to a remote VM with or without a GPU
- Create a persistent VM for interactive Docker, Git, and shell work
- Compare GPU prices across clouds and pick the cheapest
- Use spot/preemptible instances with automatic checkpoint recovery
- Monitor, debug, or inspect deployments already submitted — status, logs, spend, events, or ad-hoc read-only SQL queries on deployment state

**Don't use AnyCloud for:**

- Using a VM as a managed production endpoint — use `anycloud service` for a long-running HTTP Service, or `anycloud api serve` for a hosted Anycloud control plane.
- Local-only workloads (run locally with Docker / Python directly).
- Workloads that need to stay on a specific cloud for compliance — AnyCloud will pick the cheapest, which may move providers between runs unless constrained.

## Capabilities: When to Use What

Choose the workload shape first. Jobs and Services run published container
images containing the application code and dependencies.

### 1. Job — finite container workload

Use for training, evaluation, preprocessing, batch inference, and other work
that should exit. Build and push the image yourself, then run the reference:

```bash
anycloud job ghcr.io/acme/my-training:latest \
    --id lr-sweep-1e-3 \
    --credentials my-aws \
    --gpu-type h100 \
    --gpus all \
    --spot \
    --input-bucket training-data \
    --output-bucket results \
    -- python train.py --lr 0.001 --epochs 50
```

`anycloud login` logs your local Docker CLI into GHCR, so private GHCR images pull automatically. On VM-backed providers, deployments start from the provider base image and pull the requested image reference; some retry paths can reuse an existing VM.

The equivalent Python SDK submission is:

```python
import anycloud

ac = anycloud.Client()
job = ac.submit(
    "ghcr.io/acme/my-training:latest",
    gpu="h100:8",
    cloud_config=anycloud.CloudConfig(
        credentials="my-aws",
        spot=True,
        input_bucket="training-data",
        output_bucket="results",
    ),
    command=["python", "train.py", "--lr", "0.001", "--epochs", "50"],
    deployment_id="lr-sweep-1e-3",
)
job.wait()
print(job.logs())
```

### 2. Service — long-running HTTP workload

Use when the process should listen on `PORT` and remain available at a stable
URL:

```bash
anycloud service ghcr.io/acme/model-api:latest \
    --id model-api \
    --credentials my-aws \
    --gpu-type l40s \
    -- python -m myapp
```

### 3. Persistent VM — interactive Docker environment

Use when the user wants a long-lived interactive machine rather than a finite
Job or a managed HTTP Service:

```bash
anycloud vm ghcr.io/acme/dev:latest --credentials my-aws --vm-type t3.large
anycloud ssh <id>             # VM container
anycloud ssh <id> --host      # underlying host
```

The image is required; Anycloud does not provide a default. It must provide
`/bin/sh` and POSIX `sleep` and tolerate entrypoint, command, and working-
directory replacement. The image supplies all optional development tooling,
including a Docker CLI when the user wants to control host Docker. VMs are
SSH-only and do not publish a managed HTTP endpoint; use a Service when a stable
public URL is required.

Treat the host as the persistence and trust boundary. `/workspace` survives VM
container recreation and host reboot, but `terminate` destroys it with the VM
disk. The image's configured user and `HOME` are preserved. The
mounted rootful Docker socket gives the container host-level control, so this
is not a security sandbox. A stopped container stays stopped; recover it with
`ssh --host` and `docker start <id>`. VMs do not support spot, buckets, startup
commands, arbitrary Docker flags, resubmit, snapshots, or automatic host
replacement.

## Building and pushing your image

Building and pushing is plain Docker — the only AnyCloud command in this step is `anycloud login`, which logs your local Docker CLI into GHCR so pushes (and later private pulls) just work. AnyCloud runs a prebuilt image; it does not build one for you.

Use a public image directly only when its existing contents plus the submitted
command are the complete workload. Otherwise build application code and
dependencies into a custom image.

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

Then run it with the `anycloud job` flags shown above.

- **`--platform linux/amd64` is mandatory.** A plain `docker build` on an Apple Silicon Mac may publish an `arm64` image that pulls fine but can't run on the VM.
- **GPU images must be Linux-tested.** Start `FROM nvidia/cuda:*`, `pytorch/pytorch:*cuda*`, or an NVIDIA image you've run on Linux — a Mac build won't validate GPU access.
- **Push rejected (`denied` / `401`)?** Re-run `anycloud login` — the stored Docker credential is only as fresh as your GitHub token, which Docker never refreshes on its own.
- **GHCR image names must be lowercase.**
- **For repeatable, commit-pinned builds, use CI.** GitHub Actions gives a `packages: write` `GITHUB_TOKEN` (no `anycloud login` needed). Full workflow: https://anycloud.sh/platform/container-images#build-and-publish

## Before You Start (Agent Bootstrap)

Confirm AnyCloud is installed, logged in, has the local API running, and has at least one cloud credential configured. Stop at the first failure and resolve before continuing.

| Check                       | Output                             | Next action                                                                                                           |
| --------------------------- | ---------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `anycloud --version`        | Version printed                    | Continue                                                                                                              |
|                             | `command not found: anycloud`      | Install with Homebrew: `brew install anycloud-sh/tap/anycloud`; otherwise follow https://anycloud.sh/getting-started/ |
| `anycloud job --help`       | Help printed                       | Continue                                                                                                              |
|                             | `unknown command`                  | `anycloud update` — this CLI predates the `job` / `service` names; older releases spell them `submit` / `serve`       |
| `anycloud api status`       | `running`                          | Continue                                                                                                              |
|                             | `not running` / connection refused | `anycloud api start` (runs the local API as a Docker container)                                                       |
| `anycloud credentials list` | Non-empty list                     | Continue                                                                                                              |
|                             | Empty                              | Add a credential — see "Credentials" below                                                                            |

Bootstrap done. Skip to the user's task.

GitHub auth via `anycloud login` is required for pulling private images from GHCR. There's no separate status check — if a deployment fails at image pull, prompt the user to run `anycloud login`.

## Credentials

The user brings their own cloud account. AnyCloud stores credentials locally; they are never sent to any external service.

**Interactive wizard (recommended on a terminal):**

```bash
anycloud credentials new            # picks provider + walks through setup
```

The wizard can import durable static AWS access-key profiles and GCP service-account key JSON, accept existing provider values, or explicitly create an Azure service principal with the logged-in Azure CLI. AWS SSO, role, session-token, and other indirect profiles cannot be imported, GCP user ADC is not durable service-account material, and AWS/GCP `--generate` is unsupported.

**Non-interactive (CI or scripted):**

```bash
# Populate the documented provider variables from the CI secret store first.
anycloud credentials new my-aws --provider aws
```

Secret values accept provider-specific environment-variable fallbacks; see
`anycloud credentials new --help` for the names. Never ask the user to paste
credential values into chat or emit them in an agent-generated command.

## Secrets

Secret values are write-only and never returned. Do not ask the user to paste
them into chat or put them in an agent-generated command. Ask the user to create
a named bundle privately in their terminal using `anycloud secrets new --help`,
then inject only the bundle name:

```bash
anycloud secrets list                       # names only, no values
anycloud job ghcr.io/acme/app:latest --secret hf -- python train.py
```

## Common Flags

For `anycloud job`:

| Flag                      | Effect                                                                                                                 |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `--spot`                  | Use spot/preemptible instances. Cheapest; restores `/mnt/checkpoint` on preemption (your code must write it).          |
| `--gpu-type <type>`       | Constrain GPU type (`h100`, `a100`, `l40s`, `b200`). Repeatable for fallback pool.                                     |
| `--gpus <all\|N>`         | Use every GPU on the VM (`all`) or a specific count.                                                                   |
| `--shm-size <size>`       | Shared memory (e.g. `8g`). Bump for PyTorch DataLoader / NCCL, else multi-GPU can hang.                                |
| `--credentials <name>`    | Cloud credentials to use. Repeatable for an ordered fallback list.                                                     |
| `--region <region>`       | Pin to a cloud region.                                                                                                 |
| `--input-bucket <name>`   | Read-only mount at `/mnt/input`. **Must exist + be populated before the Job starts** — see Moving data.                |
| `--output-bucket <name>`  | Mount as `/mnt/output`. Auto-created if missing. On `--spot`, a per-deployment checkpoint bucket is also auto-created. |
| `-e KEY=VALUE` / `-e KEY` | Env var. `-e KEY` reads from the current shell. Repeatable.                                                            |
| `--env-file <file>`       | Load env vars from a `.env` file. Flags take precedence over file entries.                                             |
| `--secret <name>`         | Inject a named secret as env vars (create with `anycloud secrets new`). Repeatable.                                    |
| `-i, --id <id>`           | Custom deployment ID (otherwise auto-generated).                                                                       |

Other Docker-runtime / targeting flags: `--memory`, `--cpus`, `--ipc`, `--runtime`, `--disk-size`, `--vm-type` (repeatable, explicit instance types), `--zone` — see the CLI reference.

For CI, provide authentication through the CI platform's secret store before
running `anycloud`; never place token values in workflow commands or logs.

## Moving data (buckets)

Three mount points, synced automatically — request them with `--input-bucket` / `--output-bucket`:

- `/mnt/input` — **read-only, and the bucket must exist + be populated before the Job starts.** Create and fill it first: `anycloud bucket create <name> --credentials <cred>`, then `anycloud bucket upload <name> <local> <remote> --credentials <cred>`.
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

To answer "what's the cheapest H100 across clouds," run `anycloud gpus` / `pricing` per provider and compare. Or just run a Job with `--gpu-type h100 --spot` and let AnyCloud's optimizer place it on the cheapest available GPU at dispatch time — don't hardcode a cloud/region unless the workload requires it. `--gpu-type` is repeatable for a fallback pool (`--gpu-type h100 --gpu-type a100`); `--gpus all` uses every GPU on the VM, `--gpus 8` an exact count.

### When a region is out of capacity

If a Job fails because the cloud has no quota for the GPU, request an increase:

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
anycloud cost [<id>] [--period 1d..90d]           # Job + Service + VM spend
```

**A hit cap doesn't fail `job`** — it returns an id, but the deployment stays `Queued` with a `blocked by throttle|budget …` reason in `anycloud status` / `ls`, then dispatches automatically once the cap clears (a VM ends, the window rolls over, or you raise the cap). Don't mistake a spend-blocked job for a stuck one — check `status`.

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
2. If environment-related, `anycloud exec <id> "<command>"` while the job is still running to inspect the live environment.
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

Only `SELECT` / `WITH` / `EXPLAIN` / `PRAGMA` run; results cap at 10,000 rows (`--json` sets `truncated: true` when the cap fires — add `LIMIT`). Don't hardcode columns — run `anycloud db schema --json` to introspect, since the schema can change between releases. Mutate state with the regular commands (`job` / `terminate` / `resubmit`), never SQL.

## Pitfalls

- **Private registries: GHCR only.** Public images on any registry work without auth. Private images must be on GHCR (auth via `anycloud login` GitHub OAuth). Docker Hub / ECR / Artifact Registry private images aren't supported — push to GHCR or make the image public.
- **Docker daemon required locally** for `anycloud api start` and for building/pushing your own image. Image validation runs server-side, so submitting a prebuilt image doesn't need local Docker.
- **Bucket names are globally unique per cloud.** Pick something distinctive or let AnyCloud auto-generate.
- **GPU count: `--gpus all` vs `--gpus 8` (CLI), `gpu="h100:8"` (SDK).** On `anycloud job`, `--gpus all` uses every GPU on whatever VM is provisioned (varies by quota); use an explicit count when N matters. In the Python SDK, pass an explicit `gpu="<type>:<count>"` to `Client.submit()` or `Client.serve()`.
- **Multi-cloud picks cheapest at dispatch time** — the same Job may land on different providers across runs unless `--credentials` or `--region` constrains it.
- **A Job's VM is released when it finishes.** Inspect a running job with `anycloud exec` / `anycloud ssh` before it exits; afterwards, read `anycloud status <id> --verbose` and `anycloud logs <id>`.
- **Agent runs are session-scoped.** Invoked non-interactively (as you are), `anycloud ls` / `status` list only the current agent session's deployments — an empty list doesn't mean no jobs exist. Pass `--session <id>` or `--agent <name>` to widen.

## Reference

- Install + first job: https://anycloud.sh/getting-started
- CLI reference: https://anycloud.sh/reference/cli-reference
- VM reference: https://anycloud.sh/reference/cli/vm
- Python SDK: https://anycloud.sh/reference/python-sdk
- Build & push images (Docker/GHCR): https://anycloud.sh/platform/container-images
- Spot instances guide: https://anycloud.sh/platform/jobs#spot-recovery
- Spend controls (budget/throttle): https://anycloud.sh/platform/spend-controls
- Bucket sync: https://anycloud.sh/platform/buckets
- Tutorial (training MACE): https://anycloud.sh/examples/train-mace
