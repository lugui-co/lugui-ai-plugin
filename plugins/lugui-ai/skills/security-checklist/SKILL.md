---
name: security-checklist
description: Pre-publication security gate for pages.lugui.ai — what must NEVER appear in a public page (secrets, client PII, internal endpoints) and the auth requirement (@lugui.ai only). Load this BEFORE running /lugui-ai:pages:publish.
---

# Security checklist (run BEFORE publishing)

A published page lives on the **public internet** at `pages.lugui.ai/<slug>`,
served by CloudFront with no auth in front of it. Treat every byte as world-
readable forever. Review the HTML against this list before calling
`/lugui-ai:pages:publish`. If anything matches, STOP and fix it first.

## MUST NOT appear in the HTML

### Secrets / credentials (the publish gate rejects these → HTTP 422)

- **AWS access key id** — pattern `AKIA` + 16 chars.
- **PEM private keys** — `-----BEGIN ... PRIVATE KEY-----` blocks.
- **Google API keys** — `AIza` + 35 chars.
- **Obvious assignments** — `secret`, `token`, `password`, `api_key`/`api-key`
  set to a quoted value of 8+ chars (`password = "hunter2longer"`).
- Also (gate won't catch all of these — you must): OAuth client secrets,
  bearer/session tokens, DB connection strings with passwords, signed S3 URLs,
  webhook secrets, JWTs, `.env` contents, basic-auth `user:pass@host`.

### Client / personal data (PII) — no automated gate, your responsibility

- Customer names, emails, phones, **CPF/CNPJ**, addresses.
- Contract values, proposal details, payment/bank data.
- Anything that could identify a specific Lugui customer.

### Internal-only information

- Internal hostnames / IPs (e.g. `*.internal`, `*.coolify.lugui.ai` admin URLs,
  RDS endpoints, SQS/SNS ARNs, DynamoDB table names).
- Internal API endpoints, infra topology, repo paths, ticket internals.
- Comments left in the HTML/JS that reveal any of the above.

## Auth & access reality

- Publishing requires a valid **personal access token** (`lgp_...`) obtained
  via self-service web login (`/lugui-ai:setup` → `{pages_api}/login`, sign in
  with @lugui.ai) and saved to `~/.lugui/config.json`; the backend returns
  `401`/`403` for a missing/invalid token or a token without permission. The
  token is a secret — never paste it or the contents of `~/.lugui/config.json`
  into chat.
- The published page itself is **unauthenticated and public** — there is no
  login wall on `pages.lugui.ai/<slug>`. Do not use it for anything that should
  be access-controlled.
- **Permanent vs ephemeral:** ephemeral pages still go public; they only expire
  later (server-side TTL). "Ephemeral" is not "private."

## Two-layer model (defense in depth)

1. **You + this checklist** (catches PII, internal data, and secrets the regex
   misses). This is the important layer.
2. **The publish gate** (the quick secret sanity check in `/lugui-ai:pages:publish`
   plus the backend) re-scans for the secret patterns above. It is a backstop,
   not a substitute for review.

## Go / no-go

- [ ] No secrets/credentials of any kind.
- [ ] No customer or personal data (PII).
- [ ] No internal hostnames, endpoints, or infra identifiers.
- [ ] Content is genuinely OK to be public and indexable.
- [ ] If unsure → do NOT publish; ask the owner.
