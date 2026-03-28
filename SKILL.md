---
name: gmail
description: "Read, search, send, and manage Gmail messages and threads."
version: 1.0.0

# Claude Code fields
argument-hint: "--endpoint /<resource> --intent \"user's goal, not the API call\""
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
  [--method  GET|POST|DELETE] \
  [--scope   read|write|update|delete|all] \
  [--payload <json>]
```

- **endpoint** — Gmail API path, including query parameters (e.g. `"/v1/users/me/messages?maxResults=10&q=is:unread"`)
- **intent** — the session intent: the user's overall goal for this conversation (not a description of the API call)
- **method** — defaults to `GET`; set `POST`/`DELETE` for mutations
- **scope** — inferred from method if omitted (`GET`→read, `POST`→write, `DELETE`→delete). **Always pass `--scope` explicitly for POST endpoints that require `update`, `delete`, or `all`** — auto-inference only gives `write`.
- **payload** — JSON body for POST requests only. **Never use `--payload` with GET or HEAD** — pass filters and options as query parameters in `--endpoint` instead.

## Key endpoints

| Action | Method | Endpoint | Min scope |
|--------|--------|----------|-----------|
| List messages (IDs only) | GET | `/v1/users/me/messages?maxResults=10` | `read` |
| Search messages (IDs only) | GET | `/v1/users/me/messages?q=<query>&maxResults=10` | `read` |
| Get message with metadata | GET | `/v1/users/me/messages/<id>?format=metadata&metadataHeaders=Subject&metadataHeaders=From&metadataHeaders=Date` | `read` |
| Get message full body | GET | `/v1/users/me/messages/<id>?format=full` | `read` |
| Get thread | GET | `/v1/users/me/threads/<id>` | `read` |
| List threads | GET | `/v1/users/me/threads` | `read` |
| List labels | GET | `/v1/users/me/labels` | `read` |
| Get profile | GET | `/v1/users/me/profile` | `read` |
| Send message | POST | `/v1/users/me/messages/send` | `write` |
| Create draft | POST | `/v1/users/me/drafts` | `write` |
| Modify labels | POST | `/v1/users/me/messages/<id>/modify` | `update` ⚠ pass `--scope update` |
| Trash message | POST | `/v1/users/me/messages/<id>/trash` | `delete` ⚠ pass `--scope delete` |
| Delete message (permanent) | DELETE | `/v1/users/me/messages/<id>` | `delete` |

> ⚠ POST auto-infers `write` scope. For `modify` and `trash` you **must** pass `--scope` explicitly or the proxy will reject the request with 403.

## Default behaviour

**The list endpoint returns IDs only.** Always follow up with a metadata fetch for each ID to get Subject, From, and Date. Unless the user explicitly asks for the full body, use `?format=metadata` — it's much faster than `?format=full`.

```bash
# Step 1 — list (returns IDs)
skills/gmail/scripts/gmail.sh \
  --endpoint "/v1/users/me/messages?maxResults=10&q=is:inbox" \
  --intent "summarise my inbox"

# Step 2 — fetch metadata for each ID (same intent — same user request)
skills/gmail/scripts/gmail.sh \
  --endpoint "/v1/users/me/messages/<id>?format=metadata&metadataHeaders=Subject&metadataHeaders=From&metadataHeaders=Date" \
  --intent "summarise my inbox"
```

## Typical workflows

**Search and read emails:**
```bash
# Search by query
skills/gmail/scripts/gmail.sh --endpoint "/v1/users/me/messages?q=is:unread" --intent "show me my unread emails"
# Full body (only when user needs the content)
skills/gmail/scripts/gmail.sh --endpoint "/v1/users/me/messages/<id>?format=full" --intent "show me my unread emails"
```

**Send an email:**
```bash
# Build an RFC 2822 message, base64url-encode it, then send.
# The raw string must include To, From, Subject headers followed by a blank line and the body.
# Example (bash): raw=$(printf 'To: user@example.com\r\nFrom: me@example.com\r\nSubject: Hello\r\n\r\nBody text' | base64 | tr '+/' '-_' | tr -d '=\n')
skills/gmail/scripts/gmail.sh --method POST --endpoint /v1/users/me/messages/send \
  --intent "reply to Alice's email about the project deadline" \
  --payload "{\"raw\":\"$raw\"}"
```

**Create a draft:**
```bash
# Same base64url encoding as send; body is wrapped in {"message": {...}}
skills/gmail/scripts/gmail.sh --method POST --endpoint /v1/users/me/drafts \
  --intent "draft a response to the board update thread" \
  --payload "{\"message\":{\"raw\":\"$raw\"}}"
```

**Modify labels (mark as read, star, etc.):**
```bash
# Must pass --scope update — POST alone would only get write scope
skills/gmail/scripts/gmail.sh --method POST --scope update \
  --endpoint /v1/users/me/messages/<id>/modify \
  --intent "mark the newsletters as read" \
  --payload '{"removeLabelIds":["UNREAD"]}'
```

**Move to trash:**
```bash
# Must pass --scope delete — POST alone would only get write scope
skills/gmail/scripts/gmail.sh --method POST --scope delete \
  --endpoint /v1/users/me/messages/<id>/trash \
  --intent "clean up the promotional emails from last week"
```

## Token & scope

`OKORO_SERVICE_TOKEN` must have at least the required scope level:
`read` < `write` < `update` < `delete` < `all`

**Scope auto-inference:** `GET`→`read`, `POST`→`write`, `DELETE`→`delete`. POST-based endpoints that require `update` or `delete` scope (modify, trash) will not work with the inferred `write` scope — always pass `--scope` explicitly for those.

## Intent

`--intent` is the **session intent** — the user's overall goal for this conversation, not a description of the individual API call. It is logged by the proxy as the audit reason for every token issued in this session. Pass the same value for every call you make within a single user request.

```
--intent "summarise unread emails from this morning"   ✓  (why the user asked)
--intent "fetch email metadata"                         ✗  (describes the API call)
--intent "list messages"                                ✗  (too vague, still call-level)
```
