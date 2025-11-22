# MCP Integration: Monetize ChatGPT/Claude Tools

Model Context Protocol (MCP) lets you expose tools directly inside AI assistants. MachPay adds pricing, solvency, and settlement so you can turn those tools into paid services.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]

## The Opportunity

- **Paid Search**: Charge \$0.05 for curated search results.
- **Premium Compute**: Rent GPU inference per request.
- **Private Data Access**: Meter access to proprietary corpuses or APIs.

## Python Wrapper Example

```python
from machpay.mcp import PaidTool, gateway

@PaidTool(price="0.05", currency="USDC")
def premium_weather(city: str):
    data = gateway.get("/weather/pro", params={"city": city})
    return data["forecast"]
```

The `machpay.mcp` wrapper handles x402 negotiation, checks the agent’s bond, and emits receipts for settlement.

## Manifest Snippet

Agents discover pricing through the MCP manifest:

```json
{
  "name": "premium_weather",
  "description": "High-resolution forecast with proprietary sensors",
  "pricing": {
    "price": "0.05",
    "currency": "USDC",
    "payment_provider": "machpay"
  },
  "inputs": [
    { "name": "city", "type": "string", "required": true }
  ]
}
```

When ChatGPT or Claude sees this manifest, they know the tool requires a MachPay payment before execution. The orchestrator uses the agent’s bond to sign the payment intent and seamlessly returns the result to the user.
