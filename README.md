# hermes-agent

Self-hosted [Hermes Agent](https://github.com/NousResearch/hermes-agent) on a
Raspberry Pi 5 — one Docker container, managed via Portainer, cloud LLM backend,
Telegram as the messaging interface.

This repo holds the **deployment** (compose + config templates), not the agent
source. The image is built from the upstream project.

## Layout

| Path | Purpose |
|------|---------|
| `docker-compose.yml` | The single Hermes `gateway` service, its own stack. |
| `config.example.yaml` | Template for `~/.hermes/config.yaml` (non-secret behaviour). |
| `.env.example` | Template for `~/.hermes/.env` (API keys, bot token — never committed). |

Host state lives under `~/.hermes/` (mounted at `/opt/data`): `config.yaml`,
`.env`, `MEMORY.md`, `USER.md`, skills, and the session DB.

## Architecture

Read [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) first. In short: the agent runs
as the **unprivileged `hermes-runner` user (uid 1000)** and executes code through
**that user's rootless Podman socket** — never the root Docker socket. Code runs in
throwaway sandboxes; changes to real repos go through GitHub PRs you merge.

## First-time setup (on the Pi)

```bash
# 0. Create the unprivileged runner user + enable its rootless Podman socket.
sudo useradd -m -u 1000 hermes-runner   # or reuse an existing uid-1000 user
sudo loginctl enable-linger hermes-runner
sudo -u hermes-runner systemctl --user enable --now podman.socket   # -> /run/user/1000/podman/podman.sock

# 1. Prepare host state directory (owned by uid 1000).
mkdir -p ~/.hermes

# 2. Drop in config + secrets from the templates, then edit both.
cp config.example.yaml ~/.hermes/config.yaml
cp .env.example        ~/.hermes/.env
chmod 600 ~/.hermes/.env
#   Set GOOGLE_API_KEY + TELEGRAM_BOT_TOKEN in ~/.hermes/.env (Gemini is the default provider)
#   HERMES_UID/HERMES_GID default to 1000; DOCKER_HOST points at the rootless socket.
sudo chown -R 1000:1000 ~/.hermes

# 3. Build the image (no registry image is published upstream; build for arm64).
git clone https://github.com/NousResearch/hermes-agent.git /tmp/hermes-src
docker build -t hermes-agent:latest /tmp/hermes-src

# 4. Bring the stack up (or deploy this compose file as a Portainer stack).
docker compose up -d
docker compose logs -f hermes    # confirm it reads YOUR config + finds the socket
```

## Guardrails

- **Never commit** `.env` or anything under `~/.hermes/` — secrets and personal memory.
- `terminal.backend` is a **single global** — check `config.yaml` before assuming
  where the agent's code runs.
- **Only ever mount the rootless Podman socket** (`/run/user/1000/podman/podman.sock`),
  **never** `/var/run/docker.sock` — the root socket is root-equivalent.
- **Never** set `HERMES_DASHBOARD_INSECURE=1` or expose the dashboard on a public
  interface. This stack does not run the dashboard at all.
- Keep the stack **off `public-net`** / the Cloudflare tunnel. It's outbound-only
  (Telegram + cloud API) — no inbound port is needed.
- When editing `docker-compose.yml`, **preserve** the `~/.hermes:/opt/data` bind
  mount, the rootless socket mount, and the resource limits.
- **Don't auto-mount `hermes-runner`'s home into sandboxes** — projects only.
  The real blast radius is the whole `hermes-runner` user, not one container.

## Notes / deviations from upstream

- Upstream ships a two-service compose (`gateway` + `dashboard`) on
  `network_mode: host`. This deployment runs **gateway only** on a dedicated
  **bridge** network, with **Pi 5 resource limits** — per the guardrails above.
- On Portainer, if `~` isn't expanded for the bind mount, replace `~/.hermes`
  with the absolute host path (e.g. `/home/<user>/.hermes`).

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the design and rationale,
and [`docs/IMPROVEMENTS.md`](docs/IMPROVEMENTS.md) for what can be added, changed,
and improved.
