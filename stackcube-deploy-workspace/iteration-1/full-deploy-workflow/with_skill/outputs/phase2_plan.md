# Phase 2: Deploy Plan

Phase 2 requires a live stackcube server and interactive user input. Below is the step-by-step plan documenting what would be done and what the user would be asked at each stage.

## Step 1: Check CLI initialization

**Action:** Run `stackcube list` to check if the CLI is configured.

**If not initialized, ask the user:**
- "What is the address (host:port) of your stackcube server?"
- "What SSH identity file should I use? (default: ~/.ssh/id_ed25519)"
- "Are you connecting through a reverse proxy that requires TLS? (yes/no)"

**Then run:**
```bash
stackcube init --server <host:port> [--identity <path>] [--tls]
```

**Verify:** Run `stackcube list` again to confirm the connection works.

## Step 2: Handle deploy key

**Action:** Run `stackcube deploy-key` to get the server's SSH public key.

**Detect git host:** Run `git remote get-url origin` to parse the remote URL.

**If GitHub repo:** Extract OWNER/REPO from the remote and run:
```bash
gh repo deploy-key add <key-file> --title stackcube -R OWNER/REPO
```
If the key already exists, continue (non-fatal error).

**If not GitHub:** Print the deploy key to the user and say:
> "Add this as a deploy key in your git host's settings (read access is sufficient). Let me know when done."

Wait for user confirmation before proceeding.

## Step 3: Ask for domain

**Ask the user:** "What domain should this app be available at? (e.g., myapp.example.com)"

This is required -- there is no default value.

## Step 4: Check if app already exists

**Action:** Run `stackcube list` and check if an app named `studious-meme` already exists.

- If it exists, skip to Step 6 (deploy).
- If not, proceed to Step 5.

## Step 5: Create the app

**Confirm with user:**
> "I'll register this as `studious-meme` on stackcube with domain `<domain>`. Good to go?"

**Then run:**
```bash
stackcube create studious-meme --domain <domain>
```

## Step 6: Check git state

**Action:** Run `git status`.

The adapted compose file and new .env file from Phase 1 would be uncommitted changes. Tell the user:
> "There are uncommitted changes (docker-compose.yml and .env from Phase 1). Stackcube deploys the current commit, so these changes need to be committed and pushed first."

**Ask:** "Would you like me to commit these changes now?"

If yes, commit and push:
```bash
git add docker-compose.yml .env
git commit -m "Make repo stackcube-compatible"
git push
```

## Step 7: Deploy

**Action:** Run:
```bash
stackcube deploy studious-meme
```

This streams build and deploy progress to the terminal.

## Step 8: Report

Confirm deploy succeeded and tell the user:
> "Your app is live at <domain>."

---

## Interactive steps summary

The following steps require user input and cannot be automated without interaction:

1. **Server address** (Step 1) -- no sensible default, must ask
2. **SSH identity path** (Step 1) -- has default ~/.ssh/id_ed25519, confirm with user
3. **TLS setting** (Step 1) -- depends on their server setup
4. **Deploy key confirmation** (Step 2) -- if non-GitHub host, user must manually add the key
5. **Domain** (Step 3) -- no default, must ask
6. **App creation confirmation** (Step 5) -- confirm name and domain before registering
7. **Commit approval** (Step 6) -- user may want to review changes before committing

## Steps requiring a live server

All Phase 2 steps require connectivity to a running stackcube daemon:
- `stackcube list` (Steps 1, 4)
- `stackcube init` (Step 1)
- `stackcube deploy-key` (Step 2)
- `stackcube create` (Step 5)
- `stackcube deploy` (Step 7)
