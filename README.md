# Proxmox LXC CI stack

An Ansible automation that provisions LXC containers on a remote Proxmox VE node through the Proxmox REST API, then configures CI/CD services inside those containers over SSH. The entire infrastructure — from container creation to service configuration — is declared in YAML and managed through two playbooks.

## Architecture

### Execution flow

Three machines are involved. The **control machine** (your workstation) runs Ansible. The **Proxmox node** is a separate server running Proxmox VE. The **LXC containers** are guests created on that Proxmox node.

```
                         ┌─────────────────────────┐
                         │    Proxmox VE node       │
                         │    (192.168.100.20)      │
                         │                           │
 ┌──────────────┐  HTTP  │  ┌───────────────────┐   │
 │   Control    │ ──────►│  │  Proxmox API :8006 │   │
 │   machine    │  API   │  └───────────────────┘   │
 │              │        │                           │
 │  proxmox.yml │        │  ┌─────────┐ ┌─────────┐ │
 │  site.yml    │  SSH   │  │ LXC 201 │ │ LXC 202 │ │
 │              │ ──────►│  │postgres │ │ valkey  │ │
 └──────────────┘        │  └─────────┘ └─────────┘ │
                         │  ┌─────────┐ ┌─────────┐ │
                         │  │ LXC 203 │ │ LXC 204 │ │
                         │  │runner-ci│ │runner-  │ │
                         │  │         │ │  build  │ │
                         │  └─────────┘ └─────────┘ │
                         └─────────────────────────┘
```

The workflow runs in two stages:

1. **`proxmox.yml`** — Runs on the control machine (`hosts: localhost`, `connection: local`). The `proxmox_lxc` role sends HTTP requests to the Proxmox API on the remote node to create and start each declared container. No SSH connection to the Proxmox node or to any container is needed at this stage.

2. **`site.yml`** — Connects to each running container over SSH using the addresses from `inventory.yml`. Applies configuration roles based on inventory group membership:

   | Inventory group | Role applied | Container |
   |---|---|---|
   | `all` | `common` | All four containers |
   | `postgres_ci` | `postgres_ci` | `postgres-ci` (192.168.100.21) |
   | `valkey_ci` | `valkey_ci` | `valkey-ci` (192.168.100.22) |
   | `gitlab_runner` | `gitlab_runner_shell` | `runner-ci` (192.168.100.23) |
   | `build_runner` | `build_runner` | `runner-build` (192.168.100.24) |

No Docker socket, Docker daemon, or Docker-in-Docker is used anywhere.

### Directory structure

```
.
├── proxmox.yml                          # Stage 1: provision LXC via Proxmox API
├── site.yml                             # Stage 2: configure services inside LXC
├── requirements.yml                     # Ansible Galaxy collection dependencies
├── inventory.example.yml                # Inventory template (copy to inventory.yml)
├── .gitignore                           # Excludes inventory.yml, vault, proxmox.yml
├── group_vars/
│   ├── all.yml                          # Shared variables loaded for all hosts
│   └── proxmox.yml.example              # Proxmox API and container declarations template
└── roles/
    ├── proxmox_lxc/                     # Provision containers via Proxmox API
    │   └── tasks/main.yml
    ├── common/                          # Base OS packages and time sync
    │   └── tasks/main.yml
    ├── postgres_ci/                     # PostgreSQL database server
    │   ├── tasks/main.yml
    │   └── handlers/main.yml
    ├── valkey_ci/                       # Valkey cache server
    │   ├── tasks/main.yml
    │   └── handlers/main.yml
    ├── gitlab_runner_shell/             # GitLab Runner (shell executor)
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   ├── defaults/main.yml
    │   └── templates/config.toml.j2
    └── build_runner/                    # Rootless Buildah/Podman image builder
        ├── tasks/main.yml
        └── defaults/main.yml
```

### File details

#### Playbooks

- **`proxmox.yml`** — The provisioning playbook. Targets `localhost` because the `community.proxmox.proxmox` module runs locally and communicates with the remote Proxmox API over HTTPS. It loads variables from `group_vars/proxmox.yml` (API credentials and the container list) and applies the `proxmox_lxc` role.

- **`site.yml`** — The configuration playbook. Contains five plays, each targeting a different inventory group (`all`, `postgres_ci`, `valkey_ci`, `gitlab_runner`, `build_runner`). Every play uses `become: true` for root-level package installation and service management on the remote containers.

#### Variable files

- **`group_vars/all.yml`** — Variables available to every host in the inventory. Contains the CI PostgreSQL database name (`jaringusahaqu`), database user, listen addresses for PostgreSQL and Valkey (`0.0.0.0`), the GitLab Runner executor type (`shell`), shell type (`bash`), and the image builder username (`imagebuilder`). Sensitive values like `ci_postgres_password` should reference Ansible Vault variables.

- **`group_vars/proxmox.yml.example`** — Template for Proxmox-specific variables. Contains the API endpoint (`proxmox_api_host`), API user and token ID, TLS verification flag, Proxmox node name, and the `proxmox_lxc_containers` list. Each container entry declares its VM ID, hostname, OS template path, storage backend, disk size, CPU cores, memory, swap, network interface configuration, SSH public key, and an optional `features` list. Copy this to `group_vars/proxmox.yml` and customize; it is excluded from Git via `.gitignore`.

- **`group_vars/vault.yml`** (user-created, encrypted) — Stores secrets used by both playbooks: `vault_proxmox_api_token_secret`, `vault_lxc_password`, `gitlab_runner_token`, and `ci_postgres_password`.

#### Inventory

- **`inventory.example.yml`** — Defines four inventory groups matching the plays in `site.yml`. Each group contains one host with a static IP on the `192.168.100.0/24` bridge subnet. Also sets shared connection variables (`ansible_user: ansible`, `ansible_become: true`) and service discovery variables (PostgreSQL/Valkey hostnames and ports, GitLab URL, runner description and tags).

#### Dependencies

- **`requirements.yml`** — Lists two Ansible Galaxy collections: `community.postgresql` (used by the `postgres_ci` role to manage databases and users) and `community.proxmox` (used by the `proxmox_lxc` role to create containers via the API).

### Roles

#### `proxmox_lxc` — Container provisioning

Runs on the control machine. Three tasks:

1. **Validate** — Asserts that every item in `proxmox_lxc_containers` has the required fields (`vmid`, `hostname`, `ostemplate`, `netif`). Fails early with a descriptive message if any are missing.
2. **Create or update** — Calls `community.proxmox.proxmox` with `state: present` for each container. Configures disk, CPU, memory, swap, network, SSH keys, unprivileged mode, and optional LXC features (e.g., `nesting=1` for the build runner). Uses `no_log: true` to prevent API tokens and passwords from appearing in Ansible output.
3. **Start** — Calls `community.proxmox.proxmox` with `state: started` for each container. Also uses `no_log: true`.

#### `common` — Base OS setup

Runs on all containers. Installs essential packages (`ca-certificates`, `curl`, `git`, `gnupg`, `jq`, `python3`, `python3-apt`, `unzip`) and enables `systemd-timesyncd` for time synchronization.

#### `postgres_ci` — PostgreSQL CI database

Runs on the `postgres_ci` group. Installs PostgreSQL 16, configures `listen_addresses` to the private bridge interface (default `0.0.0.0`), creates the CI database and user with the vault-encrypted password, and verifies readiness with `pg_isready`. The handler restarts PostgreSQL when the listen address configuration changes.

#### `valkey_ci` — Valkey CI cache

Runs on the `valkey_ci` group. Installs the `valkey-server` package, binds the listener to the configured private address, enables the service, and verifies connectivity with `valkey-cli ping`. The handler restarts Valkey when the bind address changes.

#### `gitlab_runner_shell` — GitLab Runner

Runs on the `gitlab_runner` group. Downloads the GitLab Runner `.deb` package from the official S3 URL (architecture and version are configurable via defaults), installs it, checks for existing registration with `gitlab-runner verify`, and registers a new runner if no valid registration exists. Registration uses the shell executor, the configured tags (default: `lxc-shell`), and the vault-encrypted runner token. The `config.toml.j2` template sets `concurrent = 1` and disables caching. The handler enables and starts the `gitlab-runner` service.

#### `build_runner` — Rootless image builder

Runs on the `build_runner` group. Installs container image tooling (`buildah`, `podman`, `fuse-overlayfs`, `uidmap`, `slirp4netns`, `ca-certificates`, `git`), creates a dedicated `imagebuilder` user, configures subordinate UID/GID ranges in `/etc/subuid` and `/etc/subgid` (default: `100000:65536`), and verifies that both `buildah info` and `podman info` run successfully under the unprivileged user. This container requires `nesting=1` in its LXC features to support user namespaces.

## Requirements

Install Ansible Galaxy collections on the control machine:

```bash
ansible-galaxy collection install -r requirements.yml
```

The control machine needs:

- **HTTPS access to the Proxmox API** (port 8006) on the Proxmox VE node for container provisioning.
- **SSH access to each LXC container** after provisioning for service configuration.

Each LXC container needs Python 3, `sudo`, and Debian/Ubuntu package repositories. The LXC bridge network must provide:

- Outbound HTTPS to `gitlab.com` (or your self-hosted GitLab instance) for runner registration and CI jobs.
- Private connectivity from the runner containers to the PostgreSQL and Valkey containers on the same bridge subnet.

## Quick start

### 1. Create a Proxmox API token

In the Proxmox web UI (Datacenter → Permissions → Users / API Tokens), create a dedicated user and API token. Assign the minimum required permissions:

| Permission | Purpose |
|---|---|
| `VM.Allocate` | Create new containers |
| `VM.Config.*` | Set CPU, memory, disk, network |
| `VM.PowerMgmt` | Start and stop containers |
| `Datastore.AllocateSpace` | Allocate disk storage |
| `Datastore.Audit` | List available storage |
| `Sys.Audit` | Query node status |

### 2. Configure Proxmox variables

```bash
cp group_vars/proxmox.yml.example group_vars/proxmox.yml
```

Edit `group_vars/proxmox.yml` and set:

- `proxmox_api_host` — the FQDN or IP of your Proxmox node.
- `proxmox_node` — the Proxmox node name (as shown in the Proxmox UI).
- Each container's `ostemplate`, `storage`, `disk_size`, `cores`, `memory`, network bridge, and static IP.

### 3. Create vault secrets

```bash
ansible-vault create group_vars/vault.yml
```

Contents:

```yaml
vault_proxmox_api_token_secret: <token-secret>
vault_lxc_password: <container-root-password>
gitlab_runner_token: <protected-runner-token>
ci_postgres_password: <database-password>
```

### 4. Provision LXC containers

```bash
ansible-playbook proxmox.yml --ask-vault-pass
```

This creates and starts all four containers on the Proxmox node via the API. No SSH is involved.

### 5. Configure inventory

```bash
cp inventory.example.yml inventory.yml
```

Verify that the IP addresses in `inventory.yml` match the static IPs assigned in `group_vars/proxmox.yml`. Ensure SSH access works:

```bash
ansible -i inventory.yml all -m ping
```

### 6. Configure services

Dry-run first, then apply:

```bash
ansible-playbook -i inventory.yml site.yml --check --diff --ask-vault-pass
ansible-playbook -i inventory.yml site.yml --ask-become-pass --ask-vault-pass
```

## Security

- **Never commit** `inventory.yml`, `group_vars/proxmox.yml`, or `group_vars/vault.yml` to Git. They are excluded via `.gitignore`.
- **Use `ansible-vault`** for all sensitive values: API token secrets, container passwords, runner tokens, database passwords.
- **Scope API token permissions** to the minimum listed above. Avoid granting `Sys.Admin` or root-level access.
- **Restrict database listeners**: PostgreSQL and Valkey bind to `0.0.0.0` by default. Use Proxmox firewall rules or iptables inside the containers to limit access to the runner subnet only.
- **Rootless containers**: All four containers run unprivileged (`unprivileged: true`). The build runner additionally requires `nesting=1` in its LXC features for user namespace support. Never grant LXC containers access to the Proxmox host filesystem.
- **`no_log: true`**: The `proxmox_lxc` role suppresses task output on API calls to prevent tokens and passwords from appearing in terminal logs.

## GitLab Runner notes

- The runner registers with the **shell executor**, meaning CI jobs run directly on the container OS (no Docker isolation).
- Default tags: `lxc-shell`. Jobs in `.gitlab-ci.yml` must reference this tag.
- The shell executor does not support `image:` or `services:` directives in `.gitlab-ci.yml`.
- Mark runners as **protected** in GitLab if they access production secrets or deploy artifacts.
- `concurrent = 1` in `config.toml` — only one job runs at a time. Increase this if the container has sufficient resources.

## References

- [community.proxmox collection](https://docs.ansible.com/ansible/latest/collections/community/proxmox/index.html) — Collection index and module list.
- [community.proxmox.proxmox module (LXC)](https://docs.ansible.com/ansible/devel/collections/community/proxmox/proxmox_module.html) — Module parameters for container management.
- [Proxmox VE API](https://pve.proxmox.com/wiki/Proxmox_VE_API) — REST API overview, authentication methods, and endpoint reference.
- [Proxmox User Management & API Tokens](https://pve.proxmox.com/pve-docs/chapter-pveum.html) — How to create users, API tokens, and assign permissions.
- [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html) — Encrypting sensitive data in playbooks and variable files.
- [Ansible Inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html) — Inventory file format, groups, and host variables.
