# MachPay Python Quickstart

Build a MachPay-enabled agent that can buy a `Weather Forecast` for `\$0.001` in just a few steps. This guide targets Base Sepolia, matching the testnet architecture described in the whitepaper.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]

## 1. Install the SDK

```bash
pip install machpay-sdk
```

## 2. Initialize the Client

```python
from machpay_sdk import MachPayClient

client = MachPayClient(
    private_key="0xabc123...your_dev_key...",
    network="base-sepolia",  # matches the Layer 2.5 testnet deployment
)
```

## 3. Fund Your Global Bond

Agents must post a solvency bond before streaming payments. Check the balance and top up \$5 USDC if needed.

```python
balance = client.bond.balance()
if balance == 0:
    tx_hash = client.bond.deposit(amount=5, token="USDC")
    client.wait_for_tx(tx_hash)
    print("Bond funded:", tx_hash)
else:
    print("Existing bond balance:", balance)
```

## 4. Execute the Weather Purchase

```python
response = client.get("https://api.machpay.xyz/weather", params={"city": "Reykjavik"})
print(response.json())
```

The SDK handles the 402 negotiation, signs an EIP-712 payment intent, and streams a \$0.001 receipt to the vendor while keeping your bond collateralized.

> **Under the hood**: When the API returns `402 Payment Required`, the SDK replays the request with the MachPay token, attaches the signed header, and updates the local channel state—no manual signing logic needed.

## 5. Next Steps

- Cache intent receipts locally so you can reconcile usage versus spend.
- Use `client.intent.history()` to audit negotiations if you suspect a faulty vendor.
- Promote to production by pointing `network="base-mainnet"` once the protocol exits Epoch 1.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]
