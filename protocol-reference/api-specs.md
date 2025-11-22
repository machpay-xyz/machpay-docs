# MachPay API Specs

These headers and status codes define the x402 negotiation surface for HTTP vendors, aligning with the whitepaper’s Optimistic Delivery model.\[file:///Users/abhishektomar/Desktop/git/machpay-docs/whitepaper/machpay_whitepaper.pdf]

## Required Headers

| Header            | Description                                                                 |
|-------------------|-----------------------------------------------------------------------------|
| `X-MachPay-Auth`  | Signed intent header: `sig=0x...; agent=0x...; nonce=...`.                 |
| `X-MachPay-Cost`  | (Optional response) Declares the unit price returned with `402` or `200`.   |

Gateways also include a JSON `Challenge` body when returning `402 Payment Required`, containing `service_id`, `price`, `nonce`, and `deadline`.

## Status Codes

- **200 OK** – Request authenticated and forwarded to the backend.
- **402 Payment Required** – Gateway issues a payment challenge; agent must sign and replay.
- **403 Forbidden** – Signature invalid, nonce reused, or bond insufficient; no service is rendered.

Clients should treat other 4xx/5xx responses as vendor-specific errors outside the MachPay protocol.
