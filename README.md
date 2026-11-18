# LiteLLM server

A local LiteLLM gateway with Microsoft Entra SSO, PostgreSQL, and Redis. Docker Compose runs the services; ngrok optionally provides HTTPS access from another computer.

## Stack

| Service | Container | Image | Purpose |
| --- | --- | --- | --- |
| LiteLLM | `litellm` | `ghcr.io/berriai/litellm:v1.110.0` | Gateway and SSO dashboard |
| PostgreSQL | `litellm-db` | `postgres:16.16` | Users, keys, configuration, and spend data |
| Redis | `litellm-redis` | `redis:7.4.12` | Shared coordination and routing state |

All services use the project's `litellm_network` bridge network. Only LiteLLM publishes a host port: `127.0.0.1:4000`. PostgreSQL and Redis are available within the Docker network at `db:5432` and `redis:6379`.

LiteLLM runs one worker. Redis authentication and append-only persistence are enabled. Model response caching is not enabled.

## Project layout

```text
compose.yaml
.env.example
.env                   # Local secrets; ignored by Git
docs/entra_sso.md      # Microsoft application and role setup
config/
├── config.yaml        # Loaded by LiteLLM
├── credentials.yaml   # Provider credentials referenced from environment variables
├── models.yaml        # Chat, embedding, and reranking models
├── routing.yaml       # Empty placeholder
├── guardrails.yaml    # Empty placeholder
└── mcp.yaml           # Empty placeholder
scripts/               # Reserved for future tooling
```

The `config/` directory is mounted read-only. LiteLLM loads `config/config.yaml`, which includes `credentials.yaml` and `models.yaml`. The routing, guardrails, and MCP files remain unused placeholders. Models reference named credentials; secret values come from the container environment.

## 1. Prepare the environment

Install Docker with Compose v2 and OpenSSL. For remote testing, also install and authenticate ngrok. Local Python, uv, and Make are not required to run this stack.

Create `.env` only if it does not already exist:

```bash
cp -n .env.example .env
chmod 600 .env
```

Fill in every empty value using the instructions in [.env.example](.env.example):

| Variables | Source |
| --- | --- |
| `LITELLM_MASTER_KEY`, `LITELLM_SALT_KEY` | Generate separate random values |
| `POSTGRES_USER`, `POSTGRES_DB`, `POSTGRES_PASSWORD` | Choose database settings and generate a password |
| `DATABASE_URL` | Construct from the PostgreSQL settings |
| `REDIS_PASSWORD` | Generate a separate random password |
| `MICROSOFT_CLIENT_ID`, `MICROSOFT_TENANT`, `MICROSOFT_CLIENT_SECRET` | Microsoft Entra app registration |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION_NAME` | AWS credentials and region for standard Bedrock Runtime |
| `JINA_AI_API_KEY`, `OPENAI_API_KEY`, `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`, `COHERE_API_KEY` | Respective provider API consoles |
| `PROXY_BASE_URL` | Browser-facing URL, without a trailing slash or path |

Compose reads `.env` and passes each service its declared variables. Keep real credentials out of Git. Preserve `LITELLM_SALT_KEY` with database backups: changing it prevents decryption of existing encrypted credentials.

### Database connection URL

`DATABASE_URL` is supplied explicitly; Compose does not construct or URL-encode it:

```dotenv
DATABASE_URL=postgresql://<user>:<encoded-password>@db:5432/<database>
```

Percent-encode special characters in URL components. For example, the password `Example@02` becomes `Example%4002` inside the URL. Keep the original password in `POSTGRES_PASSWORD`. Changing `.env` does not change the password of an already initialized PostgreSQL user.

## 2. Configure Microsoft SSO

Follow [Microsoft SSO setup](docs/entra_sso.md) to create **LiteLLM_DEV**, configure delegated `User.Read`, create app roles, and assign people under Enterprise applications.

| Role | Entra app-role value |
| --- | --- |
| Platform administrator | `proxy_admin` |
| Platform viewer | `proxy_admin_viewer` |
| Normal developer | `internal_user` |
| Personal viewer | `internal_user_viewer` |

Assign the administrator account `proxy_admin`. LiteLLM reads the assigned role at SSO login; sign out and sign in again after changing assignments. New users are created on first successful login, subject to access rules and license limits.

Dashboard password login is disabled. The master key remains available for management API authentication.

For local login, use `PROXY_BASE_URL=http://localhost:4000` and register this **Web** redirect URI:

```text
http://localhost:4000/sso/callback
```

## 3. Optional: enable remote access with ngrok

Start the tunnel in a separate terminal:

```bash
ngrok http 4000
```

Set `PROXY_BASE_URL` in `.env` to the HTTPS forwarding URL printed by ngrok. In Entra, add that same address followed by `/sso/callback` as a Web redirect URI.

For example:

```dotenv
PROXY_BASE_URL=https://your-domain.ngrok-free.dev
```

```text
https://your-domain.ngrok-free.dev/sso/callback
```

Keep ngrok and the host computer running while testing remotely. The free-plan browser warning may require clicking **Visit Site**.

## 4. Start and verify

From the project root:

```bash
docker compose config --quiet
docker compose up -d --wait --wait-timeout 180
docker compose ps
```

All three services should become healthy. Open `<PROXY_BASE_URL>/ui`, sign in with Microsoft, and verify your role in the profile menu.

First startup downloads the images and initializes the database. LiteLLM waits for PostgreSQL and Redis health checks before starting.

## Configured models

Use the public model name in API requests. Provider routes and credential references are defined in [config/models.yaml](config/models.yaml).

| Provider | Public model name | Purpose |
| --- | --- | --- |
| AWS Bedrock | `nova-2-lite` | Chat |
| AWS Bedrock | `nova-multimodal-embeddings` | Multimodal embeddings |
| Jina AI | `jina-embeddings-v5-text-small` | Text embeddings |
| Jina AI | `jina-reranker-v3.5` | Reranking |
| OpenAI | `gpt-5-mini` | Chat |
| OpenAI | `text-embedding-3-small` | Text embeddings |
| OpenAI | `text-embedding-3-large` | Text embeddings |
| Anthropic | `claude-haiku-5-5` | Chat |
| Anthropic | `claude-sonnet-5-5` | Chat |
| Cohere | `cohere-embed-v5.0-pro` | Embeddings |
| Cohere | `cohere-rerank-v5.0-pro` | Reranking |

Chat models use `/v1/chat/completions`, embedding models use `/v1/embeddings`, and rerankers use `/v1/rerank`. Embedding input formats and query/document parameters vary by provider; use the same embedding model and dimensions when indexing and querying a vector collection.

Nova uses standard Bedrock Runtime. Nova 2 Lite uses the US inference profile and defaults to a 4,096-token output limit; configure a supported US AWS region such as `us-east-1`. Nova multimodal embeddings uses its regional model ID directly.

OpenRouter credentials are available, but no OpenRouter models are configured. Speech models are not configured.

GPT-5 mini is retained as requested. OpenAI lists its shutdown date as December 11, 2026; plan its replacement before then. See [OpenAI deprecations](https://developers.openai.com/api/docs/deprecations).

The deployment was verified to load all 11 models. Listing a model does not verify provider credentials, account access, quotas, or successful inference; those require a request to each provider.

## Everyday commands

| Command | Action |
| --- | --- |
| `docker compose ps` | Show service status |
| `docker compose logs -f --tail=100` | Follow recent service logs |
| `docker compose logs -f --tail=100 litellm` | Follow gateway logs |
| `docker compose up -d --force-recreate litellm` | Recreate the gateway with current configuration and environment |
| `docker compose restart litellm` | Reload changes to the mounted LiteLLM configuration |
| `docker compose up -d --wait --wait-timeout 180` | Apply Compose or `.env` changes |
| `docker compose down` | Remove containers and network while retaining data volumes |

A container restart does not apply changed environment variables; use `docker compose up -d` for those changes. An unhealthy status alone does not cause Docker's restart policy to restart a container.

## Reset the deployment

To intentionally erase this project's PostgreSQL and Redis data:

```bash
docker compose down --volumes --remove-orphans
```

This deletes stored LiteLLM users, keys, database configuration, spend history, and Redis state. Local files and `.env` remain. Run the startup commands again to initialize a fresh deployment.

Use ordinary `docker compose down` for routine shutdowns. A Docker-wide prune is unnecessary for resetting this project and can delete unrelated resources.

## First-start database messages

After deleting the database volume, PostgreSQL initializes a temporary server, stops it, and starts the normal server. That shutdown sequence is expected.

With this LiteLLM version, first-start checks can also log `relation does not exist` for `_prisma_migrations` and reporting views before creating them. These messages occurred in our fresh-start tests; all referenced objects subsequently existed and the services became healthy.

If missing-relation errors continue after initialization, or LiteLLM fails its health check, inspect the LiteLLM logs for migration failures. Do not assume persistent errors are harmless or delete the database as the first troubleshooting step.

## References

- [LiteLLM Docker quickstart](https://docs.litellm.ai/docs/proxy/docker_quick_start)
- [LiteLLM configuration](https://docs.litellm.ai/docs/proxy/configs)
- [LiteLLM SSO permissions](https://docs.litellm.ai/docs/proxy/access_management/sso_permissions)
- [LiteLLM production and Redis guidance](https://docs.litellm.ai/docs/proxy/prod)
