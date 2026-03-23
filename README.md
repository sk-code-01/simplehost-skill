# SimpleHost Skill

Publish any file or folder to a live URL instantly. Works with every AI agent.

## Install

**One command, no prompts:**
```bash
curl -fsSL https://simplehost.dev/install.sh | bash
```

Or if you prefer npm:
```bash
npx skills add sk-code-01/simplehost-skill -g -y
```

## What it does

Once installed, your agent can:

- **Publish** any directory to `<slug>.simplehost.dev` with a single command
- **Update** existing sites with hash-based deduplication (only uploads changed files)
- **Authenticate** via email verification to get permanent URLs
- **Manage** sites — list, delete, duplicate, update metadata

## Usage

Just tell your agent:

- "Publish this project"
- "Deploy my dist folder"
- "Give me a live URL for this site"
- "Update my site"
- "Host this HTML file"

Or use the slash command:

```
/simplehost ./dist
```

## How it works

1. Scans the directory for files
2. Computes SHA-256 hashes for deduplication
3. Creates/updates the site via the SimpleHost API
4. Uploads only new or changed files
5. Returns a live URL at `<slug>.simplehost.dev`

## Authentication

- **Without API key**: Sites expire in 24 hours (anonymous publish)
- **With API key**: Sites are permanent

The skill guides you through email-based authentication to get an API key on first use.

## Requirements

- `curl`
- `jq`

Both are pre-installed on macOS and most Linux distributions.

## Links

- [simplehost.dev](https://simplehost.dev) — Dashboard
- [API Documentation](https://simplehost.dev) — Full API reference
