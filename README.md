n# The Economic Layer for Autonomous Agents

MachPay exists to give autonomous AI agents the same economic freedom humans enjoy: instant clearing, flexible identity, and predictable costs. Traditional rails slow them down before they ever execute a task.

## The Silicon Wall

- **Stripe**: Optimized for humans with cards, not AI agents. Every transaction drags KYC, chargeback risk, and per-transaction fees that break micro-inference business models.
- **Ethereum L1**: Trustless but sluggish. Finality in minutes and fluctuating gas makes it impossible for agents to negotiate and settle during live conversations.

Together, these constraints form the Silicon Wall—an invisible barrier that keeps agents from paying or getting paid in real time.

## The MachPay Approach

MachPay is a **Layer 2.5 Optimistic State Channel** purpose-built for machine commerce. Agents negotiate capabilities over x402, stake solvency bonds on-chain, and then execute high-frequency payments off-chain via cryptographic receipts. Disputes settle back to Ethereum for slash-proof accountability.

## How It Compares

|                 | Latency                 | Cost (per tx)           | Identity Model                    |
|-----------------|------------------------|-------------------------|-----------------------------------|
| **MachPay**     | Sub-second optimistic  | Deterministic sub-cent  | Wallet-based, pseudonymous bonds  |
| Stripe          | 2-7 seconds auth/settle| 2.9% + $0.30            | Full KYC & merchant underwriting  |
| L1 Blockchain   | 12-300 seconds finality| Volatile gas fees       | Public address, no verified stake |

## Choose Your Path

- [**Build an Agent (Buyer)**](getting-started/quickstart-python.md) – Plug MachPay into your autonomous agent or orchestrator.
- [**Monetize an API (Seller)**](for-vendors/gateway-integration.md) – Run a gateway, set prices, and start streaming revenue.

MachPay collapses latency, lowers cost, and keeps identity flexible so agents can transact at machine speed. Dive into the guides to start building.
