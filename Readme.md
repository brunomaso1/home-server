# home-server

![IaC](https://img.shields.io/badge/IaC-Ansible-EE0000?logo=ansible&logoColor=white) ![Containers](https://img.shields.io/badge/containers-Docker-2496ED?logo=docker&logoColor=white) ![OS](https://img.shields.io/badge/OS-Ubuntu%20Server%2024.04-E95420?logo=ubuntu&logoColor=white)

Infrastructure and application stacks for a self-hosted home server. A physical machine running Ubuntu Server 24 — with a TerraMaster D2-320 DAS attached — is provisioned with Ansible (Infrastructure as Code) and runs applications as Docker Compose stacks behind a Traefik reverse proxy.

> **Status:** Under construction. This README documents the target architecture; the directories below are populated incrementally.

## Getting Started

**Prerequisites:**

- A control node with [Ansible](https://docs.ansible.com/) installed.
- SSH access to the server (Ubuntu Server 24) from the control node.
- The TerraMaster D2-320 DAS physically connected to the server.

**Bring-up order:**

1. Provision the host — run all Ansible playbooks at once with `site.yaml` (or individually in numbered order).
2. Deploy applications — bring up each stack under `apps/` with Docker Compose.

## Architecture

The flow from bare host to running apps:

1. **Provision the host with Ansible** — install Docker and mount the DAS.
2. **Run applications as Docker Compose stacks** — one stack per app under `apps/`.
3. **Route traffic through Traefik** — a single reverse proxy is the entry point to every app.

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

| Environment | Branch                      | Host             | Purpose                                             |
| ----------- | --------------------------- | ---------------- | --------------------------------------------------- |
| Local       | `develop` (via `feature/*`) | Your workstation | Development and local testing                       |
| Development | `develop`                   | Development VM   | Integration testing before release                  |
| Production  | `main`                      | Home server      | Live deployment — the server runs what is on `main` |

### Branching strategy

A GitFlow-inspired model adapted for DevOps, with two permanent branches:

- **`main`** — production. Reflects exactly what runs on the server.
  A **protected branch**: no direct pushes — changes only arrive through a pull request from `develop`.
- **`dev`** — integration branch. Small changes may be committed here directly; larger features get their own `feature/<name>` branch first.

Flow from work to production:

```
feature/* ──▶ dev ──▶ main
  (work)   (integrate) (release to server)
```

- For a large feature, branch `feature/<name>` off `dev`, then open a pull request into `dev`.
- Small changes can go straight onto `dev`.
- Changes merged into `dev` are validated on the Development VM.
- To release, open a pull request from `dev` into `main`. Merging it triggers the deployment pipeline to the server.

> **Promotion to `main` is done through a pull request against the remote — never a direct push.**

## Repository structure

```
├── infra/            # Infrastructure as Code
│   └── ansible/      #   Ansible — host provisioning
│       ├── site.yaml       #   Entry point — imports all playbooks in order
│       ├── inventory.yaml  #   Hosts and connection variables
│       └── playbooks/      #   Numbered playbooks (01-update-system, 02-install-docker, ...)
└── apps/             # One Docker Compose stack per application
    ├── traefik/
    ├── seaweedfs/
    └── simple-page/
```

## Applications

Each application lives in its own directory under `apps/` with a dedicated `docker-compose.yaml`.

- **Traefik** — reverse proxy / gateway; the single entry point that routes traffic to the other apps. See [`apps/traefik/docker-compose.yaml`](apps/traefik/docker-compose.yaml).
- **SeaweedFS** — object storage. See [`apps/seaweedfs/docker-compose.yaml`](apps/seaweedfs/docker-compose.yaml).
- **simple-page** — a minimal test page used as a deployment smoke test. See [`apps/simple-page/docker-compose.yaml`](apps/simple-page/docker-compose.yaml).

## Infrastructure

### Ubuntu Server 26

The server runs Ubuntu Server 26.04 LTS, a minimal installation with OpenSSH server enabled.
To install Ubuntu Server 26.04 LTS using autoinstall:

1. Download the ISO from [Ubuntu](https://ubuntu.com/download/server).
2. Create a bootable USB drive with the ISO.
> Note: You can use [Rufus](https://rufus.ie/) on Windows or `dd` on Linux/macOS.
3. Configure your ssh public key by:
   - Generate an SSH key pair on your workstation (if you don't have one already):

  ```bash
   ssh-keygen -t ed25519 -C "user@workstation"
  ```
   - Rename it `home_server` and add it to your `config` file (you can find the ip of the server later):

  ```bash
    Host home-server
      HostName 192.168.1.50
      User maso
      IdentityFile C:/Users/maso/.ssh/home_server
      IdentitiesOnly yes
  ```
4. Configure autoinstall by:
   - Create a directory called `nocloud` in the root of the USB drive.
   - Copy the `user-data` and `meta-data` from `infra\autoinstall` to the `nocloud` directory.
5. Boot the server from the USB drive. Wait for the autoinstall to complete (it should turn off the server when the installation completes), then log in via SSH.
> Note: 
> - The server should have a static IP address (e.g., `192.168.0.3`), preferably set in the router's DHCP settings. The first time you can search for the server's IP address using `arp -a` or check the router's connected devices list.

### Ansible

Host provisioning is handled by Ansible playbooks under `infra/ansible/playbooks/`, run in numbered order:

- `01-update-system` — apt cache update + dist-upgrade all packages, cleanup, reboot check.
- `02-install-docker` — add the Docker APT repository, install Docker Engine + Compose plugin, enable the service.
- `02.1-verify-docker-installation` — verify the Docker daemon and service are up; print installed versions.

Run everything at once with the parent playbook:

```bash
ansible-playbook -i infra/ansible/inventory.yaml infra/ansible/site.yaml
```

Or run a single playbook against the inventory:

```bash
ansible-playbook -i infra/ansible/inventory.yaml infra/ansible/playbooks/01-update-system.yaml
```

Using the debug (print) in a playbook is useful for troubleshooting:

```yaml
- name: Show stat result
  ansible.builtin.debug:
    var: reboot_required_file
```

> NOTE: Tip for learning: add -v (or -vvv) to the real run to see exactly what the apt module reports as changed.

## Roadmap

- [ ] Configure the development VM to mirror the production server.


## Useful commands

### Bash commands

- Connect to the server via SSH:
```bash
ssh maso@192.168.0.3
```

> Without password: `ssh home-server`

### Ansible

> Important: On Windows, ansible commands must be run from a WSL2 terminal (e.g., Ubuntu). Do not run them from PowerShell or CMD. You can use the `wsl -d Ubuntu` command to open a WSL2 terminal from PowerShell or CMD, and install ansible by running `apt install pipx -y && pipx install ansible`.
> Important: From Windows, the path to the private key file must be specified in Unix format (e.g., `/mnt/c/Users/maso/.ssh/home_server`), not Windows format (e.g., `C:/Users/maso/.ssh/home_server`).
> Important: Before running any ansible command, trust the server's SSH fingerprint by connecting to it once with `ssh maso@server_ip` and accepting the fingerprint.

- Verify the inventory -> `ansible-inventory -i ./infra/ansible/inventory.yaml --list`
- Check connectivity to all inventory hosts -> `ansible prod -m ping -i ./infra/ansible/inventory.yaml`
- Run a playbook (dry run) -> `ansible-playbook -i ./infra/ansible/inventory.yaml ./infra/ansible/playbooks/01-update-system.yaml --check`
- Run a playbook (actual run) -> `ansible-playbook -i ./infra/ansible/inventory.yaml ./infra/ansible/playbooks/01-update-system.yaml`
- Provision all the server -> `ansible-playbook -i ./infra/ansible/inventory.yaml ./infra/ansible/site.yaml`

### Docker Compose (per app)

- Start a stack in the background (run from the app directory) -> `cd apps/traefik && docker compose up -d`
- Follow logs -> `docker compose logs -f`
- Show running services -> `docker compose ps`
- Stop and remove the stack -> `docker compose down`

### Disk / DAS

- List block devices and their mount points -> `lsblk`
- Show mounted filesystems and free space -> `df -h`
- Show current mounts -> `mount | grep -i das`

## Common issues

### SSH Private Key Permissions in WSL

#### Problem

When using Ansible/SSH from **WSL** with a private SSH key stored in the Windows filesystem (`/mnt/c/...`), SSH may reject the key with:

```text
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0777 for '/mnt/c/Users/maso/.ssh/home_server' are too open.
This private key will be ignored.
```

#### Cause

WSL uses **DrvFS** to mount Windows filesystems (`/mnt/c`, `/mnt/d`, etc.).

By default, Windows files do not have Linux permission metadata. As a result, files accessed through `/mnt/c` may appear inside WSL with permissions such as `0777`.

OpenSSH requires private keys to have restrictive permissions and will refuse to use a key that appears to be accessible by other users.

#### Solution

Enable **Linux permission metadata** for DrvFS.

Edit the WSL configuration:

```bash
sudo nano /etc/wsl.conf
```

Add:

```ini
[automount]
enabled = true
options = metadata,uid=1000,gid=1000,umask=0022
```

Then completely shut down WSL from **PowerShell or CMD**:

```powershell
wsl --shutdown
```

Reopen Ubuntu and check the key permissions:

```bash
ls -l /mnt/c/Users/maso/.ssh/home_server
```

Set restrictive permissions:

```bash
chmod 600 /mnt/c/Users/maso/.ssh/home_server
```

Verify:

```bash
ls -l /mnt/c/Users/maso/.ssh/home_server
```

The expected result is similar to:

```text
-rw------- ... home_server
```

#### Verify SSH Connection

Test the connection directly from WSL:

```bash
ssh -i /mnt/c/Users/maso/.ssh/home_server maso@192.168.0.3
```

The connection should now use the private key without asking for the user's password.

#### Ansible Configuration

The private key can be referenced directly from the Windows filesystem in the Ansible inventory:

```yaml
prod:
  hosts:
    home_server:
      ansible_host: 192.168.0.3
      ansible_user: maso
      ansible_ssh_private_key_file: /mnt/c/Users/maso/.ssh/home_server
      ansible_become: true
      ansible_become_method: sudo
```

There is no need to copy the private key into the WSL filesystem.

This allows the same key to be stored in:

```text
C:\Users\maso\.ssh\home_server
```

and accessed from both Windows and WSL.

#### Verify Ansible

Run:

```bash
ansible prod -i inventory.yml -m ping
```

Expected output:

```text
home_server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

#### Alternative

Copying the private key to `~/.ssh` inside WSL would also solve the problem, but it creates a second copy of the private key.

Enabling `metadata` is the appropriate WSL/DrvFS configuration when the intention is to use Linux tools such as SSH and Ansible with files stored on the Windows filesystem.

### Setting Up a Dedicated Ansible User

This setup creates a dedicated `ansible` user on the managed server. The user authenticates exclusively through an SSH key and can execute privileged commands through `sudo` without requiring a password.

This is useful for automation because it separates Ansible's credentials from the personal user account.

#### 1. Create the `ansible` User

On the managed server, log in with an administrative user:

```bash
ssh maso@192.168.0.3
```

Create the user without a password:

```bash
sudo adduser --disabled-password --gecos "" ansible
```

The user does **not** need to be added to the `sudo` group.

#### 2. Configure Passwordless `sudo`

Create a dedicated sudoers configuration:

```bash
sudo visudo -f /etc/sudoers.d/ansible
```

Add:

```text
ansible ALL=(ALL) NOPASSWD: ALL
```

Save and exit.

Set the recommended permissions:

```bash
sudo chmod 440 /etc/sudoers.d/ansible
```

Verify the configuration:

```bash
sudo -l -U ansible
```

Expected:

```text
User ansible may run the following commands on home-server:
    (ALL) NOPASSWD: ALL
```

##### Important

Do **not** add `ansible` to the `sudo` group:

```bash
sudo usermod -aG sudo ansible
```

The `sudo` group normally has a rule that requires a password:

```text
%sudo ALL=(ALL:ALL) ALL
```

If `ansible` belongs to that group, it can result in both rules being applicable:

```text
(ALL) NOPASSWD: ALL
(ALL : ALL) ALL
```

The dedicated `/etc/sudoers.d/ansible` rule is sufficient.

If the user was previously added to the `sudo` group, remove it with:

```bash
sudo deluser ansible sudo
```

Then verify:

```bash
groups ansible
```

#### 3. Generate the SSH Key

The SSH key should be generated on the **Ansible control node**.

In this setup, Ansible runs inside WSL, but the key is stored on the Windows filesystem so it can be reused by both Windows and WSL.

From PowerShell:

```powershell
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\ansible_home_server
```

If Ansible needs to connect without interactive input, leave the passphrase empty.

This creates:

```text
C:\Users\maso\.ssh\ansible_home_server
C:\Users\maso\.ssh\ansible_home_server.pub
```

The private key must remain on the control node. Only the `.pub` file is copied to the server.

#### 4. Configure `authorized_keys`

From WSL, the Windows key is accessible as:

```bash
/mnt/c/Users/maso/.ssh/ansible_home_server
/mnt/c/Users/maso/.ssh/ansible_home_server.pub
```

Display the public key:

```bash
cat /mnt/c/Users/maso/.ssh/ansible_home_server.pub
```

Copy the entire output.

On the managed server, logged in as an administrative user:

```bash
sudo mkdir -p /home/ansible/.ssh
```

Create the authorized keys file:

```bash
sudo nano /home/ansible/.ssh/authorized_keys
```

Paste the public key and save.

Set the correct ownership and permissions:

```bash
sudo chown -R ansible:ansible /home/ansible/.ssh
sudo chmod 700 /home/ansible/.ssh
sudo chmod 600 /home/ansible/.ssh/authorized_keys
```

#### 5. Test SSH Authentication

From WSL:

```bash
ssh -i /mnt/c/Users/maso/.ssh/ansible_home_server ansible@192.168.0.3
```

The connection should work without asking for a password.

Verify the user:

```bash
whoami
```

Expected:

```text
ansible
```

Verify passwordless privilege escalation:

```bash
sudo whoami
```

Expected:

```text
root
```

with no password prompt.

You can also test both operations in a single command:

```bash
ssh -i /mnt/c/Users/maso/.ssh/ansible_home_server ansible@192.168.0.3 sudo whoami
```

Expected:

```text
root
```

#### 6. Configure Ansible

The inventory can now use the dedicated user:

```yaml
prod:
  hosts:
    home_server:
      ansible_host: 192.168.0.3
      ansible_user: ansible
      ansible_ssh_private_key_file: /mnt/c/Users/maso/.ssh/ansible_home_server
      ansible_become: true
      ansible_become_method: sudo
```

Ansible can now execute privileged tasks without `-K`:

```bash
ansible prod -i inventory.yaml -m ping
```

Expected:

```text
home_server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

#### Final Architecture

```text
Windows
└── C:\Users\maso\.ssh\
    ├── ansible_home_server          # Private key
    └── ansible_home_server.pub      # Public key
             │
             │ /mnt/c
             ▼
       WSL / Ansible
             │
             │ SSH
             ▼
       home-server
             │
             └── ansible
                   │
                   └── sudo (NOPASSWD)
                         │
                         ▼
                        root
```

The private key exists only on the control node, while the managed server stores only the corresponding public key. The dedicated `ansible` account is independent from the personal `maso` account and is used exclusively for automation.
