---
name: simplehost
description: Publish any file or folder to the web instantly via simplehost.dev. Use when the user wants to publish, deploy, host, or share HTML/CSS/JS files, static sites, images, PDFs, or any files as a live website. Also triggers on "make this live", "give me a URL", "host this", "deploy this", "add a domain", "claim a handle", or "check my account".
argument-hint: "[directory-or-file]"
allowed-tools: Read, Glob, Grep, Bash, Write
---

# SimpleHost — Instant Web Publishing

Publish any file or folder to a live URL at `<slug>.simplehost.dev`. Everything works from the terminal.

Base URL: `https://simplehost.dev`
Auth header (when needed): `Authorization: Bearer $SIMPLEHOST_API_KEY`

---

## Publish a Site

**Just publish. Don't ask questions first.** Get the site live, then offer options.

### Step 1: Find what to publish

- Use `$ARGUMENTS` if provided
- Otherwise look for: `dist/`, `build/`, `out/`, `public/`, `.next/out/`
- If nothing found, ask the user

### Step 2: Download the publish script

```bash
curl -fsSL https://simplehost.dev/publish.sh -o /tmp/simplehost-publish.sh && chmod +x /tmp/simplehost-publish.sh
```

### Step 3: Publish

```bash
# If SIMPLEHOST_API_KEY is set, it publishes permanently. If not, it publishes temporarily.
/tmp/simplehost-publish.sh <directory>

# To update an existing site
/tmp/simplehost-publish.sh <directory> <existing-slug>
```

### Step 4: After publishing, show the result

Show the live URL prominently: `https://<slug>.simplehost.dev/`

**If the publish was anonymous** (no API key), tell the user:
> "Your site is live! This link expires in 24 hours. Want me to make it permanent? It's free — I just need your email and it takes 30 seconds."

If they say yes, run the Sign Up flow below, then claim the site.

---

## Sign Up / Login

Only do this when the user wants to make a site permanent or explicitly asks to sign up. Never ask for this upfront.

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

Response gives `apiKey`. Tell the user to add it to their shell profile so it works forever:
```bash
export SIMPLEHOST_API_KEY="sh_live_..."
```

---

## Check Account

```bash
curl -s https://simplehost.dev/api/v1/account \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"
```

Returns: plan, usage (sites, storage, domains), limits, email, handle.

---

## Custom Domains

Only when the user asks to connect a domain.

```bash
# Add domain
curl -s -X POST https://simplehost.dev/api/v1/domains \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "example.com"}'

# Verify after DNS setup
curl -s -X POST https://simplehost.dev/api/v1/domains/example.com/verify \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"

# List domains
curl -s https://simplehost.dev/api/v1/domains \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"

# Remove domain
curl -s -X DELETE https://simplehost.dev/api/v1/domains/example.com \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"
```

---

## Custom Handle

A vanity URL like `yourname.simplehost.dev`. Requires Hobby plan ($5/mo).

```bash
# Claim
curl -s -X PUT https://simplehost.dev/api/v1/handle \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"handle": "yourname"}'

# Check current
curl -s https://simplehost.dev/api/v1/handle \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"

# Release
curl -s -X DELETE https://simplehost.dev/api/v1/handle \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"
```

---

## Manage Sites

```bash
# List all sites
curl -s https://simplehost.dev/api/v1/publishes \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"

# Get site details
curl -s https://simplehost.dev/api/v1/publish/<slug> \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"

# Delete
curl -s -X DELETE https://simplehost.dev/api/v1/publish/<slug> \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY"

# Duplicate
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

API errors include an `upgrade` object. When you see it:

1. Show the user `upgrade.message`
2. If `upgrade.action` is `"checkout"`:
   ```bash
   curl -s -X POST https://simplehost.dev/api/billing/checkout \
     -H "Authorization: Bearer $SIMPLEHOST_API_KEY"
   ```
   Tell the user: "Open this link to upgrade: <checkoutUrl>"
   They pay in the browser, plan upgrades automatically.

---

## Quick Facts

- Static files only (HTML, CSS, JS, images, fonts, PDFs, videos)
- No server-side code
- Max 1,000 files per site
- **Free**: 500 sites, 10 GB storage, 250 MB file size, 1 domain
- **Hobby ($5/mo)**: Unlimited sites, 100 GB storage, 5 GB file size, 5 domains, custom handle
