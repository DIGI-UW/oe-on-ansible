# Ansible Docker Compose Management

A simple Ansible setup that provisions three Linux servers and manages Docker Compose based applications on them.

## Overview

This project uses Ansible to:

1. Bootstrap three remote servers with common packages and hardened SSH.
2. Install Docker Engine and the Docker Compose plugin.
3. Deploy per-host `docker-compose.yml` stacks (web, database, monitoring).
4. Provide idempotent playbooks for updates, restarts, and teardown.

## Project Structure

```
ansible/
├── README.md
├── ansible.cfg               # Ansible defaults (inventory path, SSH, roles path)
├── inventory/
│   └── hosts.yml             # Three managed servers grouped by role
├── group_vars/
│   └── all.yml               # Variables shared across all hosts
├── host_vars/
│   ├── web01.yml             # Per-host overrides
│   ├── db01.yml
│   └── monitor01.yml
├── playbooks/
│   ├── site.yml              # Full provisioning entrypoint
│   ├── bootstrap.yml         # Base packages, users, SSH
│   ├── docker.yml            # Install Docker + Compose plugin
│   ├── deploy.yml            # Render compose files and bring stacks up
│   ├── update.yml            # Pull new images and recreate containers
│   └── down.yml              # Stop and remove stacks
├── roles/
│   ├── common/               # OS baseline (packages, timezone, users)
│   ├── docker/               # Docker engine + compose plugin install
│   └── app/                  # Per-host compose stack management
└── files/
    └── compose/              # Static compose fragments referenced by role
```

## Managed Servers

| Host       | Role       | Compose Stack                          |
|------------|------------|----------------------------------------|
| web01      | web        | nginx + application container          |
| db01       | database   | postgres + backup sidecar              |
| monitor01  | monitoring | prometheus + grafana + node exporter   |

Hostnames, IPs, and SSH users are defined in `inventory/hosts.yml`. Override per-host settings in `host_vars/<hostname>.yml`.

## Requirements

- Ansible 2.14+ on the control node
- Python 3.9+ on all managed hosts
- SSH key-based access to all three servers as a user with sudo
- Ubuntu 22.04 / Debian 12 (adjust `roles/docker` for other distros)

Install collections used by the docker role:

```
ansible-galaxy collection install community.docker
```

## Quick Start

1. Edit `inventory/hosts.yml` and set the `ansible_host` addresses and `ansible_user` for each server.
2. Confirm connectivity:

   ```
   ansible all -m ping
   ```

3. Run the full provisioning playbook:

   ```
   ansible-playbook playbooks/site.yml
   ```

## Common Operations

Provision everything from scratch:

```
ansible-playbook playbooks/site.yml
```

Only install Docker on new hosts:

```
ansible-playbook playbooks/docker.yml --limit web01
```

Deploy or update compose stacks:

```
ansible-playbook playbooks/deploy.yml
```

Pull newer images and recreate containers:

```
ansible-playbook playbooks/update.yml
```

Bring stacks down (containers stopped, data preserved):

```
ansible-playbook playbooks/down.yml
```

Target a single host or group:

```
ansible-playbook playbooks/deploy.yml --limit database
```

## Variables

Global defaults live in `group_vars/all.yml`. Common values you may want to change:

- `timezone`
- `docker_users` (users added to the `docker` group)
- `compose_project_root` (where compose files are rendered on each host)
- `app_env` (environment name passed into templates)

Per-host stacks are declared in `host_vars/<hostname>.yml` under the `compose_stack` key. The `app` role renders that structure into `docker-compose.yml` and calls the `community.docker.docker_compose_v2` module.

## Secrets

Do not commit plaintext credentials. Use Ansible Vault:

```
ansible-vault encrypt group_vars/all/vault.yml
ansible-playbook playbooks/site.yml --ask-vault-pass
```

Reference vaulted values from unencrypted vars files (e.g. `db_password: "{{ vault_db_password }}"`).

## Troubleshooting

- Increase verbosity with `-vvv` to see SSH and module output.
- Use `--check --diff` to preview changes before applying.
- If Docker install fails, verify the host distro matches the repo added in `roles/docker/tasks/main.yml`.
- To inspect a stack on a host: `ssh <host> "cd /opt/compose/<stack> && docker compose ps"`.

## License

Internal tooling. Adapt freely within your organization.
