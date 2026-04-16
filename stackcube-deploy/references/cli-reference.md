# Stackcube CLI Reference

Stackcube is a lightweight, self-hosted PaaS that deploys apps from a git remote onto a Docker host via a gRPC daemon.

## App Repository Requirements

- A compose file at the repo root: `compose.yml` or `docker-compose.yml`
- A `.env` file at the repo root defining `APP_PORT` (the port your app listens on inside the container)
- The compose file should use `expose` with `${APP_PORT}` (stackcube routes via its Docker network, not host ports)
- Each service must set `container_name` explicitly (stackcube routes traffic by container name)
- Containers are attached to the `stackcube-net` Docker network for front-proxy routing

## Commands

### `stackcube init`

One-time CLI setup. Configures which stackcube server to talk to.

```
stackcube init --server <host:port> [--identity <path-to-ssh-key>] [--tls]
```

- `--server` — daemon address (default: `localhost:9090`)
- `--identity` — SSH private key path (default: `~/.ssh/id_ed25519`)
- `--tls` — use TLS (required when connecting through a reverse proxy like Caddy)

### `stackcube deploy-key`

Prints the server's SSH public key. This key must be added to your git host so the daemon can fetch your code.

```
stackcube deploy-key
```

### `stackcube scaffold`

Generates starter `docker-compose.yml` and `.env` files for a new stackcube app. Sets `APP_PORT=3000` as a placeholder.

```
stackcube scaffold [--name <app-name>] [--force]
```

- `--name` — app name (defaults to current directory name)
- `--force` — overwrite existing files

### `stackcube lint`

Checks compose file and `.env` against stackcube requirements. Runs automatically before `deploy`.

**Errors (block deploy):**
- Missing `container_name` on any service
- No services defined

**Warnings (informational):**
- Missing `.env` or `APP_PORT` not set (defaults to 3000)
- Service uses `ports` instead of `expose`
- No restart policy

```
stackcube lint
```

### `stackcube create`

Registers a new app with the daemon. Must be run from inside a git repo.

```
stackcube create <appname> --domain <domain> [--port <host-port>]
```

- `--domain` — public domain for the app
- `--port` — host port (auto-assigned if not set)

### `stackcube deploy`

Deploys the current git branch/commit. Must be run from inside the app's git repo.

```
stackcube deploy <appname>
```

The daemon fetches the commit, builds the Docker image, runs `docker compose up`, and attaches containers to `stackcube-net`.

### `stackcube list`

Lists all managed apps.

```
stackcube list
```

### `stackcube config`

Updates app configuration.

```
stackcube config <appname> [--domain <domain>] [--port <host-port>]
```

### `stackcube destroy`

Destroys an app.

```
stackcube destroy <appname>
```
