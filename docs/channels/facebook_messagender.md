---
layout: default
title: Facebook Messenger
parent: Channels
nav_order: 1
---

# Facebook Messenger Integration Guide

This guide will help you integrate your Mica AI assistant with Facebook Messenger, enabling your AI assistant to interact with users through Facebook.

## Prerequisites

Before starting the configuration, please ensure you have:

1. Facebook Developer Account (register at [Facebook Developer Platform](https://developers.facebook.com/))
2. Facebook Business Page - This will serve as the interface for your AI assistant to interact with users

## Configuration Process

### 1. Create Facebook Application

1. Open [Facebook Developer Platform](https://developers.facebook.com/)
2. Click "Create App" and select "Messaging" as the app type
3. After creation, find the "Messenger" configuration option in the left menu

### 2. Set up Messenger

1. In the Messenger configuration page, select and link your Facebook Business Page
2. Click "Generate Token" to get the Page Access Token
3. Configure Webhook
   - Set callback URL (Webhook URL):
     ```
     https://{your-Mica-server-domain}/v1/facebook/webhook/{bot_name}
     ```
     For example: If your domain is `mica-labs.github.io` and your bot name is `mica`,
     then the callback URL should be: `https://mica-labs.github.io/v1/facebook/webhook/mica`
   
   - Set verification token: Create a secure verification token (this token must be consistent in both Facebook and Mica configurations)
   - Select subscription events: Must check `messages` and `messaging_postbacks`

### 3. Configure Mica

Add the following configuration to your Mica configuration file:

```yaml
bot_name: mica  # Your bot name
llm_config:
   headers:
      Authorization: Bearer xxx  # Your authorization token

facebook:
   verify_token: {your-verification-token}        # Must match the verification token in Webhook settings
   page_access_token: {page-access-token}    # Page Access Token obtained from Facebook Developer Platform
   secret: {your-app-secret}              # Your Facebook application secret
```

## Testing Steps
After completing the configuration, follow these steps to verify if the integration is successful:

1. Visit your configured Facebook Business Page
2. Find and click the "Send Message" button on the page
3. Send a test message in the dialog box
4. Observe if the AI assistant responds correctly to your message

## Common Issues and Solutions
1. Unable to Send Messages
   
   - Check if the Page Access Token has expired or is incorrectly entered
   - Verify if the Webhook URL is properly configured and accessible
   - Confirm that your server supports HTTPS (Facebook requirement)
2. Webhook Verification Failed
   
   - Carefully verify that the verification token matches exactly on both ends
   - Ensure your callback URL is accessible and responds in the correct format
   - Check if the Webhook subscription events are correctly selected
3. Message Response Delays
   
   - Check if the server network connection is stable
   - Review Mica's running status and logs
   - Verify Facebook server status

## References
- [Facebook Messenger Platform Documentation](https://developers.facebook.com/docs/messenger-platform/)
- [Messenger API Reference](https://developers.facebook.com/docs/messenger-platform/reference/)