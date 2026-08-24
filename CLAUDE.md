# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This is a **DevOps repository** for a self-hosted home server. There is no application source code, build system, linter, or test suite — `infra/` and `apps/` are being populated incrementally.

## Architecture

Bare host → running apps, in three stages:

1. **Ansible provisions the host** (`infra/ansible/`) — installs Docker Engine and mounts a TerraMaster D2-320 DAS, via numbered playbooks run in sequence (e.g. `01-install-docker`, `02-mount-das`) against `infra/ansible/inventory.yaml`.
2. **Apps run as Docker Compose stacks**, one directory per app under `apps/` (e.g. `apps/traefik`, `apps/seaweedfs`, `apps/simple-page`), each with its own `docker-compose.yaml`.
3. **Traefik is the single reverse-proxy entry point** routing to the other app stacks.

`infra/autoinstall/` holds the Ubuntu Server autoinstall config (`user-data`/`meta-data`) used to build the install USB via nocloud — this is consumed at OS-install time, before Ansible ever runs.

## Environments & branching

Three environments map to two permanent branches (GitFlow-inspired):

| Environment | Branch                      | Host           |
| ----------- | --------------------------- | -------------- |
| Local       | `develop` (via `feature/*`) | Workstation    |
| Development | `develop`                   | Development VM |
| Production  | `main`                      | Home server    |

- `main` is production and **protected** — no direct pushes; only merges via PR from `dev`.
- `dev` is the integration branch — small changes may go straight to `dev`; larger features get a `feature/<name>` branch off `dev` with a PR back into it.
- Promotion to `main` is always via PR (`dev` → `main`), which triggers deployment to the server. Never push directly to `main`.

### Commits structure

- Commit messages use Conventional Commits with gitmoji (e.g. `feat: :tada:`, `docs: :memo:`, `fix: :bug:`) and must be in English.
- Commits should be **atomic** — one commit per logical change, with a clear message. Avoid mixing unrelated changes in a single commit.
- Just commit to the `dev` branch, don't create the PR, it's up to the reviewer to decide when to merge into `main`.

## Working with Ansible (Windows-specific)

Ansible must be run from **WSL2**, not PowerShell/CMD:

```bash
wsl -d Ubuntu
```

```bash
# Verify inventory
ansible-inventory -i ./infra/ansible/inventory.yaml --list

# Check connectivity
ansible prod -m ping -i ./infra/ansible/inventory.yaml

# Run a playbook
ansible-playbook -i infra/ansible/inventory.yaml infra/ansible/playbooks/01-install-docker.yaml
```

The user is already set up with passwordless sudo, so no `sudo` is needed for Ansible commands.

## Working with app stacks

Each app under `apps/<name>/` is operated independently with Docker Compose, from that app's directory:

```bash
cd apps/<name> && docker compose up -d
docker compose logs -f
docker compose ps
docker compose down
```

## Project conventions

- Design/planning docs for larger changes live under `docs/superpowers/` (`specs/` for design specs, `plans/` for implementation plans), following the Superpowers `brainstorming` → `writing-plans` workflow. Check there for the rationale behind existing structure before re-deriving it from scratch.
- For now, this is a greenfield project. Just use the production server as the target host for Ansible playbooks. In the future, a development VM will be added to mirror the production server, and the inventory will be updated accordingly.
