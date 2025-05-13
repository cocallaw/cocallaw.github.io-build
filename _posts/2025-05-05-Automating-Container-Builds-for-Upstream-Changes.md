---
title: "Automating Container Builds with GitHub Actions for Upstream Changes"
date: 2025-05-05 08:00:00 -500
categories: [GitHub Actions, Docker, Automation, DevOps]
tags: [DevOps, Automation]
author: CLC
description: Automating container builds with GitHub Actions when the upstream image changes.
toc: true
comments: true
media_subpath: /post-content/04-ghacontainerbld
---

Keeping your container images up to date is crucial for security and performance. Whether you're running a homelab or managing production workloads, you want to ensure that your containers are built with the latest upstream changes. However, manually keeping track of updates and rebuilding containers can be tedious.

I use [Tailscale](https://tailscale.com/) to easily access services I run at home, or to easily connect to a lab environment in Azure from wherever I may be. To make this easier, I extended the Tailscale container image so I could deploy it as a [subnet router](https://github.com/cocallaw/terraform-azure-tailscale-subnet-router) and inject the address ranges that I needed advertised to my Tailnet. This made my setup more resilient, but I still had to make sure my container was using the latest version of Tailscale to make sure I was secure and prevent possible issues.

Until recently, I had a simple [GitHub Actions](https://docs.github.com/en/actions) workflow that would build and push to Docker Hub, whenever I remembered to check for updates (which I didn’t do very regularly) to the upstream Tailscale image.

In this post, I’ll walk through how I use a GitHub Actions workflow to **automatically check for updates to an upstream Docker image**, and **trigger a build and push only if a new version is available**. This kind of automation saves time, reduces human error, and keeps my environment reliably up to date.

## Why Automate Container Builds?

If you're maintaining a custom container that wraps or extends functionality from an upstream project (like Tailscale, Nginx, or Node.js), you'll often want to:

- Track upstream releases.
- Rebuild your container with the new version.
- Push the updated image to Docker Hub or a private registry.
- Optionally commit a version change back to your repo for tracking.

Doing all this manually is inefficient. Automating this with GitHub Actions ensures consistency and frees you to focus on more important tasks.

## The Workflow

Here’s a breakdown of the GitHub Actions workflow I use. It:

- Runs every 3 days or on manual trigger.
- Checks the latest stable tag of the upstream container.
- Compares it against images already published.
- If a newer version is available, builds and pushes multi-arch images.
- Commits a version file update for visibility.

## Secrets You Will Need

Before you can use this workflow, you'll need to set up a few secrets in your GitHub repository to allow the workflow to authenticate with Docker Hub. The workflow uses a [Docker Hub username and an access token](https://docs.docker.com/security/for-developers/access-tokens/) for authentication.

- `DOCKERHUB_USERNAME`: Your Docker Hub username.
- `DOCKERHUB_TOKEN`: A Docker Hub access token with write permissions.

To set these secrets, go to your GitHub repository, click on **Settings**, then **Secrets and variables** > **Actions**. Click on **New repository secret** to add each secret.

## Workflow Breakdown

Let's walk through what each section of the workflow does and how it contributes to the build and update process.

### 1. Triggers

{% raw %}

```yaml
on:
  schedule:
    - cron: '0 0 */3 * *'
  workflow_dispatch:
```

{% endraw %}

- **`schedule`**: Runs the workflow automatically every 3 days.
- **`workflow_dispatch`**: Allows you to manually trigger the workflow from the GitHub UI.

### 2. Permissions

{% raw %}

```yaml
permissions:
  contents: write
```

{% endraw %}

This grants the workflow permission to make commits to the repository, which is necessary so that the `upstream-version.txt` file can be updated and pushed to the repo if there is a new upstream container version.

### 3. Job: `check-and-build`

The entire automation is contained in a single job that runs using the latest Ubuntu GitHub Actions runner.

#### Step: Checkout Repo

{% raw %}

```yaml
jobs:
  check-and-build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v3
```

{% endraw %}

This pulls down the repository code so that subsequent steps can read/write files like `upstream-version.txt` and the `Dockerfile`.

#### Step: Get Latest Upstream Version

{% raw %}

```yaml
      - name: Get latest Tailscale stable version
        id: tailscale
        run: |
          VERSION=$(curl -s https://registry.hub.docker.com/v2/repositories/tailscale/tailscale/tags/?page_size=100 \
            | jq -r '.results[].name' \
            | grep -E '^v[0-9]+\.[0-9]+(\.[0-9]+)?$' \
            | sort -V \
            | tail -n1)
          echo "Latest stable version: $VERSION"
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"

      - name: Debug version
        run: |
            echo "Detected Tailscale version: ${{ steps.tailscale.outputs.version }}"
```

{% endraw %}

- Uses Docker Hub's API to fetch the latest stable tag from an upstream image.
- Outputs the latest version as `steps.upstream.outputs.version`.

#### Step: Check for Existing Image

{% raw %}

```yaml
      - name: Check if image with version already exists
        id: check
        run: |
          if docker manifest inspect ${{ secrets.DOCKERHUB_USERNAME }}/tailscale-sr:${{ steps.tailscale.outputs.version }} > /dev/null 2>&1; then
            echo "Image already exists. Skipping build."
            echo "build_needed=false" >> $GITHUB_OUTPUT
          else
            echo "New version. Proceeding to build."
            echo "build_needed=true" >> $GITHUB_OUTPUT
          fi
```

{% endraw %}

- Uses `docker manifest inspect` to check whether a container image for this version already exists in your Docker Hub repo.
- Sets a flag `build_needed` to `true` or `false`.

#### Conditional Build Steps

The following steps are only run if a new version is detected, resulting in `build_needed` being set to `true`.

- Set up emulation support (QEMU) for building multi-architecture containers.
- Set up Docker Buildx for advanced build features.
- Log into Docker Hub using credentials stored in GitHub Secrets.
- Build and push the container using the new version as a tag.

{% raw %}

```yaml
      - name: Set up QEMU
        if: steps.check.outputs.build_needed == 'true'
        uses: docker/setup-qemu-action@v2

      - name: Set up Docker Buildx
        if: steps.check.outputs.build_needed == 'true'
        uses: docker/setup-buildx-action@v2

      - name: Login to DockerHub
        if: steps.check.outputs.build_needed == 'true'
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push multi-arch container
        if: steps.check.outputs.build_needed == 'true'
        uses: docker/build-push-action@v3
        with:
          context: ./docker
          build-args: |
            TAILSCALE_TAG=${{ steps.tailscale.outputs.version }}
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/tailscale-sr:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/tailscale-sr:${{ steps.tailscale.outputs.version }}
```

{% endraw %}

#### Step: Commit Version Update

{% raw %}

```yaml
      - name: Commit updated version file
        if: steps.check.outputs.build_needed == 'true'
        run: |
          echo "${{ steps.tailscale.outputs.version }}" > tailscale-version.txt
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add tailscale-version.txt
          git commit -m "Update Tailscale version to ${{ steps.tailscale.outputs.version }}"
          git push
```

{% endraw %}

This writes the new version to a file, commits it, and pushes it to your GitHub repo. This gives you a version history and a way to confirm that the build occurred.

## The YAML Workflow

{% raw %}

```yaml
name: Auto Build & Publish Tailscale Container

on:
  schedule:
    - cron: '0 0 */3 * *'  # Every 3 days at 00:00 UTC
  workflow_dispatch:

permissions:
  contents: write  # Needed to commit version file

jobs:
  check-and-build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v3

      - name: Get latest Tailscale stable version
        id: tailscale
        run: |
          VERSION=$(curl -s https://registry.hub.docker.com/v2/repositories/tailscale/tailscale/tags/?page_size=100 \
            | jq -r '.results[].name' \
            | grep -E '^v[0-9]+\.[0-9]+(\.[0-9]+)?$' \
            | sort -V \
            | tail -n1)
          echo "Latest stable version: $VERSION"
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"

      - name: Debug version
        run: |
            echo "Detected Tailscale version: ${{ steps.tailscale.outputs.version }}"

      - name: Check if image with version already exists
        id: check
        run: |
          if docker manifest inspect ${{ secrets.DOCKERHUB_USERNAME }}/tailscale-sr:${{ steps.tailscale.outputs.version }} > /dev/null 2>&1; then
            echo "Image already exists. Skipping build."
            echo "build_needed=false" >> $GITHUB_OUTPUT
          else
            echo "New version. Proceeding to build."
            echo "build_needed=true" >> $GITHUB_OUTPUT
          fi

      - name: Set up QEMU
        if: steps.check.outputs.build_needed == 'true'
        uses: docker/setup-qemu-action@v2

      - name: Set up Docker Buildx
        if: steps.check.outputs.build_needed == 'true'
        uses: docker/setup-buildx-action@v2

      - name: Login to DockerHub
        if: steps.check.outputs.build_needed == 'true'
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push multi-arch container
        if: steps.check.outputs.build_needed == 'true'
        uses: docker/build-push-action@v3
        with:
          context: ./docker
          build-args: |
            TAILSCALE_TAG=${{ steps.tailscale.outputs.version }}
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/tailscale-sr:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/tailscale-sr:${{ steps.tailscale.outputs.version }}

      - name: Commit updated version file
        if: steps.check.outputs.build_needed == 'true'
        run: |
          echo "${{ steps.tailscale.outputs.version }}" > tailscale-version.txt
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add tailscale-version.txt
          git commit -m "Update Tailscale version to ${{ steps.tailscale.outputs.version }}"
          git push
```

{% endraw %}

## Final Thoughts

Automating container builds with GitHub Actions is a powerful way to keep your images up to date without manual intervention. This workflow can be adapted for any upstream project, and you can customize it further based on your needs.

- Consider adding notifications (e.g., Slack, email) to alert you when a new version is built.
- You can also extend the workflow to run tests against the new image before pushing it.
- Explore other GitHub Actions to enhance your CI/CD pipeline.

If you’ve built similar automation's or have tips for improving this setup, let me know—I’d love to hear how others are tackling this.
