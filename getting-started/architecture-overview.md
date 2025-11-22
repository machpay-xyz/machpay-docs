# MachPay Architecture: Fast Path vs. Slow Path

MachPay decouples authorization speed from settlement finality, mirroring the Layer 2.5 model outlined in the whitepaper.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]

## The Fast Path — <5 ms Authorization

- **Participants**: Agent ↔ Gateway
- **Transport**: Standard HTTP with an `X-MachPay-Auth` header
- **Flow**:
  1. Agent requests a resource.
  2. Gateway responds with `402 Payment Required` and a nonce challenge.
  3. Agent replays the call with a signed MachPay token (EIP-712 intent).
  4. Gateway validates the signature against the agent’s solvency bond and streams the resource.

Because every check is off-chain and purely cryptographic, authorization happens in under 5 ms, which lets agents stay in conversational real time.

## The Slow Path — Asynchronous Settlement

- **Participants**: Gateway ↔ Relayer ↔ Blockchain
- **Flow**:
  1. Gateways accumulate payment receipts locally.
  2. Relayers scoop thousands of receipts, build a Merkle tree, and craft a batch proof.
  3. The batch is submitted to the Layer 2 settlement contract, slashing fraudulent intents and paying vendors in one transaction.

This batching model amortizes gas costs while preserving on-chain accountability, matching the Optimistic Delivery mechanism described in the MachPay whitepaper.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]

## Visualizing the Flow

Imagine a looped diagram:

1. **Agent ↔ Gateway** (fast loop): Continuous HTTP request/response cycle with signed headers.
2. **Gateway → Relayer**: Periodic export of receipt logs.
3. **Relayer → Blockchain**: Merkle root submission, dispute window, and token transfers.

## Why the Split Matters

- **Unlimited Throughput**: Off-chain loops are bounded only by server hardware; relayers can queue any number of receipts before settling.
- **Zero Gas for Agents**: Agents never touch the blockchain directly; they only sign intents, so the relayer pays gas once per batch.

This dual-path design lets MachPay offer sub-second UX without sacrificing trustless settlement—something neither traditional payment processors nor monolithic L1 blockchains can achieve.
