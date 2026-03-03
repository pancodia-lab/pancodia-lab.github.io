---
layout: post
title: "Stop Typing Your GitHub Token: Use gh as Your Git Credential Helper"
date: 2026-03-03
categories: [devtools, github]
tags: [github, git, authentication, pat, gh-cli, systemd]
description: "How to set up GitHub CLI as your git credential helper so you never paste a PAT again — plus a pattern for headless services."
---

If you've ever found yourself pasting a GitHub Personal Access Token (PAT) into a terminal prompt for the fifth time in one afternoon, or wondering why `git push` suddenly stopped working after you rotated a token — this guide is for you.

We'll walk through how to set up **GitHub CLI (`gh`)** as the single source of truth for GitHub authentication, so that both `gh` commands (like `gh repo create`, `gh pr list`) and plain `git` operations (like `git push`, `git fetch`) authenticate seamlessly — without you ever hardcoding a token into a script or pasting one into chat.

We'll also cover a second pattern: how to provide GitHub access to a **headless service** (like an AI agent gateway running as a systemd unit) using an environment variable, and how the two approaches interact.

## The Problem

When you work with private GitHub repositories over HTTPS, every `git push` or `git fetch` needs credentials. Out of the box, git will prompt you for a username and password (which, since 2021, must be a PAT — not your actual GitHub password).

This leads to a few common pain points:

- **Repetitive prompts.** You get asked for credentials on every push/pull unless you configure a credential helper.
- **Token sprawl.** You end up with tokens copy-pasted into `.env` files, shell histories, CI configs, and chat logs.
- **Stale tokens.** You rotate or regenerate a PAT (e.g., to extend its expiration), and suddenly half your tooling breaks because the old token is cached somewhere you forgot about.
- **Subprocess surprises.** A token set via `export GITHUB_TOKEN=...` in your shell works fine — until a background service, cron job, or subprocess doesn't inherit that environment variable.

The solution is to centralize authentication through `gh` and let it handle credential distribution.

## How `gh` Authentication Works

GitHub CLI (`gh`) can authenticate in two independent ways. Understanding the difference is key to avoiding surprises.

### Method 1: Environment Variable (Stateless)

If `GITHUB_TOKEN` (or `GH_TOKEN`) is set in the current shell environment, `gh` picks it up automatically. No login step needed.

```bash
export GITHUB_TOKEN="github_pat_xxxx..."
gh auth status
# ✓ Logged in to github.com as your-user (GITHUB_TOKEN)
```

**Pros:**
- Dead simple. Set the variable, done.
- Great for CI, Docker containers, and systemd services where you control the environment.

**Cons:**
- Only works where the variable is present. Open a new terminal tab, SSH into the same machine from a different session, or spawn a subprocess that doesn't inherit env — and auth breaks.
- `gh auth status` will show the env token, but `gh` hasn't "stored" anything. Remove the variable and `gh` forgets who you are.

### Method 2: Stored Auth Context (Stateful)

Running `gh auth login` walks you through an interactive flow that stores credentials persistently — typically in `~/.config/gh/hosts.yml`.

```bash
gh auth login
# Interactive prompts:
#   → GitHub.com
#   → HTTPS
#   → Paste your PAT
gh auth status
# ✓ Logged in to github.com as your-user (~/.config/gh/hosts.yml)
```

**Pros:**
- Survives new shells, reboots, and SSH sessions (for the same OS user).
- Enables the **credential helper** pattern (see next section).

**Cons:**
- Token is stored on disk. You need to trust the machine and user account.

### Which One Wins?

If both are present, **the environment variable takes priority**. This is important: if you have a stale `GITHUB_TOKEN` in your environment but a fresh token stored via `gh auth login`, `gh` will still try (and fail with) the stale env token.

This is the most common "it was working yesterday" gotcha.

## The Key Trick: `gh` as Git's Credential Helper

Here's where it gets powerful. Git has a pluggable **credential helper** system — you can tell git to ask an external program for credentials instead of prompting you. GitHub CLI can act as that program.

After logging in, run:

```bash
gh auth setup-git
```

This modifies your global git config (usually `~/.gitconfig`) to include something like:

```ini
[credential "https://github.com"]
    helper = !/usr/bin/gh auth git-credential
```

From this point on, whenever git needs credentials for `https://github.com/...`, it shells out to `gh auth git-credential`, which returns the stored token. You never see a prompt.

### Verify the Setup

```bash
# Check gh is logged in
gh auth status

# Check git knows to ask gh for credentials
git config --global --get credential.https://github.com.helper
# Expected: !/path/to/gh auth git-credential

# Test with a real private repo
git ls-remote https://github.com/your-org/your-private-repo.git
# Should print refs without prompting for credentials
```

### Why This Matters for Robustness

Consider a scenario:
1. You set `GITHUB_TOKEN` in your shell and everything works.
2. A background process (e.g., a git hook, a VS Code extension, a cron job) runs `git fetch` but doesn't inherit your shell's environment.
3. Without the credential helper, git prompts for credentials (or just fails silently).
4. With `gh auth setup-git`, git asks `gh`, which uses its stored auth context — no env variable needed.

This is why **running both `gh auth login` and `gh auth setup-git` is good practice**, even if you also set `GITHUB_TOKEN` in your shell.

## Setup Guide: Interactive Developer Workflow

This is the recommended setup for your laptop, workstation, or any machine where you do interactive development.

### Step 1: Generate a PAT

1. Go to [GitHub → Settings → Developer settings → Personal access tokens](https://github.com/settings/tokens)
2. Choose **Fine-grained token** (recommended) or **Classic token**
3. Set an expiration (shorter is better — 30 to 90 days)
4. Grant the scopes you need:
   - For basic repo access: `repo` (classic) or repository read/write (fine-grained)
   - For creating repos in an org: may also need Administration permissions, depending on org policy

### Step 2: Authenticate `gh`

```bash
gh auth login
```

Follow the prompts:
- **Where do you use GitHub?** → GitHub.com
- **Preferred protocol?** → HTTPS
- **Authenticate with?** → Paste an authentication token
- Paste your PAT

### Step 3: Enable the Git Credential Helper

```bash
gh auth setup-git
```

### Step 4: Verify Everything

```bash
# gh can talk to GitHub
gh auth status

# git can talk to GitHub (test with a private repo you have access to)
gh repo view your-org/your-repo --json nameWithOwner,isPrivate,viewerPermission

# git operations work without prompts
git ls-remote https://github.com/your-org/your-repo.git
```

## Setup Guide: Headless Service (systemd)

For a service like an AI agent gateway that runs unattended, the environment variable approach is simpler and more appropriate. There's no human to run `gh auth login`, and the service environment is deterministic.

### Step 1: Create a Secrets File

```bash
sudo mkdir -p /etc/yourservice
sudo touch /etc/yourservice/secrets.env
sudo chmod 600 /etc/yourservice/secrets.env
sudo chown root:root /etc/yourservice/secrets.env
```

### Step 2: Add Your Token

```bash
sudo nano /etc/yourservice/secrets.env
```

Add:
```bash
GITHUB_TOKEN=github_pat_xxxx...
```

### Step 3: Reference It in the systemd Unit

In your service file (e.g., `/etc/systemd/system/your-service.service`):

```ini
[Service]
EnvironmentFile=/etc/yourservice/secrets.env
```

### Step 4: Reload and Restart

```bash
sudo systemctl daemon-reload
sudo systemctl restart your-service
sudo systemctl status your-service --no-pager
```

The service process will always have `GITHUB_TOKEN` in its environment, so `gh` and `git` commands run by the service will authenticate automatically.

### Do You Still Need `gh auth login`?

For the **service itself**: no. `GITHUB_TOKEN` in the environment is sufficient.

For **interactive use on the same machine** (e.g., SSH in and run `gh pr list`): yes, because your interactive shell may not have `GITHUB_TOKEN` set. Running `gh auth login` + `gh auth setup-git` ensures your interactive sessions also work.

## Troubleshooting

### "The token in GITHUB_TOKEN is no longer valid"

You'll see this after rotating a PAT if you forgot to update it everywhere.

**Diagnose:** Is `gh` using the env token or stored auth?

```bash
# This tells you which auth source gh is using
gh auth status

# Temporarily ignore env tokens to test stored auth
env -u GITHUB_TOKEN -u GH_TOKEN gh auth status
```

**Fix:** Update the token in whichever source is stale:
- Environment: update your secrets env file, then restart the service
- Stored auth: re-run `gh auth login` with the new PAT

### "Repository not found" on a Private Repo

This usually means the token doesn't have access to the repo (wrong scopes, wrong org, or the authenticated user isn't a collaborator).

```bash
# Quick access check
gh repo view org/repo --json nameWithOwner,isPrivate,viewerPermission
```

If this fails, the token either lacks repo scope or the authenticated user doesn't have access.

### `gh repo create` Fails with "CreateRepository" Permission Error

This is common with **fine-grained PATs** and **organization repos**. Even with "Administration: Read and write" on the token, org-level policies may block programmatic repo creation.

**Workaround:** Create the repo manually in the GitHub UI, then add the remote locally:

```bash
gh repo create org/repo --private --source . --remote origin --push
# If that fails, create in UI first, then:
git remote add origin https://github.com/org/repo.git
git push -u origin main
```

## Quick Reference

| Scenario | Auth method | Commands |
|---|---|---|
| Laptop / interactive dev | Stored auth + credential helper | `gh auth login` → `gh auth setup-git` |
| systemd service | Environment variable | `GITHUB_TOKEN=...` in `EnvironmentFile` |
| CI / Docker | Environment variable | `GITHUB_TOKEN` as secret/env |
| One-off test | Ephemeral env | `GH_TOKEN=... gh repo view ...` |

## Security Checklist

- PAT has the **shortest practical expiration** (30–90 days)
- PAT is **never committed** to a repo or pasted into chat
- Secrets env file is `chmod 600`, owned by `root:root`
- Old PATs are **revoked** after rotation (not just replaced)
- Consider migrating to **GitHub Apps** for shorter-lived, auto-rotating tokens as your setup matures
