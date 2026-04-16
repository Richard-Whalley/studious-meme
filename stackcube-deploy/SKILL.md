---
name: stackcube-deploy
description: Prepare and deploy apps to a stackcube self-hosted PaaS. Use this skill whenever the user mentions stackcube, wants to deploy to their own server using stackcube, needs to make a project stackcube-compatible, or asks about stackcube deployment. This covers the full workflow from adapting compose files through to a live deploy. Even if the user just says "deploy this" in a repo that has stackcube config, use this skill.
---

# Stackcube Deploy

Deploy apps to a stackcube self-hosted PaaS. This skill has two phases:

- **Phase 1 (Prepare)** — make the repo stackcube-compatible
- **Phase 2 (Deploy)** — configure the CLI, register the app, and deploy

If the user asks to "prepare" or "make this stackcube-compatible", run Phase 1 only. If they ask to "deploy", run both phases. If the request is ambiguous, run both.

For CLI syntax details, read `references/cli-reference.md`.

## Phase 1: Prepare

The goal is to get `stackcube lint` passing. Stackcube needs three things in the repo root: a compose file, a `.env` with `APP_PORT`, and a Dockerfile.

### Step 1: Check what exists

Look for these files in the repo root:
- `compose.yml` or `docker-compose.yml`
- `.env`
- `Dockerfile`

If there is no Dockerfile, warn the user that stackcube needs one to build the image. Do not generate a Dockerfile — it's too app-specific. Stop Phase 1 here and ask the user to create one first.

### Step 2: Infer APP_PORT

Before touching any files, figure out what port the app listens on:
1. Check the Dockerfile for an `EXPOSE` directive — this is the most reliable signal
2. Check existing compose file for port mappings (e.g., `"8080:8080"` means the app listens on 8080)
3. Check the app's source code for a hardcoded listen port (e.g., `http.ListenAndServe(":8080", nil)` in Go, `app.listen(3000)` in Node)

If you can't determine the port, ask the user.

### Step 3: Create or adapt files

**If no compose file exists:**
Run `stackcube scaffold --name <directory-name>` to generate both the compose file and `.env`. Then update `APP_PORT` in `.env` to the value from Step 2.

**If a compose file already exists:**
Adapt it to meet stackcube requirements. The changes needed are:

1. **Add `container_name`** to every service. Use the pattern `<directory-name>-<service-name>` (e.g., `my-app-web`). Stackcube routes traffic by container name, so this is mandatory.

2. **Replace `ports` with `expose`**. Stackcube routes via its Docker network (`stackcube-net`), not host ports. Change:
   ```yaml
   ports:
     - "8080:8080"
   ```
   to:
   ```yaml
   expose:
     - "${APP_PORT}"
   ```

3. **Add `env_file: .env`** to each service so the container receives `APP_PORT`.

4. **Add `restart: unless-stopped`** if no restart policy is set.

5. **Add environment pass-through** so `APP_PORT` is available inside the container:
   ```yaml
   environment:
     - APP_PORT=${APP_PORT}
   ```

**For the `.env` file:**
- If it doesn't exist, create it with `APP_PORT=<inferred-port>`
- If it exists but doesn't have `APP_PORT`, add it
- If it exists and has `APP_PORT`, leave it alone

### Step 4: Lint

Run `stackcube lint` and check the output:
- If it passes (exit code 0), Phase 1 is complete
- If there are errors, read the output, fix the issues, and run lint again
- Warnings are informational — mention them to the user but don't block on them

### Step 5: Report

Tell the user what files were created or changed and that the repo is now stackcube-compatible. If running both phases, proceed to Phase 2.

---

## Phase 2: Deploy

The goal is to get the app live on the stackcube server.

### Step 1: Check CLI initialization

Run `stackcube list`. If it errors (connection refused, not configured, etc.), the CLI hasn't been initialized yet.

If not initialized, ask the user for:
- **Server address** (`host:port`) — required, no sensible default
- **SSH identity path** — offer the default (`~/.ssh/id_ed25519`)
- **TLS** — ask if they're connecting through a reverse proxy

Then run:
```bash
stackcube init --server <host:port> [--identity <path>] [--tls]
```

Verify it worked by running `stackcube list` again.

### Step 2: Handle deploy key

The stackcube server needs SSH access to pull from the git remote. Run:
```bash
stackcube deploy-key
```

Save the output (the public key) to a temporary file.

**Detect the git host:** Parse the git remote URL (`git remote get-url origin`).

**If it's a GitHub repo** (remote contains `github.com`):
1. Extract the `OWNER/REPO` from the remote URL
2. Run: `gh repo deploy-key add <key-file> --title stackcube -R OWNER/REPO`
3. If it fails because the key already exists, that's fine — continue
4. Clean up the temp file

**If it's not GitHub:**
1. Print the deploy key to the user
2. Explain: "Add this as a deploy key in your git host's settings (read access is sufficient)"
3. Ask the user to confirm when done before continuing

### Step 3: Ask for domain

Ask the user: "What domain should this app be available at?" (e.g., `myapp.example.com`)

This is required for the `create` command. There is no default — always ask.

### Step 4: Check if app already exists

Run `stackcube list` and check if an app with this name already exists. If it does, skip to Step 6 (deploy).

### Step 5: Create the app

Use the current directory name as the app name. Confirm with the user:
> "I'll register this as `<app-name>` on stackcube with domain `<domain>`. Good to go?"

Then run:
```bash
stackcube create <app-name> --domain <domain>
```

### Step 6: Check git state

Run `git status`. If there are uncommitted changes (especially compose file and `.env` changes from Phase 1), tell the user:
> "There are uncommitted changes. Stackcube deploys the current commit, so these changes need to be committed and pushed first."

Ask if they'd like to commit now. Do not commit without their approval — they may want to review the changes first.

After committing, ensure changes are pushed to the remote (`git push`), since the stackcube daemon fetches from the remote.

### Step 7: Deploy

Run:
```bash
stackcube deploy <app-name>
```

This streams progress events. Let the output flow to the user so they can see the build and deploy progress.

### Step 8: Report

Confirm the deploy succeeded and tell the user their app is live at the domain they specified.

---

## Edge Cases

- **App already registered** — `stackcube list` before `create`. If it exists, skip to deploy.
- **Deploy key already added** — `gh repo deploy-key add` errors if the key exists. Catch the error and continue.
- **Lint still fails after edits** — read the lint output carefully, fix remaining issues, re-run. Don't give up after one attempt.
- **No Dockerfile** — warn and stop. Don't generate one.
- **Dirty git state** — always check before deploy. Prompt user to commit + push.
- **Non-GitHub remotes** — fall back to manual deploy key instructions.
- **Multiple services in compose** — add `container_name` to every service, not just the first.
