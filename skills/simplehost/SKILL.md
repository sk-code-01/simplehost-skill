---
name: simplehost
description: Publish any file or folder to the web instantly via simplehost.dev. Use when the user wants to publish, deploy, host, or share HTML/CSS/JS files, static sites, images, PDFs, or any files as a live website. Also triggers on "make this live", "give me a URL", "host this", "deploy this", "add a domain", "claim a handle", or "check my account".
argument-hint: "[directory-or-file]"
allowed-tools: Read, Glob, Grep, Bash, Write
---

# SimpleHost — Instant Web Publishing

Publish any file or folder to a live URL at `<slug>.simplehost.dev`. Everything works from the terminal.

All endpoints use: `https://simplehost.dev` as the base URL.
All authenticated endpoints need: `Authorization: Bearer $SIMPLEHOST_API_KEY`

---

## Publish a Site

### Step 1: Ask the user

> **Permanent or temporary link?**
> - **Temporary** — No signup. Live instantly. Expires in 24 hours. Can't update or edit.
> - **Permanent** — Free account (30-second one-time setup). Stays forever. Editable. Custom domains.

### Step 2: Find what to publish

- Use `$ARGUMENTS` if provided
- Otherwise look for: `dist/`, `build/`, `out/`, `public/`, `.next/out/`
- If nothing found, ask the user

### Step 3: Publish

```bash
curl -fsSL https://simplehost.dev/publish.sh -o /tmp/simplehost-publish.sh && chmod +x /tmp/simplehost-publish.sh

# Temporary (no key needed)
/tmp/simplehost-publish.sh <directory>

# Permanent (needs API key)
export SIMPLEHOST_API_KEY="sh_live_..."
/tmp/simplehost-publish.sh <directory>

# Update existing site
/tmp/simplehost-publish.sh <directory> <existing-slug>
```

After publishing, show the URL: `https://<slug>.simplehost.dev/`

For temporary: warn it expires in 24 hours and offer to set up a free account.

---

## Sign Up / Login (only when needed)

Only do this when the user wants permanent publishing and `SIMPLEHOST_API_KEY` is not set.

```bash
# 1. Send verification code
curl -s -X POST https://simplehost.dev/api/auth/agent/request-code \
  -H "Content-Type: application/json" \
  -d '{"email": "USER_EMAIL"}'

# 2. User checks email, gives you the XXXX-XXXX code

# 3. Verify and get API key
curl -s -X POST https://simplehost.dev/api/auth/agent/verify-code \
  -H "Content-Type: application/json" \
  -d '{"email": "USER_EMAIL", "code": "XXXX-XXXX"}'
```

Response gives `apiKey`. Tell the user to save it in their shell profile:
```bash
export SIMPLEHOST_API_KEY="sh_live_..."
```

This only happens once — the key works forever.

---

## When the User Asks About Their Account

```bash
curl -s https://simplehost.dev/api/v1/account \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"
```

Returns: plan, usage (sites, storage, domains), limits, email, handle.
Show it in a clean readable format.

---

## When the User Wants a Custom Domain

Only do this when the user specifically asks to connect a domain.

```bash
# Add a domain
curl -s -X POST https://simplehost.dev/api/v1/domains \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "example.com"}'
```

The response gives DNS instructions (CNAME + TXT records). Tell the user exactly what to add at their DNS provider.

After user adds DNS records:
```bash
# Verify the domain
curl -s -X POST https://simplehost.dev/api/v1/domains/example.com/verify \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"
```

Other domain commands:
```bash
# List all domains
curl -s https://simplehost.dev/api/v1/domains \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"

# Remove a domain
curl -s -X DELETE https://simplehost.dev/api/v1/domains/example.com \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"
```

---

## When the User Wants a Custom Handle

A handle gives a vanity URL: `yourname.simplehost.dev`. Requires Hobby plan.

```bash
# Claim a handle
curl -s -X PUT https://simplehost.dev/api/v1/handle \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"handle": "yourname"}'

# Check current handle
curl -s https://simplehost.dev/api/v1/handle \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"

# Release handle
curl -s -X DELETE https://simplehost.dev/api/v1/handle \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"
```

---

## When the User Wants to Manage Sites

```bash
# List all sites
curl -s https://simplehost.dev/api/v1/publishes \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"

# Get site details
curl -s https://simplehost.dev/api/v1/publish/<slug> \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"

# Delete a site
curl -s -X DELETE https://simplehost.dev/api/v1/publish/<slug> \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"

# Duplicate a site
curl -s -X POST https://simplehost.dev/api/v1/publish/<slug>/duplicate \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"

# Update title/description
curl -s -X PATCH https://simplehost.dev/api/v1/publish/<slug>/metadata \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "My Site", "description": "A cool site"}'
```

---

## When the User Hits a Limit

API errors include an `upgrade` object when upgrading would fix the issue. When you see it:

1. Show the user `upgrade.message` — it explains the problem clearly
2. If `upgrade.action` is `"checkout"`:
   ```bash
   curl -s -X POST https://simplehost.dev/api/billing/checkout \
     -H "Authorization: Bearer $SIMPLEHOST_API_KEY"
   ```
   Tell the user: "Open this link to upgrade: <checkoutUrl>"
   They pay in the browser, plan upgrades automatically. This is the only step that needs a browser.

---

## Quick Facts

- Static files only (HTML, CSS, JS, images, fonts, PDFs, videos)
- No server-side code
- Max 1,000 files per site
- **Free**: 500 sites, 10 GB storage, 250 MB file size, 1 domain
- **Hobby ($5/mo)**: Unlimited sites, 100 GB storage, 5 GB file size, 5 domains, custom handle
