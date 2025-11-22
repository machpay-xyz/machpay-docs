# EIP-712 Specification

Wallets and SDKs must encode payment intents using the following domain separator and struct, matching the `IMachPayCore` contract series described in the whitepaper.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]

## Domain Separator

```solidity
bytes32 constant DOMAIN_SEPARATOR = keccak256(
    abi.encode(
        keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
        keccak256(bytes("MachPay Protocol")),
        keccak256(bytes("1.0")),
        chainId,             // e.g., 84532 for Base Sepolia
        verifyingContract    // MachPayCore address
    )
);
```

## PaymentIntent Struct

```solidity
struct PaymentIntent {
    address agent;     // Wallet that signs the intent
    string  serviceId; // Vendor-defined resource identifier
    uint256 amount;    // Priced in wei (USDC decimals, etc.)
    address token;     // ERC-20 token contract
    uint256 nonce;     // Unique per request (UUID or counter)
    uint256 deadline;  // Unix timestamp for expiry
}
```

Type hash declaration:

```solidity
bytes32 constant PAYMENT_INTENT_TYPEHASH =
    keccak256(
        "PaymentIntent(address agent,string serviceId,uint256 amount,address token,uint256 nonce,uint256 deadline)"
    );
```

## Signing Flow

1. Serialize the challenge payload into the struct above.
2. Compute `hashStruct = keccak256(abi.encode(PAYMENT_INTENT_TYPEHASH, ...))`.
3. Sign `EIP712TypedData = keccak256(abi.encodePacked("\x19\x01", DOMAIN_SEPARATOR, hashStruct))`.

Relayers use the same encoding to verify receipts before batching signatures into the settlement Merkle tree.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]
