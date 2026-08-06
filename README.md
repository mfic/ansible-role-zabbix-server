# ansible-role-zabbix-server

Deploys the **Zabbix 7.0 LTS server** as a Docker Compose stack on Debian.

**Status: deploys.** The compose stack, the volume layout and both safety assertions are
implemented. Not yet run against a real host.

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

## Storage: bind mounts, on a filesystem of their own

Upstream's compose file uses **named volumes**. This role uses **explicit bind mounts** under
`zabbix_server_data_dir`, and the reason is assertion #1 above. A named volume lives under Docker's
data-root: the path holding the database is not visible to `ls`, not checkable by `findmnt`, and a
rebuild that loses the volume recreates it *empty* — which is exactly the failure that comes up
green. A bind mount is a directory. You can look at it, a backup job that is not Docker can read it,
and the pre-flight guard can `stat` the `PG_VERSION` inside it.

`zabbix_server_data_dir` is expected to be **its own filesystem**, and the role fails when it is not
(`zabbix_server_assert_data_dir_is_mount`). That check is only worth having *because* the data is on
a dedicated device: it turns "did the data disk mount?" into a yes/no question asked before
PostgreSQL starts. Without it, an unmounted disk means the database quietly writes to root and
nothing looks wrong until the disk fills or the machine is rebuilt.

Deployments that genuinely keep their data on root can set that variable `false` — the `PG_VERSION`
check is then the only guard left, and that is a real reduction in cover, not a formality.

## What this role does *not* configure

**Retention, and the two housekeeping override flags.** They are frontend settings stored in the
database, not `ZBX_*` environment variables, so no compose file can express them. They are applied
over the API after the stack is up. `defaults/main.yml` carries the *values* and the reasoning
(notably that both overrides must stay `false` on plain PostgreSQL), but a consuming playbook has to
apply them — assuming the compose file did is how a documented retention policy ends up never
actually in force.

## Requirements

- Docker Engine with the Compose v2 plugin on the target, and `docker.service` **enabled** — the
  stack's `restart: unless-stopped` policy is the only thing that brings Zabbix back after an
  unplanned reboot, and it cannot act if the daemon does not start.
- `community.docker` on the controller.
- `community.zabbix` on the controller, for the post-flight API assertion.

## Role variables

See `defaults/main.yml`. Two have no usable default and must come from the consuming inventory:
`zabbix_server_db_password` (the role ships none on purpose) and, in practice,
`zabbix_server_php_tz` / `zabbix_server_name`, whose upstream defaults are `Europe/Riga` and
`Composed installation`.

## Licence

MIT
