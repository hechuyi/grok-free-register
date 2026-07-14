# Local Ops and Cloudflare Integration

This document summarizes the local operations layer added around
`grok-free-register`. It is written as a public-safe overview: deployment-specific
paths, private email addresses, tokens, and account identifiers should be supplied
through local `.env` files, launchd plists, or Cloudflare secrets rather than
committed to the repository.

## Goals

- Run the custom email receiving path as local background services.
- Expose a read-only local operations panel for health checks and logs.
- Route Cloudflare Email Routing catch-all traffic through an Email Worker.
- Keep normal root-domain catch-all mail forwarding intact while sending the
  dedicated registration subdomain to the local webhook.
- Import authenticated OAuth credential files into a local CLIProxyAPI-compatible
  endpoint through an idempotent importer.

## Components

### Local Email Service

The local email service receives webhook payloads from a Cloudflare Email Worker
and stores short-lived verification codes. It supports configurable host, port,
and domain values through environment variables.

### Webhook Token Proxy

The webhook proxy sits in front of the local email service. It verifies a shared
`x-webhook-token` header, enforces the expected recipient domain, and forwards
valid webhook payloads to the local email service.

### Cloudflare Email Worker

The Worker receives Email Routing messages. Its behavior is configured by
Cloudflare secrets:

- `WEBHOOK_URL`
- `WEBHOOK_TOKEN`
- `GROK_EMAIL_DOMAIN`
- `FALLBACK_FORWARD_TO`

The Worker forwards messages for `GROK_EMAIL_DOMAIN` to `WEBHOOK_URL`. All other
catch-all messages are forwarded to `FALLBACK_FORWARD_TO`, which preserves normal
root-domain mail behavior while allowing the dedicated registration subdomain to
enter the automation pipeline.

### OAuth Auth Service

The auth service can run in daemon mode for unattended local operation. It reads
registered SSO sessions, performs the OAuth authorization flow, and writes
authenticated credential files to a private local inventory directory.

### CPA Importer

The importer scans the authenticated credential inventory and uploads new files
to a local CLIProxyAPI-compatible management endpoint. Uploads are tracked in a
SQLite ledger so retries are idempotent.

### Read-Only Ops Panel

The local ops panel is a small HTTP server that reports:

- Python and dependency status
- Email service and webhook proxy health
- launchd service states
- credential inventory counts
- log file tails
- copyable maintenance commands

The panel intentionally does not expose a button that starts account registration.

## Deployment Notes

Use private local files for environment values and credentials. Keep these out of
Git:

- `.env`
- runtime logs
- credential inventories
- Cloudflare tunnel tokens
- webhook tokens
- CPA management secrets
- generated account/session files

For macOS background operation, launchd plists can invoke small wrapper scripts
that load the project runtime and execute the relevant Python module.

## Validation

Recommended local checks:

```bash
python -m pytest tests -q
node --check cloudflare/email-worker.js
curl http://127.0.0.1:<email-service-port>/health
curl http://127.0.0.1:<webhook-proxy-port>/health
```

Cloudflare-side checks:

- Worker deployment succeeds.
- Required Worker secrets exist.
- Email Routing catch-all points to the Worker.
- Worker logs show webhook delivery for the dedicated subdomain and fallback
  forwarding for other catch-all mail.
