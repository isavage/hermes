# Zeno — Hermes Agent deployment

A [Hermes Agent](https://github.com/NousResearch/hermes-agent) running in Docker on your VPS, with **all secrets injected from Doppler**.

## How it works


- **This repo only owns the container definition** (`docker-compose.yml`) and the deploy pipeline (`.github/workflows/deploy.yml`).
- **All Hermes configuration lives on the server**, under `./.hermes/` (mounted into the container at `/opt/data`). `config.yaml`, `SOUL.md` (personality), and `skills/` are edited through the **Hermes dashboard** ( — reverse-proxied to the container). Logs, sessions, memories, the SQLite DB, and credentials are runtime state.
- **`.hermes/` is fully gitignored** — nothing in it is tracked or deployed from this repo. On first boot, Hermes seeds a default `config.yaml` and `SOUL.md` automatically.
- **Secrets come only from Doppler** — never from the repo, never from files.

## Directory layout

| Path | Tracked? | Purpose |
|------|----------|---------|
| `docker-compose.yml` | ✅ | Container definition |
| `scripts/bootstrap-hermes-config.sh` | ✅ | First-run config bootstrap (dashboard auth from Doppler) |
| `.github/workflows/deploy.yml` | ✅ | Deploy pipeline |
| `.env.example` | ✅ | Doppler secret schema reference (no real values) |
| `README.md` | ✅ | This doc |
| `.hermes/` (config.yaml, SOUL.md, skills/, logs/, sessions/, state.db, .env, …) | ❌ | Everything Hermes manages on the server — dashboard + runtime state |

## Secrets: Doppler only

1. Create a Doppler project + config (e.g. `zeno-hermes` / `prod-IN`) and add the variables listed in [`.env.example`](.env.example) — provider keys, `TELEGRAM_BOT_TOKEN`, `HERMES_UID/GID`, etc.
2. Generate a **service token** in Doppler and store it as the GitHub secret `DOPPLER_TOKEN` (alongside `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`).
3. The workflow runs `doppler run -- docker compose ...` on the VPS, so `docker-compose.yml`'s `${VAR}` are substituted from Doppler and passed to the container as environment variables. Hermes reads them at runtime (they override `/opt/data/.env`).

Nothing secret is ever committed — the repo contains no keys, tokens, or `.env`.

## Deploy

Push to `main` (or `workflow_dispatch`):

```sh
git push
```

Manual refresh of the image (keep config/data):

```sh
gh workflow run "Deploy Zeno Hermes Agent" -f cleanup=true
```

## Manage the agent

Everything except the container definition is managed on the server:

- **Dashboard** (config, personality, skills): https://zeno.varunrs.in (reverse proxy → http://hermes:9119 over the `hermes.net` network)
- **SSH** directly to the VPS if you prefer files:
  ```sh
  ssh user@<vps>
  cd /docker/zeno-hermes
  nano .hermes/config.yaml    # or SOUL.md, or add skills/ entries
  doppler run -- docker compose restart hermes
  ```

### First run is fully automated

On the first deploy, `scripts/bootstrap-hermes-config.sh` (run by the workflow under `doppler run`) generates a valid `config.yaml` on the server — dashboard basic auth from `DASHBOARD_USERNAME` / `DASHBOARD_PASSWORD`, `_config_version: 12`, and `toolsets: [all]` (terminal tool enabled). No manual `hermes setup` needed.

- Add `DASHBOARD_USERNAME` and `DASHBOARD_PASSWORD` to your Doppler config.
- The bootstrap is **idempotent**: once `config.yaml` is provisioned, deploys never overwrite it — you manage it via the dashboard.
- To re-bootstrap (fresh VPS, or you deleted the config), remove `.hermes/config.yaml` on the server and redeploy.

## Agent access to your other Docker containers

ZenoVerify it works:** `docker exec hermes docker ps` should list your containers. (The first-run bootstrap sets `toolsets: [all]`, so the terminal tool is already enabled.)cket, mounted `:ro`.
- It enables only read endpoints (`CONTAINERS`, `LOGS`, `INFO`, `IMAGES`, `VOLUMES`, `NETWORKS`). Create/exec/delete are denied, and it's reachable only on `hermes.net` (no published ports).
- The `hermes` container talks to it over the private network via `DOCKER_HOST=tcp://docker-proxy:2375`.

Just ask Zeno, e.g. *"list my containers"* or *"show the last 50 lines of the nginx logs"* — it runs `docker ps` / `docker logs` through its terminal tool.

> **Two things to configure:**
> 1. Enable the **terminal** tool for the agent in the dashboard (a toolset that includes it, e.g. `all`) — that's how it runs `docker` commands.
> 2. Verify the path works: `docker exec hermes docker ps` should list your containers.

> **Security note:** this is still a powerful capability. It's scoped read-only, but `CONTAINERS`/`LOGS` can expose data from your other apps, so keep the proxy off the internet (it is — private network only).

## Local development

```sh
cp .env.example .env      # fill in locally (dev only; not committed)
doppler run -- docker compose up -d
docker compose ps
docker compose logs -f hermes
```

## Useful commands

```sh
# Open an interactive chat inside the running container
docker exec -it hermes hermes

# Check gateway health (no host ports published — run from inside the network)
docker exec hermes hermes status
```
