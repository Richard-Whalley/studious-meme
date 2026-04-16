# Stackcube Preparation Summary

## What was done

This Go web application (Go + HTMX + Alpine.js + Pico.css) was prepared for deployment on stackcube, a self-hosted PaaS.

### APP_PORT inference

The app listens on port **8080**, determined from three sources:
- `Dockerfile`: `EXPOSE 8080`
- `docker-compose.yml`: port mapping `"8080:8080"`
- `main.go`: `http.ListenAndServe(":8080", nil)`

### Files created

- **`.env`** — Created with `APP_PORT=8080`

### Files modified

- **`docker-compose.yml`** — Adapted for stackcube requirements:
  1. Added `container_name: studious-meme-web` (required by stackcube for traffic routing)
  2. Replaced `ports: - "8080:8080"` with `expose: - "${APP_PORT}"` (stackcube routes via its Docker network, not host ports)
  3. Added `env_file: .env` so the container receives the APP_PORT variable
  4. Added `environment: - APP_PORT=${APP_PORT}` to pass the port into the container
  5. Kept existing `restart: unless-stopped` policy (already compliant)

### Files unchanged (copied as-is)

- `main.go`
- `go.mod`
- `Dockerfile`
- `templates/index.html`

## Lint expectations

Based on the stackcube lint rules, this configuration should pass with:
- **No errors**: all services have `container_name`, services are defined
- **No warnings**: `.env` exists with `APP_PORT`, `expose` is used instead of `ports`, restart policy is set

## Next steps

To deploy, run Phase 2 which involves:
1. Initializing the stackcube CLI (`stackcube init --server <host:port>`)
2. Adding the deploy key to the git remote
3. Registering the app (`stackcube create`)
4. Committing and pushing changes
5. Deploying (`stackcube deploy`)
