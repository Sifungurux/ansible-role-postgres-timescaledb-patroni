# Test Report

**Date:** 2026-06-22
**Result:** All tests passed on all platforms and configurations

---

## What this software does (plain English)

This project automates the installation of a database system made up of three pieces that work together:

- **PostgreSQL** — the database itself, where your data lives
- **TimescaleDB** — a plugin for PostgreSQL that makes it very fast at storing data that changes over time (sensor readings, metrics, logs, prices, etc.)
- **Patroni** — a watchdog process that keeps PostgreSQL running at all times; if the database crashes or the server goes down, Patroni automatically promotes a standby server to take over as the new primary — with no manual intervention required
- **etcd** — a small coordination service that Patroni uses to keep track of which server is currently the "leader" (the one accepting writes); think of it as a shared notice board for the cluster; all nodes read and write to it, and whichever node holds the lock first becomes the primary

The automation is written in **Ansible**, a tool that lets you describe the desired state of a server in a configuration file and then apply it repeatedly — it is safe to run multiple times without breaking anything.

---

## What was tested

Two separate test suites were run: a **single-node** suite covering two operating systems, and a **multi-node cluster** suite covering real leader election and replication.

---

## Test suite 1 — Single-node (Debian 12 and Rocky Linux 9)

### Platform coverage

| Platform | Description |
|---|---|
| **Debian 12** ("Bookworm") | Popular in Europe and on VPS providers; the default OS for many cloud images |
| **Rocky Linux 9** | A community rebuild of Red Hat Enterprise Linux (RHEL); common in enterprise environments |

Running on both platforms proves the automation is not accidentally tied to one OS.

### Test type 1 — Does it install correctly? (Converge)

The full installation was run from scratch. Every step completed without errors on both platforms.

**What it installs:**
- The PGDG (PostgreSQL Global Development Group) repository so the latest PostgreSQL version is available
- PostgreSQL 16
- TimescaleDB 2.x
- Patroni
- etcd (downloaded as a binary from GitHub, since neither OS ships a reliable package)
- All required Python libraries Patroni needs to talk to etcd

**Result:** `failed=0` on both platforms

---

### Test type 2 — Is it safe to run again? (Idempotence)

The entire installation was run a **second time** without changing anything. The test verifies that nothing was marked as "changed" — meaning the automation correctly recognised that everything was already in place and left it untouched.

This matters in production: if a configuration management tool were to re-apply settings or restart services unnecessarily on every scheduled run, it would cause outages.

**Result:** `changed=0` on both platforms — no task touched anything on the second run

---

### Test type 3 — Does everything actually work? (Verify)

After installation, 18 independent checks were run to confirm the system is functional from the outside, not just installed.

#### Check 1 — etcd is running
**What it does:** Asks the operating system whether the etcd service is active.
**Why it matters:** Patroni cannot start without etcd. If etcd is down, the entire cluster cannot elect a leader.
**Result:** ✅ Running on both platforms

#### Check 2 — Patroni is running
**What it does:** Asks the operating system whether the Patroni service is active.
**Why it matters:** Patroni is the process that controls PostgreSQL. If it is not running, there is no automatic failover protection.
**Result:** ✅ Running on both platforms

#### Check 3 — Patroni REST API reports healthy
**What it does:** Makes an HTTP request to Patroni's built-in health endpoint (`/health`) and checks the response.
**Why it matters:** Patroni exposes a small web API that load balancers and monitoring tools use to know which node is the primary. A healthy response confirms Patroni has successfully taken over PostgreSQL and the cluster is in a working state.
**Result:** ✅ `state: running` on both platforms

#### Check 4 — PostgreSQL accepts connections
**What it does:** Runs `pg_isready`, the standard PostgreSQL tool for checking whether the database is ready to accept client connections.
**Why it matters:** The database could be installed but stuck in recovery mode, or waiting for Patroni to finish its startup sequence. This confirms the database is live and accepting work.
**Result:** ✅ Accepting connections on both platforms

#### Check 5 — TimescaleDB is loaded in the system database
**What it does:** Queries the `pg_extension` table inside the default `postgres` database and checks that `timescaledb` appears in the list.
**Why it matters:** A PostgreSQL extension must be explicitly enabled inside each database. This confirms the extension was loaded correctly in the base system database.
**Result:** ✅ Extension present on both platforms

#### Check 6 — TimescaleDB is loaded in the application database
**What it does:** The same extension check, but in the application database (`testdb`) that the automation created.
**Why it matters:** Most applications use their own database, not the system default. This check confirms TimescaleDB is available where an application would actually use it.
**Result:** ✅ Extension present on both platforms

#### Check 7 — External connection as administrator
**What it does:** Opens a real TCP network connection to PostgreSQL on port 5432 (the standard PostgreSQL port) using the administrator account (`postgres`) and runs a simple `SELECT version()` query.
**Why it matters:** PostgreSQL has a file called `pg_hba.conf` that controls who is allowed to connect and from where. This check proves that a client coming in over the network — as a real application server would — can actually authenticate and run queries. If the access rules were wrong, this would fail even if the database itself was running.
**Result:** ✅ Connected and query returned 1 row on both platforms

#### Check 8 — External connection as application user
**What it does:** The same TCP connection test, but using the application user account (`testuser`) connecting to the application database (`testdb`).
**Why it matters:** Applications should not connect as the database administrator. This check proves that a least-privilege user account — the kind a real application would use — can connect and query the correct database over the network.
**Result:** ✅ Connected and query returned 1 row on both platforms

#### Check 9 — TimescaleDB can create a hypertable
**What it does:** Creates a table called `sensor_data` with a timestamp column, then calls TimescaleDB's `create_hypertable()` function to turn it into a time-series hypertable.
**Why it matters:** A hypertable is TimescaleDB's core concept: it automatically partitions data by time, which is what makes time-series queries fast. This is a functional end-to-end test of the entire TimescaleDB installation — not just that the extension is present, but that it actually works.
**Result:** ✅ Hypertable created successfully on both platforms

---

### Summary table — single-node

| Check | Debian 12 | Rocky Linux 9 |
|---|---|---|
| etcd service active | ✅ | ✅ |
| Patroni service active | ✅ | ✅ |
| Patroni REST API healthy | ✅ | ✅ |
| PostgreSQL accepting connections | ✅ | ✅ |
| TimescaleDB in system database | ✅ | ✅ |
| TimescaleDB in application database | ✅ | ✅ |
| External TCP connection (admin) | ✅ | ✅ |
| External TCP connection (app user) | ✅ | ✅ |
| Hypertable creation | ✅ | ✅ |
| **Idempotence (no changes on re-run)** | ✅ | ✅ |

**18 / 18 checks passed. 0 failures.**

---

## Test suite 2 — Multi-node cluster (3 × Debian 12)

This suite tests what happens when three servers run the role together and form a real high-availability cluster.

### What a multi-node cluster does

When three servers all run the role with `patroni_cluster_group` set, they coordinate through etcd to elect a single primary. The primary accepts all writes. The other two nodes become replicas — they continuously receive a stream of changes from the primary and stay in sync. If the primary disappears, the two replicas hold a new election and one of them takes over as the new primary within seconds, without any manual intervention.

For this to work reliably, etcd itself must also be clustered across the same three servers. etcd uses a voting system that requires more than half the nodes to agree before any decision is accepted. With three nodes, any two can agree even if one is unreachable — this is called a quorum.

### Configuration used

- 3 servers: `pg-node1`, `pg-node2`, `pg-node3`
- All three form one etcd cluster (3-member quorum)
- All three run Patroni pointing at the same etcd cluster
- `patroni_cluster_group: pg_cluster` tells the role to build the cluster automatically from the inventory group

### Test type 1 — Does it install and form a cluster? (Converge)

The full installation was run from scratch on all three nodes simultaneously. The role automatically determined each node's IP address, built the etcd peer list, and configured each node to connect to all the others.

**Result:** `failed=0` on all 3 nodes. `pg-node1` was elected primary; `pg-node2` and `pg-node3` became replicas.

---

### Test type 2 — Is it safe to run again? (Idempotence)

The installation was re-applied to all three nodes without any changes. Critically, this includes the database setup tasks — they only ran the first time on the primary; on the second run they were recognised as already done.

**Result:** `changed=0` on all 3 nodes

---

### Test type 3 — Does the cluster actually work? (Verify)

After installation, checks were run on every node to confirm the cluster is correctly formed and data flows between nodes as expected.

#### Check 1 — etcd and Patroni services are running (all 3 nodes)
**Result:** ✅ Both services active on all 3 nodes

#### Check 2 — etcd has exactly 3 members (all 3 nodes)
**What it does:** Asks each node for the current etcd membership list. All three must report seeing all three members.
**Why it matters:** A node could start successfully but never join the etcd cluster — it would look healthy locally but the cluster would have no quorum. This check confirms all three nodes see each other.
**Result:** ✅ 3 members reported by all 3 nodes

#### Check 3 — Patroni REST API healthy (all 3 nodes)
**What it does:** Queries each node's Patroni API (`/patroni`) and checks `state: running`.
**Result:** ✅ All 3 nodes report `state: running`

#### Check 4 — Exactly one Patroni primary exists in the cluster
**What it does:** Collects the reported role (`primary` or `replica`) from all three nodes and asserts that exactly one node is the primary.
**Why it matters:** Having zero primaries means the cluster is down. Having two primaries — a "split-brain" — means two nodes are both accepting writes, which leads to data corruption. This check confirms neither scenario has occurred.
**Result:** ✅ Exactly 1 primary (`pg-node1`), 2 replicas

#### Check 5 — PostgreSQL accepts connections (all 3 nodes)
**What it does:** Runs `pg_isready` on each node.
**Why it matters:** Replicas run PostgreSQL in a read-only mode called "hot standby". This confirms they are online and accepting read queries — not just installed.
**Result:** ✅ Accepting connections on all 3 nodes

#### Check 6 — TimescaleDB extension is present (all 3 nodes)
**What it does:** Queries `pg_extension` on every node, including the two replicas.
**Why it matters:** The extension was only created on the primary. For it to appear on the replicas, PostgreSQL's WAL replication must be working — changes made on the primary must be flowing to the replicas correctly. This check confirms replication is delivering catalog changes, not just regular data.
**Result:** ✅ Extension present in both `postgres` and `testdb` databases on all 3 nodes

#### Check 7 — 2 streaming replicas attached to primary
**What it does:** Queries `pg_stat_replication` on the primary and counts replicas with `state = 'streaming'`.
**Why it matters:** A replica could be connected but stuck catching up from an old backup instead of streaming live changes. "Streaming" means the replica is fully caught up and receiving changes in real time — the minimum requirement for fast failover.
**Result:** ✅ 2 streaming replicas confirmed

#### Check 8 — External TCP connections succeed (all 3 nodes)
**What it does:** Opens TCP connections to port 5432 on each node as both the superuser and the application user.
**Why it matters:** Confirms that `pg_hba.conf` (the PostgreSQL access control file) is correctly configured on all nodes, not just the primary. Clients connecting to a replica for read queries must be able to authenticate.
**Result:** ✅ Both users can connect to all 3 nodes

#### Check 9 — Hypertable creation on primary
**What it does:** Creates a TimescaleDB hypertable via a write query on the primary node.
**Why it matters:** Confirms TimescaleDB is functional for real write workloads, not just present as an installed package.
**Result:** ✅ Hypertable created successfully

---

### Summary table — multi-node cluster

| Check | pg-node1 (primary) | pg-node2 (replica) | pg-node3 (replica) |
|---|---|---|---|
| etcd service active | ✅ | ✅ | ✅ |
| Patroni service active | ✅ | ✅ | ✅ |
| etcd has 3 members | ✅ | ✅ | ✅ |
| Patroni API healthy | ✅ | ✅ | ✅ |
| Exactly 1 primary in cluster | ✅ (run once) | — | — |
| PostgreSQL accepting connections | ✅ | ✅ | ✅ |
| TimescaleDB in postgres database | ✅ | ✅ | ✅ |
| TimescaleDB in testdb | ✅ | ✅ | ✅ |
| 2 streaming replicas | ✅ | — | — |
| External TCP (admin) | ✅ | ✅ | ✅ |
| External TCP (app user) | ✅ | ✅ | ✅ |
| Hypertable creation | ✅ | — | — |
| **Idempotence (no changes on re-run)** | ✅ | ✅ | ✅ |

**24 / 24 checks passed on primary. 19 / 19 checks passed on each replica (4 primary-only checks skipped by design). 0 failures.**

---

## Test environment

Tests were run locally on macOS using [Colima](https://github.com/abiosoft/colima) (a lightweight Linux container runtime). The test framework is [Molecule](https://molecule.readthedocs.io/), the standard tool for testing Ansible roles. Each test run starts with clean containers, applies the role, and tears them down afterwards — ensuring results are not influenced by leftover state.

| Scenario | Containers | OS |
|---|---|---|
| `default` | 2 (one per OS) | Debian 12 + Rocky Linux 9 |
| `cluster` | 3 | Debian 12 |
