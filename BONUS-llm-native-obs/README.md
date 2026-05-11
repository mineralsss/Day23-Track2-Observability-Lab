# BONUS: LLM-Native Observability with Langfuse (Self-Hosted)

Self-hosted Langfuse for tracing LLM calls, prompts, and evaluations.

## Quick Start

```bash
# Start Langfuse services
make up-langfuse

# Verify Langfuse is healthy (wait ~30s for first start)
make smoke-langfuse

# Stop Langfuse
make down-langfuse
```

## Access

- **Langfuse UI**: http://localhost:3001 or http://127.0.0.1:3001
- **PostgreSQL**: localhost:5433 (internal: `langfuse-postgres:5432`)
- Default credentials: Sign up via UI on first launch

### Browser Redirect Loop Fix

If you experience a redirect loop in your browser:
1. Clear cookies for `localhost:3001` and `127.0.0.1:3001`
2. Or use **incognito/private browsing mode**
3. Or try the alternate URL (`http://127.0.0.1:3001` if using `localhost:3001`, or vice versa)

This is a known NextAuth.js behavior with self-hosted Langfuse and is safe to bypass.

## Environment Variables

Copy to `.env` (gitignored):

```bash
# Langfuse database
LANGFUSE_DB_PASSWORD=langfuse

# Security - generate with: openssl rand -hex 32
LANGFUSE_NEXTAUTH_SECRET=your-long-random-secret-here
LANGFUSE_SALT=your-long-random-salt-here
# ENCRYPTION_KEY must be exactly 64 hex characters (32 bytes)
LANGFUSE_ENCRYPTION_KEY=your-64-char-hex-encryption-key-here
```

## Integrating with the FastAPI App

The app (`01-instrument-fastapi/app/`) is already instrumented with Langfuse! 
When you start the stack with `make up-langfuse`, the app will automatically send LLM traces to Langfuse.

### Getting API Keys

1. Access Langfuse UI at http://localhost:3001
2. Sign up (first user becomes admin)
3. Go to **Settings > API Keys**
4. Create a new API key pair (Public Key + Secret Key)

### Configuration

Update your `.env` file:

```bash
# Langfuse connection
LANGFUSE_PUBLIC_KEY=your-public-key-from-langfuse-ui
LANGFUSE_SECRET_KEY=your-secret-key-from-langfuse-ui
LANGFUSE_HOST=http://day23-langfuse:3000
```

Or override via Docker Compose:

```bash
docker compose -f docker-compose.yml -f BONUS-llm-native-obs/docker-compose.langfuse.yml up -d
```

The app automatically:
- Creates Langfuse traces for each `/predict` request
- Records input prompts, output text, token counts, quality scores
- Tags traces with model name and service metadata
- Gracefully handles missing Langfuse (app still works, just no LLM traces)

### Using Langfuse with LangChain

Install the Langfuse Python SDK:

```bash
pip install langfuse
```

Instrument your LangChain/LlamaIndex code:

```python
from langfuse import Langfuse

langfuse = Langfuse(
    public_key="YOUR_PUBLIC_KEY",
    secret_key="YOUR_SECRET_KEY", 
    host="http://day23-langfuse:3000"
)

# For LangChain
from langchain.callbacks import Callbacks
langfuse_handler = langfuse.get_langchain_handler()
llm = YourLLM(callbacks=[langfuse_handler])
```

## Docker Usage

The `docker-compose.langfuse.yml` extends the main stack. Services:

| Service | Port | Description |
|---|---|---|
| `langfuse-postgres` | 5433 | PostgreSQL for Langfuse data |
| `langfuse-server` | 3001 | Langfuse web UI and API |

Both services join the existing `obs` network, so the app can reach Langfuse at `http://day23-langfuse:3000`.

## Makefile Targets

| Target | Description |
|---|---|
| `up-langfuse` | Start Langfuse services |
| `down-langfuse` | Stop Langfuse services |
| `smoke-langfuse` | Health check Langfuse |
| `logs-langfuse` | Tail Langfuse logs |

## Cleanup

```bash
# Remove Langfuse volumes (DESTRUCTIVE - loses all trace data)
docker compose -f docker-compose.yml -f BONUS-llm-native-obs/docker-compose.langfuse.yml down -v
```

## Features Enabled

- LLM tracing (latency, tokens, cost)
- Prompt/response logging
- Evaluations and scoring
- Datasets for testing
- Observability dashboards
- Project-based organization

## Generating Secure Keys

```bash
# NEXTAUTH_SECRET (32+ bytes)
openssl rand -hex 32

# ENCRYPTION_KEY (exactly 32 bytes)
openssl rand -hex 32
```
