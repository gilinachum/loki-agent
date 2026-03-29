# BOOTSTRAP-MEMORY-SEARCH.md — Enable Semantic Memory Search with Bedrock Embeddings

> **Run this once to enable memory search.** If `memory/.bootstrapped-memory-search` exists, skip — you've already done this.

## Overview

OpenClaw's `memory_search` uses an OpenAI-compatible embeddings API. We run [embedrock](https://github.com/inceptionstack/embedrock) — a tiny Go binary that translates OpenAI `/v1/embeddings` calls into Amazon Bedrock embedding calls. No external API keys needed — uses the EC2 instance profile.

```
OpenClaw memory_search → http://127.0.0.1:8089/v1/embeddings → embedrock → Bedrock Cohere Embed v4 → vector results
```

## Prerequisites

- EC2 instance with IAM role that has `bedrock:InvokeModel` permission
- Bedrock model access enabled for `cohere.embed-v4:0` in us-east-1
- No Node.js or extra dependencies required — embedrock is a single static binary

## Step 1: Install embedrock

```bash
# Linux arm64 (EC2 Graviton)
curl -fsSL https://github.com/inceptionstack/embedrock/releases/latest/download/embedrock-linux-arm64 \
  -o /tmp/embedrock && chmod +x /tmp/embedrock && sudo mv /tmp/embedrock /usr/local/bin/embedrock

# Linux amd64
curl -fsSL https://github.com/inceptionstack/embedrock/releases/latest/download/embedrock-linux-amd64 \
  -o /tmp/embedrock && chmod +x /tmp/embedrock && sudo mv /tmp/embedrock /usr/local/bin/embedrock

# Verify
embedrock --version
```

## Step 2: Create a systemd service

```bash
sudo tee /etc/systemd/system/embedrock.service > /dev/null << 'EOF'
[Unit]
Description=embedrock - Bedrock embedding proxy
After=network.target

[Service]
Type=simple
User=ec2-user
ExecStart=/usr/local/bin/embedrock --port 8089 --region us-east-1 --model cohere.embed-v4:0
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable embedrock
sudo systemctl start embedrock
```

## Step 3: Configure OpenClaw

Add this to your `openclaw.json` under `agents.defaults`:

```json
"memorySearch": {
  "enabled": true,
  "provider": "openai",
  "remote": {
    "baseUrl": "http://127.0.0.1:8089/v1/",
    "apiKey": "not-needed"
  },
  "fallback": "none",
  "model": "cohere.embed-v4:0",
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

## Step 4: Verify

**Service running:**
```bash
systemctl status embedrock
# Should show: active (running)
```

**Health check:**
```bash
curl -s http://127.0.0.1:8089/
# Expected: {"status":"ok","model":"cohere.embed-v4:0"}
```

**Single embedding:**
```bash
curl -s -X POST http://127.0.0.1:8089/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"input": "test embedding", "model": "cohere.embed-v4:0"}' \
  | jq '{object, model, dims: (.data[0].embedding | length)}'
# Expected: {"object":"list","model":"cohere.embed-v4:0","dims":1536}
```

**Batch embeddings:**
```bash
curl -s -X POST http://127.0.0.1:8089/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"input": ["first text", "second text"], "model": "cohere.embed-v4:0"}' \
  | jq '{results: (.data | length), dims: [.data[].embedding | length]}'
# Expected: {"results":2,"dims":[1536,1536]}
```

**End-to-end memory search:**
Ask Loki to run `memory_search` with any query. It should return ranked results from workspace memory files using hybrid search (70% vector, 30% text).

## Step 5: Backfill existing memory

After enabling semantic search for the first time, existing memory files are not automatically indexed. Run:

```bash
openclaw memory index --force
```

This vectorizes all current memory files. Without this step, `memory_search` will only find content written *after* the setup — silently missing everything prior.

## Memory quality matters

Vector search ranks results by cosine similarity. **Low-signal, repetitive content tanks the scores of everything nearby** — making recall unreliable even for genuinely useful memories.

### What hurts search quality

High-frequency repetitive content compresses into a dense cluster in vector space. This raises the effective similarity floor and pushes useful content below the retrieval threshold.

The biggest offender is **heartbeat logs**:

```markdown
## Heartbeat 02:19 UTC
### Apps: ✅ frontend + api healthy
### Security Hub: no change — 1 CRITICAL, 94 HIGH
## Heartbeat 02:49 UTC
### Apps: ✅ frontend + api healthy
### Security Hub: no change — 1 CRITICAL, 94 HIGH
```

A single daily memory file can contain 40–50 of these. They're semantically near-identical, contribute nothing to recall, and dilute chunk quality across the entire index.

### Rule: only write what changed

**Don't write:**
- "no change", "all healthy", "nothing to report"
- Repeated status confirmations
- Routine cron completions with no notable outcome

**Do write:**
- App went down or returned unexpected status
- Security finding count changed (new CVE, severity shift)
- A decision was made
- A bug was found or fixed
- A TODO was started or completed autonomously

### Keep heartbeat files out of the index

If your agent writes verbose heartbeat logs that are useful for audit but not for recall, route them to a separate file pattern and exclude from indexing:

```bash
# Write heartbeat noise here (not indexed)
memory/heartbeat-YYYY-MM-DD.md

# Keep daily notes clean (indexed)
memory/YYYY-MM-DD.md
```

To exclude a pattern from indexing, configure the memory sources in `openclaw.json`:

```json
"memorySearch": {
  "sources": {
    "exclude": ["memory/heartbeat-*.md"]
  }
}
```

## Supported Models

embedrock auto-detects model family by ID prefix:

| Model | ID | Dims |
|-------|----|------|
| **Cohere Embed v4** (recommended) | `cohere.embed-v4:0` | 1536 |
| Cohere Embed English v3 | `cohere.embed-english-v3` | 1024 |
| Cohere Embed Multilingual v3 | `cohere.embed-multilingual-v3` | 1024 |
| Titan Embed Text V2 | `amazon.titan-embed-text-v2:0` | 1024 |
| Titan Embed G1 Text | `amazon.titan-embed-g1-text-02` | 1536 |

## Finish

```bash
mkdir -p memory && echo "Memory search bootstrapped $(date -u +%Y-%m-%dT%H:%M:%SZ)" > memory/.bootstrapped-memory-search
```
