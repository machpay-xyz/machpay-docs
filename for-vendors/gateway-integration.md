# Gateway Integration Guide

Turn any FastAPI (or HTTP) service into a MachPay merchant without touching your application code. Per the whitepaper, gateways handle the x402 negotiation and proof-of-solvency checks on your behalf.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]

## The Sidecar Model

Deploy the MachPay Gateway as a sidecar proxy in front of your existing API:

```
Internet → MachPay Gateway → Your FastAPI app (localhost:8000)
```

- No SDK rewrites.
- No auth middleware changes.
- The gateway terminates x402, validates signatures, and only forwards authorized requests.

## Sample Configuration

`config.yaml`

```yaml
target_url: http://localhost:8000
price_per_request: "0.001 USDC"
wallet_private_key: "0xabc123... (keep in env)"
network: base-sepolia
service_id: weather.pro
```

Best practice: store `wallet_private_key` in an environment variable and reference it via `${MACHPAY_KEY}`.

## What the Gateway Does

1. Receives inbound requests and checks for `X-MachPay-Auth`.
2. If missing, returns `402 Payment Required` with the challenge payload.
3. When the agent replays with a valid signature, the gateway:
   - Verifies the EIP-712 intent.
   - Confirms the agent’s bond balance from the latest solvency snapshot.
   - Logs the receipt for the relayer.
   - Forwards the request to your FastAPI backend and streams the response.

This means your service experiences authenticated traffic only, while MachPay handles pricing, accounting, and settlement behind the scenes.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]
