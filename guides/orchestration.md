# Multi-Skill Orchestration

Chain multiple skills together with dependency graphs, parallel execution, and error recovery. Build complex AI workflows by composing simple skills.

**Time:** 12 minutes

## Why Orchestration?

Real-world AI applications often require multiple capabilities:

```
User Query → Translate → Analyze Sentiment → Generate Response → Translate Back
```

Orchestration lets you:
- Chain skills sequentially
- Run skills in parallel
- Handle failures gracefully
- Optimize cost and latency

## Basic Chaining

### Sequential Execution

```python
from machpay import MachPay

client = MachPay()

# Step 1: Translate to English
translation = client.skills.execute(
    skill_id="translator",
    input={"text": "Bonjour le monde", "target": "en"}
)

# Step 2: Analyze sentiment
sentiment = client.skills.execute(
    skill_id="sentiment-analyzer",
    input={"text": translation.data["translated_text"]}
)

# Step 3: Generate response
response = client.skills.execute(
    skill_id="chat-assistant",
    input={
        "prompt": f"The user said '{translation.data['translated_text']}' "
                  f"with {sentiment.data['sentiment']} sentiment. Respond appropriately."
    }
)

print(response.data["message"])
```

### Node.js

```typescript
const client = new MachPay();

// Sequential chain
const translation = await client.skills.execute({
  skillId: 'translator',
  input: { text: 'Bonjour le monde', target: 'en' }
});

const sentiment = await client.skills.execute({
  skillId: 'sentiment-analyzer',
  input: { text: translation.data.translatedText }
});

const response = await client.skills.execute({
  skillId: 'chat-assistant',
  input: {
    prompt: `User said '${translation.data.translatedText}' with ${sentiment.data.sentiment} sentiment.`
  }
});
```

## Parallel Execution

Run independent skills simultaneously:

### Python

```python
import asyncio
from machpay import AsyncMachPay

async def analyze_content(text: str):
    client = AsyncMachPay()
    
    # Run all analyses in parallel
    results = await asyncio.gather(
        client.skills.execute(
            skill_id="sentiment-analyzer",
            input={"text": text}
        ),
        client.skills.execute(
            skill_id="toxicity-detector",
            input={"text": text}
        ),
        client.skills.execute(
            skill_id="language-detector",
            input={"text": text}
        ),
        client.skills.execute(
            skill_id="entity-extractor",
            input={"text": text}
        ),
    )
    
    return {
        "sentiment": results[0].data,
        "toxicity": results[1].data,
        "language": results[2].data,
        "entities": results[3].data,
    }

# Usage
analysis = asyncio.run(analyze_content("MachPay is amazing!"))
```

### Node.js

```typescript
async function analyzeContent(text: string) {
  const client = new MachPay();
  
  const [sentiment, toxicity, language, entities] = await Promise.all([
    client.skills.execute({ skillId: 'sentiment-analyzer', input: { text } }),
    client.skills.execute({ skillId: 'toxicity-detector', input: { text } }),
    client.skills.execute({ skillId: 'language-detector', input: { text } }),
    client.skills.execute({ skillId: 'entity-extractor', input: { text } }),
  ]);
  
  return {
    sentiment: sentiment.data,
    toxicity: toxicity.data,
    language: language.data,
    entities: entities.data,
  };
}
```

## Using the Orchestrator

MachPay provides a built-in orchestrator for complex workflows:

### Define a Workflow

```python
from machpay import MachPay, Workflow, Step

client = MachPay()

# Define workflow
workflow = Workflow(
    name="content-moderation",
    steps=[
        Step(
            id="analyze",
            skill_id="content-analyzer",
            input_mapping={"text": "$.input.text"}
        ),
        Step(
            id="moderate",
            skill_id="content-moderator",
            input_mapping={
                "text": "$.input.text",
                "analysis": "$.steps.analyze.output"
            },
            depends_on=["analyze"]
        ),
        Step(
            id="respond",
            skill_id="response-generator",
            input_mapping={
                "original": "$.input.text",
                "moderation": "$.steps.moderate.output"
            },
            depends_on=["moderate"],
            condition="$.steps.moderate.output.approved == true"
        ),
    ]
)

# Execute workflow
result = client.workflows.execute(
    workflow=workflow,
    input={"text": "User generated content here..."}
)

print(result.output)
```

### Workflow Features

| Feature | Description |
|---------|-------------|
| `depends_on` | Wait for specified steps to complete |
| `condition` | Skip step if condition is false |
| `retry` | Auto-retry on failure |
| `timeout` | Per-step timeout |
| `parallel` | Run steps in parallel |

## DAG (Directed Acyclic Graph)

For complex dependencies:

```
       ┌─────────┐
       │ Extract │
       └────┬────┘
            │
    ┌───────┴───────┐
    ▼               ▼
┌───────┐     ┌───────────┐
│Analyze│     │ Summarize │
└───┬───┘     └─────┬─────┘
    │               │
    └───────┬───────┘
            ▼
      ┌───────────┐
      │  Report   │
      └───────────┘
```

```python
from machpay import Workflow, Step

workflow = Workflow(
    name="document-processor",
    steps=[
        Step(id="extract", skill_id="text-extractor"),
        Step(id="analyze", skill_id="analyzer", depends_on=["extract"]),
        Step(id="summarize", skill_id="summarizer", depends_on=["extract"]),
        Step(
            id="report",
            skill_id="report-generator",
            depends_on=["analyze", "summarize"]  # Wait for both
        ),
    ]
)
```

## Error Handling in Workflows

### Retry Configuration

```python
Step(
    id="unreliable-step",
    skill_id="external-api",
    retry=RetryConfig(
        max_retries=3,
        backoff_factor=2.0,
        retry_on=[500, 503]
    )
)
```

### Fallback Steps

```python
Step(
    id="primary",
    skill_id="premium-llm",
    fallback=Step(
        id="fallback",
        skill_id="basic-llm"
    )
)
```

### Error Handlers

```python
workflow = Workflow(
    name="with-error-handling",
    steps=[...],
    on_error=ErrorHandler(
        action="continue",  # or "abort", "retry"
        notify_webhook="https://your-server.com/workflow-error"
    )
)
```

## Fan-Out / Fan-In Pattern

Process multiple items in parallel:

```python
from machpay import MachPay, Workflow, Step

client = MachPay()

# Fan-out: Process each document in parallel
documents = ["doc1.txt", "doc2.txt", "doc3.txt"]

async def process_documents(docs):
    tasks = [
        client.skills.execute(
            skill_id="document-analyzer",
            input={"document": doc}
        )
        for doc in docs
    ]
    
    # Fan-in: Collect all results
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    # Filter successful results
    successful = [r for r in results if not isinstance(r, Exception)]
    
    # Aggregate
    aggregation = await client.skills.execute(
        skill_id="result-aggregator",
        input={"analyses": [r.data for r in successful]}
    )
    
    return aggregation.data
```

## Map-Reduce Pattern

```python
from machpay import MachPay

async def map_reduce(items, map_skill, reduce_skill):
    client = AsyncMachPay()
    
    # Map phase: Process each item
    mapped = await asyncio.gather(*[
        client.skills.execute(skill_id=map_skill, input={"item": item})
        for item in items
    ])
    
    # Reduce phase: Aggregate results
    reduced = await client.skills.execute(
        skill_id=reduce_skill,
        input={"results": [m.data for m in mapped]}
    )
    
    return reduced.data

# Example: Word count across documents
result = await map_reduce(
    items=["doc1.txt", "doc2.txt", "doc3.txt"],
    map_skill="word-counter",
    reduce_skill="count-aggregator"
)
```

## Cost Optimization

### Conditional Execution

Only run expensive skills when needed:

```python
# Check with cheap classifier first
classification = client.skills.execute(
    skill_id="quick-classifier",  # $0.0001
    input={"text": user_input}
)

# Only use expensive skill if needed
if classification.data["requires_deep_analysis"]:
    analysis = client.skills.execute(
        skill_id="deep-analyzer",  # $0.01
        input={"text": user_input}
    )
else:
    analysis = classification  # Use quick result
```

### Caching Results

```python
from functools import lru_cache
import hashlib

def get_cache_key(skill_id: str, input_data: dict) -> str:
    return hashlib.sha256(
        f"{skill_id}:{json.dumps(input_data, sort_keys=True)}".encode()
    ).hexdigest()

@lru_cache(maxsize=1000)
def execute_cached(cache_key: str, skill_id: str, input_json: str):
    return client.skills.execute(
        skill_id=skill_id,
        input=json.loads(input_json)
    )

# Usage
result = execute_cached(
    cache_key=get_cache_key("translator", {"text": "hello"}),
    skill_id="translator",
    input_json=json.dumps({"text": "hello"})
)
```

## Monitoring Workflows

```python
# Get workflow execution status
status = client.workflows.status(execution_id="wf_abc123")

print(f"Status: {status.state}")  # running, completed, failed
print(f"Current step: {status.current_step}")
print(f"Completed steps: {status.completed_steps}")
print(f"Total cost: ${status.total_cost}")
print(f"Duration: {status.duration_ms}ms")

# Get detailed step results
for step in status.steps:
    print(f"  {step.id}: {step.state} ({step.duration_ms}ms, ${step.cost})")
```

## Best Practices

| Do ✅ | Don't ❌ |
|------|---------|
| Run independent steps in parallel | Chain everything sequentially |
| Use conditions to skip unnecessary steps | Run all steps always |
| Implement fallbacks for critical paths | Let failures cascade |
| Cache frequently-used results | Re-compute everything |
| Set per-step timeouts | Use global timeout only |
| Monitor workflow costs | Ignore billing |

## Next Steps

- [Streaming Responses](streaming.md) – Stream orchestrated outputs
- [Error Handling](error-handling.md) – Handle workflow failures
- [Webhooks](webhooks.md) – Get notified of workflow completion

---

**Questions?** Check the [API Reference](/protocol-reference/api-specs.md) or join [Discord](https://discord.gg/machpay).

