# Building Your First Skill

Step-by-step tutorial on creating, deploying, and monetizing your first skill on MachPay. By the end, you'll have a live skill that others can pay to use.

**Time:** 15 minutes

## What is a Skill?

A skill is a monetizable API endpoint on the MachPay network. When someone executes your skill:

1. They pay per execution (you set the price)
2. MachPay handles billing and payments
3. You receive funds in your account

## Prerequisites

- MachPay account with API key
- Python 3.9+ or Node.js 18+
- Basic REST API knowledge

## Step 1: Plan Your Skill

Let's build a **Sentiment Analyzer** skill that analyzes text and returns sentiment scores.

**Input:**
```json
{
  "text": "I love this product! It's amazing."
}
```

**Output:**
```json
{
  "sentiment": "positive",
  "confidence": 0.94,
  "scores": {
    "positive": 0.94,
    "neutral": 0.05,
    "negative": 0.01
  }
}
```

## Step 2: Create the Skill Handler

### Python

```python
# skill.py
from machpay import Skill, SkillInput, SkillOutput
from textblob import TextBlob

skill = Skill(
    id="sentiment-analyzer",
    name="Sentiment Analyzer",
    description="Analyze sentiment of any text",
    version="1.0.0",
    price_usd=0.001,  # $0.001 per execution
)

@skill.handler
def analyze_sentiment(input: SkillInput) -> SkillOutput:
    text = input.data.get("text", "")
    
    if not text:
        raise ValueError("Text is required")
    
    # Analyze sentiment
    blob = TextBlob(text)
    polarity = blob.sentiment.polarity  # -1 to 1
    
    # Convert to scores
    if polarity > 0.1:
        sentiment = "positive"
        scores = {"positive": polarity, "neutral": 0.1, "negative": 0}
    elif polarity < -0.1:
        sentiment = "negative"
        scores = {"positive": 0, "neutral": 0.1, "negative": abs(polarity)}
    else:
        sentiment = "neutral"
        scores = {"positive": 0.1, "neutral": 0.8, "negative": 0.1}
    
    return SkillOutput(
        data={
            "sentiment": sentiment,
            "confidence": abs(polarity) if polarity != 0 else 0.8,
            "scores": scores
        }
    )

if __name__ == "__main__":
    skill.serve(port=8080)
```

### Node.js

```typescript
// skill.ts
import { Skill, SkillInput, SkillOutput } from '@machpay/sdk';
import Sentiment from 'sentiment';

const analyzer = new Sentiment();

const skill = new Skill({
  id: 'sentiment-analyzer',
  name: 'Sentiment Analyzer',
  description: 'Analyze sentiment of any text',
  version: '1.0.0',
  priceUsd: 0.001, // $0.001 per execution
});

skill.handler(async (input: SkillInput): Promise<SkillOutput> => {
  const text = input.data.text;
  
  if (!text) {
    throw new Error('Text is required');
  }
  
  // Analyze sentiment
  const result = analyzer.analyze(text);
  const score = result.comparative; // -5 to 5
  
  let sentiment: string;
  let scores: Record<string, number>;
  
  if (score > 0.5) {
    sentiment = 'positive';
    scores = { positive: Math.min(score / 5, 1), neutral: 0.1, negative: 0 };
  } else if (score < -0.5) {
    sentiment = 'negative';
    scores = { positive: 0, neutral: 0.1, negative: Math.min(Math.abs(score) / 5, 1) };
  } else {
    sentiment = 'neutral';
    scores = { positive: 0.1, neutral: 0.8, negative: 0.1 };
  }
  
  return {
    data: {
      sentiment,
      confidence: Math.abs(score) / 5,
      scores
    }
  };
});

skill.serve({ port: 8080 });
```

## Step 3: Test Locally

### Run the skill server

```bash
# Python
pip install machpay textblob
python skill.py

# Node.js
npm install @machpay/sdk sentiment
npx ts-node skill.ts
```

### Test with curl

```bash
curl http://localhost:8080/execute \
  -H "Content-Type: application/json" \
  -d '{"text": "I love this product!"}'
```

**Expected response:**
```json
{
  "sentiment": "positive",
  "confidence": 0.85,
  "scores": {"positive": 0.85, "neutral": 0.1, "negative": 0}
}
```

## Step 4: Register the Skill

Use the CLI to register your skill on MachPay:

```bash
# Install CLI
brew install machpay/tap/machpay

# Login
machpay login

# Register skill
machpay skill register \
  --id sentiment-analyzer \
  --name "Sentiment Analyzer" \
  --description "Analyze sentiment of any text" \
  --price 0.001 \
  --category "NLP"
```

## Step 5: Deploy

### Option A: MachPay Hosting (Recommended)

```bash
# Deploy to MachPay's infrastructure
machpay skill deploy --id sentiment-analyzer --source .
```

MachPay will:
- Build and containerize your skill
- Deploy to global edge nodes
- Handle scaling automatically
- Provide a public endpoint

### Option B: Self-Hosted

If you prefer to host your own infrastructure:

1. Deploy your skill to your server
2. Register the endpoint:

```bash
machpay skill set-endpoint \
  --id sentiment-analyzer \
  --url https://your-server.com/sentiment
```

3. Configure webhook for health checks

## Step 6: Configure Pricing

### Fixed Price

```bash
machpay skill pricing set \
  --id sentiment-analyzer \
  --price 0.001
```

### Tiered Pricing

```bash
machpay skill pricing set \
  --id sentiment-analyzer \
  --tiers '[
    {"up_to": 1000, "price": 0.001},
    {"up_to": 10000, "price": 0.0008},
    {"up_to": null, "price": 0.0005}
  ]'
```

### Usage-Based

```bash
machpay skill pricing set \
  --id sentiment-analyzer \
  --base 0.0005 \
  --per-token 0.00001
```

## Step 7: Publish

Make your skill discoverable in the marketplace:

```bash
machpay skill publish --id sentiment-analyzer
```

Your skill is now live! Others can find it at:
`https://console.machpay.xyz/marketplace/sentiment-analyzer`

## Step 8: Monitor

Track usage and revenue in the Console:

```bash
# Quick stats
machpay skill stats --id sentiment-analyzer

# Output:
# Executions (24h): 1,247
# Revenue (24h): $1.25
# Avg latency: 89ms
# Success rate: 99.2%
```

Or view detailed analytics in the [Console Dashboard](https://console.machpay.xyz/studio).

## Skill Configuration Reference

```python
skill = Skill(
    # Required
    id="sentiment-analyzer",           # Unique identifier
    name="Sentiment Analyzer",         # Display name
    version="1.0.0",                   # Semantic version
    
    # Pricing
    price_usd=0.001,                   # Per-execution price
    
    # Metadata
    description="Analyze text sentiment",
    category="NLP",
    tags=["sentiment", "nlp", "text"],
    
    # Limits
    timeout_seconds=30,                # Max execution time
    max_input_size_kb=100,             # Max input payload
    
    # Advanced
    requires_auth=True,                # Require API key
    rate_limit_per_minute=100,         # Per-user rate limit
)
```

## Input Validation

Add schema validation for better error messages:

```python
from pydantic import BaseModel, Field

class SentimentInput(BaseModel):
    text: str = Field(..., min_length=1, max_length=10000)
    language: str = Field(default="en", pattern="^[a-z]{2}$")

@skill.handler(input_schema=SentimentInput)
def analyze_sentiment(input: SentimentInput) -> SkillOutput:
    # input is now validated and typed
    ...
```

## Next Steps

- [Streaming Responses](streaming.md) – Add real-time output
- [Error Handling](error-handling.md) – Handle failures gracefully
- [Webhooks](webhooks.md) – Get notified of executions
- [Billing & Metering](billing.md) – Advanced pricing strategies

## Example Skills

Browse open-source examples:

- [Text Summarizer](https://github.com/machpay/examples/tree/main/skills/text-summarizer)
- [Image Classifier](https://github.com/machpay/examples/tree/main/skills/image-classifier)
- [Code Explainer](https://github.com/machpay/examples/tree/main/skills/code-explainer)

---

**Need help?** Join our [Discord](https://discord.gg/machpay) or check the [Skill Reference](/for-vendors/gateway-integration.md).

