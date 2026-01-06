# Error Handling Best Practices

Handle rate limits, retries, and errors gracefully with production-ready patterns. This guide covers common error scenarios and how to handle them.

**Time:** 7 minutes

## Error Types

MachPay uses standard HTTP status codes with detailed error bodies:

| Code | Type | Description |
|------|------|-------------|
| 400 | `ValidationError` | Invalid input |
| 401 | `AuthenticationError` | Invalid or missing API key |
| 403 | `AuthorizationError` | Insufficient permissions |
| 404 | `NotFoundError` | Skill or resource not found |
| 429 | `RateLimitError` | Too many requests |
| 500 | `InternalError` | Server error |
| 503 | `SkillUnavailable` | Skill temporarily unavailable |

## Error Response Format

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Request rate limit exceeded",
    "type": "RateLimitError",
    "retry_after": 30,
    "request_id": "req_abc123"
  }
}
```

## Basic Error Handling

### Python

```python
from machpay import MachPay
from machpay.exceptions import (
    MachPayError,
    AuthenticationError,
    RateLimitError,
    SkillError,
    ValidationError,
    InsufficientBalanceError,
)

client = MachPay()

try:
    response = client.skills.execute(
        skill_id="text-summarizer",
        input={"text": "..."}
    )
    print(response.data)
    
except AuthenticationError:
    print("Invalid API key. Check your credentials.")
    
except RateLimitError as e:
    print(f"Rate limited. Retry after {e.retry_after} seconds.")
    
except InsufficientBalanceError as e:
    print(f"Insufficient balance. Current: ${e.current_balance}")
    
except ValidationError as e:
    print(f"Invalid input: {e.message}")
    print(f"Details: {e.details}")
    
except SkillError as e:
    print(f"Skill execution failed: {e.message}")
    
except MachPayError as e:
    print(f"Unexpected error: {e}")
```

### Node.js

```typescript
import { 
  MachPay,
  MachPayError,
  AuthenticationError,
  RateLimitError,
  SkillError,
  ValidationError,
  InsufficientBalanceError,
} from '@machpay/sdk';

const client = new MachPay();

try {
  const response = await client.skills.execute({
    skillId: 'text-summarizer',
    input: { text: '...' }
  });
  console.log(response.data);
  
} catch (error) {
  if (error instanceof AuthenticationError) {
    console.log('Invalid API key. Check your credentials.');
    
  } else if (error instanceof RateLimitError) {
    console.log(`Rate limited. Retry after ${error.retryAfter} seconds.`);
    
  } else if (error instanceof InsufficientBalanceError) {
    console.log(`Insufficient balance. Current: $${error.currentBalance}`);
    
  } else if (error instanceof ValidationError) {
    console.log(`Invalid input: ${error.message}`);
    console.log(`Details:`, error.details);
    
  } else if (error instanceof SkillError) {
    console.log(`Skill execution failed: ${error.message}`);
    
  } else if (error instanceof MachPayError) {
    console.log(`Unexpected error: ${error}`);
  }
}
```

## Automatic Retries

Configure automatic retries for transient failures:

### Python

```python
from machpay import MachPay
from machpay.config import RetryConfig

client = MachPay(
    retry_config=RetryConfig(
        max_retries=3,
        backoff_factor=2.0,       # Exponential backoff multiplier
        initial_delay=1.0,        # Initial delay in seconds
        max_delay=60.0,           # Maximum delay cap
        retry_on=[429, 500, 502, 503, 504],  # Status codes to retry
        retry_on_timeout=True,    # Retry on timeout errors
    )
)
```

### Node.js

```typescript
const client = new MachPay({
  retryConfig: {
    maxRetries: 3,
    backoffFactor: 2.0,
    initialDelay: 1000,     // ms
    maxDelay: 60000,        // ms
    retryOn: [429, 500, 502, 503, 504],
    retryOnTimeout: true,
  }
});
```

## Manual Retry with Backoff

```python
import time
from machpay import MachPay
from machpay.exceptions import RateLimitError, MachPayError

def execute_with_retry(client, skill_id, input, max_retries=3):
    delay = 1.0
    
    for attempt in range(max_retries):
        try:
            return client.skills.execute(skill_id=skill_id, input=input)
            
        except RateLimitError as e:
            # Use server-provided retry delay if available
            wait_time = e.retry_after or delay
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
            delay *= 2  # Exponential backoff
            
        except MachPayError as e:
            if e.is_retryable and attempt < max_retries - 1:
                print(f"Retrying in {delay}s... ({attempt + 1}/{max_retries})")
                time.sleep(delay)
                delay *= 2
            else:
                raise
    
    raise Exception("Max retries exceeded")
```

## Rate Limit Handling

### Understanding Rate Limits

| Tier | Requests/min | Burst | Concurrent |
|------|--------------|-------|------------|
| Free | 60 | 10 | 5 |
| Standard | 1,000 | 100 | 50 |
| Enterprise | 10,000 | 1,000 | 500 |

### Rate Limit Headers

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1704067200
```

### Proactive Rate Limiting

```python
from machpay import MachPay
import time

client = MachPay()

def execute_with_rate_awareness(skill_id, inputs):
    for i, input_data in enumerate(inputs):
        response = client.skills.execute(skill_id=skill_id, input=input_data)
        
        # Check rate limit headers
        remaining = response.headers.get('X-RateLimit-Remaining', 100)
        
        if int(remaining) < 10:
            # Slow down when approaching limit
            print("Approaching rate limit, slowing down...")
            time.sleep(1)
        
        yield response
```

### Token Bucket Pattern

```python
import time
from threading import Lock

class RateLimiter:
    def __init__(self, tokens_per_second: float, max_tokens: float):
        self.tokens_per_second = tokens_per_second
        self.max_tokens = max_tokens
        self.tokens = max_tokens
        self.last_update = time.monotonic()
        self.lock = Lock()
    
    def acquire(self):
        with self.lock:
            now = time.monotonic()
            elapsed = now - self.last_update
            self.tokens = min(
                self.max_tokens,
                self.tokens + elapsed * self.tokens_per_second
            )
            self.last_update = now
            
            if self.tokens >= 1:
                self.tokens -= 1
                return True
            
            # Calculate wait time
            wait = (1 - self.tokens) / self.tokens_per_second
            time.sleep(wait)
            self.tokens = 0
            return True

# Usage
limiter = RateLimiter(tokens_per_second=10, max_tokens=100)

for request in requests:
    limiter.acquire()
    client.skills.execute(...)
```

## Timeout Configuration

```python
from machpay import MachPay

# Global timeout
client = MachPay(timeout=30)  # 30 seconds

# Per-request timeout
response = client.skills.execute(
    skill_id="slow-skill",
    input={"data": "..."},
    timeout=60  # Override for this request
)
```

### Handling Timeouts

```python
from machpay.exceptions import TimeoutError

try:
    response = client.skills.execute(
        skill_id="slow-skill",
        input={"data": "..."},
        timeout=10
    )
except TimeoutError:
    print("Request timed out. The skill may still be processing.")
    # Consider using idempotency keys for retry
```

## Idempotency

Prevent duplicate operations when retrying:

```python
import uuid

# Generate idempotency key
idempotency_key = str(uuid.uuid4())

# First attempt
try:
    response = client.skills.execute(
        skill_id="payment-processor",
        input={"amount": 100},
        idempotency_key=idempotency_key
    )
except MachPayError:
    # Safe to retry - same key returns same result
    response = client.skills.execute(
        skill_id="payment-processor",
        input={"amount": 100},
        idempotency_key=idempotency_key
    )
```

## Circuit Breaker Pattern

Prevent cascade failures:

```python
from datetime import datetime, timedelta

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failures = 0
        self.last_failure = None
        self.state = "closed"  # closed, open, half-open
    
    def call(self, func, *args, **kwargs):
        if self.state == "open":
            if datetime.now() - self.last_failure > timedelta(seconds=self.recovery_timeout):
                self.state = "half-open"
            else:
                raise Exception("Circuit breaker is open")
        
        try:
            result = func(*args, **kwargs)
            if self.state == "half-open":
                self.state = "closed"
                self.failures = 0
            return result
            
        except Exception as e:
            self.failures += 1
            self.last_failure = datetime.now()
            
            if self.failures >= self.failure_threshold:
                self.state = "open"
            
            raise

# Usage
breaker = CircuitBreaker(failure_threshold=3, recovery_timeout=30)

try:
    result = breaker.call(client.skills.execute, skill_id="...", input={})
except Exception as e:
    print(f"Call failed: {e}")
```

## Logging Errors

```python
import logging
from machpay import MachPay
from machpay.exceptions import MachPayError

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

client = MachPay()

try:
    response = client.skills.execute(...)
except MachPayError as e:
    logger.error(
        "MachPay API error",
        extra={
            "error_code": e.code,
            "error_type": type(e).__name__,
            "request_id": e.request_id,
            "status_code": e.status_code,
        }
    )
    raise
```

## Best Practices Summary

| Do ✅ | Don't ❌ |
|------|---------|
| Use specific exception types | Catch all exceptions blindly |
| Implement exponential backoff | Retry immediately in a loop |
| Set reasonable timeouts | Leave timeouts at infinity |
| Use idempotency keys | Retry without deduplication |
| Log request IDs | Lose error context |
| Monitor error rates | Ignore error patterns |

## Next Steps

- [Streaming Responses](streaming.md) – Handle stream errors
- [Webhooks](webhooks.md) – Reliable event delivery
- [Authentication](authentication.md) – Handle auth errors

---

**Questions?** Check the [API Reference](/protocol-reference/api-specs.md) or join [Discord](https://discord.gg/machpay).

