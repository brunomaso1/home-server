# home-server

![IaC](https://img.shields.io/badge/IaC-Ansible-EE0000?logo=ansible&logoColor=white) ![Containers](https://img.shields.io/badge/containers-Docker-2496ED?logo=docker&logoColor=white) ![Proxy](https://img.shields.io/badge/reverse--proxy-Traefik-24A1C1?logo=traefikproxy&logoColor=white) ![OS](https://img.shields.io/badge/OS-Ubuntu%20Server%2024.04-E95420?logo=ubuntu&logoColor=white)

Infrastructure and application stacks for a self-hosted home server. A physical
machine running Ubuntu Server 24 — with a TerraMaster D2-320 DAS attached — is
provisioned with Ansible (Infrastructure as Code) and runs applications as
Docker Compose stacks behind a Traefik reverse proxy.

> **Status:** Under construction. This README documents the target architecture;
> the directories below are populated incrementally.

## Architecture

The flow from bare host to running apps:

1. **Provision the host with Ansible** — install Docker and mount the DAS.
2. **Run applications as Docker Compose stacks** — one stack per app under `apps/`.
3. **Route traffic through Traefik** — a single reverse proxy is the entry point
   to every app.

```
Ansible (infra/)  ──provisions──▶  Ubuntu Server 24 host
                                    ├── Docker Engine
                                    └── TerraMaster D2-320 (mounted)
                                              │
                                    Docker Compose stacks (apps/)
                                              │
                                    Traefik ──▶ SeaweedFS, simple-page, ...
```

## Environments & workflow

Three environments, each mapped to a Git branch:

| Environment | Branch | Host | Purpose |
| --- | --- | --- | --- |
| Local | `dev` (via `feature/*`) | Your workstation | Development and local testing |
| Development | `dev` | Development VM | Integration testing before release |
| Production | `main` | Home server | Live deployment — the server runs what is on `main` |

### Branching strategy

A GitFlow-inspired model adapted for DevOps, with two permanent branches:

- **`main`** — production. Reflects exactly what runs on the server.
  A **protected branch**: no direct pushes — changes only arrive through a
  pull request from `dev`.
- **`dev`** — integration branch. Small changes may be committed here directly;
  larger features get their own `feature/<name>` branch first.

Flow from work to production:

```
feature/* ──▶ dev ──▶ main
  (work)   (integrate) (release to server)
```

- For a large feature, branch `feature/<name>` off `dev`, then open a pull
  request into `dev`.
- Small changes can go straight onto `dev`.
- Changes merged into `dev` are validated on the Development VM.
- To release, open a pull request from `dev` into `main`. Merging it triggers
  the deployment pipeline to the server.

> **Promotion to `main` is done through a pull request against the remote —
> never a direct push.**

### Branch protection

Branch protection is enforced on the Git platform (GitHub/GitLab), **not by Git
itself** — a local clone cannot block a push to `main`. A local `pre-push` hook
can act as a safety net, but it is not the real barrier. Setting protection up
requires a remote to be configured first. Order of operations:

1. Create the repository on GitHub/GitLab.
2. `git remote add origin <url>`, then push `main` and `dev`.
3. Configure the protection rules on the platform.

Recommended rules for `main` (and usually `dev`):

- **GitHub** — *Settings → Branches → Branch protection rules* (or *Rulesets*):
  enable "Require a pull request before merging" and "Do not allow bypassing the
  above settings"; optionally require approvals and green status checks (CI).
- **GitLab** — *Settings → Repository → Protected branches*: set "Allowed to
  push" to *No one* and "Allowed to merge" to the appropriate role.

Note: PR merges produce a merge commit (or a squash), not a fast-forward — so in
the real remote flow you rarely run `git merge --ff-only` by hand; the platform
handles the merge.

## Repository structure

```
├── infra/            # Ansible — host provisioning (Infrastructure as Code)
│   ├── playbooks/    #   01-install-docker, 02-mount-das
│   ├── inventory/    #   hosts and group/host variables
│   └── ...
└── apps/             # One Docker Compose stack per application
    ├── traefik/
    ├── seaweedfs/
    └── simple-page/
```

## Applications

Each application lives in its own directory under `apps/` with a dedicated
`docker-compose.yaml`.

- **Traefik** — reverse proxy / gateway; the single entry point that routes
  traffic to the other apps. See [`apps/traefik/docker-compose.yaml`](apps/traefik/docker-compose.yaml).
- **SeaweedFS** — object storage. See [`apps/seaweedfs/docker-compose.yaml`](apps/seaweedfs/docker-compose.yaml).
- **simple-page** — a minimal test page used as a deployment smoke test. See [`apps/simple-page/docker-compose.yaml`](apps/simple-page/docker-compose.yaml).

## Infrastructure (Ansible)

Host provisioning is handled by Ansible playbooks under `infra/playbooks/`,
run in sequence:

- **`01-install-docker`** — installs Docker Engine on the host.
- **`02-mount-das`** — mounts the TerraMaster D2-320 DAS.

Run a playbook against the inventory:

```bash
ansible-playbook -i infra/inventory infra/playbooks/01-install-docker.yaml
```

## Getting Started

**Prerequisites:**

- A control node with [Ansible](https://docs.ansible.com/) installed.
- SSH access to the server (Ubuntu Server 24) from the control node.
- The TerraMaster D2-320 DAS physically connected to the server.

**Bring-up order:**

1. Provision the host — run the Ansible playbooks in order (`01-install-docker`,
   then `02-mount-das`).
2. Deploy applications — bring up each stack under `apps/` with Docker Compose.

## Useful commands

### Ansible

```bash
# Check connectivity to all inventory hosts
ansible -i infra/inventory all -m ping

# Run a playbook
ansible-playbook -i infra/inventory infra/playbooks/01-install-docker.yaml
ansible-playbook -i infra/inventory infra/playbooks/02-mount-das.yaml
```

### Docker Compose (per app)

```bash
# Start a stack in the background (run from the app directory)
cd apps/traefik && docker compose up -d

# Follow logs
docker compose logs -f

# Show running services
docker compose ps

# Stop and remove the stack
docker compose down
```

### Disk / DAS

```bash
# List block devices and their mount points
lsblk

# Show mounted filesystems and free space
df -h

# Show current mounts
mount | grep -i das
```
