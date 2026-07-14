# home-server README — Design Spec

**Date:** 2026-07-14
**Status:** Approved
**Scope:** Replace the template `Readme.md` with a project-specific README documenting the target architecture of the home-server DevOps project, including a "Useful commands" section.

## Context

`home-server` is a greenfield DevOps project. A physical server running Ubuntu Server 24 has been acquired, with a TerraMaster D2-320 DAS attached (mount to be provisioned). The goal is an infrastructure to deploy applications via Docker, provisioned with Ansible (Infrastructure as Code).

The repository currently contains only a generic template `Readme.md` (Python/Django/Dropwizard boilerplate) and no commits. This spec covers rewriting that README only — no infra or app code is created in this task.

## Decisions

- **Language:** English.
- **Scope of content:** Document the *target architecture* (aspirational). Directories like `infra/` and `apps/` are described even though they don't exist yet.
- **Approach:** Full replacement of the template README (not section-by-section adaptation). Keep only the template's good habits: badges at the top and a clear heading structure.
- **Traefik routing:** Left as generic placeholders — no domain defined yet.
- **App descriptions:** Purpose only; do not describe the contents of each `docker-compose.yaml`. Reference the compose file by relative path (a link to be filled/refined later).
- **Ansible playbooks:** Named with a sequential prefix — `01-install-docker`, `02-mount-das`.

## README Structure

1. **Title + badges** — `home-server`, with badges for the real stack: Ansible, Docker, Traefik, Ubuntu Server 24.04.
2. **Overview** — One paragraph: physical server with Ubuntu Server 24, TerraMaster D2-320 DAS, oriented to deploying apps with Docker; infrastructure managed as code with Ansible.
3. **Architecture** — Short description of the flow: Ansible provisions the host (installs Docker + mounts the DAS) → apps run as Docker Compose stacks → Traefik acts as the entry point / reverse proxy.
4. **Repository structure** — Commented directory tree:
   ```
   ├── infra/            # Ansible (host provisioning)
   │   ├── playbooks/    #   01-install-docker, 02-mount-das
   │   ├── inventory/
   │   └── ...
   └── apps/             # One Docker Compose stack per app
       ├── traefik/
       ├── seaweedfs/
       └── simple-page/
   ```
5. **Applications** — One subsection per app, purpose only (no compose-file contents), each referencing its compose file by relative path:
   - **Traefik** — reverse proxy / gateway. → `apps/traefik/docker-compose.yaml`
   - **SeaweedFS** — object storage. → `apps/seaweedfs/docker-compose.yaml`
   - **simple-page** — test/smoke-test page. → `apps/simple-page/docker-compose.yaml`
6. **Infrastructure (Ansible)** — Planned playbooks with sequential prefixes:
   - `01-install-docker` — installs Docker Engine on the host.
   - `02-mount-das` — mounts the TerraMaster D2-320 DAS.

   Includes a note on how playbooks are run.
7. **Getting Started** — Prerequisites (control node with Ansible, SSH access to the server, DAS physically connected) and the bring-up order: run playbooks → bring up apps.
8. **Useful commands / Comandos útiles** — Grouped command blocks:
   - **Ansible** — connectivity ping, running playbooks.
   - **Docker Compose** — up / down / logs / ps per app.
   - **Disk / DAS utilities** — `lsblk`, `df`, `mount`.
9. **Roadmap / Status** — Short note that the project is under construction; directories will be populated over time.

## Excluded (dropped from template — YAGNI for a personal infra repo)

Screenshots, Running the tests, Contributing, Versioning, Authors, License, Acknowledgments. (A License can be added later if desired.)

## Deliverable

A single rewritten `Readme.md` at the repository root, replacing the current template.
