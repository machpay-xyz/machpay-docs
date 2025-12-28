# MachPay Python Agent SDK - Implementation Design

## Comprehensive E2E Integration Guide

---

## 1. Executive Summary

The **MachPay Python Agent SDK** (`machpay-py`) enables autonomous AI agents to:

1. **Pay for API services** using the x402 protocol
2. **Emit telemetry** to the MachPay Receiver for analytics
3. **Track spending** and prevent insolvency
4. **Discover vendors** dynamically via MCP manifests

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MachPay Agent SDK Architecture                        │
└─────────────────────────────────────────────────────────────────────────────┘

                    ┌─────────────┐
                    │   AI Agent  │
                    │  (Python)   │
                    └──────┬──────┘
                           │
                           ▼
              ┌────────────────────────┐
              │   machpay-py SDK       │
              │  ┌──────────────────┐  │
              │  │ PaymentClient    │  │◄─── x402 Negotiation
              │  │ TelemetryEmitter │  │◄─── Metrics to Receiver
              │  │ BalanceTracker   │  │◄─── Solvency Checks
              │  │ VendorDiscovery  │  │◄─── MCP Search
              │  └──────────────────┘  │
              └───────────┬────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
    ┌───────────┐   ┌───────────┐   ┌───────────┐
    │  Gateway  │   │ Receiver  │   │ Blockchain│
    │  :8080    │   │  :8080    │   │  (Solana) │
    └───────────┘   └───────────┘   └───────────┘
         │               │               │
         │               ▼               │
         │        ┌───────────┐          │
         │        │ PostgreSQL│          │
         │        └───────────┘          │
         │               │               │
         └───────────────┼───────────────┘
                         ▼
                  ┌───────────┐
                  │  Console  │
                  │   (UI)    │
                  └───────────┘
```

---

## 2. Installation & Setup

### 2.1 Package Installation

```bash
# From PyPI (production)
pip install machpay

# From source (development)
git clone https://github.com/machpay/machpay-py.git
cd machpay-py
pip install -e ".[dev]"
```

### 2.2 Dependencies

```toml
# pyproject.toml
[project]
name = "machpay"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "httpx>=0.25.0",           # Async HTTP client
    "solders>=0.20.0",         # Solana crypto primitives
    "solana>=0.30.0",          # Solana RPC client
    "pydantic>=2.0",           # Data validation
    "cryptography>=41.0",      # Ed25519 signing
    "prometheus-client>=0.18", # Metrics exposition
    "structlog>=23.0",         # Structured logging
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "pytest-asyncio>=0.21",
    "pytest-httpx>=0.22",
    "ruff>=0.1.0",
]
```

### 2.3 Environment Configuration

```bash
# .env
MACHPAY_AGENT_SECRET="[64-byte-json-array]"  # Solana keypair
MACHPAY_NETWORK="devnet"                      # mainnet-beta | devnet
MACHPAY_RPC_URL="https://api.devnet.solana.com"
MACHPAY_RECEIVER_URL="https://telemetry.machpay.xyz"
MACHPAY_LOW_BALANCE_THRESHOLD="5.0"           # USD
```

---

## 3. Core SDK Architecture

### 3.1 Module Structure

```
machpay/
├── __init__.py
├── client.py           # Main MachPayClient class
├── auth/
│   ├── __init__.py
│   ├── keypair.py      # Wallet management
│   ├── signer.py       # Ed25519/EIP-712 signing
│   └── siws.py         # Sign-In With Solana
├── payment/
│   ├── __init__.py
│   ├── x402.py         # 402 negotiation logic
│   ├── intent.py       # PaymentIntent model
│   └── challenge.py    # Challenge parsing
├── telemetry/
│   ├── __init__.py
│   ├── emitter.py      # Async telemetry sender
│   ├── buffer.py       # Ring buffer for batching
│   ├── metrics.py      # Prometheus metrics
│   └── models.py       # Telemetry packet schemas
├── discovery/
│   ├── __init__.py
│   ├── mcp.py          # MCP manifest parsing
│   └── search.py       # Vendor search
├── cli/
│   ├── __init__.py
│   └── main.py         # CLI commands
├── exceptions.py       # Custom exceptions
└── config.py           # Configuration management
```

### 3.2 Main Client Interface

```python
# machpay/client.py
from __future__ import annotations

import asyncio
from typing import Optional, Callable, Any
from dataclasses import dataclass
from solders.keypair import Keypair
from solders.pubkey import Pubkey
import httpx

from machpay.auth.keypair import load_keypair
from machpay.payment.x402 import X402Negotiator
from machpay.telemetry.emitter import TelemetryEmitter
from machpay.discovery.search import VendorSearch
from machpay.config import MachPayConfig


@dataclass
class Balance:
    """Agent's current on-chain balance."""
    sol: float
    usdc: float
    runway_hours: float  # Estimated time until insolvency


@dataclass
class VendorConfig:
    """Discovered vendor information."""
    service_id: str
    authority: Pubkey
    fee_per_unit: int
    treasury_mint: Pubkey
    metadata: dict
    latency_p50: Optional[int] = None
    success_rate: Optional[float] = None


class MachPayClient:
    """
    Main entry point for the MachPay Agent SDK.
    
    Handles payment authorization, telemetry emission, and vendor discovery.
    
    Example:
        >>> client = MachPayClient.from_env()
        >>> await client.initialize()
        >>> response = await client.pay_and_call("https://api.vendor.xyz/chat", {...})
    """
    
    def __init__(self, config: MachPayConfig):
        self.config = config
        self._keypair: Optional[Keypair] = None
        self._http: Optional[httpx.AsyncClient] = None
        self._negotiator: Optional[X402Negotiator] = None
        self._telemetry: Optional[TelemetryEmitter] = None
        self._balance_cache: Optional[Balance] = None
        self._initialized = False
    
    @classmethod
    def from_env(cls) -> "MachPayClient":
        """Create client from environment variables."""
        return cls(MachPayConfig.from_env())
    
    @classmethod
    def from_file(cls, path: str) -> "MachPayClient":
        """Create client from config file."""
        return cls(MachPayConfig.from_file(path))
    
    async def initialize(self) -> None:
        """
        Initialize the client.
        
        - Loads keypair
        - Starts telemetry emitter
        - Fetches initial balance
        """
        self._keypair = load_keypair(self.config.secret_key)
        self._http = httpx.AsyncClient(timeout=30.0)
        self._negotiator = X402Negotiator(self._keypair, self._http)
        self._telemetry = TelemetryEmitter(
            agent_id=str(self._keypair.pubkey()),
            receiver_url=self.config.receiver_url,
            http=self._http,
        )
        
        # Start background telemetry flush
        await self._telemetry.start()
        
        # Fetch initial balance
        self._balance_cache = await self.get_balance()
        
        self._initialized = True
    
    async def close(self) -> None:
        """Graceful shutdown."""
        if self._telemetry:
            await self._telemetry.stop()
        if self._http:
            await self._http.aclose()
    
    @property
    def pubkey(self) -> Pubkey:
        """Agent's public key."""
        return self._keypair.pubkey()
    
    # =========================================================================
    # Payment Methods
    # =========================================================================
    
    async def pay_and_call(
        self,
        url: str,
        method: str = "POST",
        json: Optional[dict] = None,
        headers: Optional[dict] = None,
        **kwargs,
    ) -> httpx.Response:
        """
        Make an HTTP request with automatic x402 payment handling.
        
        The SDK handles the 402 negotiation loop automatically:
        1. Makes initial request
        2. If 402 received, parses challenge
        3. Signs payment intent
        4. Retries with X-MachPay-Auth header
        
        Args:
            url: Target URL (e.g., "https://gateway.vendor.xyz/v1/chat")
            method: HTTP method
            json: Request body
            headers: Additional headers
            
        Returns:
            httpx.Response with successful response
            
        Raises:
            InsufficientFundsError: Agent balance too low
            PaymentRejectedError: Gateway rejected signature
            TimeoutError: Request timed out
        """
        self._ensure_initialized()
        
        # Record attempt in telemetry
        vendor_id = self._extract_vendor_id(url)
        start_time = asyncio.get_event_loop().time()
        
        try:
            response = await self._negotiator.authorize_request(
                url=url,
                method=method,
                json=json,
                headers=headers,
                **kwargs,
            )
            
            # Record success
            latency_ms = int((asyncio.get_event_loop().time() - start_time) * 1000)
            self._telemetry.record_interaction(
                vendor_id=vendor_id,
                success=True,
                latency_ms=latency_ms,
                cost=self._negotiator.last_cost,
            )
            
            return response
            
        except Exception as e:
            # Record failure
            self._telemetry.record_interaction(
                vendor_id=vendor_id,
                success=False,
                error_type=type(e).__name__,
            )
            raise
    
    async def sign_payment(
        self,
        gateway: Pubkey,
        amount: int,
        nonce: int,
        deadline: int,
    ) -> bytes:
        """
        Sign a PaymentIntent manually.
        
        Use this for custom integrations where you handle the HTTP yourself.
        
        Args:
            gateway: Gateway's public key
            amount: Payment amount in atomic units
            nonce: Challenge nonce
            deadline: Unix timestamp deadline
            
        Returns:
            Signature bytes (64 bytes Ed25519)
        """
        self._ensure_initialized()
        return self._negotiator.sign_intent(gateway, amount, nonce, deadline)
    
    # =========================================================================
    # Balance & Solvency
    # =========================================================================
    
    async def get_balance(self) -> Balance:
        """
        Get agent's current on-chain balance.
        
        Returns:
            Balance with SOL, USDC, and estimated runway
        """
        from solana.rpc.async_api import AsyncClient
        
        async with AsyncClient(self.config.rpc_url) as rpc:
            # Get SOL balance
            sol_resp = await rpc.get_balance(self.pubkey)
            sol_lamports = sol_resp.value
            
            # Get USDC balance (SPL token)
            # TODO: Implement SPL token balance lookup
            usdc_atomic = 0  # Placeholder
            
        sol = sol_lamports / 1e9
        usdc = usdc_atomic / 1e6
        
        # Calculate runway based on burn rate
        # TODO: Track spending rate in telemetry
        runway = usdc / 0.10 if usdc > 0 else 0  # Assume $0.10/hour burn
        
        self._balance_cache = Balance(sol=sol, usdc=usdc, runway_hours=runway)
        return self._balance_cache
    
    def check_solvency(self, required_amount: int) -> bool:
        """
        Check if agent can afford a payment.
        
        Uses cached balance for speed (no RPC call).
        
        Args:
            required_amount: Amount in atomic units
            
        Returns:
            True if agent has sufficient funds
        """
        if not self._balance_cache:
            return False
        return self._balance_cache.usdc * 1e6 >= required_amount
    
    # =========================================================================
    # Vendor Discovery
    # =========================================================================
    
    async def resolve_vendor(self, service_id: str) -> VendorConfig:
        """
        Look up a vendor by their service ID.
        
        Args:
            service_id: e.g., "openai-gpt4-turbo"
            
        Returns:
            VendorConfig with pricing and metadata
        """
        self._ensure_initialized()
        return await VendorSearch(self._http).resolve(service_id)
    
    async def search_vendors(
        self,
        query: str,
        max_price: Optional[float] = None,
        min_uptime: Optional[float] = None,
    ) -> list[VendorConfig]:
        """
        Semantic search for vendors.
        
        Args:
            query: Natural language query (e.g., "cheap LLM API")
            max_price: Maximum price per request in USD
            min_uptime: Minimum uptime percentage (0-100)
            
        Returns:
            List of matching vendors, sorted by relevance
        """
        self._ensure_initialized()
        return await VendorSearch(self._http).search(
            query=query,
            max_price=max_price,
            min_uptime=min_uptime,
        )
    
    # =========================================================================
    # Console Linking
    # =========================================================================
    
    async def link(self) -> str:
        """
        Generate a pairing URL to link agent to MachPay Console.
        
        Returns:
            URL to open in browser (e.g., "https://console.machpay.xyz/link?code=ABC123")
        """
        from machpay.auth.siws import generate_link_code
        
        code = await generate_link_code(self.pubkey, self._http)
        url = f"{self.config.console_url}/link?code={code}"
        
        print(f"🔗 Link your agent: {url}")
        return url
    
    # =========================================================================
    # Internal Helpers
    # =========================================================================
    
    def _ensure_initialized(self) -> None:
        if not self._initialized:
            raise RuntimeError("Client not initialized. Call await client.initialize() first.")
    
    def _extract_vendor_id(self, url: str) -> str:
        """Extract vendor identifier from URL for telemetry grouping."""
        from urllib.parse import urlparse
        parsed = urlparse(url)
        return parsed.netloc
    
    async def __aenter__(self) -> "MachPayClient":
        await self.initialize()
        return self
    
    async def __aexit__(self, *args) -> None:
        await self.close()
```

---

## 4. x402 Payment Flow

### 4.1 Negotiation Logic

```python
# machpay/payment/x402.py
from __future__ import annotations

import struct
from dataclasses import dataclass
from typing import Optional
from solders.keypair import Keypair
from solders.pubkey import Pubkey
from solders.signature import Signature
import httpx

from machpay.payment.intent import PaymentIntent
from machpay.payment.challenge import PaymentChallenge
from machpay.exceptions import (
    PaymentRequiredError,
    InsufficientFundsError,
    PaymentRejectedError,
)


@dataclass
class X402Negotiator:
    """
    Handles the x402 payment negotiation loop.
    
    The x402 protocol:
    1. Agent requests resource
    2. Gateway returns 402 with challenge
    3. Agent signs PaymentIntent
    4. Agent retries with X-MachPay-Auth header
    5. Gateway validates and serves resource
    """
    
    keypair: Keypair
    http: httpx.AsyncClient
    last_cost: Optional[int] = None
    
    async def authorize_request(
        self,
        url: str,
        method: str = "POST",
        json: Optional[dict] = None,
        headers: Optional[dict] = None,
        max_retries: int = 3,
        **kwargs,
    ) -> httpx.Response:
        """
        Execute HTTP request with automatic 402 handling.
        
        Returns the successful response after payment.
        """
        headers = headers or {}
        
        for attempt in range(max_retries):
            response = await self.http.request(
                method=method,
                url=url,
                json=json,
                headers=headers,
                **kwargs,
            )
            
            if response.status_code == 402:
                # Parse challenge
                challenge = PaymentChallenge.from_response(response)
                
                # Sign intent
                auth_header = self._sign_challenge(challenge)
                
                # Store cost for telemetry
                self.last_cost = challenge.cost
                
                # Retry with auth
                headers["X-MachPay-Auth"] = auth_header
                continue
            
            elif response.status_code == 401:
                raise PaymentRejectedError("Gateway rejected payment signature")
            
            elif response.is_success:
                return response
            
            else:
                response.raise_for_status()
        
        raise PaymentRequiredError("Max retries exceeded for 402 negotiation")
    
    def _sign_challenge(self, challenge: PaymentChallenge) -> str:
        """
        Sign a payment challenge and format the auth header.
        
        Header format:
            X-MachPay-Auth: sig=<base58>;agent=<pubkey>;nonce=<nonce>
        """
        intent = PaymentIntent(
            agent=self.keypair.pubkey(),
            gateway=challenge.gateway_id,
            mint=challenge.mint,
            amount=challenge.cost,
            nonce=challenge.nonce,
            deadline=challenge.deadline,
        )
        
        signature = self.sign_intent(
            gateway=challenge.gateway_id,
            amount=challenge.cost,
            nonce=challenge.nonce,
            deadline=challenge.deadline,
        )
        
        # Format header
        sig_b58 = str(Signature.from_bytes(signature))
        agent_b58 = str(self.keypair.pubkey())
        
        return f"sig={sig_b58};agent={agent_b58};nonce={challenge.nonce}"
    
    def sign_intent(
        self,
        gateway: Pubkey,
        amount: int,
        nonce: int,
        deadline: int,
    ) -> bytes:
        """
        Sign a PaymentIntent using Ed25519.
        
        The message format (Borsh-compatible):
        - discriminator: u8 (0x01)
        - agent: [u8; 32]
        - gateway: [u8; 32]
        - amount: u64
        - nonce: u64
        - deadline: i64
        """
        # Build message bytes
        message = struct.pack(
            "<B32s32sQQq",
            0x01,  # discriminator
            bytes(self.keypair.pubkey()),
            bytes(gateway),
            amount,
            nonce,
            deadline,
        )
        
        # Sign with Ed25519
        signature = self.keypair.sign_message(message)
        return bytes(signature)
```

### 4.2 Challenge Model

```python
# machpay/payment/challenge.py
from dataclasses import dataclass
from solders.pubkey import Pubkey
import httpx


@dataclass
class PaymentChallenge:
    """Parsed 402 Payment Required response."""
    
    gateway_id: Pubkey
    cost: int              # Atomic units
    mint: Pubkey           # Token mint address
    nonce: int             # Unique challenge nonce
    deadline: int          # Unix timestamp
    service_id: str        # Vendor service identifier
    
    @classmethod
    def from_response(cls, response: httpx.Response) -> "PaymentChallenge":
        """Parse challenge from 402 response."""
        data = response.json()
        
        # Support both header-based and body-based challenges
        if "params" in data:
            params = data["params"]
        else:
            params = data.get("challenge", data)
        
        return cls(
            gateway_id=Pubkey.from_string(params["gateway_id"]),
            cost=int(params["cost"]),
            mint=Pubkey.from_string(params.get("mint", params.get("token"))),
            nonce=int(params["nonce"]),
            deadline=int(params.get("deadline", 0)),
            service_id=params.get("service_id", "unknown"),
        )
```

---

## 5. Telemetry Emission

### 5.1 Telemetry Emitter

```python
# machpay/telemetry/emitter.py
from __future__ import annotations

import asyncio
import time
from dataclasses import dataclass, field
from typing import Optional
import httpx
import structlog

from machpay.telemetry.buffer import RingBuffer
from machpay.telemetry.models import (
    TelemetryPacket,
    InteractionMetrics,
    WalletSnapshot,
)

logger = structlog.get_logger()


@dataclass
class TelemetryEmitter:
    """
    Async telemetry emitter with local aggregation.
    
    Collects interaction metrics locally and batches them
    every `flush_interval` seconds to the receiver.
    
    Metrics collected per vendor:
    - Request count (success/failure)
    - Latency percentiles (p50, p99)
    - Spend amount
    - Error types
    """
    
    agent_id: str
    receiver_url: str
    http: httpx.AsyncClient
    flush_interval: float = 60.0  # seconds
    
    _buffer: RingBuffer = field(default_factory=lambda: RingBuffer(1000))
    _task: Optional[asyncio.Task] = None
    _running: bool = False
    _interactions: dict = field(default_factory=dict)
    _period_start: float = field(default_factory=time.time)
    
    async def start(self) -> None:
        """Start background flush loop."""
        self._running = True
        self._period_start = time.time()
        self._task = asyncio.create_task(self._flush_loop())
        logger.info("telemetry_emitter_started", agent_id=self.agent_id)
    
    async def stop(self) -> None:
        """Stop and flush remaining data."""
        self._running = False
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        
        # Final flush
        await self._flush()
        logger.info("telemetry_emitter_stopped", agent_id=self.agent_id)
    
    def record_interaction(
        self,
        vendor_id: str,
        success: bool,
        latency_ms: int = 0,
        cost: int = 0,
        error_type: Optional[str] = None,
    ) -> None:
        """
        Record a single interaction for aggregation.
        
        Called after each pay_and_call() completes.
        """
        if vendor_id not in self._interactions:
            self._interactions[vendor_id] = InteractionMetrics(vendor_id=vendor_id)
        
        metrics = self._interactions[vendor_id]
        metrics.attempts += 1
        
        if success:
            metrics.successes += 1
            metrics.latencies.append(latency_ms)
            metrics.spend_atomic += cost
        else:
            metrics.failures += 1
            if error_type:
                metrics.errors[error_type] = metrics.errors.get(error_type, 0) + 1
    
    async def _flush_loop(self) -> None:
        """Background loop that flushes telemetry periodically."""
        while self._running:
            try:
                await asyncio.sleep(self.flush_interval)
                await self._flush()
            except asyncio.CancelledError:
                break
            except Exception as e:
                logger.error("telemetry_flush_error", error=str(e))
    
    async def _flush(self) -> None:
        """Build and send telemetry packet."""
        if not self._interactions:
            return
        
        period_end = time.time()
        
        # Build packet
        packet = TelemetryPacket(
            source="sdk",
            agent_id=self.agent_id,
            period_start=int(self._period_start),
            period_end=int(period_end),
            interactions={
                vid: self._aggregate_metrics(m)
                for vid, m in self._interactions.items()
            },
            wallet=await self._get_wallet_snapshot(),
        )
        
        # Send to receiver
        try:
            response = await self.http.post(
                f"{self.receiver_url}/v1/telemetry",
                json=packet.to_dict(),
                headers={
                    "Content-Type": "application/json",
                    "X-MachPay-Agent": self.agent_id,
                },
            )
            
            if response.is_success:
                logger.info(
                    "telemetry_flushed",
                    vendors=len(self._interactions),
                    period_seconds=period_end - self._period_start,
                )
            else:
                logger.warn(
                    "telemetry_flush_rejected",
                    status=response.status_code,
                )
        except Exception as e:
            logger.error("telemetry_send_error", error=str(e))
        
        # Reset for next period
        self._interactions.clear()
        self._period_start = period_end
    
    def _aggregate_metrics(self, m: InteractionMetrics) -> dict:
        """Compute percentiles and format for transmission."""
        latencies = sorted(m.latencies) if m.latencies else [0]
        
        return {
            "attempts": m.attempts,
            "success": m.successes,
            "failures": m.failures,
            "payment_latency_p50": self._percentile(latencies, 50),
            "payment_latency_p99": self._percentile(latencies, 99),
            "spend_atomic": m.spend_atomic,
            "errors": m.errors,
        }
    
    @staticmethod
    def _percentile(data: list[int], p: int) -> int:
        """Calculate percentile from sorted list."""
        if not data:
            return 0
        k = (len(data) - 1) * p / 100
        f = int(k)
        c = f + 1 if f < len(data) - 1 else f
        return int(data[f] + (k - f) * (data[c] - data[f]))
    
    async def _get_wallet_snapshot(self) -> WalletSnapshot:
        """Get current wallet balance for telemetry."""
        # TODO: Implement actual balance fetch
        return WalletSnapshot(
            sol_balance_lamports=0,
            usdc_balance_atomic=0,
            low_balance_warnings=0,
        )
```

### 5.2 Telemetry Models

```python
# machpay/telemetry/models.py
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class InteractionMetrics:
    """Per-vendor interaction metrics (in-memory)."""
    vendor_id: str
    attempts: int = 0
    successes: int = 0
    failures: int = 0
    latencies: list[int] = field(default_factory=list)
    spend_atomic: int = 0
    errors: dict[str, int] = field(default_factory=dict)


@dataclass
class WalletSnapshot:
    """Agent wallet state at flush time."""
    sol_balance_lamports: int
    usdc_balance_atomic: int
    low_balance_warnings: int


@dataclass
class TelemetryPacket:
    """
    SDK Telemetry Packet sent to MachPay Receiver.
    
    Matches the schema expected by /v1/telemetry endpoint.
    """
    source: str  # "sdk"
    agent_id: str
    period_start: int
    period_end: int
    interactions: dict[str, dict]
    wallet: WalletSnapshot
    
    def to_dict(self) -> dict:
        """Convert to JSON-serializable dict."""
        return {
            "specversion": "1.0",
            "type": "agent.telemetry",
            "source": f"agent:{self.agent_id}",
            "id": f"{self.period_start}-{self.agent_id[:8]}",
            "time": self._timestamp(self.period_end),
            "datacontenttype": "application/json",
            "data": {
                "source": self.source,
                "agent_id": self.agent_id,
                "period_start": self.period_start,
                "period_end": self.period_end,
                "wallet": {
                    "sol_balance_lamports": self.wallet.sol_balance_lamports,
                    "usdc_balance_atomic": self.wallet.usdc_balance_atomic,
                    "low_balance_warnings": self.wallet.low_balance_warnings,
                },
                "interactions": self.interactions,
            },
        }
    
    @staticmethod
    def _timestamp(unix: int) -> str:
        from datetime import datetime, timezone
        return datetime.fromtimestamp(unix, tz=timezone.utc).isoformat()
```

---

## 6. Metrics Collected

### 6.1 Agent-Level Metrics

| Metric | Type | Unit | Description |
|--------|------|------|-------------|
| `machpay_agent_requests_total` | Counter | count | Total API requests made |
| `machpay_agent_payments_total` | Counter | count | Total payments signed |
| `machpay_agent_spend_total` | Counter | atomic | Total amount spent |
| `machpay_agent_latency_seconds` | Histogram | seconds | Payment negotiation latency |
| `machpay_agent_errors_total` | Counter | count | Payment/request errors by type |
| `machpay_agent_balance_usdc` | Gauge | USD | Current USDC balance |
| `machpay_agent_runway_hours` | Gauge | hours | Estimated time until insolvency |

### 6.2 Per-Vendor Metrics (in telemetry packet)

| Field | Type | Description |
|-------|------|-------------|
| `attempts` | int | Total requests to this vendor |
| `success` | int | Successful requests |
| `failures` | int | Failed requests |
| `payment_latency_p50` | int | 50th percentile latency (ms) |
| `payment_latency_p99` | int | 99th percentile latency (ms) |
| `spend_atomic` | int | Total spent on this vendor |
| `errors` | dict | Error breakdown by type |

### 6.3 Prometheus Exposition

```python
# machpay/telemetry/metrics.py
from prometheus_client import Counter, Histogram, Gauge

# Request metrics
REQUESTS_TOTAL = Counter(
    "machpay_agent_requests_total",
    "Total API requests made",
    ["vendor", "status"],
)

PAYMENTS_TOTAL = Counter(
    "machpay_agent_payments_total",
    "Total payments signed",
    ["vendor"],
)

SPEND_TOTAL = Counter(
    "machpay_agent_spend_total",
    "Total amount spent (atomic units)",
    ["vendor", "mint"],
)

# Latency
LATENCY = Histogram(
    "machpay_agent_latency_seconds",
    "Payment negotiation latency",
    ["vendor"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
)

# Balance
BALANCE_USDC = Gauge(
    "machpay_agent_balance_usdc",
    "Current USDC balance",
)

RUNWAY_HOURS = Gauge(
    "machpay_agent_runway_hours",
    "Estimated hours until insolvency",
)

# Errors
ERRORS_TOTAL = Counter(
    "machpay_agent_errors_total",
    "Payment/request errors",
    ["vendor", "error_type"],
)
```

---

## 7. CLI Commands

### 7.1 Command Reference

```bash
# Identity management
machpay init                    # Generate new keypair
machpay link                    # Link to MachPay Console
machpay whoami                  # Show agent public key

# Balance & funding
machpay balance                 # Check SOL/USDC balance
machpay fund <amount>           # Request devnet airdrop (devnet only)
machpay runway                  # Show estimated runway

# Vendor discovery
machpay search "cheap LLM"      # Search for vendors
machpay info <service_id>       # Get vendor details
machpay test <service_id>       # Make test request

# Telemetry
machpay logs                    # Stream telemetry logs from Console
machpay stats                   # Show spending statistics

# Config
machpay config show             # Display current config
machpay config set <key> <val>  # Update config
```

### 7.2 CLI Implementation

```python
# machpay/cli/main.py
import asyncio
import click
from machpay.client import MachPayClient


@click.group()
def cli():
    """MachPay Agent SDK CLI"""
    pass


@cli.command()
def init():
    """Generate a new agent keypair."""
    from solders.keypair import Keypair
    import json
    import os
    
    keypair = Keypair()
    secret = list(bytes(keypair))
    
    # Save to default location
    config_dir = os.path.expanduser("~/.config/machpay")
    os.makedirs(config_dir, exist_ok=True)
    
    secret_path = os.path.join(config_dir, "agent-keypair.json")
    with open(secret_path, "w") as f:
        json.dump(secret, f)
    
    click.echo(f"✅ Generated new agent keypair")
    click.echo(f"   Public Key: {keypair.pubkey()}")
    click.echo(f"   Saved to: {secret_path}")
    click.echo("")
    click.echo("Next steps:")
    click.echo("  1. Fund your agent with SOL and USDC")
    click.echo("  2. Run 'machpay link' to connect to Console")


@cli.command()
def link():
    """Link agent to MachPay Console."""
    async def _link():
        async with MachPayClient.from_env() as client:
            url = await client.link()
            click.echo(f"\n🔗 Open this URL to link your agent:\n   {url}\n")
    
    asyncio.run(_link())


@cli.command()
def balance():
    """Check agent balance."""
    async def _balance():
        async with MachPayClient.from_env() as client:
            bal = await client.get_balance()
            click.echo(f"💰 Agent Balance:")
            click.echo(f"   SOL:  {bal.sol:.4f}")
            click.echo(f"   USDC: ${bal.usdc:.2f}")
            click.echo(f"   Runway: {bal.runway_hours:.1f} hours")
    
    asyncio.run(_balance())


@cli.command()
@click.argument("query")
def search(query: str):
    """Search for vendors."""
    async def _search():
        async with MachPayClient.from_env() as client:
            results = await client.search_vendors(query)
            click.echo(f"🔍 Found {len(results)} vendors:\n")
            for v in results[:5]:
                click.echo(f"   {v.service_id}")
                click.echo(f"      Price: {v.fee_per_unit} units")
                click.echo(f"      Latency P50: {v.latency_p50}ms")
                click.echo("")
    
    asyncio.run(_search())


if __name__ == "__main__":
    cli()
```

---

## 8. E2E Integration Example

### 8.1 Complete Agent Example

```python
# examples/openai_agent.py
"""
Example: AI Agent that uses MachPay to pay for OpenAI API calls.

This demonstrates the full integration flow:
1. Initialize SDK with wallet
2. Search for OpenAI vendor
3. Make paid API calls
4. Telemetry automatically sent to receiver
"""
import asyncio
from machpay import MachPayClient


async def main():
    # Initialize client from environment
    async with MachPayClient.from_env() as client:
        
        # 1. Check balance first
        balance = await client.get_balance()
        print(f"Agent balance: ${balance.usdc:.2f} USDC")
        print(f"Estimated runway: {balance.runway_hours:.1f} hours")
        
        if balance.usdc < 1.0:
            print("⚠️  Low balance! Please fund your agent.")
            return
        
        # 2. Discover OpenAI vendor
        vendors = await client.search_vendors(
            "OpenAI GPT-4 API",
            max_price=0.05,
        )
        
        if not vendors:
            print("No vendors found!")
            return
        
        vendor = vendors[0]
        print(f"\n✅ Found vendor: {vendor.service_id}")
        print(f"   Price: ${vendor.fee_per_unit / 1e6:.4f} per request")
        
        # 3. Make a paid request (402 negotiation handled automatically)
        print("\n📤 Making paid API call...")
        
        response = await client.pay_and_call(
            url=f"https://gateway.{vendor.service_id}/v1/chat/completions",
            method="POST",
            json={
                "model": "gpt-4",
                "messages": [
                    {"role": "user", "content": "What is MachPay?"}
                ],
                "max_tokens": 100,
            },
        )
        
        if response.is_success:
            data = response.json()
            print(f"\n📥 Response:")
            print(data["choices"][0]["message"]["content"])
        else:
            print(f"❌ Error: {response.status_code}")
        
        # 4. Check updated balance
        new_balance = await client.get_balance()
        spent = balance.usdc - new_balance.usdc
        print(f"\n💸 Spent: ${spent:.4f}")
        print(f"📊 Telemetry will be sent to receiver in ~60 seconds")


if __name__ == "__main__":
    asyncio.run(main())
```

### 8.2 MCP Tool Integration

```python
# examples/mcp_tool.py
"""
Example: Expose a MachPay-enabled tool for ChatGPT/Claude via MCP.
"""
from machpay import MachPayClient
from machpay.mcp import PaidTool


# Global client (initialized once)
client: MachPayClient = None


async def initialize():
    global client
    client = MachPayClient.from_env()
    await client.initialize()


@PaidTool(
    name="premium_weather",
    description="High-resolution weather forecast with proprietary data",
    price=0.05,
    currency="USDC",
)
async def premium_weather(city: str) -> dict:
    """
    Get premium weather data for a city.
    
    The @PaidTool decorator handles:
    - x402 negotiation
    - Payment signing
    - Error handling
    - Telemetry emission
    """
    response = await client.pay_and_call(
        url="https://gateway.weather.pro/v1/forecast",
        method="GET",
        params={"city": city, "days": 7},
    )
    
    return response.json()


# MCP Manifest (generated automatically)
MANIFEST = {
    "name": "premium_weather",
    "description": "High-resolution weather forecast with proprietary data",
    "pricing": {
        "price": "0.05",
        "currency": "USDC",
        "payment_provider": "machpay",
    },
    "inputs": [
        {"name": "city", "type": "string", "required": True}
    ],
}
```

---

## 9. Database Schema for Agent Telemetry

The Receiver will store agent telemetry in a dedicated table:

```sql
-- Agent telemetry (stored by receiver)
CREATE TABLE IF NOT EXISTS agent_telemetry (
    time            TIMESTAMPTZ NOT NULL,
    agent_id        VARCHAR(64) NOT NULL,
    
    -- Wallet state
    sol_balance_lamports    BIGINT,
    usdc_balance_atomic     BIGINT,
    low_balance_warnings    INT,
    
    -- Aggregated metrics
    period_seconds          INT,
    total_requests          INT,
    total_payments          INT,
    total_spend_atomic      BIGINT,
    
    -- Per-vendor JSONB (denormalized for flexibility)
    interactions            JSONB,
    
    PRIMARY KEY (time, agent_id)
) PARTITION BY RANGE (time);

-- Create index for agent lookups
CREATE INDEX idx_agent_telemetry_agent_id ON agent_telemetry (agent_id, time DESC);

-- Sample interactions JSONB structure:
-- {
--   "openai-gpt4": {
--     "attempts": 100,
--     "success": 98,
--     "payment_latency_p99": 45,
--     "spend_atomic": 500000,
--     "errors": {"timeout": 2}
--   }
-- }
```

---

## 10. Testing Strategy

### 10.1 Unit Tests

```python
# tests/test_x402.py
import pytest
from unittest.mock import AsyncMock
from machpay.payment.x402 import X402Negotiator
from machpay.payment.challenge import PaymentChallenge


@pytest.fixture
def mock_keypair():
    from solders.keypair import Keypair
    return Keypair()


@pytest.fixture
def mock_http():
    return AsyncMock()


@pytest.mark.asyncio
async def test_402_negotiation_success(mock_keypair, mock_http):
    """Test successful 402 -> 200 flow."""
    negotiator = X402Negotiator(mock_keypair, mock_http)
    
    # First call returns 402
    challenge_response = AsyncMock()
    challenge_response.status_code = 402
    challenge_response.json.return_value = {
        "params": {
            "gateway_id": "11111111111111111111111111111111",
            "cost": 1000,
            "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
            "nonce": 12345,
            "deadline": 9999999999,
        }
    }
    
    # Second call returns 200
    success_response = AsyncMock()
    success_response.status_code = 200
    success_response.is_success = True
    
    mock_http.request.side_effect = [challenge_response, success_response]
    
    result = await negotiator.authorize_request(
        url="https://test.com/api",
        method="GET",
    )
    
    assert result.status_code == 200
    assert mock_http.request.call_count == 2
```

### 10.2 E2E Tests

```python
# tests/e2e/test_full_flow.py
import pytest
import asyncio
from machpay import MachPayClient


@pytest.fixture
async def client():
    """Create test client with devnet config."""
    client = MachPayClient.from_env()
    await client.initialize()
    yield client
    await client.close()


@pytest.mark.e2e
@pytest.mark.asyncio
async def test_full_payment_flow(client):
    """
    E2E test: Make a real paid request on devnet.
    
    Requirements:
    - Devnet RPC access
    - Funded test wallet
    - Test gateway running
    """
    # Check balance
    balance = await client.get_balance()
    assert balance.usdc > 0, "Test wallet needs USDC"
    
    # Make paid request
    response = await client.pay_and_call(
        url="https://gateway.test.machpay.xyz/v1/echo",
        method="POST",
        json={"test": True},
    )
    
    assert response.status_code == 200
    
    # Verify telemetry was recorded
    # (Check receiver DB or metrics)
```

---

## 11. Deployment Checklist

### 11.1 Agent Operator Checklist

- [ ] Install SDK: `pip install machpay`
- [ ] Generate keypair: `machpay init`
- [ ] Fund wallet with SOL (gas) and USDC (payments)
- [ ] Link to Console: `machpay link`
- [ ] Configure low balance alerts in Console
- [ ] Set environment variables in production
- [ ] Monitor balance with `machpay balance` or Console dashboard

### 11.2 Production Configuration

```yaml
# production.yaml
network: mainnet-beta
rpc_url: https://api.mainnet-beta.solana.com
receiver_url: https://telemetry.machpay.xyz
console_url: https://console.machpay.xyz

# Safety limits
low_balance_threshold: 10.0  # USD
max_payment_amount: 1.0      # USD per request
daily_spend_limit: 100.0     # USD

# Telemetry
flush_interval: 60           # seconds
enable_prometheus: true
prometheus_port: 9090
```

---

## 12. Comparison: Gateway vs Agent SDK

| Aspect | Gateway (Go) | Agent SDK (Python) |
|--------|--------------|-------------------|
| **Role** | Vendor proxy | Agent client |
| **Direction** | Receives payments | Makes payments |
| **Protocol** | Validates x402 | Initiates x402 |
| **Telemetry** | Emits business metrics | Emits agent metrics |
| **Balance** | N/A | Tracks solvency |
| **Discovery** | N/A | Searches vendors |
| **Settlement** | Queues for relayer | N/A |

---

## 13. Next Steps

### Phase 1: Core SDK (Week 1-2)
- [ ] Implement `MachPayClient` class
- [ ] Implement `X402Negotiator`
- [ ] Implement `TelemetryEmitter`
- [ ] Unit tests for all components

### Phase 2: CLI & Tooling (Week 2-3)
- [ ] Implement CLI commands
- [ ] Add Prometheus metrics
- [ ] Create Docker dev environment
- [ ] Documentation and examples

### Phase 3: E2E Integration (Week 3-4)
- [ ] Integration tests with devnet
- [ ] Test with real Gateway
- [ ] Verify telemetry in Receiver
- [ ] Console link flow testing

### Phase 4: Production Hardening (Week 4+)
- [ ] Mainnet configuration
- [ ] Rate limiting and retries
- [ ] Error recovery strategies
- [ ] Performance optimization

---

*Document Version: 1.0*
*Last Updated: December 2024*
*Author: MachPay Engineering*


