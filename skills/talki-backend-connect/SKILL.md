---
name: talki-backend-connect
description: Integrate Talki Backend Connect into your project. Use when the user wants to show customer data (name, email, plan, custom fields) in Talki's support dashboard when users open a chat widget. Creates the backend endpoint, verifies signature, and returns customer profile data.
argument-hint: [optional: framework hint e.g. express / fastapi / rails]
disable-model-invocation: false
---

# Talki Backend Connect Integration

Your goal: create a working Backend Connect endpoint so Talki can fetch customer data when a user opens a support ticket. The operator will see name, email, plan, and any custom fields you return — live in the ticket sidebar.

## How it works

1. User sends first widget message → Talki POSTs to your endpoint with `{ "userToken": "<user's ID>" }`
2. Your endpoint verifies the HMAC-SHA256 signature, looks up the user, returns their profile JSON
3. Talki saves and displays it in the ticket panel instantly
4. Operators can hit Refresh any time to re-fetch

## What Talki sends to your endpoint

```
POST <your-endpoint-url>
Content-Type: application/json
X-Talki-Signature: <hmac-sha256-hex of raw body>

{ "userToken": "<visitorId from widget config>" }
```

- 5-second timeout — respond fast
- `userToken` field name is configurable in Admin → Backend Connect (default: `userToken`)

## What your endpoint must return

```json
{
  "name":  "Alice Johnson",
  "email": "alice@acme.com",
  "plan":  "pro",
  "fields": {
    "company":      "Acme Inc",
    "mrr":          149,
    "signedUpAt":   "2024-01-15",
    "lastPurchase": "2025-03-10",
    "seats":        12
  }
}
```

- `name`, `email`, `plan` — shown with special UI treatment (`plan` renders as a green badge)
- `fields` — any key-value pairs, displayed as a list; return whatever is useful for support operators
- All fields optional — return what you have

## Signature verification (REQUIRED)

Always verify `X-Talki-Signature` before trusting the payload. Use the **raw request body** (before JSON parse) to compute HMAC-SHA256 with your signing secret. Use constant-time comparison.

**Node.js / Express:**
```javascript
const crypto = require('crypto');

app.post('/talki/customer', express.raw({ type: 'application/json' }), (req, res) => {
  const sig = req.headers['x-talki-signature'];
  const expected = crypto
    .createHmac('sha256', process.env.TALKI_SECRET)
    .update(req.body) // raw Buffer — NOT parsed JSON
    .digest('hex');

  if (!sig || !crypto.timingSafeEqual(Buffer.from(sig), Buffer.from(expected))) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  const { userToken } = JSON.parse(req.body);
  // look up user by userToken (your DB ID, Firebase UID, etc.)
  const user = await db.users.findById(userToken);
  if (!user) return res.status(404).json({ error: 'User not found' });

  res.json({
    name: user.name,
    email: user.email,
    plan: user.plan,
    fields: {
      company:      user.company,
      mrr:          user.mrr,
      signedUpAt:   user.createdAt,
      lastPurchase: user.lastPurchaseAt,
    },
  });
});
```

**Python / FastAPI:**
```python
import hmac, hashlib, os, json
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()

@app.post("/talki/customer")
async def talki_customer(request: Request):
    raw = await request.body()
    sig = request.headers.get("x-talki-signature", "")
    expected = hmac.new(
        os.environ["TALKI_SECRET"].encode(),
        raw,
        hashlib.sha256,
    ).hexdigest()

    if not hmac.compare_digest(sig, expected):
        raise HTTPException(status_code=401, detail="Invalid signature")

    body = json.loads(raw)
    user = db.get_user(body["userToken"])
    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    return {
        "name":  user.name,
        "email": user.email,
        "plan":  user.plan,
        "fields": {
            "company":      user.company,
            "mrr":          user.mrr,
            "signedUpAt":   str(user.created_at),
            "lastPurchase": str(user.last_purchase_at),
        },
    }
```

**Python / Flask:**
```python
import hmac, hashlib, json, os
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/talki/customer', methods=['POST'])
def talki_customer():
    raw = request.get_data()
    sig = request.headers.get('X-Talki-Signature', '')
    expected = hmac.new(
        os.environ['TALKI_SECRET'].encode(),
        raw,
        hashlib.sha256,
    ).hexdigest()

    if not hmac.compare_digest(sig, expected):
        return jsonify(error='Invalid signature'), 401

    body = json.loads(raw)
    user = db.get_user(body['userToken'])

    return jsonify(
        name=user.name,
        email=user.email,
        plan=user.plan,
        fields={'mrr': user.mrr, 'company': user.company},
    )
```

## Step-by-step instructions

### Step 1 — Explore the project

Read the codebase to understand:
- What framework/language is used (Express, FastAPI, Rails, Laravel, etc.)
- How users are stored and looked up (DB model, table name, ID field)
- What useful fields exist (name, email, plan, subscription status, MRR, company, last purchase, etc.)
- Where environment variables / secrets are stored (`.env`, config files)
- Where existing API routes are defined — add the new endpoint there

If `$ARGUMENTS` contains a framework hint, prioritize that.

### Step 2 — Create the endpoint

Add the Backend Connect endpoint to the appropriate file in the project:
- Follow the existing code style exactly
- Use raw body for HMAC verification (not parsed JSON)
- Look up user by the `userToken` field (this is the visitor ID from the widget)
- Return all useful fields: `name`, `email`, `plan` + anything valuable in `fields`
- Good `fields` candidates: MRR, company name, subscription start, last purchase date, seat count, usage stats, billing email, account tier

### Step 3 — Add the secret to environment config

Add `TALKI_SECRET` to:
- `.env` or `.env.example` (with a placeholder value)
- Any existing secret/config documentation in the project

### Step 4 — Report what was done

After implementing, report:
1. **File changed** — which file the endpoint was added to, line number
2. **Endpoint URL** — the full path (e.g. `POST /talki/customer`) that the user needs to enter in Talki Admin
3. **Fields returned** — list every field your endpoint returns so the user knows what will appear in the dashboard
4. **Next steps for the user:**
   - Go to Talki Console → Admin → Backend Connect
   - Enter the full public URL of the endpoint
   - Paste in the signing secret from `.env`
   - Click Save → Test to verify it works
   - Make sure `visitorId` is set in the widget config to your user's ID

## Important rules

- NEVER hardcode secrets — always use environment variables
- Use constant-time comparison for HMAC (`crypto.timingSafeEqual` / `hmac.compare_digest`)
- Use raw body bytes for signature check, not parsed JSON
- Respond with HTTP 401 for invalid signatures, 404 for unknown users
- Keep response under 5 seconds
- Return `200` with JSON on success — no other status code
