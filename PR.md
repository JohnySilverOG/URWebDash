# PR: Webhook management section in the dashboard UI

## Summary

Adds a **🔔 Webhook** section to the web dashboard so webhook alerts can be
managed from the browser — paste an `https://` Discord webhook URL, save it,
send a test message, or clear it — without touching the filesystem, env vars,
or restarting the service.

Resolves the "manage webhook from the dashboard" ask; completes the webhook
alerting feature that shipped in the installer (v0.0.7) by giving it a UI.

---

## Features

### Webhook section (UI, `index.html`)

- New section in the dashboard nav with status header, usage note, and a
  dedicated `.webhook-panel` card.
- **Status line** shows whether a webhook is configured, with the secret token
  **masked** (only the base URL prefix is shown — never the full token).
- **URL input box** (`type="url"`, spellcheck off) to paste a Discord webhook
  `https://` URL. Input validation enforces the Discord webhook host prefixes:
  `discord.com`, `discordapp.com`, `ptb.discord.com`, `canary.discord.com`.
- **Action buttons**:
  - 💾 **Save** — stores the URL server-side (`POST /api/webhook`)
  - 📣 **Send Test** — posts a sample alert; tests the URL currently in the
    box before saving, or the configured one if the box is empty
  - 🗑️ **Clear** — removes the stored URL
- **On-screen instructions** for getting the URL (Discord → Server Settings →
  Integrations → Webhooks → New Webhook → Copy Webhook URL).
- Toast feedback on success/failure; buttons disable while a request is in
  flight.

### Env-var overlap warning (UI + backend)

- If `DISCORD_WEBHOOK_URL` is set in the environment, the section renders a
  warning banner: the env var takes precedence over the saved URL, and the
  saved URL will only be used once the env var is removed. The overlap is now
  visible instead of silent.

### API endpoints (`main.go`)

- `GET /api/webhook` — current config state; responses **mask** the secret
  token before it ever reaches the browser.
- `POST /api/webhook` — validates and saves the URL.
- `POST /api/webhook-test` — sends a test Discord message and returns
  Discord's actual response body or a success result; the URL in the input
  box is tested before saving.
- Saved to `~/.urnetwork/discord_webhook` (mode `0600`); Docker:
  `/data/.urnetwork/discord_webhook`, persisted in the volume. Takes effect
  immediately — no restart needed.

### Refactor (no behavior change)

- Notification posting refactored around a shared helper; `urwebdash
  testwebhook` CLI output is unchanged.
- `DISCORD_WEBHOOK_URL` env var still takes precedence when set (documented
  in the UI warning).

---

## Security

- Webhook token is masked in all API responses — the full secret never
  returns to the client.
- Saved URL file is written with `0600` permissions.
- URL validation rejects anything that isn't a Discord webhook endpoint.

## Files changed

| File | Change |
|---|---|
| `index.html` | New webhook section: status, URL input, Save/Test/Clear, env-var warning, masked-token rendering, toasts |
| `main.go` | `GET`/`POST /api/webhook`, `POST /api/webhook-test`, shared notification helper, masking, validation |
| `CHANGELOG.md` | Documented under **Unreleased** |

## How to test

1. Open the dashboard → **🔔 Webhook** section → status should show "not configured".
2. Paste an invalid URL → **Save** → expect a validation toast.
3. Paste a real Discord webhook URL → **Save** → status shows the masked URL.
4. Click **📣 Send Test** → sample message appears in the Discord channel.
5. Test with the box cleared → test posts to the previously saved URL.
6. Set `DISCORD_WEBHOOK_URL` env var → warning banner appears in the section.
7. **Clear** → status returns to "not configured"; file removed.
8. Docker: save a URL, restart the container → URL persists via the volume.

## Checklist

- [x] UI section with https URL input, save, test, clear
- [x] Masked token in API responses
- [x] Env-var precedence warning surfaced in UI
- [x] Works in Docker (persisted path)
- [x] CHANGELOG updated
