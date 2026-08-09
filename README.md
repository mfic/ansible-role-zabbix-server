# ansible-role-zabbix-server

Deploys the **Zabbix 7.0 LTS server** as a Docker Compose stack on Debian.

**Status: in production.** The compose stack, the volume layout, the nightly dump and both safety
assertions are implemented and running against a real server.

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

`tasks/assert_monitoring.yml` reads the server's own host object out of PostgreSQL — in the
container this role deployed, with the credentials this role already holds — and fails the run if
`monitored_by` is anything but the server.

Note this cannot fire in a proxy-less deployment, which is exactly why it is asserted rather than
documented: by the time proxies are added, the interaction is easy to forget, and the symptom is a
switch that simply never alerts.

Set **`zabbix_server_self_host_name`** to the *technical* host name of that object if it is not the
`inventory_hostname`.

> ⚠ **This assertion did nothing for its first several releases, and reported `ok` while doing it.**
> Worth reading if you write guards. It called `community.zabbix.zabbix_host_info`, whose modules run
> over the `httpapi` connection plugin **on the control node** — where the frontend this role
> deploys, bound to the guest's loopback, does not exist; and it carried `failed_when: false`. Under
> that it tested `proxy_hostid`, a Zabbix **6.x** column that 7.0 replaced with `monitored_by` /
> `proxyid` / `proxy_groupid`, so the missing key fell through `| default('0')` to *not proxied* and
> the guard would have passed even over a working connection. Both defects share one shape: **a read
> whose failure mode defaults to the value that passes is not a check.** The rewrite therefore fails
> when it cannot ask the question, and prints when it can — an assertion that is only ever visible on
> failure cannot be told apart from one that is not running.

It does **not** use the Zabbix API, deliberately: an API call needs a Super admin login, which would
make a Zabbix password a mandatory variable of a role that otherwise only needs a database one, and
would put the PHP frontend among the guard's dependencies. `deploy.yml` already reads Zabbix's own
tables (`select count(*) from users`) for the same reason. Reading that schema is fine; nothing here
writes to it.

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

## Publishing the frontend

The frontend is published on **`127.0.0.1` only** by default, and enabling either front door below
does **not** change that. The loopback binding is the rung underneath both of them — what an
operator reaches over `ssh -L` when the proxy, the tunnel, the certificate or DNS is itself the
thing that has broken. A break-glass path that shares a dependency with the normal path is not one.

Both are optional services in the *same* Compose project, not a second stack, so one
`docker compose up` restores the frontend and its front door together after a reboot. There is no
separate unit that can be down while the other still looks healthy.

**`zabbix_server_enable_traefik`** — TLS reverse proxy on the host, certificate from Let's Encrypt
by **DNS-01**. Configured through the **file provider**; no Docker socket is mounted. Label
discovery would hand a network-facing container root-equivalent control of the daemon running the
database, in order to route one fixed backend already known when the file is rendered.

> ⚠ **`zabbix_server_traefik_acme_dns_resolvers` is load-bearing on any split-horizon estate.**
> lego uses these resolvers both to find the zone that should hold the challenge record and to check
> that it propagated. Left to the container's own resolver, a local server that is authoritative for
> the sub-zone ends the SOA walk in the wrong place, the record is requested in a zone the DNS
> provider has never heard of, and the propagation check then asks a server that answers NXDOMAIN
> with full authority. It never succeeds — and it fails looking exactly like a bad API token.

**`zabbix_server_enable_cloudflared`** — outbound-only tunnel to the Cloudflare edge,
**token-configured** ("remotely managed"). One secret, `zabbix_server_cloudflared_tunnel_token`; no
config file and no credentials file. It reaches the frontend **directly over the Compose network,
not through Traefik** — Cloudflare terminates TLS at its own edge, so the Traefik path would demand
a second certificate for a name whose whole purpose is being reachable without the LAN.

> ⚠ **The routing is not in this repo.** A token-configured tunnel keeps its ingress rules in the
> Cloudflare dashboard, so a rebuild from source restores a connector that serves whatever the
> dashboard says. The credential is reproducible; the routing is not. A locally-managed tunnel
> (credentials file + rendered config) is the trade in the other direction — ingress in version
> control, at the cost of a second secret and a config to keep in step.

> ⚠ **The dashboard's public hostname must point at `http://zabbix-web:8080`**, resolved by Docker
> on the Compose network — *not* at localhost. Pointing it at `127.0.0.1` reaches the connector
> container's own loopback, where nothing listens: every request 502s while the tunnel reports
> perfectly healthy.

The connector runs on the Compose network rather than `network_mode: host` deliberately. Host
networking would reach the frontend through its `127.0.0.1` publish — the break-glass path — and
coupling the public route to the emergency route means one change can remove both.

The role verifies the certificate Traefik actually serves, rather than that it is listening. Until
DNS-01 completes, Traefik answers with its own `TRAEFIK DEFAULT CERT` — a working HTTPS endpoint
that every browser rejects, and a silent failure otherwise.

Neither front door authenticates anyone. **Access policy is out of this role's scope**: it belongs
to whatever sits in front of the tunnel, and to Zabbix's own user database.

## The nightly database dump

Off by default (`zabbix_server_backup_enabled`). When on, a systemd timer runs `pg_dump -Fc` plus
`pg_dumpall --globals-only` into `zabbix_server_backup_dir`, keeping `zabbix_server_backup_keep`
pairs and hard-linking the newest to stable `*-latest.*` names.

**It is not redundant with a snapshot of the guest, and consistency is the least of the reason.** A
snapshot preserves a corrupt page faithfully every night for as long as retention keeps it;
`pg_dump` reads every row, so it *fails* on that page instead of archiving it. The dump also
survives a PostgreSQL major-version change, which a copied `PGDATA` does not, and `pg_restore` can
pull back a single table without standing up a guest.

Four details are load-bearing rather than incidental:

- **The globals are a separate artifact and not optional.** They are a few KiB, and without them a
  restore onto fresh PostgreSQL fails before it reads a single row, because the role that owns every
  object does not exist yet.
- **The archive is written to a `.part` name and renamed.** `rename(2)` inside one filesystem is
  atomic, so the monitored name never refers to a half-written archive. This is what makes
  *overrunning the capture window safe*: a guest snapshot taken while tonight's dump is still running
  captures yesterday's complete pair rather than today's stump. The cost is a day of RPO for that
  night — which is what a dump-age trigger is for.
- **It is verified before it is promoted.** `pg_restore -l` reads the archive's table of contents, so
  a truncated dump is rejected. Size and mtime both look healthy on a truncated archive, so the
  checks most likely to be monitoring it cannot catch this.
- **`zabbix_server_backup_read_group` is what makes the dump monitorable.** The files are written by
  the database container's uid; without a group an agent belongs to, `vfs.file.*` items on them
  return *unsupported*, and an unsupported item is not an alert. The role asserts the group exists
  rather than letting `chgrp` fail nightly after a successful dump.

`zabbix_server_backup_dir` is bind-mounted read-write into the database container. Both directions
are used: `pg_dump` writes there and `pg_restore` reads from there during a recovery — a restore that
cannot see the dump from inside the container is a restore that does not happen.

**What it is not: an off-host backup.** The dump lands on the same filesystem as the database it came
from. Something outside this role has to capture that directory, and prove it has.

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
