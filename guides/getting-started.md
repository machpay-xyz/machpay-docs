# Getting Started with MachPay

A comprehensive introduction to MachPay, covering setup, authentication, and your first API call. By the end of this guide, you'll have a working integration that can execute skills on the MachPay network.

**Time:** 5 minutes

## Prerequisites

- Python 3.9+ or Node.js 18+
- A MachPay account (sign up at [console.machpay.xyz](https://console.machpay.xyz))
- Basic familiarity with REST APIs

## Step 1: Get Your API Key

1. Log in to the [MachPay Console](https://console.machpay.xyz)
2. Navigate to **Keys** in the sidebar
3. Click **Create New Key**
4. Copy your API key and store it securely

> ⚠️ **Security Note**: Never commit your API key to version control. Use environment variables instead.

## Step 2: Install the SDK

### Python

```bash
pip install machpay
```

### Node.js

```bash
npm install @machpay/sdk
```

## Step 3: Initialize the Client

### Python

```python
from machpay import MachPay
import os

# Initialize with API key from environment
client = MachPay(api_key=os.environ.get("MACHPAY_API_KEY"))

# Or let the SDK find it automatically
client = MachPay()  # Uses MACHPAY_API_KEY env var
```

### Node.js

```typescript
import { MachPay } from '@machpay/sdk';

// Initialize with API key from environment
const client = new MachPay({
  apiKey: process.env.MACHPAY_API_KEY
});

// Or let the SDK find it automatically
const client = new MachPay(); // Uses MACHPAY_API_KEY env var
```

## Step 4: Execute Your First Skill

### Python

```python
# Execute a text summarization skill
response = client.skills.execute(
    skill_id="text-summarizer",
    input={
        "text": "MachPay is a payment protocol designed for autonomous AI agents...",
        "max_length": 50
    }
)

print(response.data["summary"])
# Output: "MachPay enables AI agents to make instant payments..."
```

### Node.js

```typescript
// Execute a text summarization skill
const response = await client.skills.execute({
  skillId: 'text-summarizer',
  input: {
    text: 'MachPay is a payment protocol designed for autonomous AI agents...',
    maxLength: 50
  }
});

console.log(response.data.summary);
// Output: "MachPay enables AI agents to make instant payments..."
```

## Step 5: Check Your Balance

After executing a skill, you can check your account balance:

### Python

```python
balance = client.account.balance()
print(f"Available: ${balance.available}")
print(f"Pending: ${balance.pending}")
```

### Node.js

```typescript
const balance = await client.account.balance();
console.log(`Available: $${balance.available}`);
console.log(`Pending: $${balance.pending}`);
```

## Understanding the Response

Every skill execution returns a standardized response:

```json
{
  "success": true,
  "data": {
    "summary": "MachPay enables AI agents to make instant payments...",
    "tokens_used": 145,
    "processing_time_ms": 89
  },
  "meta": {
    "skill_id": "text-summarizer",
    "version": "1.2.0",
    "execution_id": "exec_abc123",
    "cost_usd": 0.0012
  }
}
```

| Field | Description |
|-------|-------------|
| `success` | Whether the execution completed successfully |
| `data` | The skill's output (varies by skill) |
| `meta.skill_id` | The skill that was executed |
| `meta.execution_id` | Unique ID for this execution |
| `meta.cost_usd` | Cost deducted from your balance |

## Next Steps

Now that you've made your first API call, explore these guides:

- [Authentication & API Keys](authentication.md) – Learn about scoped keys and OAuth
- [Building Your First Skill](first-skill.md) – Create and monetize your own skill
- [Error Handling](error-handling.md) – Handle failures gracefully

## Troubleshooting

### "Invalid API Key" Error

- Verify your API key is correct
- Check that the key hasn't been revoked
- Ensure you're using the right environment (test vs. production)

### "Insufficient Balance" Error

- Add funds in the [Console](https://console.machpay.xyz/funds)
- Check your current balance with `client.account.balance()`

### Connection Timeout

- Check your internet connection
- Verify MachPay status at [status.machpay.xyz](https://status.machpay.xyz)
- Try again with increased timeout: `MachPay(timeout=30)`

---

**Need help?** Join our [Discord](https://discord.gg/machpay) or check the [FAQ](/guides/faq).

