# Proxmox LXC Infrastructure and GitLab CI Connectivity Validation

## 1. Executive Summary

This project provisions CI infrastructure in Proxmox LXC containers
using Ansible and validates connectivity from a GitLab Shell Runner to
three supporting services:

-   PostgreSQL 17
-   Percona Valkey 9.1
-   Mailpit 1.27.4

The infrastructure validation pipeline initially failed because the
PostgreSQL and Valkey services were not available on their expected
network endpoints. The Ansible playbooks were then run to provision and
configure the services. After provisioning, the Shell Runner could reach
all services, authenticate successfully, execute a PostgreSQL query,
receive a successful Valkey `PING` response, and send a test email
through Mailpit over SMTP.

**Final result: both the `test-infrastructure-connectivity` and
`mailpit-connectivity` GitLab CI jobs succeeded.**

The work did not require changes to the CI test logic to resolve the
service availability failures. The underlying issue was the state of the
service containers, which was corrected through Ansible.

------------------------------------------------------------------------

## 2. Architecture Overview

### 2.1 Environment

  --------------------------------------------------------------------------
  Component                   Proxmox CT IP address         Service /
                                                            purpose
  ---------------- --------------------- ------------------ ----------------
  PostgreSQL CI                      201 `192.168.100.21`   PostgreSQL 17,
                                                            TCP 5432

  Valkey CI                          202 `192.168.100.22`   Percona Valkey
                                                            9.1, TCP 6379

  GitLab Shell                       203 `192.168.100.23`   Runs shell-based
  Runner                                                    CI jobs

  GitLab Build                       204 `192.168.100.24`   Rootless Buildah
  Runner                                                    image builds

  Mailpit CI                         205 `192.168.100.25`   Mailpit 1.27.4,
                                                            SMTP 1025 / UI 8025
  --------------------------------------------------------------------------

All listed containers are Debian 12 and unprivileged LXC containers. The
Shell Runner is the source of the connectivity tests described in this
document.

### 2.2 Network flow

``` text
                         GitLab CI/CD
                              |
                              | Shell executor
                              v
                 +--------------------------+
                 | GitLab Shell Runner      |
                 | CT 203                   |
                 | 192.168.100.23           |
                 +--------------------------+
                  /           |            \
        TCP 5432 /   TCP 6379 |             \ TCP 1025
                v             v              v
     +-----------------+ +-----------------+ +-----------------+
     | PostgreSQL CI   | | Valkey CI       | | Mailpit CI      |
     | CT 201          | | CT 202          | | CT 205          |
     | 192.168.100.21  | | 192.168.100.22  | | 192.168.100.25  |
     | PostgreSQL 17   | | Percona Valkey  | | Mailpit 1.27.4  |
     +-----------------+ +-----------------+ +-----------------+
```

### 2.3 Relevant network settings

  Setting                 Value
  ----------------------- -----------------------
  LXC network             `192.168.100.0/24`
  Gateway                 `192.168.100.1`
  PostgreSQL endpoint     `192.168.100.21:5432`
  Valkey endpoint         `192.168.100.22:6379`
  Mailpit SMTP endpoint   `192.168.100.25:1025`
  Mailpit Web UI endpoint `192.168.100.25:8025`
  Shell Runner endpoint   `192.168.100.23`

------------------------------------------------------------------------

## 3. Ansible Project Structure

The Ansible project is located at:

``` text
~/Developments/DevOps/ansible
```

The project uses an inventory file named `inventory.yml`, a top-level
`site.yml`, and role-based configuration.

A simplified structure:

``` text
ansible/
├── ansible.cfg
├── inventory.yml
├── site.yml
├── group_vars/
│   └── all/
│       ├── main.yml
│       └── vault.yml
└── roles/
    ├── common/
    ├── postgres_ci/
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   └── handlers/main.yml
    ├── valkey_ci/
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   └── handlers/main.yml
    ├── gitlab_runner_shell/
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/config.toml.j2
    ├── gitlab_build_runner/
    └── mailpit_ci/
        ├── defaults/main.yml
        ├── tasks/main.yml
        ├── handlers/main.yml
        └── templates/
            ├── mailpit.env.j2
            └── mailpit.service.j2
```

### 3.1 Inventory groups

The inventory groups used for the relevant services and runner are:

``` yaml
all:
  children:
    postgres_ci:
      hosts:
        postgres-ci:
          ansible_host: 192.168.100.21

    valkey_ci:
      hosts:
        valkey-ci:
          ansible_host: 192.168.100.22

    gitlab_runner_shell:
      hosts:
        gitlab-runner-shell-01:
          ansible_host: 192.168.100.23

    gitlab_build_runner:
      hosts:
        gitlab-runner-build-01:
          ansible_host: 192.168.100.24

    mailpit_ci:
      hosts:
        mailpit-ci:
          ansible_host: 192.168.100.25
```

### 3.2 Shared variables

The shared variables in `group_vars/all/main.yml` include:

``` yaml
---
ansible_user: ansible
ansible_become: true
ansible_python_interpreter: /usr/bin/python3

ci_postgres_host: postgres-ci
ci_postgres_port: 5432

ci_valkey_host: valkey-ci
ci_valkey_port: 6379

ci_mailpit_host: mailpit-ci
ci_mailpit_smtp_port: 1025
ci_mailpit_ui_port: 8025
```

Secrets, including database passwords, GitLab Runner authentication
tokens, and Mailpit credentials, are stored in the encrypted Ansible
Vault file:

``` text
group_vars/all/vault.yml
```

**Security:** Never commit plaintext secrets or paste passwords and
runner tokens into logs, tickets, or chat. If a runner authentication
token has been exposed, revoke/rotate it in GitLab and update the
encrypted Ansible Vault value.

### 3.3 Main playbook

The top-level playbook applies roles to the corresponding inventory
groups:

``` yaml
---
- name: Configure shared LXC prerequisites
  hosts: all
  become: true
  roles:
    - common

- name: Configure PostgreSQL CI service
  hosts: postgres_ci
  become: true
  roles:
    - postgres_ci

- name: Configure Valkey CI service
  hosts: valkey_ci
  become: true
  roles:
    - valkey_ci

- name: Configure GitLab shell runner
  hosts: gitlab_runner_shell
  become: true
  roles:
    - gitlab_runner_shell

- name: Configure rootless image build runner
  hosts: gitlab_build_runner
  become: true
  roles:
    - gitlab_build_runner

- name: Configure Mailpit CI service
  hosts: mailpit_ci
  become: true
  roles:
    - mailpit_ci
```

------------------------------------------------------------------------

## 4. PostgreSQL CI Service

### 4.1 Configuration

The relevant values from `roles/postgres_ci/defaults/main.yml` are:

``` yaml
---
ci_postgres_version: "17"
ci_postgres_port: 5432
ci_postgres_listen_address: "127.0.0.1,192.168.100.21"

ci_postgres_database: "ci_database"
ci_postgres_user: "ci_user"
```

The database password is provided separately through Ansible Vault. Do
not store it in this documentation or in a plaintext defaults file.

### 4.2 Provisioning behavior

The PostgreSQL role performs the following tasks:

1.  Installs repository prerequisites.
2.  Adds the PostgreSQL PGDG repository and signing key.
3.  Installs PostgreSQL 17 and its client package.
4.  Configures PostgreSQL to listen on localhost and the LXC service IP.
5.  Adds a `pg_hba.conf` rule for the CI subnet and the CI database
    user.
6.  Enables and starts PostgreSQL.
7.  Installs the Python PostgreSQL driver used by Ansible modules.
8.  Creates the CI database and user.
9.  Grants the CI user database privileges.
10. Checks local PostgreSQL readiness.

The relevant authentication rule is:

``` text
host ci_database ci_user 192.168.100.0/24 scram-sha-256
```

The role configures the listener with:

``` text
listen_addresses = '127.0.0.1,192.168.100.21'
```

### 4.3 Service verification

The PostgreSQL service was confirmed to be listening on both:

``` text
127.0.0.1:5432
192.168.100.21:5432
```

The Debian `postgresql.service` unit showed `active (exited)`. This is
expected for the Debian service wrapper in this environment; the actual
PostgreSQL server process was running and holding the listening sockets.

The following checks were used:

``` bash
ansible postgres_ci -i inventory.yml -b --ask-vault-pass \
  -m shell \
  -a 'systemctl status postgresql --no-pager; ss -lntp | grep 5432'
```

Runner-side readiness check:

``` bash
ansible gitlab_runner_shell -i inventory.yml -b --ask-vault-pass \
  -m shell \
  -a 'pg_isready -h 192.168.100.21 -p 5432 -t 5'
```

Successful result:

``` text
192.168.100.21:5432 - accepting connections
```

### 4.4 PostgreSQL CI variables

Configure these as GitLab CI/CD variables:

  Variable                 Value
  ------------------------ ----------------------------------------
  `CI_POSTGRES_HOST`       `192.168.100.21`
  `CI_POSTGRES_PORT`       `5432`
  `CI_POSTGRES_DB`         `ci_database`
  `CI_POSTGRES_USER`       `ci_user`
  `CI_POSTGRES_PASSWORD`   The matching secret from Ansible Vault

Mark the password variable as masked and protected according to the
project's GitLab branch and pipeline policy.

------------------------------------------------------------------------

## 5. Valkey CI Service

### 5.1 Configuration

The relevant values from `roles/valkey_ci/defaults/main.yml` are:

``` yaml
---
ci_valkey_port: 6379
ci_valkey_listen_address: "127.0.0.1 192.168.100.22"

ci_valkey_package: percona-valkey-server
ci_valkey_service: valkey
ci_valkey_config_path: /etc/valkey/valkey.conf
```

The Valkey password is supplied through Ansible Vault and is not
included here.

### 5.2 Provisioning behavior

The Valkey role performs the following tasks:

1.  Installs the Percona repository package.
2.  Enables the Percona Valkey 9.1 release repository.
3.  Refreshes the APT cache.
4.  Installs `percona-valkey-server`.
5.  Creates a systemd drop-in to relax selected namespace restrictions
    for the unprivileged LXC environment.
6.  Reloads systemd configuration.
7.  Configures Valkey to listen on localhost and `192.168.100.22`.
8.  Validates that the Valkey password exists and meets the role's
    minimum length requirement.
9.  Configures `requirepass`.
10. Restarts Valkey when configuration changes.
11. Enables and starts the service.
12. Authenticates locally and checks that `PING` returns `PONG`.

The systemd unit actually reported by the installed package is:

``` text
valkey-server.service
```

The service process reported:

``` text
Ready to accept connections
```

The service listened on:

``` text
127.0.0.1:6379
192.168.100.22:6379
```

The role's `ci_valkey_service` default is `valkey`, while the systemd
unit shown in the service status is `valkey-server.service`. The current
playbook completed successfully and the service is running. For clarity
and long-term maintainability, consider aligning the variable and
handler with the actual unit name after confirming the package's unit
aliases on the target system.

### 5.3 Valkey CI variables

Configure these as GitLab CI/CD variables:

  Variable               Value
  ---------------------- ----------------------------------------
  `CI_VALKEY_HOST`       `192.168.100.22`
  `CI_VALKEY_PORT`       `6379`
  `CI_VALKEY_PASSWORD`   The matching secret from Ansible Vault

Use a masked GitLab variable for the password. The CI script exports it
as `REDISCLI_AUTH`, avoiding putting the password directly in the
command-line arguments.

------------------------------------------------------------------------

## 6. Mailpit CI Service

### 6.1 Configuration

The relevant values from `roles/mailpit_ci/defaults/main.yml` are:

``` yaml
---
ci_mailpit_version: "1.27.4"
ci_mailpit_user: "mailpit"
ci_mailpit_group: "mailpit"
ci_mailpit_home: "/var/lib/mailpit"

ci_mailpit_smtp_bind: "192.168.100.25:1025"
ci_mailpit_ui_bind: "192.168.100.25:8025"

ci_mailpit_db_file: "/var/lib/mailpit/mailpit.db"
ci_mailpit_max_messages: 500
```

The Web UI and SMTP authentication credentials are provided separately
through Ansible Vault (`vault_mailpit_ui_username`,
`vault_mailpit_ui_password`, `vault_mailpit_smtp_username`,
`vault_mailpit_smtp_password`). Do not store them in plaintext defaults
or documentation.

### 6.2 Provisioning behavior

The Mailpit role performs the following tasks:

1.  Installs prerequisites (`curl`, `tar`).
2.  Creates the dedicated `mailpit` system user and group.
3.  Ensures data and configuration directories exist (`/var/lib/mailpit`,
    `/etc/mailpit`).
4.  Downloads and extracts the official Mailpit binary to
    `/usr/local/bin/mailpit`.
5.  Templates the environment file `/etc/mailpit/mailpit.env` with
    configured listen addresses, database path, and Vault credentials.
6.  Deploys a hardened systemd service unit (`mailpit.service`).
7.  Reloads systemd configuration.
8.  Enables and starts the Mailpit service.
9.  Verifies service readiness via authenticated HTTP API check against
    `http://192.168.100.25:8025/api/v1/info`.

The systemd unit for the service is:

``` text
mailpit.service
```

The service listened on:

``` text
192.168.100.25:1025
192.168.100.25:8025
```

### 6.3 Mailpit CI variables

Configure these as GitLab CI/CD variables:

  Variable               Value
  ---------------------- ----------------------------------------
  `MAIL_HOST`         `192.168.100.25`
  `MAIL_PORT`         `1025`
  `MAIL_USERNAME`     The matching SMTP username from Vault
  `MAIL_PASSWORD`     The matching SMTP password from Vault

Use a masked GitLab variable for the password.

------------------------------------------------------------------------

## 7. GitLab Shell Runner

### 7.1 Runner details

  Property     Value
  ------------ --------------------------
  Container    `gitlab-runner-shell-01`
  Proxmox CT   203
  IP address   `192.168.100.23`
  OS           Debian 12
  CPU          2 cores
  Memory       2048 MB
  Disk         16 GB
  Executor     `shell`
  Shell        `bash`

The runner was installed and registered using the Ansible role.

### 7.2 Runner verification

The following command verified the runner service, registration, and
GitLab connectivity:

``` bash
ansible gitlab_runner_shell -i inventory.yml -b --ask-vault-pass \
  -m shell \
  -a 'systemctl is-active gitlab-runner && gitlab-runner list && gitlab-runner verify'
```

The observed status was:

-   `gitlab-runner` service: active
-   Runner listed in `/etc/gitlab-runner/config.toml`
-   GitLab runner verification: valid

### 7.3 Installed CI tools

The Shell Runner role installs the tools required by the connectivity
test. The package list includes:

``` yaml
gitlab_runner_packages:
  - git
  - curl
  - ca-certificates
  - postgresql-client
  - redis-tools
```

The installed clients were verified:

``` text
psql (PostgreSQL) 15.19
pg_isready (PostgreSQL) 15.19
redis-cli 7.0.15
```

These are client tools; the PostgreSQL server itself runs in CT201, Valkey
runs in CT202, and Mailpit runs in CT205.

### 7.4 Runner tags

The CI job uses the tag:

``` yaml
tags:
  - shell
```

The runner must have a matching `shell` tag configured in GitLab. Runner
tags are managed in the GitLab UI in this setup rather than being
overwritten by the Ansible config template.

------------------------------------------------------------------------

## 8. GitLab CI Connectivity Tests

The connectivity jobs validate both the network path and service-level
authentication from the Shell Runner to PostgreSQL, Valkey, and
Mailpit.

The first job tests PostgreSQL and Valkey:
``` yaml
test-infrastructure-connectivity:
  stage: test
  tags:
    - shell

  script:
    - |
      set -eu

      echo "=================================="
      echo "Testing PostgreSQL connectivity"
      echo "=================================="

      pg_isready \
        -h "$CI_POSTGRES_HOST" \
        -p "$CI_POSTGRES_PORT" \
        -t 5

      PGPASSWORD="$CI_POSTGRES_PASSWORD" \
      psql \
        -h "$CI_POSTGRES_HOST" \
        -p "$CI_POSTGRES_PORT" \
        -U "$CI_POSTGRES_USER" \
        -d "$CI_POSTGRES_DB" \
        -v ON_ERROR_STOP=1 \
        -c "SELECT 1 AS postgres_connection_test;"

      echo "PostgreSQL authentication: OK"

      echo "=================================="
      echo "Testing Valkey connectivity"
      echo "=================================="

      export REDISCLI_AUTH="$CI_VALKEY_PASSWORD"

      VALKEY_RESULT=$(
        redis-cli \
          -h "$CI_VALKEY_HOST" \
          -p "$CI_VALKEY_PORT" \
          PING
      )

      test "$VALKEY_RESULT" = "PONG"

      unset REDISCLI_AUTH

      echo "Valkey authentication: OK"
      echo "=================================="
      echo "All infrastructure tests passed"
      echo "=================================="
```

The second job validates Mailpit SMTP connectivity, authentication, and
email delivery using a Python script:

``` yaml
mailpit-connectivity:
  stage: test
  tags:
    - shell

  variables:
    PYTHONDONTWRITEBYTECODE: "1"

  script:
    - |
      python3 - <<'PY'
      import os
      import smtplib
      import socket
      from email.message import EmailMessage

      host = os.environ["MAIL_HOST"]
      port = int(os.environ.get("MAIL_PORT", "1025"))
      username = os.environ["MAIL_USERNAME"]
      password = os.environ["MAIL_PASSWORD"]

      print(f"Testing Mailpit connectivity: {host}:{port}")

      try:
          with smtplib.SMTP(host, port, timeout=10) as smtp:
              smtp.ehlo()
              print("[OK] SMTP connection established")

              smtp.login(username, password)
              print("[OK] SMTP authentication successful")

              message = EmailMessage()
              message["Subject"] = "GitLab CI - Mailpit Connectivity Test"
              message["From"] = "ci-test@example.local"
              message["To"] = "mailpit-test@example.local"
              message.set_content(
                  "Mailpit connectivity test successful.\n"
                  f"Pipeline: {os.environ.get('CI_PIPELINE_URL', 'local')}\n"
                  f"Job: {os.environ.get('CI_JOB_ID', 'local')}\n"
              )

              smtp.send_message(message)
              print("[OK] Test email sent successfully")

      except (smtplib.SMTPException, socket.error, OSError) as error:
          print(f"[FAILED] Mailpit test failed: {type(error).__name__}: {error}")
          raise SystemExit(1)

      print("Mailpit integration test completed successfully.")
      PY
```

Ensure that `test` is included in the pipeline's top-level `stages:`
list if the project explicitly defines stages.

### 8.1 What the jobs verify

**PostgreSQL**
-   `pg_isready` checks that the server accepts connections.
-   `psql` connects using the configured host, port, database, username,
    and password.
-   `SELECT 1` confirms that an authenticated SQL query can be executed.

**Valkey**

-   `redis-cli` connects to the Valkey endpoint.
-   `REDISCLI_AUTH` supplies the password.
-   `PING` must return `PONG`.
-   The shell test fails if the returned value is not `PONG`.

**Mailpit**

-   `smtplib.SMTP` establishes a TCP connection to port 1025.
-   `ehlo()` performs standard SMTP handshake.
-   `login()` validates SMTP basic authentication credentials.
-   `send_message()` confirms that an email message is successfully
    delivered and stored in Mailpit.

This is stronger than a simple TCP port check: it validates actual
application-protocol access and authentication.
------------------------------------------------------------------------

## 9. Troubleshooting Timeline

### 9.1 Initial PostgreSQL failure

The first pipeline attempt reported:

``` text
192.168.100.21:5432 - no response
ERROR: Job failed: exit status 2
```

An initial service inspection reported:

``` text
Unit postgresql.service could not be found.
```

The `pg_lsclusters` utility was also unavailable at that time. A
separate socket inspection returned no listener on port 5432.

**Diagnosis:** PostgreSQL was not yet available on the expected
endpoint.

**Resolution:** The PostgreSQL Ansible playbook was run against the
PostgreSQL inventory host:

``` bash
ansible-playbook -i inventory.yml site.yml \
  --ask-vault-pass \
  --limit postgres-ci
```

After provisioning, PostgreSQL was active and listening on the expected
service IP.

### 9.2 PostgreSQL network and authentication validation

The runner-side `pg_isready` check returned:

``` text
192.168.100.21:5432 - accepting connections
```

The subsequent CI job successfully executed:

``` sql
SELECT 1 AS postgres_connection_test;
```

The output showed one row with the value `1`, followed by:

``` text
PostgreSQL authentication: OK
```

This confirmed that PostgreSQL connectivity and database authentication
were working.

### 9.3 Initial Valkey failure

The next pipeline attempt failed with:

``` text
Could not connect to Redis at 192.168.100.22:6379: Connection refused
```

Service inspection showed:

``` text
Unit valkey.service could not be found.
Unit valkey-server.service could not be found.
```

No process was listening on port 6379, and the runner's TCP connection
attempt was refused.

**Diagnosis:** Valkey was not available on the expected endpoint at that
time.

**Resolution:** The Valkey Ansible role was applied:

``` bash
ansible-playbook -i inventory.yml site.yml \
  --ask-vault-pass \
  --limit valkey-ci
```

The play recap reported:

``` text
valkey-ci : ok=21 changed=10 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Subsequent inspection showed the Valkey server running and listening on
both the loopback and LXC service IP.

### 9.4 Valkey TCP validation

The Shell Runner successfully connected to:

``` text
192.168.100.22:6379
```

The `nc` output ended with:

``` text
6379 (redis) open
```

This confirmed TCP reachability, but did not by itself prove
authentication. The final CI job performed the authenticated `PING`
test.

### 9.5 Final successful pipeline

The final pipeline output for `test-infrastructure-connectivity`
included:

``` text
Testing PostgreSQL connectivity
192.168.100.21:5432 - accepting connections

 postgres_connection_test
--------------------------
                        1
(1 row)

PostgreSQL authentication: OK

Testing Valkey connectivity
Valkey authentication: OK

All infrastructure tests passed

Job succeeded
```

### 9.6 Mailpit deployment and SMTP verification

The Mailpit CI role was provisioned:

``` bash
ansible-playbook -i inventory.yml site.yml \
  --ask-vault-pass \
  --limit mailpit-ci
```

The systemd service was verified active and listening on ports 1025 and
8025. The `mailpit-connectivity` GitLab CI job was executed from the
Shell Runner using Python `smtplib`, successfully connecting,
authenticating, and sending a test email:

``` text
Testing Mailpit connectivity: 192.168.100.25:1025
[OK] SMTP connection established
[OK] SMTP authentication successful
[OK] Test email sent successfully
Mailpit integration test completed successfully.
Job succeeded
```

**Final diagnosis:** The CI test jobs were functioning. The initial
failures were caused by the PostgreSQL and Valkey services not being
available on their expected endpoints. Applying the Ansible service
roles brought PostgreSQL, Valkey, and Mailpit into the required state.
No change to the test jobs was necessary to resolve those service
availability failures.
------------------------------------------------------------------------

## 10. Useful Operational Commands

Run these from the Ansible project directory.

### 10.1 Provision PostgreSQL

``` bash
ansible-playbook -i inventory.yml site.yml \
  --ask-vault-pass \
  --limit postgres-ci
```

### 10.2 Provision Valkey

``` bash
ansible-playbook -i inventory.yml site.yml \
  --ask-vault-pass \
  --limit valkey-ci
```

### 10.3 Provision Mailpit

``` bash
ansible-playbook -i inventory.yml site.yml \
  --ask-vault-pass \
  --limit mailpit-ci
```

### 10.4 Provision the Shell Runner

``` bash
ansible-playbook -i inventory.yml site.yml \
  --ask-vault-pass \
  --limit gitlab-runner-shell-01
```

### 10.5 Verify PostgreSQL service and listener

``` bash
ansible postgres_ci -i inventory.yml -b --ask-vault-pass \
  -m shell \
  -a 'systemctl status postgresql --no-pager; ss -lntp | grep 5432'
```

### 10.6 Verify PostgreSQL readiness from the runner

``` bash
ansible gitlab_runner_shell -i inventory.yml -b --ask-vault-pass \
  -m shell \
  -a 'pg_isready -h 192.168.100.21 -p 5432 -t 5'
```

### 10.7 Verify Valkey service and listener

The observed systemd unit is `valkey-server.service`:

``` bash
ansible valkey_ci -i inventory.yml -b --ask-vault-pass \
  -m shell \
  -a 'systemctl status valkey-server --no-pager; ss -lntp | grep 6379'
```

### 10.8 Verify Valkey TCP connectivity from the runner

``` bash
ansible gitlab_runner_shell -i inventory.yml -b --ask-vault-pass \
  -m shell \
  -a 'nc -vz -w 5 192.168.100.22 6379'
```

### 10.9 Verify Mailpit service and listener

The systemd unit is `mailpit.service`:

``` bash
ansible mailpit_ci -i inventory.yml -b --ask-vault-pass \
  -m shell \
  -a 'systemctl status mailpit --no-pager; ss -lntp | grep -E "(1025|8025)"'
```

### 10.10 Verify Mailpit HTTP API from the runner

``` bash
ansible gitlab_runner_shell -i inventory.yml -b --ask-vault-pass \
  -m shell \
  -a 'curl -fsSL http://192.168.100.25:8025/api/v1/info'
```

### 10.11 Run the complete site playbook

``` bash
ansible-playbook -i inventory.yml site.yml --ask-vault-pass
```

------------------------------------------------------------------------

## 11. Security and Operational Recommendations

1.  **Keep secrets in Ansible Vault and GitLab CI/CD variables.** Do not
    place passwords in source-controlled defaults, plaintext YAML, shell
    command arguments, or job output.
2.  **Mask and protect GitLab secret variables.** Apply protection in a
    way that matches the branches and environments that are allowed to
    run the job.
3.  **Rotate exposed runner tokens.** If an authentication token has
    appeared in terminal output, logs, or chat, revoke or rotate it in
    GitLab and update the Vault value.
4.  **Restrict database and cache access.** The current PostgreSQL
    access rule is scoped to the CI subnet. Keep network exposure
    limited to the required LXC network and clients.
5.  **Retain authenticated checks.** TCP checks such as `nc` are useful
    for diagnosing network reachability, but they do not replace
    PostgreSQL queries, authenticated Valkey commands, or SMTP
    handshakes.
6.  **Keep infrastructure changes in Ansible.** Avoid one-off manual
    changes inside containers so that the declared configuration remains
    reproducible.
7.  **Review the Valkey systemd service variable.** The observed unit is
    `valkey-server.service`; verify aliases and consider using the exact
    unit name consistently in role defaults and handlers.
8.  **Consider pinning package versions.** If deterministic rebuilds are
    important, pin or otherwise control package versions and repository
    changes.
9.  **Keep CI jobs small and diagnostic.** Separate infrastructure
    connectivity checks from application integration tests so failures
    are easier to isolate.
10. **Protect Mailpit UI and SMTP credentials.** Keep web UI and SMTP
    passwords in Ansible Vault and supply them to CI jobs via masked
    GitLab CI/CD variables.

------------------------------------------------------------------------

## 12. Current Status and Next Steps

### 12.1 Completed

-   [x] Proxmox LXC service and runner containers provisioned.
-   [x] PostgreSQL 17 installed and configured through Ansible.
-   [x] PostgreSQL listens on `192.168.100.21:5432`.
-   [x] CI database `ci_database` and user `ci_user` configured by the
    PostgreSQL role.
-   [x] PostgreSQL authenticated query from the Shell Runner succeeded.
-   [x] Percona Valkey 9.1 installed and configured through Ansible.
-   [x] Valkey listens on `192.168.100.22:6379`.
-   [x] Shell Runner can reach the Valkey TCP endpoint.
-   [x] Authenticated Valkey `PING` test succeeded in GitLab CI.
-   [x] Mailpit 1.27.4 installed and configured through Ansible.
-   [x] Mailpit listens on `192.168.100.25:1025` and `8025`.
-   [x] Authenticated Mailpit SMTP test succeeded in GitLab CI.
-   [x] Final `test-infrastructure-connectivity` and
    `mailpit-connectivity` jobs succeeded.

### 12.2 Recommended next steps

1.  Add Go application integration tests that use the actual PostgreSQL,
    Valkey, and Mailpit SMTP endpoints used by the application.
2.  Run those tests from GitLab CI using the same CI service endpoints
    and credentials.
3.  Add health checks and clear failure diagnostics to the service
    roles.
4.  Review the Valkey systemd unit naming consistency.
5.  Continue with the build and deployment pipeline for the Go
    application and the ARM64 K3s target.

------------------------------------------------------------------------

## 13. Final Conclusion

The Proxmox LXC-based CI environment now has a working GitLab Shell
Runner connection to PostgreSQL, Valkey, and Mailpit.

The final pipeline validated:

-   Network reachability to PostgreSQL, Valkey, and Mailpit.
-   PostgreSQL authentication and SQL query execution.
-   Valkey authentication and Redis-compatible `PING` behavior.
-   Mailpit SMTP authentication and email delivery.

The initial failures were resolved by provisioning and configuring the
service containers with Ansible. The successful final pipeline provides
a verified baseline for the next stage: Go application integration
testing.

**Overall status: PostgreSQL, Valkey, and Mailpit infrastructure
connectivity validation is complete and successful.**
