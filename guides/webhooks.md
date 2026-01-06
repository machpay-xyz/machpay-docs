# Webhooks & Event Handling

Set up webhooks to receive real-time notifications for skill executions, payments, and errors. Build event-driven integrations that respond instantly to MachPay events.

**Time:** 8 minutes

## Overview

Webhooks push events to your server in real-time:

```
MachPay Event → Your Webhook Endpoint → Your Business Logic
```

Instead of polling, webhooks notify you immediately when:
- A skill execution completes
- A payment is received
- An error occurs
- Your balance changes

## Supported Events

| Event | Description |
|-------|-------------|
| `skill.executed` | Skill execution completed |
| `skill.failed` | Skill execution failed |
| `payment.received` | Payment credited to your account |
| `payment.sent` | Payment deducted from your account |
| `balance.low` | Balance dropped below threshold |
| `balance.depleted` | Balance is zero |
| `key.created` | New API key created |
| `key.revoked` | API key revoked |

## Creating Webhooks

### Via Console

1. Go to [Console → Settings → Webhooks](https://console.machpay.xyz/settings)
2. Click **Add Webhook**
3. Enter your endpoint URL
4. Select events to subscribe to
5. Copy the signing secret

### Via API

```python
from machpay import MachPay

client = MachPay()

webhook = client.webhooks.create(
    url="https://your-server.com/webhooks/machpay",
    events=["skill.executed", "payment.received", "balance.low"],
    description="Production webhook"
)

print(f"Webhook ID: {webhook.id}")
print(f"Signing Secret: {webhook.secret}")  # Store securely!
```

### Via CLI

```bash
machpay webhook create \
  --url https://your-server.com/webhooks/machpay \
  --events skill.executed,payment.received,balance.low
```

## Webhook Payload

All webhooks follow this structure:

```json
{
  "id": "evt_abc123",
  "type": "skill.executed",
  "created_at": "2025-01-06T12:00:00Z",
  "data": {
    "skill_id": "text-summarizer",
    "execution_id": "exec_xyz789",
    "status": "success",
    "duration_ms": 245,
    "cost_usd": 0.0012,
    "input": {"text": "..."},
    "output": {"summary": "..."}
  }
}
```

### Event-Specific Payloads

**skill.executed**
```json
{
  "type": "skill.executed",
  "data": {
    "skill_id": "text-summarizer",
    "execution_id": "exec_xyz789",
    "status": "success",
    "duration_ms": 245,
    "cost_usd": 0.0012
  }
}
```

**payment.received**
```json
{
  "type": "payment.received",
  "data": {
    "payment_id": "pay_def456",
    "amount_usd": 10.50,
    "from_account": "acc_buyer123",
    "skill_id": "your-skill",
    "execution_id": "exec_xyz789"
  }
}
```

**balance.low**
```json
{
  "type": "balance.low",
  "data": {
    "current_balance_usd": 5.00,
    "threshold_usd": 10.00
  }
}
```

## Handling Webhooks

### Python (Flask)

```python
from flask import Flask, request, jsonify
import hmac
import hashlib

app = Flask(__name__)
WEBHOOK_SECRET = "whsec_..."

def verify_signature(payload: bytes, signature: str) -> bool:
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)

@app.route("/webhooks/machpay", methods=["POST"])
def handle_webhook():
    # Verify signature
    signature = request.headers.get("X-MachPay-Signature")
    if not verify_signature(request.data, signature):
        return jsonify({"error": "Invalid signature"}), 401
    
    event = request.json
    
    # Handle different event types
    if event["type"] == "skill.executed":
        handle_skill_executed(event["data"])
    elif event["type"] == "payment.received":
        handle_payment_received(event["data"])
    elif event["type"] == "balance.low":
        handle_low_balance(event["data"])
    
    return jsonify({"received": True}), 200

def handle_skill_executed(data):
    print(f"Skill {data['skill_id']} executed in {data['duration_ms']}ms")

def handle_payment_received(data):
    print(f"Received ${data['amount_usd']} from {data['from_account']}")

def handle_low_balance(data):
    print(f"Low balance alert: ${data['current_balance_usd']}")
    # Send notification, trigger auto-reload, etc.
```

### Node.js (Express)

```typescript
import express from 'express';
import crypto from 'crypto';

const app = express();
const WEBHOOK_SECRET = 'whsec_...';

function verifySignature(payload: Buffer, signature: string): boolean {
  const expected = crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(payload)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(`sha256=${expected}`),
    Buffer.from(signature)
  );
}

app.post('/webhooks/machpay', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-machpay-signature'] as string;
  
  if (!verifySignature(req.body, signature)) {
    return res.status(401).json({ error: 'Invalid signature' });
  }
  
  const event = JSON.parse(req.body.toString());
  
  switch (event.type) {
    case 'skill.executed':
      console.log(`Skill ${event.data.skill_id} executed`);
      break;
    case 'payment.received':
      console.log(`Received $${event.data.amount_usd}`);
      break;
    case 'balance.low':
      console.log(`Low balance: $${event.data.current_balance_usd}`);
      break;
  }
  
  res.json({ received: true });
});
```

### Using the SDK

```python
from machpay.webhooks import WebhookHandler

handler = WebhookHandler(secret="whsec_...")

@handler.on("skill.executed")
def on_skill_executed(event):
    print(f"Skill executed: {event.data['execution_id']}")

@handler.on("payment.received")
def on_payment_received(event):
    print(f"Payment received: ${event.data['amount_usd']}")

# In your Flask/FastAPI route:
@app.post("/webhooks/machpay")
def webhook():
    return handler.handle(request.data, request.headers)
```

## Signature Verification

**Always verify webhook signatures** to ensure requests are from MachPay.

The signature is in the `X-MachPay-Signature` header:

```
X-MachPay-Signature: sha256=abc123...
```

Verification steps:
1. Get the raw request body (don't parse JSON first)
2. Compute HMAC-SHA256 with your webhook secret
3. Compare with the provided signature (timing-safe)

```python
def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    computed = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    # Use constant-time comparison to prevent timing attacks
    return hmac.compare_digest(f"sha256={computed}", signature)
```

## Retry Policy

MachPay retries failed webhook deliveries:

| Attempt | Delay |
|---------|-------|
| 1 | Immediate |
| 2 | 1 minute |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |

Webhooks are considered failed if:
- Your endpoint returns a non-2xx status
- Connection times out (30 seconds)
- DNS/SSL errors occur

## Idempotency

Webhooks may be delivered more than once. Use the event ID to deduplicate:

```python
from redis import Redis

redis = Redis()

@app.post("/webhooks/machpay")
def webhook():
    event = request.json
    event_id = event["id"]
    
    # Check if already processed
    if redis.get(f"webhook:{event_id}"):
        return jsonify({"status": "already_processed"}), 200
    
    # Process the event
    process_event(event)
    
    # Mark as processed (expire after 24 hours)
    redis.setex(f"webhook:{event_id}", 86400, "1")
    
    return jsonify({"received": True}), 200
```

## Testing Webhooks

### Using the Console

1. Go to **Settings → Webhooks**
2. Click **Test** on your webhook
3. Select an event type
4. View the request and response

### Using the CLI

```bash
# Send a test event
machpay webhook test \
  --id whk_abc123 \
  --event skill.executed

# Check recent deliveries
machpay webhook deliveries --id whk_abc123
```

### Local Development

Use a tunnel for local testing:

```bash
# Using ngrok
ngrok http 3000

# Then create webhook with ngrok URL
machpay webhook create \
  --url https://abc123.ngrok.io/webhooks/machpay \
  --events skill.executed
```

## Managing Webhooks

```python
# List webhooks
webhooks = client.webhooks.list()

# Update webhook
client.webhooks.update(
    webhook_id="whk_abc123",
    events=["skill.executed", "skill.failed"],
    enabled=True
)

# Delete webhook
client.webhooks.delete("whk_abc123")

# View delivery history
deliveries = client.webhooks.deliveries(
    webhook_id="whk_abc123",
    limit=10
)

for d in deliveries:
    print(f"{d.event_type}: {d.status} ({d.response_code})")
```

## Best Practices

| Do ✅ | Don't ❌ |
|------|---------|
| Verify signatures | Trust unverified requests |
| Respond quickly (< 5s) | Do heavy processing synchronously |
| Implement idempotency | Process duplicates |
| Log all webhook events | Lose audit trail |
| Use HTTPS endpoints | Accept HTTP |
| Handle retries gracefully | Return errors for transient issues |

### Async Processing

For heavy workloads, queue events and respond immediately:

```python
from celery import Celery

celery = Celery()

@app.post("/webhooks/machpay")
def webhook():
    event = request.json
    
    # Verify signature first
    if not verify_signature(request.data, request.headers.get("X-MachPay-Signature")):
        return jsonify({"error": "Invalid signature"}), 401
    
    # Queue for async processing
    process_webhook.delay(event)
    
    # Respond immediately
    return jsonify({"received": True}), 200

@celery.task
def process_webhook(event):
    # Heavy processing here
    ...
```

## Next Steps

- [Error Handling](error-handling.md) – Handle webhook failures
- [Authentication](authentication.md) – Secure your endpoints
- [Billing & Metering](billing.md) – Track payments

---

**Questions?** Check the [API Reference](/protocol-reference/api-specs.md) or join [Discord](https://discord.gg/machpay).

