---
name: simplehost
version: 11
description: Publish any file or folder to the web instantly via simplehost.dev. Use when the user wants to publish, deploy, host, or share HTML/CSS/JS files, static sites, images, PDFs, or any files as a live website. Also triggers on "make this live", "give me a URL", "host this", "deploy this", "claim a handle", "check my account", "password protect", "set up a link", or "manage subscription".
argument-hint: "[directory-or-file]"
allowed-tools: Read, Glob, Grep, Bash, Write
---

<!-- Keeping the skill up to date: this skill is installed from a file. To update it, the
     user can re-run their installer (`npx skills add sk-code-01/simplehost-skill`, or
     download and run https://simplehost.dev/install.sh). Do not fetch or execute remote
     scripts on your own — only run an install/update step the user has explicitly asked for. -->

## CRITICAL RULE — API key handling (read first, applies always)

Before any other action that needs authentication, run:

```bash
echo "${SIMPLEHOST_API_KEY:-NOT SET}"
```

**Decision tree (check in this order):**

1. **The user pasted a key in chat** (anything matching the regex `sh_live_[a-f0-9]{32}` in their message) → this is the highest-priority signal. The user is explicitly directing you to publish under that account (common pattern: a team lead shares their key with teammates so all team sites land in one account). Treat the pasted key as the ONLY key for the rest of this conversation. See "Pasted-key flow" below.

2. **`SIMPLEHOST_API_KEY` env var starts with `sh_live_`** → use it directly. Stop here. Do NOT ask for email, do NOT request a code, do NOT call `/api/auth/agent/*`. The env var is the source of truth — ignore any dashboard text that says "lost your key".

3. **Env var is `NOT SET`** AND the user is doing a permanent publish → run the **email-auth recovery flow** in Step 3b. Works for both new users (signup) and returning users (key recovery). The same key is returned every time, so this never breaks other machines.

4. **The user explicitly says "regenerate my API key"** OR a publish call returns 401 with an existing key → run the regenerate flow at `/api/auth/agent/regenerate-key`. Warn the user this invalidates the key everywhere.

**Never silently regenerate.** Email-auth gets the user's existing key back; only regenerate when the user asks or the existing key is genuinely broken.

### Pasted-key flow (Branch 1 above)

When the user pastes an `sh_live_…` key in chat:

```bash
# 1. Validate the key + identify the account it belongs to
PASTED_KEY="sh_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"   # what they pasted
ACCOUNT=$(curl -s https://simplehost.dev/api/v1/account \
  -H "Authorization: Bearer $PASTED_KEY")
ACCOUNT_EMAIL=$(echo "$ACCOUNT" | python3 -c "import json,sys; print(json.load(sys.stdin).get('email',''))" 2>/dev/null)
ACCOUNT_PLAN=$(echo "$ACCOUNT" | python3 -c "import json,sys; print(json.load(sys.stdin).get('plan',''))" 2>/dev/null)
```

- If `ACCOUNT_EMAIL` is empty, the key is invalid/revoked — tell the user *"That key didn't work — double-check it or paste a different one."* and stop.
- Otherwise, acknowledge once (no confirmation question — the user pasted it, they know what they're doing) and proceed:

> "Got it — publishing to **$ACCOUNT_EMAIL**'s SimpleHost account ($ACCOUNT_PLAN plan) for the rest of this conversation."

```bash
# 2. Use it for THIS process only — DO NOT write to ~/.zshrc, ~/.bashrc, or any file
export SIMPLEHOST_API_KEY="$PASTED_KEY"
```

**Critical rules for pasted keys:**
- **Never write the pasted key to `~/.zshrc` / `~/.bashrc` / `~/.config/fish/config.fish` / any `.env` file.** It belongs to someone else (typically a team lead). Persisting it would silently hijack the user's own future publishes and is a security/UX disaster.
- **Use it for the entire current conversation**, including any later anonymous claims, updates, or new publishes — the user's intent is "everything in this chat goes to that account."
- If the user later in the same conversation pastes a different `sh_live_…` key, switch to the new one (re-validate, re-announce the new account).
- If they say *"use my own account"* / *"go back to mine"* / similar, drop back to the original `$SIMPLEHOST_API_KEY` (or fall through to the email-auth flow if there isn't one).

# SimpleHost — Instant Web Publishing

Publish any file or folder to a live URL at `<slug>.simplehost.dev`.

## Modes

Use this skill in three modes:

1. **Execution mode** — when the user wants to publish, update, delete, or manage something in SimpleHost
2. **Support mode** — when the user is asking a question about how SimpleHost works, what a feature means, what happened, or what to click
3. **Troubleshooting mode** — when the user has an error, failed publish, broken link, auth problem, or dashboard confusion

If the user is asking a question instead of asking you to perform an action, do not jump straight into commands. Answer the question first in plain language.

## Docs-Aware Q&A Rule

When the user asks anything about SimpleHost, use the docs as your product knowledge source.

Check docs in this order:

1. `https://docs.simplehost.dev/llms.txt` for the fastest high-level routing
2. `https://docs.simplehost.dev/llms-full.txt` when you need fuller product context
3. the specific docs page that best matches the question when you need precision

Use the live API only for account-specific or site-specific facts, such as:

- what sites the user owns
- the user's plan, handle, or usage
- whether a specific site exists
- details for a specific slug

Use docs for product knowledge, not memory. If docs and memory differ, trust the docs.

## How To Answer Product Questions

When answering a SimpleHost question:

1. Start with the direct answer in plain English
2. Explain only the next layer of detail the user likely needs
3. Tell them the exact next step if there is one
4. Include a relevant docs page only when it helps

Keep the tone warm, clear, and non-technical by default.

Good examples of questions to answer from docs:

- "How does anonymous publishing work?"
- "What is a handle?"
- "Why was my publish blocked?"
- "How do I manage secrets?"
- "What are the plan limits?"
- "How do I make a site permanent?"
- "Where do I find my API key?"

If the user seems non-technical:

- avoid internal terms unless needed
- explain what happened before explaining why
- say what you are checking or what they should click next
- do not overwhelm them with endpoint names unless they asked for technical details

## Troubleshooting Rule

When the user reports a problem:

1. Say clearly what you think is happening
2. Check the docs or live API depending on whether the issue is product behavior or account/site state
3. Give the most likely fix first
4. If there are multiple possible causes, present the most likely ones in simple order

If the user gives you an error code like `SECRET_DETECTED`, explain:

- what it means in simple terms
- why SimpleHost blocked the publish
- what you can do next

If the user is confused about the dashboard, explain the click path in plain UI language, for example:

> "Open the Sites page, find your site, click the three dots on the right, then choose Manage Secrets."

## Security Rule

If a publish is blocked with `code: "SECRET_DETECTED"`, do **not** retry blindly and do **not** publish the site as-is.

Instead:

1. Explain the issue in plain English. Say you found a private key or secret in the website files, and that publishing it as-is could expose it publicly.
2. Reassure the user that you can secure it for publishing without changing their original local project files.
3. Ask explicit permission before changing anything.
4. If the user says yes:
   - first confirm that this deployment explicitly supports secure public site proxying
   - only if that secure proxy path is available, store the secret as a site variable, add the destination API hostname to the site's proxy allowlist, and rewrite only the published copy or generated output to call `POST /api/v1/proxy/<slug>`
   - if secure public proxying is unavailable, explain that SimpleHost refused the publish to avoid exposing the user's secret and do not publish
5. If the user says no:
    - refuse to publish it

Never expose or echo secret values back to the user after storing them.
Keep the tone calm, friendly, and non-technical unless the user asks for deeper details.

Recommended message:

> "I found a private key in your website files. If I publish it like this, other people could see it and misuse it. I can secure it for you before publishing by storing it safely in SimpleHost and connecting your site to it in the safe way. I will not change your original project files. Would you like me to do that?"

If the user says yes, say:

> "I’m securing that for you now and publishing a safe version of your site. Your original files will stay untouched."

If the user says no, say:

> "I can’t safely publish this version because it contains private keys that would be publicly visible. If you want, I can secure it first and then publish it for you."

## Step 1: Ask the User What They Want

Before doing anything, ask:

> **Do you want a permanent link or a temporary link?**
>
> - **Temporary** — No signup needed. Your site goes live instantly, but it expires in 24 hours and you won't be able to edit or update it.
> - **Permanent** — Free account needed (takes 30 seconds, one-time setup). Your link stays forever and you can update it anytime.

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

### If `SIMPLEHOST_API_KEY` is set (per the CRITICAL RULE at the top):
Skip directly to "Publish" below — never run the email/code flow.

### If `SIMPLEHOST_API_KEY` is NOT SET — Email-auth recovery flow:
This same flow handles both first-time signup and returning users on a new machine. The server returns the **same** API key the user had before (no rotation). Tell the user one short message before starting:

> "I don't see your SimpleHost API key in this terminal. I'll send a code to your email so I can fetch it for you — what's your email?"

Then:

```bash
# 1. Request the verification code
curl -s -X POST https://simplehost.dev/api/auth/agent/request-code \
  -H "Content-Type: application/json" \
  -d '{"email": "USER_EMAIL"}'
```

Tell the user: *"Check your inbox for a code like `XXXX-XXXX` and paste it back to me."*

```bash
# 2. Verify the code and receive the API key
RESPONSE=$(curl -s -X POST https://simplehost.dev/api/auth/agent/verify-code \
  -H "Content-Type: application/json" \
  -d '{"email": "USER_EMAIL", "code": "XXXX-XXXX"}')

# Extract the apiKey from the JSON response (use jq, python, or similar parser
# already available — do NOT echo the key to the terminal)
API_KEY=$(echo "$RESPONSE" | python3 -c "import json,sys; print(json.load(sys.stdin).get('apiKey',''))")
```

If `apiKey` is empty, the response also tells you why (wrong code, expired, etc.) — show the user the message field and ask them to retry.

If the response includes `"keyRotated": true`, that means this is a legacy account being upgraded for the first time. Warn the user:
> "Heads up — your account predates our recoverable-key system, so I had to generate a fresh key. Any other terminals or scripts using your old key will need to be updated. From now on, the same key will be returned every time."

#### 3. Save the key — the agent does this automatically:

```bash
# Detect the user's shell config file
SHELL_RC=""
case "$(basename "$SHELL")" in
  zsh)  SHELL_RC="$HOME/.zshrc" ;;
  bash) [ -f "$HOME/.bashrc" ] && SHELL_RC="$HOME/.bashrc" || SHELL_RC="$HOME/.bash_profile" ;;
  fish) SHELL_RC="$HOME/.config/fish/config.fish" ;;
  *)    SHELL_RC="$HOME/.profile" ;;
esac

# Make sure the file exists, then append ONLY if the line isn't already there
touch "$SHELL_RC"
if ! grep -q "SIMPLEHOST_API_KEY" "$SHELL_RC"; then
  {
    echo ""
    echo "# Added by SimpleHost on $(date +%Y-%m-%d)"
    echo "export SIMPLEHOST_API_KEY=\"$API_KEY\""
  } >> "$SHELL_RC"
fi

# Use it for the current process too
export SIMPLEHOST_API_KEY="$API_KEY"
```

Then tell the user:
> "Done — I saved your API key to `$SHELL_RC`. Future sessions will pick it up automatically. Open a new terminal (or run `source $SHELL_RC`) once we're finished if you want it loaded everywhere."

**Never echo the actual key value** to the terminal in plain text after writing it — the user can find it in `$SHELL_RC` if they need it.

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
| List site variables | GET | `/api/v1/publish/<slug>/variables` |
| Store site variables | PUT | `/api/v1/publish/<slug>/variables` |
| Delete site variable | DELETE | `/api/v1/publish/<slug>/variables/<NAME>` |
| List proxy hosts | GET | `/api/v1/publish/<slug>/proxy-hosts` |
| Add proxy hosts | PUT | `/api/v1/publish/<slug>/proxy-hosts` |
| Delete proxy host | DELETE | `/api/v1/publish/<slug>/proxy-hosts/<hostname>` |
| Public site proxy | POST | `/api/v1/proxy/<slug>` |

All authenticated endpoints require: `Authorization: Bearer $SIMPLEHOST_API_KEY`

## Secure Variables & Proxy

Use this flow when a site needs Stripe, OpenAI, Supabase, Resend, or any other external API key.

```bash
# 1. Store one or more secret values on the site
curl -s -X PUT https://simplehost.dev/api/v1/publish/<slug>/variables \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"OPENAI_API_KEY":"sk-...","STRIPE_KEY":"sk_live_..."}'

# 2. Approve exact outbound hosts this site may proxy to
curl -s -X PUT https://simplehost.dev/api/v1/publish/<slug>/proxy-hosts \
  -H "Authorization: Bearer $SIMPLEHOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"hosts":["api.openai.com","api.stripe.com"]}'
```

Then have the published site call:

```bash
POST https://simplehost.dev/api/v1/proxy/<slug>
```

JSON body example:

```json
{
  "url": "https://api.openai.com/v1/responses",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer {{OPENAI_API_KEY}}",
    "Content-Type": "application/json"
  },
  "body": "{\"model\":\"gpt-4.1-mini\",\"input\":\"hello\"}"
}
```

Rules:
- Only exact approved hosts are allowed.
- Only use `{{VARIABLE_NAME}}` placeholders for secrets.
- Never put raw secrets into HTML, JS, or JSON files.
- If the user refuses the secure rewrite, do not publish.
- When explaining this flow to the user, avoid technical terms like "proxy", "placeholder substitution", or "rewrite fetch calls" unless they ask for more detail.
- If secure public site proxying is disabled on this deployment, do not promise this flow to the user.

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
- **Hobby plan ($4/mo)**: Unlimited sites, 100 GB storage, 5 GB max file size, custom handle.
