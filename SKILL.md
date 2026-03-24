---
name: gmail
description: "Read, search, send, and manage Gmail messages and threads."
version: 1.0.0

# Claude Code fields
argument-hint: "--endpoint /<resource> --intent \"describe action\""
allowed-tools: Bash

# OpenClaw fields
metadata:
  openclaw:
    requires:
      env:
        - OKORO_SERVICE_TOKEN
      bins:
        - curl
        - jq
    primaryEnv: OKORO_SERVICE_TOKEN
    homepage: https://okoro.ai
    os:
      - darwin
      - linux
---

You have access to the Gmail skill via the Okoro proxy. Use `scripts/gmail.sh` for all
Gmail operations. The script caches the session token and refreshes it automatically on expiry.

## Usage

```bash
skills/gmail/scripts/gmail.sh \
  --endpoint <path> \
  --intent   <reason> \
  [--method  GET|POST|PUT|DELETE] \
  [--scope   read|write|update|delete] \
  [--payload <json>]
```

- **endpoint** — Gmail API path
- **intent** — why Claude is making this call (5–10 words, reflects the user's goal)
- **method** — defaults to `GET`; set `POST`/`PATCH`/`PUT`/`DELETE` for mutations
- **scope** — inferred from method if omitted
- **payload** — JSON body for POST/PATCH/PUT requests

## Key endpoints

| Action | Method | Endpoint | Min scope |
|--------|--------|----------|-----------|
| List messages (IDs only) | GET | `/v1/users/me/messages` | `read` |
| Search messages (IDs only) | GET | `/v1/users/me/messages?q=<query>` | `read` |
| Get message with metadata | GET | `/v1/users/me/messages/<id>?format=metadata&metadataHeaders=Subject&metadataHeaders=From&metadataHeaders=Date` | `read` |
| Get message full body | GET | `/v1/users/me/messages/<id>?format=full` | `read` |
| Get thread | GET | `/v1/users/me/threads/<id>` | `read` |
| List threads | GET | `/v1/users/me/threads` | `read` |
| List labels | GET | `/v1/users/me/labels` | `read` |
| Get profile | GET | `/v1/users/me/profile` | `read` |
| Send message | POST | `/v1/users/me/messages/send` | `write` |
| Create draft | POST | `/v1/users/me/drafts` | `write` |
| Modify labels | POST | `/v1/users/me/messages/<id>/modify` | `update` |
| Trash message | POST | `/v1/users/me/messages/<id>/trash` | `delete` |

## Default behaviour

**The list endpoint returns IDs only.** Always follow up with a metadata fetch for each ID to get Subject, From, and Date. Unless the user explicitly asks for the full body, use `?format=metadata` — it's much faster than `?format=full`.

```bash
# Step 1 — list (returns IDs)
skills/gmail/scripts/gmail.sh \
  --endpoint "/v1/users/me/messages?maxResults=10&q=is:inbox" \
  --intent "list inbox emails"

# Step 2 — fetch metadata for each ID
skills/gmail/scripts/gmail.sh \
  --endpoint "/v1/users/me/messages/<id>?format=metadata&metadataHeaders=Subject&metadataHeaders=From&metadataHeaders=Date" \
  --intent "fetch email subject and sender"
```

## Typical workflows

**Search and read emails:**
```bash
# Search by query
skills/gmail/scripts/gmail.sh --endpoint "/v1/users/me/messages?q=is:unread" --intent "find unread emails"
# Full body (only when user needs the content)
skills/gmail/scripts/gmail.sh --endpoint "/v1/users/me/messages/<id>?format=full" --intent "read email body"
```

**Send an email:**
```bash
# Payload must be a base64url-encoded RFC 2822 message
skills/gmail/scripts/gmail.sh --method POST --endpoint /v1/users/me/messages/send \
  --intent "send email to user" \
  --payload '{"raw":"<base64url-encoded message>"}'
```

**Move to trash:**
```bash
skills/gmail/scripts/gmail.sh --method POST --endpoint /v1/users/me/messages/<id>/trash \
  --intent "trash spam email"
```

## Token & scope

`OKORO_SERVICE_TOKEN` must have at least the required scope level:
`read` < `write` < `update` < `delete` < `all`

## Intent

Always pass `--intent` with the user's actual reason — not a description of the API call.
