---
name: simplehost
description: Publish any file or folder to the web instantly via simplehost.dev. Use when the user wants to publish, deploy, host, or share HTML/CSS/JS files, static sites, images, PDFs, or any files as a live website. Also triggers on "make this live", "give me a URL", "host this", or "deploy this".
argument-hint: "[directory-or-file]"
allowed-tools: Read, Glob, Grep, Bash, Write
---

# SimpleHost — Instant Web Publishing

Publish any file or folder to a live URL at `<slug>.simplehost.dev`.

## Step 1: Ask the User What They Want

Before doing anything, ask:

> **Do you want a permanent link or a temporary link?**
>
> - **Temporary** — No signup needed. Your site goes live instantly, but it expires in 24 hours and you won't be able to edit or update it.
> - **Permanent** — Free account needed (takes 30 seconds, one-time setup). Your link stays forever, you can update it anytime, and you can add a custom domain later.

Wait for the user to choose before proceeding.

## Step 2: Determine What to Publish

- If `$ARGUMENTS` is provided, use that as the directory or file path.
- If no argument, look for common build output directories: `dist/`, `build/`, `out/`, `public/`, `.next/out/`.
- If none exist, ask the user what to publish.
- If publishing a single file, create a temp directory, copy it there, and publish that.

## Step 3a: Temporary (Anonymous) Publish

No API key needed. Just publish directly:

```bash
# Download publish script if needed
curl -fsSL https://simplehost.dev/publish.sh -o /tmp/simplehost-publish.sh && chmod +x /tmp/simplehost-publish.sh

# Publish (no API key = anonymous)
/tmp/simplehost-publish.sh <directory>
```

After publishing, tell the user:
> "Your site is live at `https://<slug>.simplehost.dev/`
> ⚠️ This link expires in 24 hours. If you want to keep it permanently, I can help you create a free account."

**Done. Stop here for anonymous publishes.**

## Step 3b: Permanent Publish

### Check for API key first:
```bash
echo "${SIMPLEHOST_API_KEY:-NOT SET}"
```

### If API key exists:
Skip ahead to "Publish" below.

### If API key is NOT set (first time only):
Ask the user for their email address, then:

```bash
# Request verification code
curl -s -X POST https://simplehost.dev/api/auth/agent/request-code \
  -H "Content-Type: application/json" \
  -d '{"email": "USER_EMAIL"}'
```

Tell the user to check their email for a code (format: `XXXX-XXXX`). Once they give it:

```bash
# Verify code and get API key
curl -s -X POST https://simplehost.dev/api/auth/agent/verify-code \
  -H "Content-Type: application/json" \
  -d '{"email": "USER_EMAIL", "code": "XXXX-XXXX"}'
```

The response contains `apiKey`. Save it:
```bash
export SIMPLEHOST_API_KEY="sh_live_..."
```

Tell the user to add `SIMPLEHOST_API_KEY` to their shell profile (`.bashrc`, `.zshrc`) so they never have to do this again.

### Publish:
```bash
# Download publish script if needed
curl -fsSL https://simplehost.dev/publish.sh -o /tmp/simplehost-publish.sh && chmod +x /tmp/simplehost-publish.sh

# New site
/tmp/simplehost-publish.sh <directory>

# Update existing site
/tmp/simplehost-publish.sh <directory> <existing-slug>
```

After publishing, show the user the live URL prominently:
> "Your site is live at `https://<slug>.simplehost.dev/`"

Save the slug so the user can update later.

## Updating a Site

If the user says "update", "redeploy", or "publish again" and you know the slug from a previous publish:

```bash
/tmp/simplehost-publish.sh <directory> <slug>
```

Unchanged files are automatically skipped (hash dedup). Only modified files are re-uploaded.

## API Reference

For actions beyond publishing (list sites, delete, etc.), use the API directly:

| Action | Method | Endpoint |
|--------|--------|----------|
| List sites | GET | `/api/v1/publishes` |
| Site details | GET | `/api/v1/publish/<slug>` |
| Delete site | DELETE | `/api/v1/publish/<slug>` |
| Duplicate site | POST | `/api/v1/publish/<slug>/duplicate` |
| Update metadata | PATCH | `/api/v1/publish/<slug>/metadata` |

All authenticated endpoints require: `Authorization: Bearer $SIMPLEHOST_API_KEY`

## Handling Limits & Upgrades

API errors include an `upgrade` object when the user can fix the issue by upgrading. When you see it:

1. **Show the user the `upgrade.message`** — it explains what happened and the next step
2. **If `upgrade.action` is `"signup"`** — tell the user to create a free account at simplehost.dev/login
3. **If `upgrade.action` is `"checkout"`** — get a payment link:
   ```bash
   curl -s -X POST https://simplehost.dev/api/billing/checkout \
     -H "Authorization: Bearer $SIMPLEHOST_API_KEY" \
     -H "Content-Type: application/json"
   ```
   This returns `{"checkoutUrl": "https://..."}`. Tell the user:
   > "Open this link to upgrade: <checkoutUrl>"

   The user opens it in their browser, pays, and their plan upgrades automatically.

## Important Notes

- **Static hosting only**: HTML, CSS, JS, images, fonts, PDFs, videos, and other static files.
- **No server-side code**: No Node.js, Python, PHP, etc.
- **Max 1,000 files** per site.
- **Free plan**: 500 sites, 10 GB storage, 250 MB max file size.
- **Hobby plan ($4/mo)**: Unlimited sites, 100 GB storage, 5 GB max file size, custom domains.
