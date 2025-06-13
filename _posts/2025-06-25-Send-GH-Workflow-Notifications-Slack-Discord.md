---
title: "Send GitHub Actions Workflow Notifications to Slack and Discord"
date: 2025-07-25 12:00:00 -500
categories: [PowerShell, Azure, Microsoft Graph]
tags: [GitHub Actions, Slack, Discord, CI/CD, Automation, Webhooks]
author: CLC
description: Learn how to enhance your GitHub Actions workflows with real-time notifications to Slack and Discord, complete with status, duration, links, and more.
toc: true
comments: true
media_subpath: /post-content/04-gh-notifyslackdiscord
---

## Why Send Workflow Notifications?

Whether you're running CI/CD pipelines, deployment scripts, or automated tasks, it's helpful to receive real-time notifications about your workflow status. Slack and Discord are great platforms to stay updated without checking GitHub constantly. In this guide, you'll learn how to integrate **Slack** and **Discord** notifications into your GitHub Actions workflows.

---

## Prerequisites

* A GitHub repository using GitHub Actions
* A Discord server or a Slack workspace
* Permissions to create incoming webhooks

---

## Step 1: Create Incoming Webhooks

### Discord

1. Create or open a Discord server.
2. Go to a text channel > *Edit Channel* > *Integrations* > *Webhooks*.
3. Click *New Webhook*, name it, choose a channel, and copy the URL.

### Slack

1. Visit [Slack API: Apps](https://api.slack.com/apps) and create a new app.
2. Enable *Incoming Webhooks*.
3. Add a new webhook to a channel and copy the generated URL.

> 📌 You cannot send messages directly to a Discord or Slack user via webhooks — only to channels.

---

## Step 2: Add Webhook URLs to GitHub Secrets

In your GitHub repo:

* Navigate to **Settings > Secrets and variables > Actions**
* Add:

  * `DISCORD_WEBHOOK_URL`
  * `SLACK_WEBHOOK_URL`

---

## Step 3: Enhanced Workflow Example

The following workflow sends detailed status updates, including:

* ✅/❌ Status
* 🕒 Build duration
* 🔗 Link to the run
* 📦 Branch name
* 🧑 Author
* 💬 Commit message

### Discord Example

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set start time
        id: start
        run: echo "start_time=$(date +%s)" >> $GITHUB_OUTPUT

      - name: Run your script
        run: echo "Simulating build..."

      - name: Calculate duration
        id: duration
        run: |
          end_time=$(date +%s)
          start_time=${{ steps.start.outputs.start_time }}
          duration=$((end_time - start_time))
          echo "duration=${duration}" >> $GITHUB_OUTPUT

      - name: Send Discord Notification
        if: always()
        run: |
          duration="${{ steps.duration.outputs.duration }}"
          minutes=$((duration / 60))
          seconds=$((duration % 60))
          duration_formatted="${minutes}m ${seconds}s"

          if [ "${{ job.status }}" == "success" ]; then
            emoji="✅"
            status="Success"
          else
            emoji="❌"
            status="Failure"
          fi

          curl -H "Content-Type: application/json" \
            -X POST \
            -d "{
              \"content\": \"${emoji} **${status}** - [${{ github.workflow }}](https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }})\\n
              **Branch:** \`${{ github.ref_name }}\`\\n
              **Author:** ${{ github.actor }}\\n
              **Duration:** ${duration_formatted}\\n
              **Message:** _${{ github.event.head_commit.message }}_ \"
            }" \
            ${{ secrets.DISCORD_WEBHOOK_URL }}
```

### Slack Example

```yaml
- name: Send Slack Notification
  if: always()
  run: |
    duration="${{ steps.duration.outputs.duration }}"
    minutes=$((duration / 60))
    seconds=$((duration % 60))
    duration_formatted="${minutes}m ${seconds}s"

    if [ "${{ job.status }}" == "success" ]; then
      color="#2eb886"
      status="✅ *Success*"
    else
      color="#e01e5a"
      status="❌ *Failure*"
    fi

    curl -X POST -H 'Content-type: application/json' \
    --data "{
      \"attachments\": [
        {
          \"color\": \"${color}\",
          \"title\": \"GitHub Actions – ${status}\",
          \"title_link\": \"https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}\",
          \"fields\": [
            {
              \"title\": \"Workflow\",
              \"value\": \"${{ github.workflow }}\",
              \"short\": true
            },
            {
              \"title\": \"Branch\",
              \"value\": \"${{ github.ref_name }}\",
              \"short\": true
            },
            {
              \"title\": \"Author\",
              \"value\": \"${{ github.actor }}\",
              \"short\": true
            },
            {
              \"title\": \"Duration\",
              \"value\": \"${duration_formatted}\",
              \"short\": true
            },
            {
              \"title\": \"Commit Message\",
              \"value\": \"${{ github.event.head_commit.message }}\"
            }
          ]
        }
      ]
    }" ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Final Thoughts

This setup is easy to implement and gives you fast feedback on your CI/CD process, especially when working across teams. You can also:

* Add emojis to differentiate build types 🧪🔧🚀
* Use conditionals to notify only on failure or success
* Extend to Microsoft Teams, Telegram, or email with similar logic

Let your bots work for you — and stop refreshing GitHub 😉

---

Need a working repo or want to add colored embeds? Let me know!
