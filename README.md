# OpenELIS on Ansible

Ansible playbooks that provision Linux hosts and deploy standalone
[OpenELIS Global 2](https://github.com/DIGI-UW/OpenELIS-Global-2) instances
on each one, driven by the compose stack that lives in that upstream
repository.

## Overview

Each managed host runs the full OpenELIS stack (database, webapp, FHIR
server, frontend, nginx proxy, cert bootstrap) as defined by
`docker-compose.yml` in `DIGI-UW/OpenELIS-Global-2`. Ansible:

1. Bootstraps the OS (packages, timezone, firewall).
2. Installs Docker Engine and the Compose plugin.
3. Clones (or updates) the OpenELIS repo at a configurable ref into
   `/opt/openelis` on each host.
4. Renders a per-host `.env` file consumed by that compose stack.
5. Brings the stack up with `community.docker.docker_compose_v2`.

## Project Structure

```
.
├── README.md
├── ansible.cfg
├── inventory/
│   └── hosts.yml             # openElis01, openElis02, openElis03
├── group_vars/
│   └── all.yml               # Repo URL, ref, install dir, default .env values
├── host_vars/
│   ├── openElis01.yml        # Per-host env overrides
│   ├── openElis02.yml
│   └── openElis03.yml
├── playbooks/
│   ├── site.yml              # Full provisioning entrypoint
│   ├── bootstrap.yml         # Base packages, timezone, firewall
│   ├── docker.yml            # Install Docker + Compose plugin
│   ├── deploy.yml            # Clone repo, render .env, bring stack up
│   ├── update.yml            # git pull, image pull, recreate
│   └── down.yml              # Stop and remove the stack
└── roles/
    ├── common/               # OS baseline
    ├── docker/               # Docker engine + compose plugin install
    └── openelis/             # Clone OpenELIS repo, render .env, run compose
```

## Managed Hosts

| Host         | Role              |
|--------------|-------------------|
| openElis01   | OpenELIS instance |
| openElis02   | OpenELIS instance |
| openElis03   | OpenELIS instance |

Hostnames and IPs are set in `inventory/hosts.yml`. Each host runs the
same compose stack independently (no inter-host dependencies).

## Requirements

- Ansible 2.14+ on the control node
- Python 3.9+ on all managed hosts
- SSH key-based access to each host as a user with sudo
- Ubuntu 22.04 / Debian 12 (adjust `roles/docker` for other distros)

Install the required collection:

```
ansible-galaxy collection install community.docker
```

## Quick Start

1. Set `ansible_host` for each host in `inventory/hosts.yml`.
2. Confirm connectivity:

   ```
   ansible all -m ping
   ```

3. Provision everything:

   ```
   ansible-playbook playbooks/site.yml
   ```

## Common Operations

Deploy or redeploy the stack on every host:

```
ansible-playbook playbooks/deploy.yml
```

Pull repo changes and newer images, then recreate containers:

```
ansible-playbook playbooks/update.yml
```

Stop the stack (containers removed, named volumes preserved):

```
ansible-playbook playbooks/down.yml
```

Target a single host:

```
ansible-playbook playbooks/deploy.yml --limit openElis02
```

## Configuration

`group_vars/all.yml` defines the shared defaults:

- `openelis_repo_url` — git URL of OpenELIS-Global-2 (default: the
  DIGI-UW fork).
- `openelis_repo_ref` — branch, tag, or commit to deploy
  (default: `develop`).
- `openelis_install_dir` — where the repo is cloned on each host
  (default: `/opt/openelis`).
- `openelis_env` — key/value map rendered into
  `{{ openelis_install_dir }}/.env` and consumed by the compose file.
  Keys mirror the variables referenced in the upstream
  `docker-compose.yml` (`OE_HTTP_PORT`, `OE_DB_PASSWORD`,
  `SSL_KEYSTORE_PASSWORD`, `OE_ENCRYPTION_PASSWORD`, etc.).

Per-host overrides go in `host_vars/<hostname>.yml`. Anything set under
`openelis_env` there merges on top of the group defaults, so a single
host can run on a different port or with a different DB password without
duplicating the whole block.

## Secrets

Do not commit plaintext credentials. Use Ansible Vault:

```
ansible-vault create group_vars/all/vault.yml
ansible-playbook playbooks/site.yml --ask-vault-pass
```

The default `openelis_env` references vault variables such as
`vault_oe_db_password`, `vault_ssl_keystore_password`,
`vault_ssl_truststore_password`, `vault_oe_admin_password`, and
`vault_oe_encryption_password`. Define them in the vaulted file.

## Troubleshooting

- Verbose output: `ansible-playbook ... -vvv`.
- Preview changes: `--check --diff`.
- Inspect the stack on a host:

  ```
  ssh openElis01 "cd /opt/openelis && docker compose ps"
  ```

- Logs for a service:

  ```
  ssh openElis01 "cd /opt/openelis && docker compose logs oe.openelis.org"
  ```

