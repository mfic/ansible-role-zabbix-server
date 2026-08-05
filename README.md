# ansible-role-zabbix-server

Deploys the **Zabbix 7.0 LTS server** as a Docker Compose stack on Debian.

**Status: skeleton.** The interface and the two safety assertions below are settled; the deployment
tasks are still to be filled in.

## Design

Pure logic, no site data. Every address, hostname, credential and sizing value comes from the
inventory that consumes this role — nothing here is specific to a site or an estate.

## The two assertions this role must never lose

Both defend against a **silent** failure: something that looks healthy while being useless. They live
here rather than in a playbook because a role should travel with its guarantees.

### 1. The database must not be silently empty

The upstream `zabbix-docker` entrypoint (`pgsql.sh`) recreates the database and schema on an `info`
log line and **exits 0**. So `pg_isready`, `service_healthy`, `service_completed_successfully` and
the frontend `/ping` all pass against a **brand new, empty Zabbix**. If you also run an external
heartbeat, it stays quiet — the server is genuinely alive. Nothing upstream catches this.

Two independent routes into it:

- a rebuild where the durable volume is not mounted, or is mounted empty;
- mounting at `/var/lib/postgresql` instead of `/var/lib/postgresql/data`, which silently creates an
  anonymous volume that vanishes on the next `docker compose up --force-recreate`.

**Only a pre-flight guard closes this.** `tasks/assert_db.yml` refuses to start the stack when a
database was expected to exist but the data directory is absent or empty.

### 2. An external heartbeat must not be answerable by a proxy

A common dead-man's-switch design is an HTTP agent item on the server's own host object, pinging an
external service. If that host object is monitored by a **Zabbix proxy** rather than the server, the
proxy executes the item and — because proxies buffer results while the server is unreachable —
**keeps pinging the external service for as long as `ProxyOfflineBuffer` allows**, commonly a week.
The switch reports green throughout an outage it exists to detect.

`tasks/assert_monitoring.yml` verifies via the API that the server's own host object is monitored by
the server, and fails the run otherwise.

Note this cannot fire in a proxy-less deployment, which is exactly why it is asserted rather than
documented: by the time proxies are added, the interaction is easy to forget, and the symptom is a
switch that simply never alerts.

## Corrections to the upstream defaults

`defaults/main.yml` overrides several upstream choices deliberately — a floating image tag, an
unset `shm_size`, a `stop_grace_period` shorter than the database image's own guidance, and two
optional components that ship enabled. Each is commented in place.

## Role variables

See `defaults/main.yml`.

## Licence

MIT
