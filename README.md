# OpenELIS on Ansible

Ansible playbooks that provision Linux hosts and deploy standalone
[OpenELIS Global 2](https://github.com/DIGI-UW/OpenELIS-Global-2) instances
on each one, driven by the compose stack that lives in that upstream
repository. A separate monitoring host runs Prometheus, Blackbox exporter,
Alertmanager, and Grafana to watch the OpenELIS hosts.

## Overview

Each OpenELIS host runs the full OpenELIS stack (database, webapp, FHIR
server, frontend, nginx proxy, cert bootstrap) as defined by
`docker-compose.yml` in `DIGI-UW/OpenELIS-Global-2`, plus a
`node_exporter` sidecar for host metrics. One monitoring host runs the
Prometheus stack and probes the OpenELIS hosts. Ansible:

1. Bootstraps the OS (packages, timezone, firewall).
2. Installs Docker Engine and the Compose plugin.
3. On OpenELIS hosts: clones the OpenELIS repo at a configurable ref into
   `/opt/openelis`, renders a per-host `.env`, brings the stack up, and
   starts `node_exporter` on port 9100 (firewalled to monitoring hosts only).
4. On the monitoring host: renders Prometheus/Blackbox/Alertmanager/Grafana
   configs into `/opt/monitoring`, brings that stack up, and opens the
   Grafana/Prometheus/Alertmanager ports through UFW.

## Project Structure

```
.
├── README.md
├── ansible.cfg
├── inventory/
│   └── hosts.yml             # openElis01-03 + monitor01
├── group_vars/
│   └── all.yml               # Shared paths, OpenELIS .env defaults
├── host_vars/
│   ├── openElis01.yml
│   ├── openElis02.yml
│   ├── openElis03.yml
│   └── monitor01.yml
├── playbooks/
│   ├── site.yml              # Full provisioning entrypoint
│   ├── bootstrap.yml         # Base packages, timezone, firewall
│   ├── docker.yml            # Install Docker + Compose plugin
│   ├── deploy.yml            # Deploy OpenELIS + node_exporter + monitoring
│   ├── update.yml            # git pull, image pull, recreate (all stacks)
│   └── down.yml              # Stop and remove every stack
└── roles/
    ├── common/               # OS baseline
    ├── docker/               # Docker engine + compose plugin install
    ├── openelis/             # Clone OpenELIS repo, render .env, run compose
    ├── node_exporter/        # Host metrics exporter on each OpenELIS host
    └── monitoring/           # Prometheus + Blackbox + Alertmanager + Grafana
```

## Managed Hosts

| Host         | Group       | What runs                                                    |
|--------------|-------------|--------------------------------------------------------------|
| openElis01   | openelis    | OpenELIS compose stack + node_exporter                       |
| openElis02   | openelis    | OpenELIS compose stack + node_exporter                       |
| openElis03   | openelis    | OpenELIS compose stack + node_exporter                       |
| monitor01    | monitoring  | Prometheus, Blackbox, Alertmanager, Grafana                  |

Hostnames and IPs are set in `inventory/hosts.yml`. Each OpenELIS host
runs independently (no inter-host dependencies).

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

1. Set `ansible_host` for each host in `inventory/hosts.yml` (including
   `monitor01`).
2. Confirm connectivity:

   ```
   ansible all -m ping
   ```

3. Provision everything:

   ```
   ansible-playbook playbooks/site.yml
   ```

4. Open Grafana at `http://<monitor01>:3000` (admin password is
   `monitoring_grafana_admin_password`, default `admin`). The Prometheus
   datasource is pre-provisioned. Import community dashboards:
   - **1860** — Node Exporter Full (OpenELIS host metrics)
   - **7587** — Prometheus Blackbox Exporter (up/down + probe latency)

## Common Operations

Deploy or redeploy every stack:

```
ansible-playbook playbooks/deploy.yml
```

Pull repo changes and newer images, then recreate containers:

```
ansible-playbook playbooks/update.yml
```

Stop all stacks (containers removed, named volumes preserved):

```
ansible-playbook playbooks/down.yml
```

Target one host or group:

```
ansible-playbook playbooks/deploy.yml --limit openElis02
ansible-playbook playbooks/deploy.yml --limit monitoring
```

## Configuration

`group_vars/all.yml` defines the shared defaults:

- `openelis_repo_url` / `openelis_repo_ref` / `openelis_install_dir` —
  which upstream repo and ref to deploy on each OpenELIS host and where.
- `openelis_env` — key/value map rendered into
  `{{ openelis_install_dir }}/.env` and consumed by the upstream compose
  file. Keys mirror the variables referenced in that `docker-compose.yml`
  (`OE_HTTP_PORT`, `OE_DB_PASSWORD`, `SSL_KEYSTORE_PASSWORD`, etc.).
- `node_exporter_port` — port node_exporter listens on (default 9100).
- `monitoring_project_dir` / `monitoring_data_dir` — where monitoring
  configs and data volumes live on `monitor01`.

Per-host overrides go in `host_vars/<hostname>.yml`.

### Monitoring role variables (`roles/monitoring/defaults/main.yml`)

- `monitoring_prometheus_port` (9090), `monitoring_grafana_port` (3000),
  `monitoring_alertmanager_port` (9093), `monitoring_blackbox_port` (9115).
- `monitoring_grafana_admin_password` — defaults to
  `vault_grafana_password` with fallback `admin`.
- `monitoring_openelis_targets_group` — inventory group to probe
  (default `openelis`).

### What Prometheus scrapes

- `node_exporter` on each OpenELIS host (host CPU/memory/disk).
- Blackbox HTTP probes against `http://<host>/` (OpenELIS proxy).
- Blackbox ICMP probes against each host IP.

### Alerts

`alerts.yml` ships two rules:

- `HostDown` — `probe_success == 0` for 2m (critical).
- `NodeExporterDown` — scrape target down for 2m (warning).

The Alertmanager receiver is a stub — add `slack_configs`, `email_configs`,
or `webhook_configs` under the `default` receiver in
`roles/monitoring/templates/alertmanager.yml.j2` to actually deliver alerts.

## Secrets

Do not commit plaintext credentials. Use Ansible Vault:

```
ansible-vault create group_vars/all/vault.yml
ansible-playbook playbooks/site.yml --ask-vault-pass
```

The defaults reference these vault variables:

- `vault_oe_db_password`, `vault_oe_admin_password`,
  `vault_oe_encryption_password`
- `vault_ssl_keystore_password`, `vault_ssl_truststore_password`
- `vault_grafana_password`

## Troubleshooting

- Verbose output: `ansible-playbook ... -vvv`.
- Preview changes: `--check --diff`.
- Inspect the stacks on a host:

  ```
  ssh openElis01 "cd /opt/openelis && docker compose ps"
  ssh openElis01 "cd /opt/node_exporter && docker compose ps"
  ssh monitor01  "cd /opt/monitoring && docker compose ps"
  ```

- Logs:

  ```
  ssh openElis01 "cd /opt/openelis && docker compose logs oe.openelis.org"
  ssh monitor01  "cd /opt/monitoring && docker compose logs prometheus"
  ```

- Verify a probe directly on the monitoring host:

  ```
  curl 'http://localhost:9115/probe?target=http://10.0.0.11/&module=http_2xx'
  ```
