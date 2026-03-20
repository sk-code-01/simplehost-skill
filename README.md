# Supalink Skill for Claude Code

Publish any file or folder to a live URL instantly. Built for AI agents.

## Install

```bash
npx skills add supalink/skill --skill supalink -g
```

For project-local install (only available in this repo):

```bash
npx skills add supalink/skill --skill supalink
```

## What it does

Once installed, Claude Code can:

- **Publish** any directory to `<slug>.supalink.dev` with a single command
- **Update** existing sites with hash-based deduplication (only uploads changed files)
- **Authenticate** via email verification to get permanent URLs
- **Manage** sites — list, delete, duplicate, update metadata

## Usage

Just ask Claude:

- "Publish this project"
- "Deploy my dist folder"
- "Give me a live URL for this site"
- "Update my site"
- "Host this HTML file"

Or use the slash command:

```
/supalink ./dist
```

## How it works

1. Scans the directory for files
2. Computes SHA-256 hashes for deduplication
3. Creates/updates the site via the Supalink API
4. Uploads only new or changed files
5. Returns a live URL at `<slug>.supalink.dev`

## Authentication

- **Without API key**: Sites expire in 24 hours (anonymous publish)
- **With API key**: Sites are permanent

The skill guides you through email-based authentication to get an API key on first use.

## Requirements

- `curl`
- `jq`

Both are pre-installed on macOS and most Linux distributions.

## Links

- [supalink.dev](https://supalink.dev) — Dashboard
- [API Documentation](https://supalink.dev) — Full API reference
