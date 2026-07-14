# home-server README Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the generic template `Readme.md` at the repo root with a project-specific README documenting the target architecture of the home-server DevOps project, including a "Useful commands" section.

**Architecture:** Single-file deliverable. The current template README (Python/Django/Dropwizard boilerplate) is fully replaced by a new English README describing the *target* architecture: Ansible provisions an Ubuntu Server 24 host (Docker + DAS mount) and apps run as Docker Compose stacks fronted by Traefik. Directories described (`infra/`, `apps/`) are aspirational — they do not exist yet.

**Tech Stack:** Markdown. Documents Ansible, Docker / Docker Compose, Traefik, SeaweedFS, Ubuntu Server 24.04, TerraMaster D2-320 DAS.

## Global Constraints

- Language: **English** (the entire README).
- Content is the **target architecture** (aspirational); `infra/` and `apps/` are described though absent.
- Full replacement of the template — do not preserve template sections. Keep only: badges at top + clear heading structure.
- Traefik routing: **generic placeholders only** — no domain defined.
- App descriptions: **purpose only**, no `docker-compose.yaml` contents; reference each compose file by relative path.
- Ansible playbooks named with sequential prefix: `01-install-docker`, `02-mount-das`.
- Excluded sections (YAGNI): Screenshots, Running the tests, Contributing, Versioning, Authors, License, Acknowledgments.
- Source of truth: `docs/superpowers/specs/2026-07-14-home-server-readme-design.md`.

---

### Task 1: Rewrite `Readme.md` with the full project structure

**Files:**
- Modify (full replacement): `Readme.md`

**Interfaces:**
- Consumes: the approved spec at `docs/superpowers/specs/2026-07-14-home-server-readme-design.md`.
- Produces: final `Readme.md`. No downstream code depends on it.

- [ ] **Step 1: Replace the entire contents of `Readme.md` with the content below**

````markdown
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
````

- [ ] **Step 2: Verify the README against the spec**

Check the rendered file covers every spec section, in order:
Title+badges, Overview, Architecture, Repository structure (tree),
Applications (3 apps, purpose only, relative links), Infrastructure/Ansible
(`01-install-docker`, `02-mount-das`), Getting Started (prereqs + bring-up
order), Useful commands (Ansible / Docker Compose / Disk-DAS), Status note.

Run: `grep -nE "01-install-docker|02-mount-das|SeaweedFS|simple-page|Traefik|Useful commands" Readme.md`
Expected: matches for each of the playbook names, all three apps, and the
Useful commands heading.

- [ ] **Step 3: Verify no template leftovers remain**

Run: `grep -niE "Dropwizard|PurpleBooth|Billie Thompson|django|SemVer|One Paragraph" Readme.md`
Expected: no output (exit code 1) — none of the template boilerplate survives.

- [ ] **Step 4: Verify relative links point to sensible paths**

Run: `grep -oE "apps/[a-z-]+/docker-compose.yaml" Readme.md`
Expected: `apps/traefik/docker-compose.yaml`, `apps/seaweedfs/docker-compose.yaml`,
`apps/simple-page/docker-compose.yaml` (paths are aspirational; files need not
exist yet).

- [ ] **Step 5: Commit**

```bash
git add Readme.md
git commit -m "docs: rewrite README for home-server architecture"
```

---

## Self-Review

- **Spec coverage:** All nine spec sections map to content in Task 1 Step 1;
  Steps 2–4 verify coverage, template removal, and links. ✅
- **Placeholder scan:** No TBD/TODO in the deliverable; the "Status: Under
  construction" note is intentional content, not a plan placeholder. ✅
- **Type consistency:** Playbook names (`01-install-docker`, `02-mount-das`),
  app names (Traefik, SeaweedFS, simple-page), and compose paths are identical
  across the tree, Applications, Infrastructure, and Useful commands sections. ✅
