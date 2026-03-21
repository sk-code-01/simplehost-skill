---
name: supalink
description: Publish any file or folder to the web instantly via supalink.dev. Use when the user wants to publish, deploy, host, or share HTML/CSS/JS files, static sites, images, PDFs, or any files as a live website. Also triggers on "make this live", "give me a URL", "host this", or "deploy this".
argument-hint: "[directory-or-file]"
allowed-tools: Read, Glob, Grep, Bash, Write
---

# Supalink — Instant Web Publishing

Publish any file or folder to a live URL at `<slug>.supalink.dev`.

## Before You Start

1. Check if `SUPALINK_API_KEY` is set in the environment:
   ```bash
   echo "${SUPALINK_API_KEY:-NOT SET}"
   ```

2. **If NOT SET**, ask the user:
   > "You need a Supalink API key for permanent publishing. Would you like me to set one up? I'll need your email address."

   If they agree, run the auth flow:
   ```bash
   # Step 1: Request verification code
   curl -s -X POST https://supalink.dev/api/auth/agent/request-code \
     -H "Content-Type: application/json" \
     -d '{"email": "USER_EMAIL"}'
   ```
   Ask the user to check their email and paste the `XXXX-XXXX` code, then:
   ```bash
   # Step 2: Verify code and get API key
   curl -s -X POST https://supalink.dev/api/auth/agent/verify-code \
     -H "Content-Type: application/json" \
     -d '{"email": "USER_EMAIL", "code": "XXXX-XXXX"}'
   ```
   The response contains `apiKey`. Tell the user to save it, then set it:
   ```bash
   export SUPALINK_API_KEY="sl_live_..."
   ```

3. **If the user declines auth**, publish anonymously (site expires in 24 hours).

## How to Publish

### Determine what to publish

- If `$ARGUMENTS` is provided, use that as the directory or file path.
- If no argument, look for common build output directories: `dist/`, `build/`, `out/`, `public/`, `.next/out/`.
- If none exist, ask the user what to publish.
- If publishing a single file, create a temp directory, copy it there, and publish that.

### Publish using the shell script

Download `publish.sh` if it doesn't exist locally:

```bash
curl -fsSL https://supalink.dev/publish.sh -o /tmp/supalink-publish.sh && chmod +x /tmp/supalink-publish.sh
```

Then publish:

```bash
# New site
/tmp/supalink-publish.sh <directory>

# Update existing site
/tmp/supalink-publish.sh <directory> <existing-slug>
```

The script handles everything: scanning files, computing hashes, uploading, and finalizing.

### After publishing

1. Show the user the live URL prominently: `https://<slug>.supalink.dev/`
2. If anonymous, warn that the site expires in 24 hours and show how to claim it.
3. Save the slug so the user can update later.

## Updating a Site

If the user says "update", "redeploy", or "publish again" and you know the slug from a previous publish:

```bash
/tmp/supalink-publish.sh <directory> <slug>
```

Unchanged files are automatically skipped (hash dedup).

## API Reference

If you need to do something beyond publish (list sites, delete, etc.), use the API directly:

| Action | Method | Endpoint |
|--------|--------|----------|
| List sites | GET | `/api/v1/publishes` |
| Site details | GET | `/api/v1/publish/<slug>` |
| Delete site | DELETE | `/api/v1/publish/<slug>` |
| Duplicate site | POST | `/api/v1/publish/<slug>/duplicate` |
| Update metadata | PATCH | `/api/v1/publish/<slug>/metadata` |

All authenticated endpoints require: `Authorization: Bearer $SUPALINK_API_KEY`

## Handling Limits & Upgrades

API errors include an `upgrade` object when the user can fix the issue by upgrading. When you see it:

1. **Show the user the `upgrade.message`** — it explains what happened and the next step
2. **If `upgrade.action` is `"signup"`** — tell the user to create a free account at supalink.dev/login
3. **If `upgrade.action` is `"checkout"`** — get a payment link:
   ```bash
   curl -s -X POST https://supalink.dev/api/billing/checkout \
     -H "Authorization: Bearer $SUPALINK_API_KEY" \
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
