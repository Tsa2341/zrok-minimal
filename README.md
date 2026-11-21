# zrok-minimal

A trimmed, self-hosted [zrok2](https://github.com/openziti/zrok) stack for a small
server, deployed with Ansible.

It is derived from upstream `docker/compose/zrok2-instance` (commit `d8e5316`,
zrok2 `v2.0.11`) with everything not needed for personal tunnelling removed.

## What is in the stack

| Service | Role | Exposed |
|---|---|---|
| `ziti-controller` | OpenZiti control plane; zrok clients talk to it directly | `1280/tcp` |
| `ziti-router` | OpenZiti data plane | `3022/tcp` |
| `rabbitmq` | Share-name updates for the frontend (required by zrok2) | internal |
| `zrok2-controller` | zrok API and web console | via Caddy |
| `zrok2-frontend` | Serves public shares | via Caddy |
| `caddy` | HTTPS with a wildcard certificate (Cloudflare DNS challenge) | `443/tcp` |
| `ziti-router-init`, `zrok2-init`, `zrok2-frontend-files` | One-shot setup containers | — |

Measured memory with no shares running: about 290 MB for all containers.

### Removed from upstream

- PostgreSQL: zrok2 uses SQLite instead.
- InfluxDB and the metrics bridge: the console's usage graphs stay empty.
- RabbitMQ management UI and Prometheus plugin (`rabbitmq:3-alpine`, plugins off).
- The Caddy compile at container start: Caddy is built once on your machine.
- The public route to the Ziti management API on 443 (it is still on 1280,
  password-protected, because clients need that port).

### Changed from upstream

- Pinned image versions, a memory limit per container, log rotation.
- RabbitMQ runs with its own generated user instead of `guest`, and a fixed
  hostname so its data survives container recreation.
- `zrok2-controller` waits for the Ziti router. Upstream can start it first, and
  then public share names never resolve until the controller is restarted. The
  playbook also checks for this and restarts the controller if needed.
- Port 1280 is published publicly. Upstream's Caddy overlay binds it to
  localhost, which stops clients from connecting.
- The internet-facing frontend gets its own volume holding only its config and
  identity. Upstream mounts the whole config volume, which exposes `ctrl.yaml`
  (admin token, Ziti and RabbitMQ passwords), the database and the CLI session.
- Health checks are TCP or HTTP probes every 60 s. Upstream's checks start a
  `ziti` CLI and an Erlang VM every 5–10 s, which piles up and overloads a
  throttled 1-OCPU VM.
- `entrypoint-init.bash` reads the frontend token from the right table column.
  Upstream takes the table border (`│`) as the token, so public share names never
  resolve.

## Layout

```text
caddy/Dockerfile          Caddy with the Cloudflare DNS plugin
compose/                  compose.yml, Caddyfile, upstream entrypoint-init.bash
ansible/
  site.yml                Full deploy (roles: base, docker, zrok)
  account.yml             Create a zrok account
  password.yml            Set a new password for an account
  group_vars/all/main.yml Settings (versions, limits)
  group_vars/all/zone.yml   Your DNS zone (git-ignored; from zone.yml.example)
  group_vars/all/vault.yml  Cloudflare token (encrypted; you create it)
  .secrets/               Generated secrets (git-ignored)
  .build/                 Built Caddy image (git-ignored)
```

## Requirements

**Your machine:** Docker (for building Caddy), Python 3.13, and SSH access to the server.

**Server:** Ubuntu (tested on 26.04; the playbook uses apt and Docker's Ubuntu
repository) with SSH key login, x86_64 or arm64. The playbook adds 2 GB of
swap, which a 1 GB server needs.

**DNS (Cloudflare):** two A records pointing to the server, both **DNS only**
(grey cloud, so ports 1280 and 3022 are not proxied):

```text
<zone>     A  <server-ip>
*.<zone>   A  <server-ip>
```

and an API token from the **Edit zone DNS** template, limited to that zone.

**Cloud firewall**: allow inbound TCP
`22`, `443`, `1280` and `3022`. Docker-published ports bypass the server's own
iptables `INPUT` rules, so no host firewall change is needed.

## Setup

1. Install Ansible in a local virtualenv:

   ```bash
   python3.13 -m venv .venv && .venv/bin/pip install ansible-core
   ```

2. Point the inventory at your server. `ansible/inventory.ini` uses an SSH config
   host named `zrok-srv`:

   ```text
   Host zrok-srv
     HostName <server-ip>
     User ubuntu
     ServerAliveInterval 30
     ServerAliveCountMax 3
   ```

3. Set your zone (the file is git-ignored):

   ```bash
   cp ansible/group_vars/all/zone.yml.example ansible/group_vars/all/zone.yml
   ```

   and edit `zrok_dns_zone` in it.

4. Store the Cloudflare token:

   ```bash
   cd ansible && ../.venv/bin/ansible-vault create group_vars/all/vault.yml
   ```

   with the content:

   ```yaml
   vault_cloudflare_api_token: "<token>"
   ```

## Deploy

From `ansible/`:

```bash
../.venv/bin/ansible-playbook site.yml --ask-vault-pass
```

It prepares the server (swap, automatic security updates, key-only SSH, Docker),
builds Caddy locally for the server's CPU and copies it over, starts the stack,
and waits until `https://zrok2.<zone>` answers. Re-running it is safe and only
applies what changed. Use `--tags base`, `--tags docker` or `--tags zrok` to run
one part.

The admin token, Ziti password and RabbitMQ password are generated on the first
run into `ansible/.secrets/` and reused afterwards. Back that folder up: the
stack's data depends on them.

While testing, set `caddy_acme_ca` to the Let's Encrypt staging URL in
`main.yml` to avoid certificate rate limits.

## Accounts

```bash
../.venv/bin/ansible-playbook account.yml --ask-vault-pass
```

It prints an enable token. If the account already exists, it stops and says so.

To set a new password:

```bash
../.venv/bin/ansible-playbook password.yml --ask-vault-pass
```

zrok's password rules: at least 8 characters, a capital letter, a number and a
special character from `` !@#$%^&*()_+-=[]{};':"\|,.<> ``. zrok enforces them when
changing a password, so `password.yml` refuses a weak one. Creating an account
does not enforce them, so `account.yml` only warns.

## Using it

Web console: `https://zrok2.<zone>`.

On a client with the `zrok2` CLI:

```bash
zrok2 config set apiEndpoint https://zrok2.<zone>
```

```bash
zrok2 enable <enable-token>
```

```bash
zrok2 create name myapp
```

```bash
zrok2 share public http://127.0.0.1:3000 --headless --name-selection public:myapp
```

The share is then at `https://myapp.<zone>`. Creating the name and sharing are
separate commands in zrok2 2.0.11.

### Private names

Protect a named share with a password the browser asks for:

```bash
zrok2 share public http://127.0.0.1:3000 --headless --name-selection public:myapp --basic-auth-from-stdin
```

or with Google login (set up below):

```bash
zrok2 share public http://127.0.0.1:3000 --headless --name-selection public:myapp --oauth-provider google --oauth-email-domain 'you@gmail.com'
```

### Google login

1. In Google Cloud Console → APIs & Services, set up the OAuth consent screen
   with the `.../auth/userinfo.email` scope (add yourself as a test user while
   the app is in testing), then create an **OAuth client ID** of type **Web
   application** with this redirect URI:

   ```text
   https://oauth.<zone>/google/auth/callback
   ```

2. Add the client ID and secret to the vault
   (`../.venv/bin/ansible-vault edit group_vars/all/vault.yml`):

   ```yaml
   vault_oauth_google_client_id: "<client-id>"
   vault_oauth_google_client_secret: "<client-secret>"
   ```

3. Re-run `site.yml`. The signing and encryption keys are generated into
   `ansible/.secrets/`, and `https://oauth.<zone>` serves the login callbacks.

Without those vault entries, Google login stays off.

## Operating

On the server, the stack lives in `/opt/zrok-minimal` (root-only):

```bash
sudo bash -c 'cd /opt/zrok-minimal && docker compose ps'
```

```bash
sudo bash -c 'cd /opt/zrok-minimal && docker compose logs --tail=50 zrok2-frontend'
```

```bash
sudo docker stats --no-stream
```

**Upgrading:** change the tags in `main.yml` and re-run `site.yml`. For a new
Caddy version, change `caddy_version`; the image is rebuilt because the file name
includes the version.

## License

Apache-2.0; see `LICENSE`. `compose/entrypoint-init.bash` is from openziti/zrok
with a marked change to the frontend-token lookup, also Apache-2.0 (`compose/LICENSE.zrok`). This project is not
affiliated with OpenZiti or NetFoundry.
