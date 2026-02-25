# OpenClaw + AWS Bedrock AgentCore Browser: Complete Setup Guide

**Date:** 2026-02-25  
**Purpose:** Enable any OpenClaw agent to browse the web via AWS Bedrock AgentCore Browser — a cloud-hosted remote browser with persistent login sessions.

---

## What is AgentCore Browser?

AgentCore Browser is an AWS Bedrock service that provides a cloud-hosted Chromium browser your AI agent can control programmatically. Key features:

- **Cloud-hosted** — no local Chrome needed, runs in AWS
- **Persistent profiles** — save cookies/logins across sessions
- **Automation-ready** — click, type, scroll, screenshot, eval JS
- **AI-optimized** — `snapshot` command returns accessibility tree with element refs

This is ideal for OpenClaw agents that need to browse websites requiring login (X/Twitter, Reddit, etc.) or interact with web UIs.

---

## Prerequisites

### 1. Install `agent-browser` CLI

```bash
npm install -g @anthropic/agent-browser
# Verify
agent-browser --version  # Should be v0.14.0+
```

### 2. AWS Credentials

Your OpenClaw host needs AWS credentials with these IAM permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock-agentcore:StartBrowserSession",
        "bedrock-agentcore:ConnectBrowserAutomationStream",
        "bedrock-agentcore:StopBrowserSession"
      ],
      "Resource": "*"
    }
  ]
}
```

If running on EC2, attach this policy to the instance role. Otherwise, configure `~/.aws/credentials`.

### 3. Verify Access

```bash
aws sts get-caller-identity
# Should return your account/role info without errors
```

---

## Install the Skill

Create the skill directory and SKILL.md:

```bash
mkdir -p ~/.openclaw/skills/agentcore-browser
```

Create `~/.openclaw/skills/agentcore-browser/SKILL.md` with the content below:

```markdown
---
name: agentcore-browser
description: "Browse the web using AWS Bedrock AgentCore Browser via agent-browser CLI. Use when you need to visit websites (Reddit, X/Twitter, etc.), read page content, search, or extract information from the web. Supports persistent login sessions."
metadata:
  openclaw:
    emoji: "🌐"
    requires:
      anyBins: ["agent-browser"]
---

# AgentCore Browser Skill

Browse the web using AWS Bedrock AgentCore Browser — a cloud-hosted remote browser with persistent login sessions.

## Prerequisites

- agent-browser CLI installed (v0.14.0+ with agentcore provider)
- AWS credentials configured (EC2 role, env vars, or ~/.aws/credentials)
- IAM permissions: bedrock-agentcore:StartBrowserSession, ConnectBrowserAutomationStream, StopBrowserSession

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| AGENTCORE_REGION | Yes | us-east-1 | AWS region |
| AGENTCORE_PROFILE_ID | Recommended | — | Profile ID for persistent cookies/login |
| AGENTCORE_BROWSER_ID | No | aws.browser.v1 | Browser identifier |
| AGENTCORE_SESSION_TIMEOUT | No | 3600 | Session timeout (seconds) |

## Browser Profiles

Each profile has its own persistent cookies/login state. Add your profiles here:

| Profile ID | Purpose | Saved Logins | Notes |
|---|---|---|---|
| (your-profile-id) | (description) | (sites) | (notes) |

**Creating a new profile:**
1. Choose a profile ID matching: [a-zA-Z][a-zA-Z0-9_]{0,47}-[a-zA-Z0-9]{10}
2. Start a session with that ID (see Quick Start)
3. Use the Live View URL to manually log in to sites
4. The profile now persists those logins
5. Add it to the table above

**When no profile is needed:** Omit AGENTCORE_PROFILE_ID for a clean session (no saved logins).

## Quick Start

# Always clean up first
agent-browser close 2>/dev/null || true
pkill -f "node.*daemon" 2>/dev/null || true
rm -f ~/.agent-browser/default.sock ~/.agent-browser/default.pid 2>/dev/null
sleep 2

# Open a page with a specific profile
export AGENTCORE_REGION=us-east-1
export AGENTCORE_PROFILE_ID=your-profile-id
agent-browser -p agentcore open https://example.com

# Or without a profile (clean session)
export AGENTCORE_REGION=us-east-1
agent-browser -p agentcore open https://example.com

## Core Commands

### Navigation
agent-browser -p agentcore open <URL>     # Open URL (first call needs -p)
agent-browser open <URL>                   # Navigate within session

### Read Page Content
agent-browser snapshot                     # Accessibility tree with refs (AI-optimized)
agent-browser eval "document.title"        # Get page title
agent-browser eval "document.body.innerText.slice(0, 2000)"  # Get text

### Interact with Page
agent-browser click @e1                    # Click element by ref
agent-browser type @e3 "search query"      # Type into input
agent-browser press Enter                  # Press key
agent-browser scroll down                  # Scroll down

### Close Session
agent-browser close                        # Always close when done!

## Safety & Guardrails

- Always close sessions when done — agent-browser close
- Never post, like, or interact with content without explicit user permission
- Never share Session IDs or Live View URLs publicly
- Read-only by default — only browse and extract information
- Session timeout is 1 hour; close and reopen if needed for longer tasks

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Failed to start session | Kill daemon: pkill -f "node.*daemon" + remove socket + retry |
| 403 Forbidden | Check AWS credentials: aws sts get-caller-identity |
| Login expired | Profile may need refresh — ask your human to re-login via Live View |
| Daemon stuck | pkill -9 -f "node.*daemon" && rm -f ~/.agent-browser/default.sock ~/.agent-browser/default.pid |
| AGENTCORE_PROFILE_ID format error | Must match: [a-zA-Z][a-zA-Z0-9_]{0,47}-[a-zA-Z0-9]{10} |
```

---

## How It Works with OpenClaw

Once the skill file is in place, **all OpenClaw agents** automatically get the `agentcore-browser` skill. When an agent needs to browse the web:

1. Agent reads the SKILL.md instructions
2. Cleans up any existing browser daemon
3. Sets `AGENTCORE_REGION` and optionally `AGENTCORE_PROFILE_ID`
4. Uses `agent-browser -p agentcore open <URL>` to start browsing
5. Uses `snapshot` / `eval` to extract page content
6. Closes the session with `agent-browser close`

### Token-Efficient Patterns

**Use `eval` instead of `snapshot` when possible.** Snapshot returns the full accessibility tree which can be large. For structured extraction:

```bash
# Extract Reddit comments (saves ~80% tokens vs snapshot)
agent-browser eval "
  const comments = document.querySelectorAll('shreddit-comment');
  const results = [];
  for (let i = 0; i < Math.min(comments.length, 10); i++) {
    const c = comments[i];
    const author = c.getAttribute('author') || 'unknown';
    const score = c.getAttribute('score') || '0';
    const text = c.querySelector('[slot=\"comment-body\"]')?.innerText?.slice(0, 500) || '';
    results.push({ author, score, text: text.trim() });
  }
  JSON.stringify(results, null, 2);
"
```

```bash
# Extract X/Twitter posts
agent-browser eval "
  const tweets = document.querySelectorAll('article[data-testid=\"tweet\"]');
  const results = [];
  for (let i = 0; i < Math.min(tweets.length, 10); i++) {
    const t = tweets[i];
    const text = t.querySelector('[data-testid=\"tweetText\"]')?.innerText || '';
    const user = t.querySelector('[data-testid=\"User-Name\"]')?.innerText || '';
    results.push({ user: user.split('\\n')[0], text: text.slice(0, 500) });
  }
  JSON.stringify(results, null, 2);
"
```

**Use scoped snapshots for interactive pages:**

```bash
agent-browser snapshot --selector "main" --interactive --compact
```

---

## Profile Management

### Creating Your First Profile

1. **Generate a profile ID** — format: `myagent-<10 random alphanumeric chars>`  
   Example: `mybrowser-aB3cD4eF5g`

2. **Start a session with the new profile:**
   ```bash
   export AGENTCORE_REGION=us-east-1
   export AGENTCORE_PROFILE_ID=mybrowser-aB3cD4eF5g
   agent-browser -p agentcore open https://x.com/login
   ```

3. **Get the Live View URL** — the agent-browser output will include a URL you can open in your local browser to see and interact with the remote browser.

4. **Log in manually** via Live View — your cookies are now saved in the profile.

5. **Close the session:**
   ```bash
   agent-browser close
   ```

6. **Update SKILL.md** — add your profile to the Browser Profiles table.

### Multiple Profiles

You can create separate profiles for different purposes:

| Profile ID | Use Case |
|---|---|
| `work-xXxXxXxXxX` | Work accounts (LinkedIn, etc.) |
| `social-yYyYyYyYyY` | Personal social media |
| `research-zZzZzZzZzZ` | Clean profile for research |

Each agent can choose which profile to use based on the task.

### When Your Agent Needs a New Profile

If the agent encounters a login wall and no suitable profile exists, it should:

1. Notify the human: "I need a browser profile with login access to [site]. Can you set one up?"
2. The human creates the profile, logs in via Live View
3. The human adds the profile ID to the SKILL.md table
4. The agent can now use it

---

## Architecture

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│  OpenClaw   │────▶│  agent-browser   │────▶│  AWS AgentCore      │
│  Agent      │     │  CLI (local)     │     │  Browser (cloud)    │
│             │◀────│                  │◀────│                     │
└─────────────┘     └──────────────────┘     └─────────────────────┘
                           │
                     Local daemon
                     (WebSocket relay)
```

- **OpenClaw Agent** — issues `exec` commands to `agent-browser`
- **agent-browser CLI** — local daemon that relays commands to AWS via WebSocket
- **AgentCore Browser** — cloud Chromium instance with your profile's cookies

---

## Cost & Limits

- AgentCore Browser sessions have a configurable timeout (default: 1 hour)
- Always close sessions when done to avoid unnecessary charges
- Check [AWS Bedrock AgentCore pricing](https://aws.amazon.com/bedrock/pricing/) for current rates

---

## Related

- [OpenClaw Documentation](https://docs.openclaw.ai)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [agent-browser npm](https://www.npmjs.com/package/@anthropic/agent-browser)
