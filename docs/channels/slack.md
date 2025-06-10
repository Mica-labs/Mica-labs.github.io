---
layout: default
title: Slack
parent: Channels
nav_order: 2
---

# Slack Integration Guide

This guide will help you integrate the Mica AI assistant into your Slack workspace, enabling intelligent conversational interactions with Slack users.

## Prerequisites

1. A Slack workspace with administrator privileges
2. Basic understanding of Slack app configuration process

## Configuration Steps

### 1. Create a Slack App

1. Visit [Slack API](https://api.slack.com/apps)
2. Click "Create New App"
   - Choose "From Scratch"
   - Name your app
   - Select your workspace

### 2. Configure Bot User

1. Select "Bot Users" from the left menu
2. Click "Add a Bot User"
3. Set the Bot's display name and username
4. Save settings
5. Configure callback URL in "Event Subscriptions":
   - Set callback URL (Webhook URL):
     ```
     https://{your-Mica-server-domain}/v1/slack/webhook/{bot_name}
     ```
     For example: If your domain is `mica-labs.github.io` and bot name is `mica`,
     the callback URL should be: `https://mica-labs.github.io/v1/slack/webhook/mica`

6. Add the following events in "Subscribe to bot events":
- `message.channels`
- `message.groups`
- `message.im`
- `message.mpim`


### 3. Configure Incoming Webhook

1. Select "Incoming Webhooks" from the left menu
2. Enable Incoming Webhooks
3. Click "Add New Webhook to Workspace"
4. Choose the channel for sending messages
5. Copy the generated Webhook URL

### 4. Install App to Workspace

1. Select "Install App" from the left menu
2. Click "Install App to Workspace"
3. Authorize the app to access your workspace
4. Save the generated Bot User OAuth Token

### 5. Configure Mica

Add the following configuration to your Mica configuration file:

```yaml
bot_name: mica  # your bot name
llm_config:
   headers:
      Authorization: Bearer xxx  # your authorization token

slack:
   incoming_webhook: {your_incoming_webhook}  # Webhook URL obtained from Slack
```

## Test Steps
1. Send a test message in the configured target channel
2. Verify that the Mica AI assistant can correctly receive and respond to messages

## Common Issues
1. Message Sending Failure
- Check if the Webhook URL is correct
- Confirm if the Webhook is still valid
- Verify if the target channel exists and has proper permissions

2. Connection Issues
- Check network connection status
- Confirm if the server can access Slack API

## References
- [Slack API Documentation](https://api.slack.com/docs)
- [Incoming Webhooks Guide](https://api.slack.com/messaging/webhooks)