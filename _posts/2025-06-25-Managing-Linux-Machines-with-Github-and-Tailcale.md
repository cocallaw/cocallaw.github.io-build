---
title: "Manage Linux Machines with Github Actions and Tailscale"
date: 2025-06-25 12:00:00 -500
categories: [Tailscale, Github]
tags: [GitHub Actions, Slack, Discord, CI/CD, Automation, Webhooks]
author: CLC
description: How to manage Linux machines using GitHub Actions and Tailscale with secure SSH access and automation.
toc: true
comments: true
media_subpath: /post-content/05-gh-tailscale
---

## Managing Homelab Linux Servers with GitHub Actions and Tailscale

Maintaining a homelab with multiple Linux servers often leads to repetitive tasks like updates, container cleanup, and application upgrades. With GitHub Actions and [Tailscale SSH](https://tailscale.com/kb/1190/tailscale-ssh/), we can automate these tasks across any private network — without opening a single port or exposing services.

In this post, I’ll show you how I use GitHub Actions + Tailscale to manage my Pi-hole boxes, Docker hosts, and Ubuntu servers — safely and automatically.

---

## ✅ Overview

We’ll cover:

1. Creating a dedicated `github-runner` user
2. Restricting sudo access using `visudo`
3. Setting up Tailscale:
   - Enabling SSH
   - Applying tags and ACLs
   - Creating an OAuth client for GitHub Actions
4. Automating:
   - OS updates
   - Pi-hole upgrades
   - Docker cleanup
   - Using GitHub Actions + JSON-based machine tagging

---

## 👤 Step 1: Create the `github-runner` user

On each machine:

```bash
sudo adduser --disabled-password --gecos "" github-runner
```

---

## 🔒 Step 2: Limit `sudo` access with visudo

We only want `github-runner` to run specific commands with `sudo`.

Run:

```bash
sudo visudo -f /etc/sudoers.d/github-runner
```

Then paste:

```sudoers
Defaults:github-runner env_keep += "DEBIAN_FRONTEND"
github-runner ALL=(ALL) NOPASSWD: /usr/bin/apt, /usr/bin/apt-get, /usr/bin/sh, /usr/bin/pihole, /usr/bin/docker
```

This grants:
- Passwordless sudo only for update commands, Pi-hole, and Docker
- `DEBIAN_FRONTEND` to suppress interactive prompts

---

## 🛜 Step 3: Set up Tailscale

### Enable Tailscale SSH

On each machine:

```bash
sudo tailscale up --ssh --authkey tskey-...
```

Or, use ephemeral keys via GitHub Actions.

### Configure ACLs in the Tailscale admin:

```json
{
  "tagOwners": {
    "tag:ghup": ["autogroup:admin"],
    "tag:ghci": ["autogroup:admin"]
  },
  "ssh": [
    {
      "action": "accept",
      "src": ["tag:ghci"],
      "dst": ["tag:ghup"],
      "users": ["github-runner"]
    }
  ]
}
```

### Tag each machine in the Tailscale admin

Example:

- `usvr.tail9990.ts.net`: `tag:ghup`
- GitHub runner (when connected): `tag:ghci`

### Create a Tailscale OAuth client

Visit [https://login.tailscale.com/admin/oauth](https://login.tailscale.com/admin/oauth)

- Grant these scopes:
  - `device:create`
  - `device:read`
  - `device:delete`
- Save the **Client ID** and **Client Secret** in GitHub repo secrets:
  - `TS_OAUTH_CLIENT_ID`
  - `TS_OAUTH_SECRET`

---

## ⚙️ Step 4: Define Machines in JSON

Create `machines.json` in the root of your repo:

```json
{
  "machines": [
    { "name": "dsvr.tail0990.ts.net", "tags": ["update-os", "containers"] },
    { "name": "pi-dns-01.tail0990.ts.net", "tags": ["pihole"] }
  ]
}
```

---

## 🚀 Step 5: GitHub Workflows

### 🔧 1. OS Updates: `.github/workflows/update-os.yml`

```yaml
name: Update OS
on: [workflow_dispatch]

jobs:
  generate-matrix:
    ...
    run: |
      MACHINE_LIST=$(jq '[.machines[] | select(.tags | index("update-os")) | .name]' machines.json)
      echo "matrix={\"machine\": $MACHINE_LIST}" >> "$GITHUB_OUTPUT"

  update:
    ...
    run: |
      ssh -o StrictHostKeyChecking=no github-runner@${{ matrix.machine }} << 'EOF'
        echo "Running on: $(hostname)"
        echo "User: $(whoami)"
        sudo DEBIAN_FRONTEND=noninteractive apt update -y
        sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y
        sudo DEBIAN_FRONTEND=noninteractive apt autoremove -y
EOF
```

---

### 🧼 2. Clean Docker Containers: `.github/workflows/clean-containers.yml`

```yaml
name: Clean Docker Containers
on: [workflow_dispatch]

jobs:
  generate-matrix:
    ...
    run: |
      MACHINE_LIST=$(jq '[.machines[] | select(.tags | index("containers")) | .name]' machines.json)
      echo "matrix={\"machine\": $MACHINE_LIST}" >> "$GITHUB_OUTPUT"

  clean-containers:
    ...
    run: |
      ssh -o StrictHostKeyChecking=no github-runner@${{ matrix.machine }} << 'EOF'
        echo "Cleaning on: $(hostname)"
        sudo docker system prune -f
EOF
```

---

### 🛠 3. Pi-hole Update: `.github/workflows/update-pihole.yml`

```yaml
name: Update Pi-hole
on: [workflow_dispatch]

jobs:
  generate-matrix:
    ...
    run: |
      MACHINE_LIST=$(jq '[.machines[] | select(.tags | index("pihole")) | .name]' machines.json)
      echo "matrix={\"machine\": $MACHINE_LIST}" >> "$GITHUB_OUTPUT"

  update:
    ...
    run: |
      ssh -o StrictHostKeyChecking=no github-runner@${{ matrix.machine }} << 'EOF'
        echo "Running on: $(hostname)"
        sudo pihole -up
EOF
```

---

## 📌 Recap

- ✅ Tailscale gives secure, zero-config SSH access across any network
- ✅ GitHub Actions orchestrates updates on a schedule or on-demand
- ✅ Tags and JSON-based config make targeting servers flexible
- ✅ Principle of least privilege enforced with `visudo`

---

## 📎 Bonus Ideas

- Add Discord/Slack notifications
- Include logging to a GitHub artifact or centralized log store
- Add reboot scheduling for updates that require it
- Use `tailscale logout` to explicitly clean up ephemeral runners

---

## 💬 Final Thoughts

This setup has been rock solid in my homelab. I don’t log into boxes for routine updates anymore — GitHub Actions does it safely, and Tailscale ensures it's secure and private.

Let me know if you'd like to see a template repo with this structure — or a GUI front-end to trigger workflows per machine group.