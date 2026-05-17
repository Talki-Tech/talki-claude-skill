# Talki Claude Skill

AI skill for [Claude Code](https://claude.ai/code) that integrates [Talki](https://talki.tech) Backend Connect into your project automatically.

**What it does:** Opens your codebase, finds your user model, writes the Backend Connect endpoint in your framework's style, adds the secret to your `.env`, and tells you exactly what to paste into Talki Admin — all in one command.

---

## Install

### Option A — Project skill (one project)

Copy the skill into your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp -r talki-backend-connect .claude/skills/
```

Or clone directly:

```bash
git clone https://github.com/Talki-Tech/talki-claude-skill.git /tmp/talki-skill
mkdir -p .claude/skills
cp -r /tmp/talki-skill/skills/talki-backend-connect .claude/skills/
```

### Option B — Personal skill (all your projects)

Install once, available everywhere:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/Talki-Tech/talki-claude-skill.git /tmp/talki-skill
cp -r /tmp/talki-skill/skills/talki-backend-connect ~/.claude/skills/
```

---

## Use

Open Claude Code in your backend project and run:

```
/talki-backend-connect
```

Or just describe what you want:

```
integrate Talki Backend Connect
```

Claude will:
1. Read your codebase — detect framework, find user model, find useful fields
2. Create the `/talki/customer` endpoint with proper HMAC-SHA256 signature verification
3. Add `TALKI_SECRET` to your `.env` / `.env.example`
4. Tell you the exact URL to paste into Talki Admin and which fields will show in the dashboard

You can pass a framework hint if Claude needs guidance:

```
/talki-backend-connect express
/talki-backend-connect fastapi
/talki-backend-connect rails
```

---

## What you get in the Talki dashboard

After integrating, operators see this in every ticket sidebar:

| Field | Source | Display |
|-------|--------|---------|
| `name` | your endpoint | plain text |
| `email` | your endpoint | plain text |
| `plan` | your endpoint | green badge |
| `fields.*` | your endpoint | key-value list |

Example `fields` you can return:

```json
{
  "company":      "Acme Inc",
  "mrr":          149,
  "seats":        12,
  "signedUpAt":   "2024-01-15",
  "lastPurchase": "2025-03-10",
  "accountTier":  "enterprise"
}
```

Any key-value pair in `fields` appears in the dashboard — return whatever your support team needs.

---

## How Backend Connect works

```
Widget user sends message
        ↓
Talki POSTs to your endpoint
  { "userToken": "<user's ID>" }
  X-Talki-Signature: <hmac-sha256>
        ↓
Your endpoint verifies signature
looks up user, returns JSON
        ↓
Talki shows data in ticket sidebar
```

The `userToken` is the `visitorId` you set when initializing the widget:

```html
<script>
  window.TalkiConfig = {
    orgId:     'YOUR_ORG_ID',
    apiKey:    'YOUR_PUBLIC_API_KEY',
    visitorId: currentUser.id,  // ← your user's unique ID
  };
</script>
```

---

## Manual setup (without Claude)

If you prefer to write the endpoint yourself, see the full guide:
[talki.tech/docs/backend-connect](https://talki.tech/docs/backend-connect)

---

## Requirements

- [Claude Code](https://claude.ai/code) desktop app or CLI
- A Talki account — [talki.tech](https://talki.tech)


## Help 
If you have any questions, please contact us at hello@talki.tech or use the chat widget on the [talki.tech](https://talki.tech) website.