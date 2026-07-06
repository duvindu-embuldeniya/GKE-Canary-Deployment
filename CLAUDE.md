# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

An end-to-end demo of **replica-based canary deployment** of a Django app on GKE. It contains four independently-versioned layers that share one convention (`django-stable` / `django-canary` Deployments behind a single Service):

- `demo-application/` — the Django app being deployed
- `terraform/` — provisions the GKE cluster + VPC on GCP
- `k8s-manifests/` — raw Kubernetes YAML (dev namespace, hardcoded)
- `helm-charts/deployments/` — templated equivalent of the manifests, parameterized by `values.yaml`
- `.github/workflows/` — CI (build image), CD (canary rollout), and Terraform pipelines

## The canary mechanism (most important architecture)

Traffic splitting is done **purely by replica count**, not a service mesh or ingress weighting. Understanding this is essential before touching the CD workflow or manifests:

- Two Deployments — `django-stable` and `django-canary` — run the same app on different image tags. Both carry the label `app: django`.
- The Service (`django-svc`) selects `app: django` only (**not** the `track: stable`/`track: canary` label). So it load-balances across the combined pod set, and the traffic ratio equals the replica ratio.
- Total replicas are held at 5. [.github/workflows/cd.yaml](.github/workflows/cd.yaml) walks stable→canary in 20% steps (4/1 → 3/2 → 2/3 → 1/4 → 0/5) with a `sleep 60` between each, then **promotes** by setting `django-stable` to the new image, scaling it back to 5, and scaling `django-canary` to 0.
- Rollback is a separate job gated on `if: failure()`. It captures the stable image tag *before* the rollout (`save-image` step → job output) and restores stable to that image at 5 replicas, canary to 0.

When editing replica math or step counts, keep stable+canary summing to the intended total or the traffic percentages will be wrong.

## Health / probe endpoints

The app exposes four endpoints used by Kubernetes probes ([demo-application/home/views.py](demo-application/home/views.py), wired in [home/urls.py](demo-application/home/urls.py)):

- `/start/` — startupProbe; returns 200 unconditionally (just signals the process booted)
- `/ready/` — readinessProbe; 200 only if all DB connections open, else 503
- `/live/` — livenessProbe; same DB check as ready
- `/health/` — general healthcheck (used by docker-compose)

Probe paths and thresholds are duplicated in three places that must stay in sync: the k8s manifests, `helm-charts/deployments/values.yaml`, and the deployment templates. There is no app-side test suite beyond Django's default `tests.py`.

## Configuration & secrets flow

- The Django app is **entirely env-driven** ([simply/settings.py](demo-application/simply/settings.py)): `SECRET_KEY`, `DB_NAME`, `DB_USER`, `DB_PASS`, `DB_HOST`, `DB_PORT`. `DEBUG=False` and `ALLOWED_HOSTS=['*']` are hardcoded; DB engine is PostgreSQL (the committed `db.sqlite3` is unused legacy).
- In Kubernetes those env vars come from a single `postgres-secret` Secret; `DB_HOST` points at the headless Postgres service. Postgres itself runs as a StatefulSet in-cluster.
- `helm-charts/deployments/values.yaml` carries **plaintext demo secrets** (DB password, Django `SECRET_KEY`) — this is a demo repo, but do not treat these as safe to reuse.
- CI/CD get all GCP config (project, region, WIF provider, image repo) from GitHub Actions **secrets**, authenticated via **Workload Identity Federation (OIDC)**, not a service-account key. The Terraform workflow is the exception — it still writes a `key.json` from `GCP_SA_KEY` and deletes it afterward.

## Common commands

Local dev (spins up Postgres + the app with migrations + gunicorn):
```bash
docker compose up --build          # needs DB_USER/DB_PASS/DB_NAME/DB_HOST/DB_PORT/SECRET_KEY in env or .env
```

Django app (run from `demo-application/`):
```bash
python manage.py makemigrations
python manage.py migrate
python manage.py runserver          # dev server
python manage.py test               # run tests
python manage.py test home.tests.SomeTest.test_method   # single test
```

Build the image the way CI does:
```bash
docker build -t canary-app:latest ./demo-application
```

Helm (chart lives in `helm-charts/deployments/`):
```bash
helm template ./helm-charts/deployments        # render manifests locally
helm upgrade --install django ./helm-charts/deployments -n dev
```

Terraform (run from `terraform/`, requires all `TF_VAR_*` inputs):
```bash
terraform init && terraform plan && terraform apply
```

## Pipelines

All three workflows are currently **`workflow_dispatch` only** (manual). The original event triggers (`push`, `workflow_run` chaining CI after Terraform) are commented out at the top of each file — check there before assuming a workflow is automatic.

- **CI** (`ci.yaml`): builds `./demo-application`, tags with `github.run_number`, pushes to Artifact Registry.
- **CD** (`cd.yaml`): takes `image_tag` + `namespace` (dev/qa/uat/prod) inputs and runs the canary rollout described above.
- **Terraform** (`terraform.yaml`): init/plan/apply of the cluster.

## Terraform layout

`terraform/main.tf` composes two local modules: `modules/network` (VPC + subnet) and `modules/gke` (cluster + node pool), with `gke` depending on `network`'s outputs. Everything is variable-driven with no defaults ([variables.tf](terraform/variables.tf)) — the values arrive from `TF_VAR_*` env vars set by the workflow secrets, so there is no committed `.tfvars`.
