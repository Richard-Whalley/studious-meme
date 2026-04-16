# Phase 2: Deploy Plan

Phase 2 requires a live stackcube server and interactive user input. Below is the step-by-step plan documenting what would be done at each stage.

## Step 1: Check CLI initialization

**Command:** `stackcube list`

**If it errors (not configured):** Ask the user for:
- Server address (host:port) -- required, no default
- SSH identity path -- offer default `~/.ssh/id_ed25519`
- Whether TLS is needed (reverse proxy like Caddy)

Then run:
```bash
stackcube init --server <host:port> [--identity <path>] [--tls]
```

Verify by running `stackcube list` again.

**Interactive:** This step requires the user to provide their server address and connection details.

## Step 2: Handle deploy key

**Command:** `stackcube deploy-key`

Save the output (public key) to a temp file.

**Detect git host:** Run `git remote get-url origin` to determine the host.

The repo module path is `github.com/Richard-Whalley/reimagined-journey`, suggesting it is a GitHub repo. If the remote confirms `github.com`:

```bash
gh repo deploy-key add <key-file> --title stackcube -R Richard-Whalley/reimagined-journey
```

If the key already exists, the error is safe to ignore. Clean up the temp file.

**If not GitHub:** Print the deploy key and ask the user to add it manually to their git host settings (read access is sufficient), then confirm when done.

**Interactive:** May require user confirmation if not a GitHub remote.

## Step 3: Ask for domain

**Interactive:** Ask the user: "What domain should this app be available at?" (e.g., `studious-meme.example.com`)

This is required for the `create` command. There is no default.

## Step 4: Check if app already exists

**Command:** `stackcube list`

Check if an app named `studious-meme` already exists. If it does, skip to Step 6.

## Step 5: Create the app

Confirm with the user:
> "I'll register this as `studious-meme` on stackcube with domain `<domain>`. Good to go?"

**Command:**
```bash
stackcube create studious-meme --domain <domain>
```

**Interactive:** Requires user confirmation before proceeding.

## Step 6: Check git state

**Command:** `git status`

If there are uncommitted changes (the modified docker-compose.yml and new .env from Phase 1), inform the user:
> "There are uncommitted changes. Stackcube deploys the current commit, so these changes need to be committed and pushed first."

Ask if they want to commit now. Do not commit without approval.

After committing, push to remote:
```bash
git push
```

**Interactive:** Requires user approval to commit and push.

## Step 7: Deploy

**Command:**
```bash
stackcube deploy studious-meme
```

This streams build and deploy progress to the terminal.

**Requires live server:** The stackcube daemon fetches the commit from the git remote, builds the Docker image, runs `docker compose up`, and attaches containers to `stackcube-net`.

## Step 8: Report

Confirm the deploy succeeded and tell the user their app is live at the domain they specified.
