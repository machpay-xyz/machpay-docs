# Authentication & API Keys

Learn how to securely manage API keys, implement OAuth flows, and handle token rotation. This guide covers everything you need to authenticate with MachPay in production.

**Time:** 8 minutes

## Overview

MachPay supports multiple authentication methods:

| Method | Use Case | Security Level |
|--------|----------|----------------|
| API Keys | Server-to-server | High |
| OAuth 2.0 | User-facing apps | High |
| JWT Tokens | Short-lived sessions | Medium |

## API Keys

### Creating Keys

1. Navigate to **Keys** in the [Console](https://console.machpay.xyz/keys)
2. Click **Create New Key**
3. Configure permissions (see Scopes below)
4. Copy and store the key securely

> ⚠️ The full key is only shown once. Store it in a secure location.

### Key Types

| Type | Prefix | Use Case |
|------|--------|----------|
| Live | `sk_live_` | Production environments |
| Test | `sk_test_` | Development and testing |

Test keys work against the sandbox environment and never charge real money.

### Scopes & Permissions

When creating a key, select only the permissions you need:

```
skills:execute     - Execute skills
skills:list        - List available skills
account:read       - View balance and history
account:write      - Add funds, manage payment methods
webhooks:manage    - Create and manage webhooks
```

**Example: Read-only monitoring key**
```
skills:list, account:read
```

**Example: Full automation key**
```
skills:execute, account:read, webhooks:manage
```

## Using API Keys

### Environment Variables (Recommended)

```bash
# .env file
MACHPAY_API_KEY=sk_live_abc123...
```

```python
# Python - automatically picks up env var
from machpay import MachPay
client = MachPay()
```

```typescript
// Node.js - automatically picks up env var
import { MachPay } from '@machpay/sdk';
const client = new MachPay();
```

### Explicit Configuration

```python
# Python
client = MachPay(api_key="sk_live_abc123...")
```

```typescript
// Node.js
const client = new MachPay({ apiKey: 'sk_live_abc123...' });
```

### HTTP Header

For direct API calls without the SDK:

```bash
curl https://api.machpay.xyz/v1/skills/execute \
  -H "Authorization: Bearer sk_live_abc123..." \
  -H "Content-Type: application/json" \
  -d '{"skill_id": "text-summarizer", "input": {...}}'
```

## Key Rotation

Rotate keys regularly to minimize exposure risk.

### Rotation Strategy

1. **Create new key** with the same permissions
2. **Deploy new key** to your services
3. **Verify** the new key works in production
4. **Revoke old key** after confirming everything works

### Automated Rotation

```python
from machpay import MachPay

admin = MachPay(api_key=os.environ["MACHPAY_ADMIN_KEY"])

# Create new key
new_key = admin.keys.create(
    name="production-api-v2",
    scopes=["skills:execute", "account:read"]
)

# Store new_key.secret securely (only shown once)
print(f"New key created: {new_key.id}")

# After deploying new key, revoke old one
admin.keys.revoke("key_old123")
```

## OAuth 2.0

For applications that act on behalf of users.

### Authorization Flow

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│   User   │────▶│ Your App │────▶│ MachPay  │
└──────────┘     └──────────┘     └──────────┘
     │                │                │
     │ 1. Login       │                │
     │───────────────▶│                │
     │                │ 2. Redirect    │
     │                │───────────────▶│
     │                │                │ 3. User authorizes
     │                │◀───────────────│
     │                │ 4. Auth code   │
     │                │                │
     │                │ 5. Exchange    │
     │                │───────────────▶│
     │                │◀───────────────│
     │                │ 6. Access token│
     │◀───────────────│                │
     │ 7. Logged in   │                │
```

### Implementation

**Step 1: Redirect to MachPay**

```typescript
const authUrl = `https://auth.machpay.xyz/oauth/authorize?` +
  `client_id=${CLIENT_ID}&` +
  `redirect_uri=${encodeURIComponent(REDIRECT_URI)}&` +
  `response_type=code&` +
  `scope=skills:execute account:read`;

// Redirect user to authUrl
```

**Step 2: Handle Callback**

```typescript
// In your callback route
app.get('/callback', async (req, res) => {
  const { code } = req.query;
  
  const tokens = await machpay.oauth.exchangeCode({
    code,
    redirectUri: REDIRECT_URI,
    clientId: CLIENT_ID,
    clientSecret: CLIENT_SECRET
  });
  
  // Store tokens.accessToken and tokens.refreshToken
  req.session.accessToken = tokens.accessToken;
  res.redirect('/dashboard');
});
```

**Step 3: Use Access Token**

```typescript
const client = new MachPay({
  accessToken: req.session.accessToken
});

const response = await client.skills.execute({...});
```

### Refreshing Tokens

Access tokens expire after 1 hour. Use the refresh token to get a new one:

```typescript
const newTokens = await machpay.oauth.refreshToken({
  refreshToken: storedRefreshToken,
  clientId: CLIENT_ID,
  clientSecret: CLIENT_SECRET
});
```

## Security Best Practices

### Do ✅

- Store keys in environment variables or secret managers
- Use scoped keys with minimal permissions
- Rotate keys regularly (every 90 days recommended)
- Use test keys for development
- Monitor key usage in the Console

### Don't ❌

- Commit keys to version control
- Share keys across environments
- Use admin keys in client-side code
- Log full API keys
- Reuse keys across multiple services

## IP Allowlisting

Restrict key usage to specific IP addresses:

```python
admin.keys.update(
    key_id="key_abc123",
    allowed_ips=["203.0.113.10", "203.0.113.0/24"]
)
```

## Rate Limits by Key Type

| Key Type | Requests/min | Concurrent |
|----------|--------------|------------|
| Test | 100 | 5 |
| Live (Standard) | 1,000 | 50 |
| Live (Enterprise) | 10,000 | 500 |

## Next Steps

- [Error Handling](error-handling.md) – Handle auth errors gracefully
- [Webhooks](webhooks.md) – Secure webhook signatures
- [Building Your First Skill](first-skill.md) – Start building

---

**Questions?** Check the [API Reference](/protocol-reference/api-specs.md) or join [Discord](https://discord.gg/machpay).

