# Lab 10 — Cloud Computing: Ship QuickNotes to a Real Cloud

## Environment

- Repository: `SanyaLikeIT/DevOps-Intro`
- Branch: `feature/lab10`
- Docker: `29.8.1`; Docker Compose: `v5.5.1`
- Hugging Face CLI: `2.0.0`; authenticated account: `Sane4ka01`
- Release tag: `v0.1.0` on commit `473d014514b8f4710679e7467ee64e7535c66101` in the feature branch
- GHCR image: `ghcr.io/sanyalikeit/devops-intro/quicknotes`

## Task 1 — GitHub Container Registry

### 1.1 Release workflow

The [release workflow](../.github/workflows/release.yml) runs on pushed `v*` tags, builds from `app/`, and publishes both the version tag and `latest`. It grants only `contents: read` and `packages: write`.

### 1.2 Action SHA pinning

| Action | Version | Commit SHA |
|---|---|---|
| `actions/checkout` | `v4.2.2` | `11bd71901bbe5b1630ceea73d27597364c9af683` |
| `docker/setup-buildx-action` | `v3.11.1` | `e468171a9de216ec08956ac3ada2f0791b6bd435` |
| `docker/login-action` | `v3.6.0` | `5e57cd118135c172c3672efd75eb46360885c0ef` |
| `docker/build-push-action` | `v6.18.0` | `263435318d21b8e681c14492fe198d362a7d2c83` |

The SHAs were resolved from the corresponding upstream Git tag refs.

### 1.3 Release run

- Signed tag verification: `Good "git" signature for billboard163rus@gmail.com with ED25519 key SHA256:Acrm9WJAVNML9xkwQ9n5pPrUp7EzdKyz/GG+jn8NSnw`.
- Tag target: `473d014514b8f4710679e7467ee64e7535c66101`.
- [Release run 36243218827](https://github.com/SanyaLikeIT/DevOps-Intro/actions/runs/36243218827): `completed`, `success`. Its publish job and build/push step succeeded.

### 1.4 Published image

- Immutable tag: `ghcr.io/sanyalikeit/devops-intro/quicknotes:v0.1.0`
- Moving tag: `ghcr.io/sanyalikeit/devops-intro/quicknotes:latest`
- Both pulls returned digest `sha256:551189c47afcd0d5c5e2221ce16e178999bcf7eec8c0584f40450ea72372bdad`.

### 1.5 Anonymous clean pull

An empty temporary `DOCKER_CONFIG` was created and checked for files. With that config, `docker pull ghcr.io/sanyalikeit/devops-intro/quicknotes:v0.1.0` succeeded and returned the same digest. The temporary config was removed afterward.

### 1.6 Design questions

#### a) OIDC vs GITHUB_TOKEN

Publishing to GHCR from this repository only needs the job's short-lived `GITHUB_TOKEN` with `packages: write`; GHCR does not require OIDC here. OIDC is useful for access to an external cloud or resource: a configured trust relationship exchanges the workflow identity for short-lived provider credentials, avoiding a stored, long-lived cloud secret.

#### b) latest vs immutable version

The `v0.1.0` tag identifies a specific release for traceability, reproducible deployment, and rollback. `latest` is a convenient moving pointer to the current release. Production deployments should pin the version or, more strongly, the digest so a later publication cannot silently change the deployed artifact.

#### c) Minimal permissions

The release job needs `contents: read` to check out source and `packages: write` to publish to GHCR. `write-all` would grant unrelated powers. Narrow permissions reduce the damage possible if a workflow step or dependency is compromised.

## Task 2 — Hugging Face Spaces

### 2.1 Space artifacts

The Space [Dockerfile](../cloud/hf-space/Dockerfile) pulls the immutable `v0.1.0` GHCR image; it does not rebuild source. The Space [README](../cloud/hf-space/README.md) sets `sdk: docker` and `app_port: 8080`.

A local build of this Space Dockerfile succeeded. Running that wrapper image on a temporary loopback port returned HTTP 200 for `/health` (`{"notes":4,"status":"ok"}`) and HTTP 200 for `/notes`. This validates the artifact locally; it is not evidence of a public Space.

### 2.2 Deployment status

Creating `Sane4ka01/quicknotes-lab10` as a public Docker Space with `cpu-basic` returned HTTP 402. Hugging Face stated that hosting Docker Spaces on free `cpu-basic` requires a PRO subscription for this account. No Space was created; public endpoint, warm latency, and cold-start evidence remain pending.

### 2.3 Design questions

#### d) Sleep vs scale to zero

Both models stop compute after inactivity and must start a workload for a later request. A free shared Space may wake more slowly than a request-focused serverless platform because scheduling, image transfer, and container startup can contribute to the response time. The size of each contribution needs measurement; the present account restriction prevents a timing claim.

#### e) Why `app_port: 8080`?

Docker Spaces normally expose port 7860, while QuickNotes listens on port 8080. The README metadata routes public traffic to the application's actual container port. Changing QuickNotes to 7860 solely for this deployment would be unnecessary.

#### f) Pulling the GHCR release vs building in the Space

Pulling the immutable GHCR release uses the same artifact produced by the GitHub release, gives a clear deployment identity, and avoids duplicate compilation in the Space. A cached image can also simplify deployment. It depends on GHCR availability and public access, and image transfer may add startup time; build debugging remains in the upstream workflow. Rebuilding source in the Space would produce a separate artifact and duplicate build logic.
