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
  **Never committed to directly.**
- **`dev`** — integration branch. All work lands here first.

Feature work branches off `dev` and merges back through a pull request:

```
feature/* ──▶ dev ──▶ main
  (work)   (integrate) (release to server)
```

- Create a `feature/<name>` branch from `dev` for each change.
- Open a pull request into `dev`; once merged, changes are validated on the
  Development VM.
- Promote `dev` to `main` to release to production (the home server).

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
