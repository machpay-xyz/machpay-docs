# Proof of Solvency & Slashing

MachPay’s security hinges on the **Global Bond**—a single collateral position that backs every payment an agent makes.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]

## The Global Bond

- Agents post collateral once into the `BondManager` contract.
- Any vendor can accept a signed intent as long as the agent’s on-chain bond balance covers the quoted price.
- Unlike Lightning’s per-vendor channels, a \$50 bond can serve a hundred vendors simultaneously.

## Optimistic Trust

Gateways do not query the blockchain every time. Instead, they:

1. Cache the latest bond root from relayers.
2. Verify signatures off-chain.
3. Serve data instantly because the bond proves the agent can be slashed later.

This keeps authorization under 5 ms while still inheriting L2/L1 security.

## Slashing Double-Spend Attempts

- Risk: An agent signs receipts totaling more than the bonded amount.
- Mitigation: Relayers aggregate receipts, compare to the bond snapshot, and submit a **Fraud Proof** if volume exceeds collateral.
- Outcome: The `slashAgent` function burns part or all of the bond and optionally blacklists the offender, as defined in the whitepaper’s `IMachPayCore` interface.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]

## Game Theory: Why Dine-and-Dash Fails

Let `Bond = B` and `API Cost = C`. Because the protocol requires `B >> C`, the expected penalty for cheating (losing the entire bond) dwarfs any single API call value. Rational agents maximize utility by staying solvent:

```
Expected Gain (cheat) = C
Expected Loss (caught) = B
If B ≥ 50 * C, cheating is economically irrational.
```

This incentive design scales across vendors, making MachPay safer than bilateral state channels and dramatically more capital efficient.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]
