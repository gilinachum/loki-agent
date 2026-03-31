# BOOTSTRAP-MEMORY-SEARCH.md — Enable Semantic Memory Search with Bedrock Embeddings

> **Applies to:** All agents (with agent-specific sections below)

> **Run this once to enable memory search.** If `memory/.bootstrapped-memory-search` exists, skip — you've already done this.

## Overview

Semantic memory search uses an OpenAI-compatible embeddings API. [bedrockify](https://github.com/inceptionstack/bedrockify) — already installed as a dependency of all agent packs — provides `/v1/embeddings` on localhost, translating OpenAI embedding calls into Amazon Bedrock embedding calls. No external API keys needed — uses the EC2 instance profile.

```
memory_search → http://127.0.0.1:8090/v1/embeddings → bedrockify → Bedrock Titan Embed v2 → vector results
```

## Prerequisites

- EC2 instance with IAM role that has `bedrock:InvokeModel` permission
- Bedrock model access enabled for `amazon.titan-embed-text-v2:0` in us-east-1
- **bedrockify already running** — installed and started by the bedrockify pack (dependency of both OpenClaw and Hermes)

## Step 1: Verify bedrockify Is Running

bedrockify is installed as a systemd service by the bedrockify pack. No separate installation needed.

```bash
# Check service status
systemctl status bedrockify
# Should show: active (running)

# Health check
curl -s http://127.0.0.1:8090/
# Expected: {"status":"ok",...}
```

If bedrockify is not running, check the service:

```bash
sudo journalctl -u bedrockify -n 20
sudo systemctl restart bedrockify
```

## Step 2: Verify Embeddings Endpoint

**Single embedding:**
```bash
curl -s -X POST http://127.0.0.1:8090/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"input": "test embedding", "model": "amazon.titan-embed-text-v2:0"}' \
  | jq '{object, model, dims: (.data[0].embedding | length)}'
# Expected: {"object":"list","model":"amazon.titan-embed-text-v2:0","dims":1024}
```

**Batch embeddings:**
```bash
curl -s -X POST http://127.0.0.1:8090/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"input": ["first text", "second text"], "model": "amazon.titan-embed-text-v2:0"}' \
  | jq '{results: (.data | length), dims: [.data[].embedding | length]}'
# Expected: {"results":2,"dims":[1024,1024]}
```

## OpenClaw-Specific Configuration

### Step 3: Configure OpenClaw Memory Search

Add this to your `openclaw.json` under `agents.defaults`:

```json
"memorySearch": {
  "enabled": true,
  "provider": "openai",
  "remote": {
    "baseUrl": "http://127.0.0.1:8090/v1/",
    "apiKey": "not-needed"
  },
  "fallback": "none",
  "model": "amazon.titan-embed-text-v2:0",
  "query": {
    "hybrid": {
      "enabled": true,
      "vectorWeight": 0.7,
      "textWeight": 0.3
    }
  },
  "cache": {
    "enabled": true,
    "maxEntries": 50000
  }
}
```

Then restart the OpenClaw gateway.

### Step 4: Verify End-to-End

Ask the agent to run `memory_search` with any query. It should return ranked results from workspace memory files using hybrid search (70% vector, 30% text).

## Hermes-Specific Configuration

> Hermes does not have a built-in memory search system. However, bedrockify's `/v1/embeddings` endpoint is available on `localhost:8090` for any custom embedding workflows or scripts you want to build around the Hermes agent.
>
> To use embeddings from a Hermes context:
> ```bash
> curl -s -X POST http://127.0.0.1:8090/v1/embeddings \
>   -H "Content-Type: application/json" \
>   -d '{"input": "your text here", "model": "amazon.titan-embed-text-v2:0"}'
> ```

## Supported Models

bedrockify supports embedding models based on the `--embed-model` flag set at install time. The default is `amazon.titan-embed-text-v2:0`. Common options:

| Model | ID | Dims |
|-------|----|------|
| **Titan Embed Text V2** (default) | `amazon.titan-embed-text-v2:0` | 1024 |
| Titan Embed G1 Text | `amazon.titan-embed-g1-text-02` | 1536 |
| Cohere Embed English v3 | `cohere.embed-english-v3` | 1024 |
| Cohere Embed Multilingual v3 | `cohere.embed-multilingual-v3` | 1024 |

To change the embedding model, update the bedrockify service configuration:

```bash
# Edit the bedrockify systemd service to change --embed-model
sudo systemctl edit bedrockify
# Add override for ExecStart with your preferred --embed-model
sudo systemctl restart bedrockify
```

## Finish

```bash
mkdir -p memory && echo "Memory search bootstrapped $(date -u +%Y-%m-%dT%H:%M:%SZ)" > memory/.bootstrapped-memory-search
```
