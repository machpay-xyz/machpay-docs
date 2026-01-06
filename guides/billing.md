# Billing & Metering

Understand pay-per-use billing, set up metering for your skills, and optimize costs. This guide covers both consuming and providing skills on MachPay.

**Time:** 6 minutes

## How Billing Works

MachPay uses **pay-per-use** billing:

```
Execution Cost = Base Price + (Usage Units × Unit Price)
```

- **No subscriptions** – Pay only for what you use
- **Real-time billing** – Charges deducted instantly
- **Transparent pricing** – See cost before execution

## Understanding Your Bill

### Viewing Usage

```python
from machpay import MachPay

client = MachPay()

# Get current balance
balance = client.account.balance()
print(f"Available: ${balance.available:.2f}")
print(f"Pending: ${balance.pending:.2f}")

# Get usage summary
usage = client.account.usage(period="month")
print(f"Total spent: ${usage.total_cost:.2f}")
print(f"Executions: {usage.total_executions}")
print(f"Most used skill: {usage.top_skills[0]['name']}")
```

### Cost Breakdown

```python
# Detailed breakdown by skill
breakdown = client.account.usage_breakdown(
    period="month",
    group_by="skill"
)

for skill in breakdown:
    print(f"{skill['name']}: ${skill['cost']:.4f} ({skill['executions']} executions)")
```

Output:
```
text-summarizer: $1.2340 (1234 executions)
chat-assistant: $0.8920 (89 executions)
image-classifier: $0.4500 (45 executions)
```

## Cost Estimation

Estimate cost before execution:

```python
# Get skill pricing info
skill = client.skills.get("text-summarizer")
print(f"Base price: ${skill.pricing.base_usd}")
print(f"Per-token: ${skill.pricing.per_token_usd}")

# Estimate cost for your input
estimate = client.skills.estimate_cost(
    skill_id="text-summarizer",
    input={"text": "Your long text here..." * 100}
)
print(f"Estimated cost: ${estimate.cost_usd:.4f}")
print(f"Estimated tokens: {estimate.tokens}")
```

## Budget Controls

### Set Spending Limits

```python
# Set monthly budget
client.account.set_budget(
    monthly_limit_usd=100.00,
    alert_threshold=0.8  # Alert at 80%
)

# Set per-execution limit
client.account.set_execution_limit(
    max_cost_usd=1.00  # Reject executions > $1
)
```

### Low Balance Alerts

```python
# Configure balance alerts
client.account.set_alerts(
    low_balance_threshold=10.00,  # Alert when below $10
    webhook_url="https://your-server.com/alerts"
)
```

## Pricing Your Skills

### Fixed Price

Simple per-execution pricing:

```python
from machpay import Skill

skill = Skill(
    id="my-skill",
    pricing={
        "base_usd": 0.001  # $0.001 per execution
    }
)
```

### Usage-Based Pricing

Charge based on consumption:

```python
skill = Skill(
    id="llm-wrapper",
    pricing={
        "base_usd": 0.0005,      # Base fee
        "per_token_usd": 0.00002  # Per output token
    }
)
```

### Tiered Pricing

Volume discounts:

```python
skill = Skill(
    id="high-volume-skill",
    pricing={
        "tiers": [
            {"up_to": 1000, "price_usd": 0.001},     # First 1K: $0.001
            {"up_to": 10000, "price_usd": 0.0008},   # Next 9K: $0.0008
            {"up_to": None, "price_usd": 0.0005}     # Beyond: $0.0005
        ]
    }
)
```

### Time-Based Pricing

For long-running tasks:

```python
skill = Skill(
    id="video-processor",
    pricing={
        "base_usd": 0.01,
        "per_second_usd": 0.001  # $0.001 per second
    }
)
```

## Revenue & Payouts

### Viewing Earnings

```python
# Your skill revenue
revenue = client.vendor.revenue(period="month")
print(f"Total revenue: ${revenue.total_usd:.2f}")
print(f"Pending payout: ${revenue.pending_usd:.2f}")
print(f"Next payout: {revenue.next_payout_date}")
```

### Revenue by Skill

```python
breakdown = client.vendor.revenue_breakdown(
    period="month",
    group_by="skill"
)

for skill in breakdown:
    print(f"{skill['name']}: ${skill['revenue']:.2f} ({skill['executions']} exec)")
```

### Payout Schedule

| Threshold | Payout |
|-----------|--------|
| $25+ | Weekly (Monday) |
| $100+ | Daily |
| Custom | On request |

### Configure Payout

```python
# Set payout method
client.vendor.set_payout_method(
    type="bank_account",
    details={
        "account_number": "****1234",
        "routing_number": "****5678"
    }
)

# Or crypto
client.vendor.set_payout_method(
    type="usdc",
    details={
        "address": "0x...",
        "network": "base"
    }
)
```

## MachPay Fees

| Fee | Amount | Description |
|-----|--------|-------------|
| Platform fee | 5% | Deducted from skill revenue |
| Payment processing | 2.9% + $0.30 | When adding funds via card |
| Crypto top-up | 0% | No fee for USDC deposits |
| Withdrawal | $0.25 | Per bank withdrawal |
| Crypto withdrawal | Network gas | Varies by network |

### Fee Example

```
User pays: $1.00 for skill execution
├── Skill creator receives: $0.95 (95%)
└── MachPay fee: $0.05 (5%)
```

## Cost Optimization

### 1. Use Caching

```python
from functools import lru_cache
import hashlib

@lru_cache(maxsize=1000)
def cached_execute(input_hash: str):
    return client.skills.execute(
        skill_id="expensive-skill",
        input={"key": input_hash}
    )

# Hash input for cache key
input_hash = hashlib.sha256(json.dumps(input).encode()).hexdigest()
result = cached_execute(input_hash)
```

### 2. Batch Requests

```python
# Instead of 10 separate calls
for item in items:
    client.skills.execute(skill_id="processor", input={"item": item})

# Batch into one call
client.skills.execute(
    skill_id="batch-processor",
    input={"items": items}
)
```

### 3. Choose Right Skill

```python
# Quick check with cheap skill
quick_result = client.skills.execute(
    skill_id="quick-classifier",  # $0.0001
    input={"text": text}
)

# Only use expensive skill if needed
if quick_result.data["needs_deep_analysis"]:
    deep_result = client.skills.execute(
        skill_id="deep-analyzer",  # $0.01
        input={"text": text}
    )
```

### 4. Set Execution Limits

```python
# Prevent runaway costs
response = client.skills.execute(
    skill_id="llm-skill",
    input={"prompt": "..."},
    max_cost_usd=0.05  # Fail if would cost more than $0.05
)
```

## Metering Your Skills

Track usage for billing:

```python
from machpay import Skill, SkillInput, MeterEvent

skill = Skill(id="metered-skill")

@skill.handler
def handle(input: SkillInput):
    # Do work
    result = process(input.data)
    
    # Report usage
    skill.meter(
        MeterEvent(
            metric="tokens",
            value=len(result["text"].split())
        ),
        MeterEvent(
            metric="api_calls",
            value=1
        )
    )
    
    return result
```

## Invoices & Receipts

```python
# Get invoices
invoices = client.account.invoices(year=2025)

for invoice in invoices:
    print(f"{invoice.period}: ${invoice.total_usd:.2f}")
    print(f"  Download: {invoice.pdf_url}")

# Get receipt for specific transaction
receipt = client.account.receipt(transaction_id="txn_abc123")
print(receipt.pdf_url)
```

## Best Practices

| Do ✅ | Don't ❌ |
|------|---------|
| Set budget limits | Let spending run unchecked |
| Monitor usage daily | Wait for monthly bill surprise |
| Use cost estimation | Execute blindly |
| Cache expensive results | Re-compute everything |
| Batch when possible | Make many small requests |
| Choose appropriate tiers | Overpay for volume |

## Next Steps

- [Building Your First Skill](first-skill.md) – Set up pricing
- [Webhooks](webhooks.md) – Get payment notifications
- [Error Handling](error-handling.md) – Handle billing errors

---

**Questions?** Check the [Billing FAQ](https://console.machpay.xyz/settings/billing) or join [Discord](https://discord.gg/machpay).

