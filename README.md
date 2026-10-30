# Proxmox LXC CI Infrastructure

Ansible automation for provisioning, configuring, and managing a CI infrastructure on Proxmox VE using Debian 12 LXC containers.

The project separates infrastructure provisioning from guest configuration and uses dedicated LXC containers for PostgreSQL, Valkey, GitLab Runner, and container image builds.

## Table of Contents

- [1. Architecture Overview](#1-architecture-overview)
- [2. Infrastructure](#2-infrastructure)
- [3. Tested Environment](#3-tested-environment)
- [4. Project Structure](#4-project-structure)
- [5. How the Automation Works](#5-how-the-automation-works)
- [6. Playbooks](#6-playbooks)
- [7. Ansible Roles](#7-ansible-roles)
- [8. Configuration Files](#8-configuration-files)
- [9. Requirements](#9-requirements)
- [10. Initial Setup](#10-initial-setup)
- [11. Proxmox API Configuration](#11-proxmox-api-configuration)
- [12. Deploy the Infrastructure](#12-deploy-the-infrastructure)
- [13. Verify the Deployment](#13-verify-the-deployment)
- [14. Destroy the Infrastructure](#14-destroy-the-infrastructure)
- [15. Security Considerations](#15-security-considerations)
- [16. Troubleshooting](#16-troubleshooting)
- [17. Useful Commands](#17-useful-commands)
- [18. References](#18-references)

## 1. Architecture Overview

The infrastructure consists of four Debian 12 LXC containers:

| Container | Hostname | IP Address | Purpose |
|---|---|---|---|
| PostgreSQL CI | `postgres-ci` | `192.168.100.21` | CI database service |
| Valkey CI | `valkey-ci` | `192.168.100.22` | CI cache and in-memory data store |
| GitLab Runner | `runner-ci` | `192.168.100.23` | GitLab Runner using Shell executor |
| Build Runner | `runner-build` | `192.168.100.24` | Container image builds using rootless Buildah/Podman |

### Architecture Diagram

```text
                       ┌──────────────────────┐
                       │    Control Machine   │
                       │                      │
                       │       Ansible        │
                       └──────────┬───────────┘
                                  │
                         HTTPS / REST API
                                  │
                                  ▼
                       ┌──────────────────────┐
                       │      Proxmox VE      │
                       │     server-home      │
                       │   192.168.100.20     │
                       └──────────┬───────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
     ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
     │ postgres-ci    │  │ valkey-ci      │  │ runner-ci      │
     │ 192.168.100.21 │  │ 192.168.100.22 │  │ 192.168.100.23 │
     │                │  │                │  │                │
     │ PostgreSQL     │  │ Valkey         │  │ GitLab Runner  │
     └────────────────┘  └────────────────┘  └────────────────┘

                                  ┌────────────────┐
                                  │ runner-build   │
                                  │ 192.168.100.24 │
                                  │                │
                                  │ Buildah/Podman │
                                  └────────────────┘
```

### Design Principles

- Infrastructure provisioning is managed through the Proxmox API.
- Guest operating system configuration is managed through SSH.
- Each service runs in its own LXC container.
- Sensitive credentials are encrypted using Ansible Vault.
- Rootless Buildah/Podman is used instead of Docker-in-Docker.
- Provisioning and destruction are separated into different playbooks.
- Proxmox operations use the `community.proxmox.proxmox` Ansible module.

## 2. Infrastructure

| Property | Value |
|---|---|
| Hypervisor | Proxmox VE |
| Proxmox Version | 8.4.14 |
| Proxmox Node | `server-home` |
| Proxmox API | `192.168.100.20:8006` |
| Container OS | Debian 12 |
| Container Type | LXC |
| Container Network | `vmbr0` |
| Network Gateway | `192.168.100.1` |
| Storage | `local-lvm` |
| Automation | Ansible |
| Provisioning API | Proxmox REST API |

### Container Resources

| Container | VMID | vCPU | RAM | Swap | Disk |
|---|---:|---:|---:|---:|---:|
| postgres-ci | 201 | 2 | 2048 MB | 512 MB | 16 GB |
| valkey-ci | 202 | 1 | 1024 MB | 512 MB | 8 GB |
| runner-ci | 203 | 2 | 2048 MB | 512 MB | 16 GB |
| runner-build | 204 | 2 | 4096 MB | 1024 MB | 32 GB |

All containers use unprivileged LXC configuration. The build runner additionally enables nesting where required.

## 3. Tested Environment

The project is developed and tested with the following software versions:

| Component | Version |
|---|---|
| Proxmox VE | 8.4.14 |
| Ansible Core | 2.21.1 |
| Ansible Proxmox Collection | 1.6.0 |
| Python (Project Virtual Environment) | 3.12.13 |
| Python (Ansible Runtime) | 3.13.14 |
| Jinja2 | 3.1.6 |
| PyYAML | 6.0.3 |
| Container OS | Debian 12 |

### Python Environment

The project uses a Python virtual environment with Python 3.12.13 for project dependencies.

Ansible Core is installed through the system package environment and currently runs with Python 3.13.14.

The project and Ansible runtime Python interpreters are separate.

**Note:** These are the versions used during development and testing. Other versions may work but have not necessarily been tested.

## 4. Project Structure

```text
.
├── .gitignore
├── README.md
├── ansible.cfg
├── inventory.example.yml
├── requirements.yml
├── proxmox.yml
├── site.yml
├── destroy.yml
│
├── group_vars/
│   ├── proxmox.yml.example
│   └── all/
│       └── vault.yml
│
└── roles/
    ├── common/
    │   └── tasks/
    │       └── main.yml
    │
    ├── proxmox_lxc/
    │   └── tasks/
    │       └── main.yml
    │
    ├── postgres_ci/
    │   ├── tasks/
    │   │   └── main.yml
    │   └── handlers/
    │       └── main.yml
    │
    ├── valkey_ci/
    │   ├── tasks/
    │   │   └── main.yml
    │   └── handlers/
    │       └── main.yml
    │
    ├── gitlab_runner_shell/
    │   ├── defaults/
    │   │   └── main.yml
    │   ├── tasks/
    │   │   └── main.yml
    │   ├── handlers/
    │   │   └── main.yml
    │   └── templates/
    │       └── config.toml.j2
    │
    └── build_runner/
        ├── defaults/
        │   └── main.yml
        └── tasks/
            └── main.yml
```

**Local files excluded from Git:**

- `inventory.yml`
- `group_vars/proxmox.yml`
- `group_vars/all/vault.yml`
- `.venv/`

These files contain environment-specific configuration or sensitive information.

## 5. How the Automation Works

The automation is divided into two main stages.

### Stage 1: Infrastructure Provisioning

The `proxmox.yml` playbook runs on the Ansible control machine.

It uses the Proxmox API to:

1. Validate LXC container declarations.
2. Create or update the declared containers.
3. Configure container resources and networking.
4. Configure SSH public key authentication.
5. Start the containers.

### Stage 2: Guest Configuration

The `site.yml` playbook connects to the containers through SSH.

It performs:

1. Common operating system configuration.
2. PostgreSQL installation and configuration.
3. Valkey installation and configuration.
4. GitLab Runner installation and configuration.
5. Rootless Buildah/Podman installation and configuration.

### Destruction

The `destroy.yml` playbook uses the Proxmox Ansible module to:

1. Stop the declared LXC containers.
2. Remove the declared LXC containers.

Only containers listed in `proxmox_lxc_containers` are targeted.

## 6. Playbooks

| Playbook | Description | Execution |
|---|---|---|
| `proxmox.yml` | Provision and start LXC containers | Local |
| `site.yml` | Configure services inside containers | SSH |
| `destroy.yml` | Stop and delete declared LXC containers | Local |

### Proxmox Provisioning

```bash
ansible-playbook -i inventory.yml proxmox.yml --ask-vault-pass
```

### Guest Configuration

```bash
ansible-playbook -i inventory.yml site.yml --ask-vault-pass
```

### Destroy Infrastructure

```bash
ansible-playbook -i inventory.yml destroy.yml --ask-vault-pass
```

**Warning:** The destroy playbook permanently deletes the declared LXC containers and their associated virtual disks. Verify the VMIDs and configuration before execution.

## 7. Ansible Roles

### common

Provides common operating system configuration for all containers.

Typical responsibilities include:

- Package updates.
- Base utilities.
- Common system settings.

### proxmox_lxc

Manages LXC lifecycle through `community.proxmox.proxmox`.

Responsibilities:

- Validate container declarations.
- Create or update LXC containers.
- Configure CPU, memory, swap, storage, and networking.
- Configure SSH public keys.
- Start containers.

### postgres_ci

Installs and configures PostgreSQL for CI workloads.

### valkey_ci

Installs and configures Valkey as an in-memory data store.

### gitlab_runner_shell

Installs GitLab Runner and configures the Shell executor.

The runner configuration is generated from an Ansible Jinja2 template.

### build_runner

Installs and configures Buildah/Podman for rootless container image builds.

The design avoids requiring a Docker daemon, Docker socket, or Docker-in-Docker.

## 8. Configuration Files

### inventory.yml

Defines the managed LXC hosts and their connection settings.

Create it from the example:

```bash
cp inventory.example.yml inventory.yml
```

The inventory should contain the four container hosts and their corresponding groups.

### group_vars/proxmox.yml

Contains Proxmox API settings and LXC declarations.

Create it from the example:

```bash
cp group_vars/proxmox.yml.example group_vars/proxmox.yml
```

Important variables:

| Variable | Description |
|---|---|
| `proxmox_api_host` | Proxmox API hostname or IP |
| `proxmox_api_user` | Proxmox API user |
| `proxmox_api_token_id` | API token ID |
| `proxmox_api_token_secret` | API token secret from Vault |
| `proxmox_validate_certs` | TLS certificate validation |
| `proxmox_node` | Target Proxmox node |
| `proxmox_lxc_containers` | LXC container declarations |

### group_vars/all/vault.yml

Contains encrypted credentials.

Expected variables include:

```yaml
vault_proxmox_api_token_secret: "YOUR_PROXMOX_TOKEN_SECRET"
vault_lxc_password: "YOUR_LXC_PASSWORD"

gitlab_runner_token: "YOUR_GITLAB_RUNNER_TOKEN"
ci_postgres_password: "YOUR_POSTGRES_PASSWORD"
```

The actual values must be encrypted with Ansible Vault.

## 9. Requirements

### Control Machine

- Linux
- Python 3.12.13 for the project virtual environment
- Ansible Core 2.21.1
- Proxmoxer
- Community Proxmox collection 1.6.0
- Community PostgreSQL collection

### Proxmox VE

- Proxmox VE 8.4.14.
- Proxmox API access.
- Debian 12 LXC template.
- `local-lvm` storage.
- `vmbr0` network bridge.
- API token with sufficient permissions.
- Network connectivity between the control machine and Proxmox.

## 10. Initial Setup

### 10.1 Clone the Repository

```bash
git clone github.com/mqnoy/Proxmox-LXC-CI-stack
cd Proxmox-LXC-CI-stack
```

### 10.2 Create Python Virtual Environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 10.3 Install Ansible Dependencies

```bash
pip install --upgrade pip
pip install ansible-core==2.21.1 proxmoxer
```

Install the required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

### 10.4 Create Local Configuration Files

```bash
cp inventory.example.yml inventory.yml
cp group_vars/proxmox.yml.example group_vars/proxmox.yml
```

Edit both files according to your environment.

### 10.5 Create Ansible Vault

Create the Vault directory if it does not exist:

```bash
mkdir -p group_vars/all
```

Create the encrypted Vault file:

```bash
ansible-vault create group_vars/all/vault.yml
```

Add the required credentials.

To edit the Vault later:

```bash
ansible-vault edit group_vars/all/vault.yml
```

## 11. Proxmox API Configuration

Create a dedicated API user and API token for Ansible automation.

The token requires appropriate permissions for the operations performed by the playbooks.

Typical permissions include:

- VM allocation and auditing.
- VM configuration.
- VM power management.
- Datastore allocation and auditing.
- System auditing.

The required privileges should be scoped to the appropriate Proxmox resources rather than granting unnecessary administrative access.

The API token configuration in `group_vars/proxmox.yml` should use:

```yaml
proxmox_api_user: ansiblex@pve
proxmox_api_token_id: lxc-provisioner
proxmox_api_token_secret: "{{ vault_proxmox_api_token_secret }}"
```

The token ID is only `lxc-provisioner`, not the full `user@realm!tokenid` string.

For a private lab using a self-signed certificate, certificate validation may be disabled:

```yaml
proxmox_validate_certs: false
```

For production environments, use a trusted certificate and enable validation.

## 12. Deploy the Infrastructure

### 12.1 Validate Inventory

```bash
ansible-inventory -i inventory.yml --graph
```

### 12.2 Test SSH Connectivity

```bash
ansible all -i inventory.yml -m ping
```

### 12.3 Provision LXC Containers

```bash
ansible-playbook -i inventory.yml proxmox.yml --ask-vault-pass
```

### 12.4 Configure Services

After the containers are running and reachable:

```bash
ansible-playbook -i inventory.yml site.yml --ask-vault-pass
```

## 13. Verify the Deployment

### PostgreSQL

```bash
ansible postgres_ci -i inventory.yml -m shell \
  -a "systemctl status postgresql"
```

### Valkey

```bash
ansible valkey_ci -i inventory.yml -m shell \
  -a "systemctl status valkey-server"
```

### GitLab Runner

```bash
ansible gitlab_runner -i inventory.yml -m shell \
  -a "gitlab-runner status"
```

### Build Runner

```bash
ansible build_runner -i inventory.yml -m shell \
  -a "podman info"
```

The exact service names and verification commands should match the installed packages and role configuration.

## 14. Destroy the Infrastructure

The `destroy.yml` playbook removes the containers declared in `proxmox_lxc_containers`.

Run:

```bash
ansible-playbook -i inventory.yml destroy.yml --ask-vault-pass
```

Before execution:

1. Verify the Proxmox node.
2. Verify the VMIDs.
3. Confirm that the containers are managed by this project.
4. Ensure that important data has been backed up.

The playbook does not target arbitrary containers outside the declared list.

## 15. Security Considerations

- Never commit `inventory.yml`.
- Never commit `group_vars/proxmox.yml`.
- Never commit `group_vars/all/vault.yml`.
- Use Ansible Vault for sensitive credentials.
- Use a dedicated Proxmox API account and token.
- Grant only the permissions required by the automation.
- Prefer trusted TLS certificates in production.
- Use unprivileged LXC containers.
- Use rootless Buildah/Podman for container image builds.
- Review the VMID declarations before running the destroy playbook.

## 16. Troubleshooting

### Proxmox API Authentication Failed

Verify:

- API username.
- API token ID.
- API token secret.
- Token permissions.
- Proxmox API connectivity on port `8006`.

### LXC Creation Failed

Verify:

- Debian 12 template exists.
- Storage is available.
- VMIDs are not already assigned to unrelated resources.
- Network bridge exists.
- API token has sufficient permissions.

### SSH Connection Failed

Verify:

- Containers are running.
- IP addresses are correct.
- SSH service is available.
- The public key was installed correctly.
- The inventory uses the correct SSH username.

### Ansible Vault Error

Verify that the Vault file exists:

```bash
ls -l group_vars/all/vault.yml
```

Run playbooks with:

```bash
--ask-vault-pass
```

### Missing Ansible Collection

Install dependencies again:

```bash
ansible-galaxy collection install -r requirements.yml
```

## 17. Useful Commands

Check Ansible version:

```bash
ansible --version
```

Check Python version:

```bash
python3 --version
```

List inventory:

```bash
ansible-inventory -i inventory.yml --graph
```

Check syntax:

```bash
ansible-playbook -i inventory.yml proxmox.yml --syntax-check
ansible-playbook -i inventory.yml site.yml --syntax-check
ansible-playbook -i inventory.yml destroy.yml --syntax-check
```

List available tasks:

```bash
ansible-playbook -i inventory.yml site.yml --list-tasks
```

Check Vault file:

```bash
ansible-vault view group_vars/all/vault.yml
```

## 18. References

- Proxmox VE Documentation: https://pve.proxmox.com/pve-docs/
- Ansible Documentation: https://docs.ansible.com/
- Community Proxmox Collection: https://docs.ansible.com/ansible/latest/collections/community/proxmox/
- Ansible Vault: https://docs.ansible.com/ansible/latest/vault_guide/index.html
- GitLab Runner Documentation: https://docs.gitlab.com/runner/
- PostgreSQL Documentation: https://www.postgresql.org/docs/
- Valkey Documentation: https://valkey.io/
- Podman Documentation: https://docs.podman.io/
- Buildah Documentation: https://buildah.io/

---

**Note:** This README describes the intended project architecture. Actual package versions, service names, inventory groups, and role behavior should remain consistent with the Ansible configuration in the repository.