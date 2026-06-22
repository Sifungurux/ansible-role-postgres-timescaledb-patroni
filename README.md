# ansible-role-postgres-timescaledb-patroni

Ansible role that installs and configures a high-availability PostgreSQL 16+ cluster with TimescaleDB and Patroni, using etcd as the distributed configuration store (DCS).

Supports **Debian 12** and **Rocky Linux / RHEL 9**.

## What it does

- Installs PostgreSQL 16+ from the official PGDG repository
- Installs TimescaleDB 2.x and pre-loads the extension
- Installs Patroni (via EPEL on RHEL, via apt on Debian) and configures it to own PostgreSQL exclusively
- Installs etcd from GitHub binary releases (not available in Debian 12 apt repos or EPEL 9) with a custom systemd unit
- Creates an optional application database with a dedicated user and TimescaleDB extension enabled
- Supports **single-node** and **multi-node HA** deployments (3, 5, … nodes with etcd quorum)
- Idempotent: re-running never causes service restarts or failed tasks

## Requirements

- Ansible 2.12+
- `community.postgresql` collection ≥ 4.0

Install collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Role variables

| Variable | Default | Description |
|---|---|---|
| `postgresql_version` | `16` | PostgreSQL major version |
| `postgresql_port` | `5432` | PostgreSQL listen port |
| `patroni_scope` | `postgres-cluster` | Patroni cluster name |
| `patroni_namespace` | `/service/` | etcd key namespace |
| `patroni_restapi_port` | `8008` | Patroni REST API port |
| `patroni_superuser_username` | `postgres` | PostgreSQL superuser name |
| `patroni_superuser_password` | `patroni` | PostgreSQL superuser password (**override in vault**) |
| `patroni_replication_username` | `replicator` | Replication user name |
| `patroni_replication_password` | `replicator` | Replication user password (**override in vault**) |
| `patroni_restapi_username` | `""` | REST API username (empty = no auth) |
| `patroni_restapi_password` | `""` | REST API password |
| `patroni_cluster_group` | `""` | Ansible group name containing all cluster nodes. When set, `patroni_etcd_hosts` and the etcd peer list are built automatically. Leave empty for single-node. |
| `patroni_etcd_hosts` | `["127.0.0.1:2379"]` | etcd endpoints for Patroni DCS (auto-built from `patroni_cluster_group` when set) |
| `etcd_version` | `3.5.14` | etcd binary version to download |
| `etcd_initial_cluster_state` | `new` | Set to `existing` when joining a running cluster |
| `timescaledb_db_name` | `""` | App database to create (skipped if empty) |
| `timescaledb_app_user` | `""` | App database user (skipped if empty) |
| `timescaledb_app_password` | `""` | App database user password |
| `patroni_pg_hba_extra` | `[]` | Additional `pg_hba.conf` entries |
| `postgresql_parameters` | see defaults | PostgreSQL parameters managed via Patroni DCS |

## Example playbook

### Single node

```yaml
- hosts: db_server
  become: true
  vars:
    patroni_superuser_password: "{{ vault_pg_superuser_password }}"
    patroni_replication_password: "{{ vault_pg_replication_password }}"
    timescaledb_db_name: myapp
    timescaledb_app_user: myapp
    timescaledb_app_password: "{{ vault_app_db_password }}"
  roles:
    - ansible-role-postgres-timescaledb-patroni
```

### Multi-node HA cluster (3, 5, … nodes)

```yaml
# inventory.yml
all:
  children:
    pg_cluster:
      hosts:
        db1:
        db2:
        db3:
```

```yaml
# playbook.yml
- hosts: pg_cluster
  become: true
  vars:
    patroni_cluster_group: pg_cluster        # enables multi-node mode
    patroni_superuser_password: "{{ vault_pg_superuser_password }}"
    patroni_replication_password: "{{ vault_pg_replication_password }}"
    timescaledb_db_name: myapp
    timescaledb_app_user: myapp
    timescaledb_app_password: "{{ vault_app_db_password }}"
  roles:
    - ansible-role-postgres-timescaledb-patroni
```

The role automatically:
- Builds the etcd peer list from the group members
- Writes the correct `ETCD_INITIAL_CLUSTER` for each node
- Runs TimescaleDB DDL (CREATE EXTENSION, CREATE DATABASE, CREATE USER) only on the Patroni primary; replicas inherit via WAL replication

## Testing

Tests use [Molecule](https://molecule.readthedocs.io/) with the Docker driver. On macOS, [Colima](https://github.com/abiosoft/colima) is used instead of Docker Desktop.

```bash
# Start Colima if not already running
colima start --cpu 4 --memory 8

export DOCKER_HOST="unix://${HOME}/.colima/default/docker.sock"

# Single-node test (Debian 12 + Rocky 9)
molecule test

# Multi-node cluster test (3 Debian 12 nodes)
molecule test -s cluster

# Or step by step
molecule converge [-s cluster]
molecule idempotence [-s cluster]
molecule verify [-s cluster]
molecule destroy [-s cluster]
```

The verify playbook checks:

- etcd and Patroni services are active
- Patroni REST API reports `state: running`
- PostgreSQL accepts TCP connections
- TimescaleDB extension is loaded in target databases
- External TCP connections succeed as both superuser and app user
- Hypertable creation works end-to-end

## Platform notes

**Debian 12** — etcd was removed from bookworm repos; installed as a binary from GitHub releases. The stock `patroni.service` unit requires the config file at `/etc/patroni/config.yml`. The `postgresql-N` postinst auto-creates a cluster that would block Patroni bootstrap; the role drops it before Patroni starts for the first time.

**Rocky Linux / RHEL 9** — TimescaleDB is installed as `timescaledb_16` from the PGDG repo (not packagecloud). Patroni from EPEL uses Python 3.12; `python-etcd` and `etcd3gw` are pip-installed into that interpreter so Patroni's etcd3 DCS module can load.

## License

MIT
