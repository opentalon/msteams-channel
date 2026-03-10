# msteams-channel

MS Teams channel for [OpenTalon](https://github.com/opentalon/opentalon) — pure YAML, no compiled binary.

Uses the **Bot Framework** model: Teams POSTs HTTP activities to your bot's webhook endpoint. The OpenTalon core starts a local HTTP server on port 3978, validates inbound Bot Framework JWTs, and sends replies via the Bot Framework REST API using a short-lived OAuth token fetched at startup.

## Prerequisites

- An **Azure Bot** (Bot Channels Registration) with the Microsoft Teams channel enabled
- The bot's **App ID** and **client secret** (from the Azure portal)
- A publicly reachable HTTPS endpoint on port 3978 (or ngrok for local dev)

## Setup

### 1. Create an Azure Bot

1. In the [Azure portal](https://portal.azure.com), create a **Bot Channels Registration** (or an Azure Bot resource)
2. Under **Channels**, enable **Microsoft Teams**
3. Under **Configuration**, note the **Microsoft App ID** — this is `MSTEAMS_APP_ID`
4. Under **Manage** → **Certificates & secrets**, create a new client secret — this is `MSTEAMS_APP_PASSWORD`

### 2. Set env vars

```bash
export MSTEAMS_APP_ID="your-azure-bot-app-id"
export MSTEAMS_APP_PASSWORD="your-azure-bot-client-secret"
```

### 3. Add to OpenTalon config

```yaml
channels:
  msteams:
    enabled: true
    github: "opentalon/msteams-channel"
    ref: "master"
```

### 4. Expose port 3978

The core listens on `:3978/api/messages`. Set this as the **Messaging endpoint** in the Azure Bot configuration:

```
https://<your-host>:3978/api/messages
```

**Local dev with ngrok:**

```bash
ngrok http 3978
# Copy the https:// forwarding URL and set it as the messaging endpoint in Azure
```

## How it works

```
Teams user sends message
        │
        ▼
Azure Bot Framework
        │  POST /api/messages
        ▼
OpenTalon core (:3978)
  1. Validate Bot Framework JWT
  2. Strip @mention prefix
  3. Dedup by activity ID
  4. Forward to LLM orchestrator
        │
        ▼
  LLM generates reply
        │
        ▼
  POST to serviceUrl/v3/conversations/{id}/activities
  (Bearer token from Bot Framework OAuth)
        │
        ▼
Teams user sees reply
```

- **Inbound:** HTTP POST from Teams; JWT validated against Microsoft's OIDC endpoint (`login.botframework.com`). Bot's own messages are skipped (`from.role: bot`). `<at>BotName</at>` mention tags are stripped automatically.
- **Outbound:** Bot Framework REST API using a client credentials OAuth token fetched at startup. Replies are threaded using `replyToId`.
- **No binary, no gRPC** — the core interprets `channel.yaml` directly.

## Capabilities

| Feature | Supported |
|---|---|
| Threads | Yes |
| Files | No |
| Reactions | No |
| Edits | No |
| Max message length | 28,000 chars |

Long responses are automatically chunked into multiple activities.

## Troubleshooting

**Bot doesn't respond in Teams**
- Check that the messaging endpoint in Azure matches your host/ngrok URL exactly (must be HTTPS)
- Ensure `MSTEAMS_APP_ID` and `MSTEAMS_APP_PASSWORD` are set correctly
- Check OpenTalon logs for JWT validation errors or OAuth token fetch failures

**`JWT validation failed`**
- The Bot Framework JWKS are cached for 24 hours; a key rotation mid-session will cause a one-time refetch on the next message
- Ensure the system clock is accurate (JWT expiry is time-sensitive)

**`init fetch_token: HTTP 401`**
- Double-check `MSTEAMS_APP_PASSWORD` — client secrets expire in Azure and must be rotated
