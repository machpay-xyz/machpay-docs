# Streaming Responses

Implement real-time streaming for LLM outputs and long-running tasks with Server-Sent Events. This guide covers both consuming and producing streaming responses.

**Time:** 10 minutes

## Why Streaming?

For LLMs and long-running tasks, waiting for a complete response creates poor UX:

| Approach | Time to First Byte | User Experience |
|----------|-------------------|-----------------|
| Standard | 5-30 seconds | User waits, uncertain |
| Streaming | < 100ms | Immediate feedback |

Streaming delivers tokens as they're generated, enabling:
- Real-time chat interfaces
- Progress indicators for long tasks
- Lower perceived latency

## Consuming Streams

### Python

```python
from machpay import MachPay

client = MachPay()

# Stream responses from an LLM skill
stream = client.skills.stream(
    skill_id="chat-assistant",
    input={"prompt": "Explain quantum computing in simple terms"}
)

# Process chunks as they arrive
for chunk in stream:
    print(chunk.text, end="", flush=True)

print()  # Final newline
```

### Node.js

```typescript
import { MachPay } from '@machpay/sdk';

const client = new MachPay();

// Stream responses from an LLM skill
const stream = await client.skills.stream({
  skillId: 'chat-assistant',
  input: { prompt: 'Explain quantum computing in simple terms' }
});

// Process chunks as they arrive
for await (const chunk of stream) {
  process.stdout.write(chunk.text);
}

console.log(); // Final newline
```

### Async/Await Pattern

```python
import asyncio
from machpay import AsyncMachPay

async def main():
    client = AsyncMachPay()
    
    async for chunk in client.skills.stream(
        skill_id="chat-assistant",
        input={"prompt": "Write a haiku about AI"}
    ):
        print(chunk.text, end="", flush=True)

asyncio.run(main())
```

## Stream Events

Each stream emits different event types:

```typescript
const stream = await client.skills.stream({...});

stream.on('start', (meta) => {
  console.log('Stream started:', meta.executionId);
});

stream.on('chunk', (chunk) => {
  console.log('Received:', chunk.text);
});

stream.on('end', (summary) => {
  console.log('Total tokens:', summary.tokensUsed);
  console.log('Total cost:', summary.costUsd);
});

stream.on('error', (error) => {
  console.error('Stream error:', error.message);
});
```

### Event Types

| Event | Description | Payload |
|-------|-------------|---------|
| `start` | Stream initialized | `{ executionId, skillId }` |
| `chunk` | Content chunk | `{ text, index, timestamp }` |
| `end` | Stream complete | `{ tokensUsed, costUsd, durationMs }` |
| `error` | Error occurred | `{ code, message }` |

## Building Streaming Skills

### Python

```python
from machpay import Skill, SkillInput
from typing import Generator

skill = Skill(
    id="story-generator",
    name="Story Generator",
    price_usd=0.01,
    streaming=True,  # Enable streaming
)

@skill.stream_handler
def generate_story(input: SkillInput) -> Generator[str, None, None]:
    prompt = input.data.get("prompt", "")
    
    # Simulate LLM generation
    words = ["Once", "upon", "a", "time", "in", "a", "digital", "realm", "..."]
    
    for word in words:
        yield word + " "
        time.sleep(0.1)  # Simulate generation delay

skill.serve(port=8080)
```

### Node.js

```typescript
import { Skill, SkillInput } from '@machpay/sdk';

const skill = new Skill({
  id: 'story-generator',
  name: 'Story Generator',
  priceUsd: 0.01,
  streaming: true, // Enable streaming
});

skill.streamHandler(async function* (input: SkillInput) {
  const prompt = input.data.prompt;
  
  // Simulate LLM generation
  const words = ['Once', 'upon', 'a', 'time', 'in', 'a', 'digital', 'realm', '...'];
  
  for (const word of words) {
    yield word + ' ';
    await sleep(100); // Simulate generation delay
  }
});

skill.serve({ port: 8080 });
```

## Integration with LLMs

### OpenAI Integration

```python
from machpay import Skill, SkillInput
from openai import OpenAI
from typing import Generator

skill = Skill(id="gpt-wrapper", streaming=True, price_usd=0.002)
openai = OpenAI()

@skill.stream_handler
def chat(input: SkillInput) -> Generator[str, None, None]:
    stream = openai.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": input.data["prompt"]}],
        stream=True
    )
    
    for chunk in stream:
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content
```

### Anthropic Integration

```python
from machpay import Skill, SkillInput
from anthropic import Anthropic
from typing import Generator

skill = Skill(id="claude-wrapper", streaming=True, price_usd=0.003)
anthropic = Anthropic()

@skill.stream_handler
def chat(input: SkillInput) -> Generator[str, None, None]:
    with anthropic.messages.stream(
        model="claude-3-opus-20240229",
        max_tokens=1024,
        messages=[{"role": "user", "content": input.data["prompt"]}]
    ) as stream:
        for text in stream.text_stream:
            yield text
```

## Frontend Integration

### React with SSE

```tsx
import { useState, useEffect } from 'react';

function ChatMessage({ prompt }: { prompt: string }) {
  const [response, setResponse] = useState('');
  const [isStreaming, setIsStreaming] = useState(true);

  useEffect(() => {
    const eventSource = new EventSource(
      `/api/stream?prompt=${encodeURIComponent(prompt)}`
    );

    eventSource.onmessage = (event) => {
      const chunk = JSON.parse(event.data);
      setResponse(prev => prev + chunk.text);
    };

    eventSource.onerror = () => {
      setIsStreaming(false);
      eventSource.close();
    };

    return () => eventSource.close();
  }, [prompt]);

  return (
    <div className="message">
      {response}
      {isStreaming && <span className="cursor">▋</span>}
    </div>
  );
}
```

### Vanilla JavaScript

```javascript
async function streamResponse(prompt) {
  const response = await fetch('/api/stream', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ prompt })
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    
    const text = decoder.decode(value);
    document.getElementById('output').textContent += text;
  }
}
```

## Error Handling

### Retry on Stream Failure

```python
from machpay import MachPay
from machpay.exceptions import StreamError

client = MachPay()

def stream_with_retry(skill_id: str, input: dict, max_retries: int = 3):
    for attempt in range(max_retries):
        try:
            collected = []
            for chunk in client.skills.stream(skill_id=skill_id, input=input):
                collected.append(chunk.text)
                yield chunk.text
            return  # Success
        except StreamError as e:
            if attempt == max_retries - 1:
                raise
            print(f"Stream failed, retrying... ({attempt + 1}/{max_retries})")
```

### Timeout Handling

```python
import asyncio
from machpay import AsyncMachPay

async def stream_with_timeout(prompt: str, timeout: float = 30.0):
    client = AsyncMachPay()
    
    try:
        async with asyncio.timeout(timeout):
            async for chunk in client.skills.stream(
                skill_id="chat-assistant",
                input={"prompt": prompt}
            ):
                yield chunk.text
    except asyncio.TimeoutError:
        yield "\n[Response timed out]"
```

## Billing for Streams

Streaming skills are billed based on:

1. **Base cost**: Fixed per-execution fee
2. **Token cost**: Per-token generated (optional)
3. **Duration cost**: Per-second of stream time (optional)

```python
skill = Skill(
    id="premium-chat",
    streaming=True,
    pricing={
        "base_usd": 0.001,        # Base fee
        "per_token_usd": 0.00002, # Per output token
        "per_second_usd": 0.0001  # Per second streamed
    }
)
```

## Performance Tips

1. **Buffer appropriately**: Don't yield single characters; buffer to ~10-50 tokens
2. **Use async**: Async streaming handles backpressure better
3. **Set timeouts**: Always set reasonable timeouts for streams
4. **Monitor latency**: Track time-to-first-byte and total duration

## Next Steps

- [Error Handling](error-handling.md) – Handle stream failures
- [Building Your First Skill](first-skill.md) – Create your own streaming skill
- [Billing & Metering](billing.md) – Price your streaming skills

---

**Questions?** Check the [API Reference](/protocol-reference/api-specs.md) or join [Discord](https://discord.gg/machpay).

