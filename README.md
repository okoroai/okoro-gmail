---
name: gmail
description: "Read, search, send, and manage Gmail messages and threads."
tags: [email, productivity, google, communication]
version: 1.0.0
repo: okoroai/okoro-gmail
---

# gmail

Read, search, send, and manage Gmail messages and threads.

All API calls go through the [okoro proxy](https://okoro.ai), which signs requests, enforces permission scopes, and writes an audit trail for every action your agent takes.

## Requirements

| Variable | Description |
|---|---|
| `OKORO_SERVICE_TOKEN` | Service token from the [okoro dashboard](https://hub.okoro.ai/docs/get-token) (`svc_...`) |

[How to get your token →](https://hub.okoro.ai/docs/get-token)

## How it works

Rather than giving your agent raw Gmail credentials, you configure a **service token** once. When the skill runs, it exchanges that token for a short-lived **operation token** scoped to exactly the action being performed. The okoro proxy signs the request, forwards it to Gmail, and records the intent in an audit log.

[Learn more about how okoro works →](https://okoro.ai/how-it-works)

## What your agent can do

- **Read** messages and threads, including full body content
- **Search** using Gmail query syntax (`from:`, `subject:`, `is:unread`, date ranges, etc.)
- **Send** emails (plain text or HTML, with attachments)
- **Draft** messages and manage drafts
- **Trash** and manage message lifecycle
- **Label** messages — apply, remove, or create labels
- **Fetch profile** — address, total messages, history ID

## License

MIT — see [LICENSE](./LICENSE).
