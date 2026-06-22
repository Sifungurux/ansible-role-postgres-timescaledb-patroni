# Test Report

**Date:** 2026-06-22
**Result:** All tests passed on both platforms

---

## What this software does (plain English)

This project automates the installation of a database system made up of three pieces that work together:

- **PostgreSQL** — the database itself, where your data lives
- **TimescaleDB** — a plugin for PostgreSQL that makes it very fast at storing data that changes over time (sensor readings, metrics, logs, prices, etc.)
- **Patroni** — a watchdog process that keeps PostgreSQL running at all times; if the database crashes or the server goes down, Patroni automatically restarts it or promotes a standby server to take over
- **etcd** — a small coordination service that Patroni uses to keep track of which server is currently the "leader" (the one accepting writes); think of it as a shared notice board for the cluster

The automation is written in **Ansible**, a tool that lets you describe the desired state of a server in a configuration file and then apply it repeatedly — it is safe to run multiple times without breaking anything.

---

## What was tested

### Platform coverage

Tests ran on two different Linux operating systems simultaneously:

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

## Summary table

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

## Test environment

Tests were run locally on macOS using [Colima](https://github.com/abiosoft/colima) (a lightweight Linux container runtime) to spin up isolated Debian 12 and Rocky Linux 9 containers. The test framework is [Molecule](https://molecule.readthedocs.io/), the standard tool for testing Ansible roles. Each test run starts with a clean container, applies the role, and tears the container down afterwards — ensuring results are not influenced by any leftover state.
