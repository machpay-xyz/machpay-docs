# x402 Negotiation Standard

The x402 standard defines how MachPay Agents and Gateways negotiate payments over plain HTTP before invoking the Optimistic settlement layer.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]

## Three-Step Handshake

1. **Request → 402 Payment Required**
   - Agent calls the vendor endpoint.
   - Gateway responds with status `402` and a JSON payload describing the offer plus a nonce challenge.

2. **Agent Signs the Challenge**
   - Agent serializes the payload into the `PaymentIntent` structure defined in the whitepaper (EIP-712 typed data).
   - Using its private key, the agent signs the challenge and caches the nonce to avoid double-use.

3. **Request + Signature Header → 200 OK**
   - Agent replays the request, attaching the signature in `X-MachPay-Auth`.
   - Gateway verifies the signature against the agent’s solvency bond and serves the resource (`200 OK`).

## Challenge Payload

```json
{
  "service_id": "weather.pro",
  "gateway_id": "0x1234abcd...",
  "price": "0.001",
  "currency": "USDC",
  "nonce": "ce4f3a40-07d1-4df2-9dc0-d2f62e87b6d3",
  "deadline": 1737168000,
  "chain_id": 84532,
  "token": "0xb53913...usdc",
  "metadata": {
    "city": "Reykjavik",
    "unit": "metric"
  }
}
```

This maps directly to the EIP-712 `PaymentIntent` fields (`serviceId`, `amount`, `token`, `nonce`, `deadline`).\n\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]

## Signed Header Example

```
X-MachPay-Auth: sig=0x9e1d...; agent=0xAaBbCc...; nonce=ce4f3a40-07d1-4df2-9dc0-d2f62e87b6d3
```

Gateways may optionally include a `receipt` identifier to help relayers construct Merkle leaves later.

## Security: Nonce-Based Replay Protection

Every challenge includes a globally unique `nonce`. Once an agent signs and submits a nonce, the gateway marks it as consumed. If an attacker replays the same header, the gateway rejects it because the nonce no longer matches an open challenge. Combined with EIP-712 domain separation (chain ID, verifying contract), this prevents cross-service replay attacks and keeps fraudulent agents slashable.
