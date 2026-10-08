# HomeAgents core

This repository holds the core of the HomeAgents platform, a self-hosted AI
setup for a single home. Everything runs with Docker Compose on one machine.

The repository contains the always-on infrastructure and the home-facing
application stacks. Each stack is a separate compose file, so you can start only
the parts you need.

## What runs here

| Folder | Services | Published on |
|---|---|---|
| `core/` | LiteLLM gateway, Qdrant, TEI embeddings, `qdrant-mcp`, Postgres, Redis | 4000, 6333, 8081, 8765 |
| `web-search/` | SearXNG and `web-search-mcp` (SearXNG and Tavily tools) | 8082 |
| `finance/` | Firefly III MCP server | 3002 |
| `programming/` | OpenHands coding agent | 3000 |

Every container joins one shared external Docker network named
`ai-platform-net`. Containers therefore reach each other by service name, for
example `litellm:4000`, `qdrant:6333` and `embeddings:80`.

## Companion repositories

Clone these side by side. The CLI and the stage scripts expect this layout:

```
<repos root>/
  homeagents-core/             this repository
  homeagents-cli/              the `homeagents` command
  dify-deployment/             Dify
  librechat-deployment/        LibreChat
  paperless-qdrant-ingestion/  Paperless-ngx to Qdrant worker
  homeagents-mcp-servers/      sources for the qdrant-mcp and web-search-mcp images
```

## Prerequisites

* Docker Desktop, or Docker Engine with Compose v2
* PowerShell 5.1 or newer (only for the optional CLI)
* An `.env` file. Copy `.env-example` and fill it in.

## Quick start

```powershell
# 1. Configuration
Copy-Item .env-example .env        # then edit .env

# 2. Build the two custom MCP images (sources in the sibling repository)
docker build -t qdrant-mcp:latest        ../homeagents-mcp-servers/qdrant
docker build -t web-search-mcp:latest    ../homeagents-mcp-servers/websearch

# 3. Create the shared network (idempotent)
docker network create ai-platform-net

# 4. Start the stacks you want
docker compose --env-file .env -f core/memory.yml up -d
docker compose --env-file .env -f core/litellm.yml up -d
docker compose --env-file .env -f web-search/websearch.yml up -d
```

The `homeagents` CLI wraps all of this:

```powershell
homeagents core run
homeagents websearch run
```

## Ports

| Port | Service |
|---|---|
| 3000 | OpenHands |
| 3002 | Firefly III MCP |
| 3080 | LibreChat (`librechat-deployment`) |
| 4000 | LiteLLM |
| 6333 | Qdrant |
| 8081 | TEI embeddings |
| 8082 | SearXNG |
| 8085 | Dify console (`dify-deployment`) |
| 8765 | qdrant-mcp |

Published ports bind to `127.0.0.1`, so nothing is exposed to the LAN by default.

## Data

Runtime data lives under `VOLUMES/` as bind mounts and is git-ignored:

```
VOLUMES/qdrant_data/       Qdrant vector storage
VOLUMES/postgres_data/     LiteLLM database
VOLUMES/redis_data/        Redis AOF/RDB
VOLUMES/tei_cache/         downloaded embedding model
VOLUMES/openhands_data/    OpenHands conversations
VOLUMES/librechat*/        LibreChat uploads and MongoDB data
```

Back up `qdrant_data/` and `postgres_data/` at minimum. Deleting `VOLUMES/`
resets the platform.

## Model routing

`core/config/litellm/config.yaml` declares virtual models, so callers never name
a provider directly:

| Virtual model | Backing model | Use |
|---|---|---|
| `fast` | DeepSeek V4 Flash, thinking off | everyday chat |
| `coder` | DeepSeek V4 Flash, thinking on | development |
| `architect` | GLM-5.3 | architecture and long-horizon planning |
| `premium` | Claude Sonnet 5 | occasional expensive escalation |
| `openai-fallback` | GPT-5.6 Luna | OpenAI option |

Budgets are enforced on three levels, per virtual model, per provider and by the
prepaid provider balances themselves. Treat the numbers in that file as starting
points and tune them to your own spending.

## Which parts call a model

LiteLLM is the only piece that holds provider keys and it forwards every chat
request onward. The embedding service next to Qdrant is a model too, but a local
one: it indexes memory on CPU, so recall costs no API credit. OpenHands is a
coding agent that talks to LiteLLM like anything else. Of the remaining
services, SearXNG, the Firefly III MCP server, Postgres and Redis never call a
model themselves.

## Security notes

* `.env` is git-ignored. Only `.env-example` with placeholders is committed.
* Provider API keys go to the LiteLLM container only. LibreChat talks to LiteLLM
  with a single master key.
* `web-search/config/searxng/settings.yml` ships a placeholder secret. The real
  value comes from `SEARXNG_SECRET` in `.env`, which SearXNG prefers over the
  file.
* `VOLUMES/` and TLS material are git-ignored.
