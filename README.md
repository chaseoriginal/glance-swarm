# Glance

Second fleet dashboard, alongside Homepage. Renders a single YAML config into
a grid of widgets: live status for every fleet service, release tracking, and
the service link directory.

| | |
|---|---|
| URL | `https://glance.echoinggreen.net` |
| Portainer host | ore-dock-110-1 |
| Endpoint | 3 (local swarm 110-1) |
| Swarm | True |
| Git repo | https://github.com/chaseoriginal/glance-swarm |
| Compose file | `docker-compose.yml` |
| Auto-update poll | every 15m |
| Node | `ore-dock-110-1` (pinned) |
| Port | 8084, published in host mode |
| Image | `glanceapp/glance:v0.8.7` (pinned) |

## What it is

Glance renders `config/glance.yml` into two pages:

- **Fleet** - one `monitor` widget per category (Media, Books, Servarr,
  Downloads, Infrastructure, Monitoring, Home & Network), laid out two
  across, plus a clock and a `releases` widget showing new upstream
  versions.
- **Links** - the fleet service directory, split across two columns.

**The `monitor` widget is the only source of fleet-wide status.** Glance has
no swarm mode: the `docker-containers` widget calls the Docker API
`/containers/json`, which returns only the containers on the node its socket
points at, and the names it derives leak Swarm task IDs
(`portainer-agents_portainer-agents.<taskid>`). One monitor entry per service
URL is the only way to see the whole fleet from one dashboard, so this stack
does not mount the Docker socket at all.

`server-stats` is deliberately absent: upstream flags it work-in-progress
(`widget.widgetBase.WIP = true`).

## Proxy

`glance.echoinggreen.net` is served by the local Nginx Proxy Manager on
`ore-dock-110-1`, forwarding to `10.77.110.10:8084`. The vhost carries
access list id 1 ("Private Site Access"), the same list the other internal
names use, so the name is reachable only from allowed sources.

## Placement

- **Node:** pinned to `ore-dock-110-1` by `node.hostname` constraint. The
  config volume is node-local, and the published port is node-local too.
- **Port:** 8084, published in host mode. The swarm ingress mesh drops IPv4
  to IPv6-only listeners, so host mode is the safe form here.
- **No Docker socket:** see above. Glance gets fleet status from HTTP
  checks, so it does not need the socket, and not mounting it keeps the
  container from having host-root-equivalent access.

## Files

- `docker-compose.yml` - the deployed compose. The named volume is declared
  with an explicit `name:`, because Docker Swarm prefixes a plain named
  volume with the stack name and a rename would mount a fresh EMPTY volume.
- `config/glance.yml` - **the authoritative dashboard configuration.** It is
  seeded byte-for-byte into the `glance-swarm_config` volume on the node.
  Glance only ever reads this file, so the volume stays an identical copy
  and git remains authoritative for the layout.
- `ops/add-dns-records.yml` - Ansible playbook that adds the AD DNS CNAME
  for the dashboard name. Run from `ore-peachpi1`:
  `ansible-playbook -i hosts add-dns-records.yml`. It asserts each record on
  the authoring DC before the run is considered successful.
- `.env.example` - placeholder only. The real values live in the Portainer
  stack environment and are deliberately NOT committed.

## Auth

Glance ships its own login gate, on top of the NPM vhost:

- `GLANCE_SECRET_KEY` - base64 of exactly 64 random bytes. Signs the
  session cookie. Glance refuses to start if it does not decode to exactly
  64 bytes.
- `GLANCE_PASSWORD_HASH` - bcrypt hash of the login password.

**The password is the same one that unlocks the Homepage dashboard**, so one
credential covers both dashboards. The username is `chase`.

Both values live only in the Portainer stack environment and in
`~/.hermes/secrets/glance.env` (mode 600) on the control node. If the stack
env is ever lost, rotate by updating the stack env and redeploying; there is
no recovery path through the volume.

### Secrets are not in the config file

The config file contains `${GLANCE_SECRET_KEY}` and `${GLANCE_PASSWORD_HASH}`
placeholders, not values. Glance expands `${VAR}` from the process
environment when it parses the config, so the live file on disk is
byte-identical to the one in git and carries no credential material. That is
what makes it safe for this repo to be public and cloned anonymously by
Portainer.

## DNS

Both halves are needed, matching every other fleet name:

- **Route 53** (public) - CNAME `glance.echoinggreen.net` ->
  `lproxy.echoinggreen.net`, TTL 300, in zone `Z1F3FV8MI2BKA0`.
- **AD DNS** (internal) - CNAME in the `echoinggreen.net` zone on the
  authoring DC, created by `ops/add-dns-records.yml`.

The public half resolves to the UDM WAN address; the internal half resolves
to `10.77.110.10` directly. The NPM access list is what actually restricts
who can reach the dashboard.

## Changing the dashboard

Two options, pick one and stick to it:

1. **Edit the volume in place** (simplest). The live config lives in the
   `glance-swarm_config` volume on `ore-dock-110-1`. Changes made here are
   NOT in git.
2. **Edit in git and re-seed** (keeps git authoritative). Change
   `config/glance.yml` in the repo, push, then re-run the seed step below.
   The seed overwrites the volume's copy of the file.

Glance watches the config file and reloads on change, so a re-seed does not
need the service to be restarted.

## Re-seeding the config from git

```bash
# on ore-dock-110-1
rsync -az config/ ladmin@ore-dock-110-1:/tmp/glance-config/
ssh ladmin@ore-dock-110-1 'docker run --rm -v glance-swarm_config:/data \
  -v /tmp/glance-config:/src:ro alpine sh -c "cp -a /src/. /data/" && \
  rm -rf /tmp/glance-config'
```

## Verifying a deployment

```bash
docker service ls --filter name=glance-swarm
curl -s -o /dev/null -w '%{http_code}\n' http://10.77.110.10:8084/api/healthz
curl -sk -o /dev/null -w '%{http_code}\n' https://glance.echoinggreen.net/
```

A healthy stack shows `1/1`, the healthcheck returns `200`, and the vhost
returns `200` for the healthcheck path.

**Widget content is not in the page HTML.** `GET /` returns the app shell.
Probe the content endpoint instead:

```bash
curl -s http://10.77.110.10:8084/api/pages/fleet/content/ | head -40
```

**Test the vhost from a client VLAN, not from the proxy host itself.** A
probe from `ore-dock-110-1` traverses a different path than a real client
and can produce a false negative.

To check the monitors actually work, run the fetch from INSIDE the
container - that is the vantage the monitors use:

```bash
docker exec <container> wget -qO- https://radarr.echoinggreen.net/ >/dev/null && echo OK
```

Every proxied monitor returning `403` means the container's egress address
is missing from NPM's access list, not that the service is down.
