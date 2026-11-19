# LiteLLM server

A local LiteLLM gateway for 10 AWS Bedrock models, two Jina AI models, and the managed AWS MCP server. Microsoft Entra provides dashboard SSO; PostgreSQL stores gateway data and Redis shares routing state. Docker Compose runs the services, and ngrok optionally provides HTTPS access from another computer.

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
└── mcp.yaml           # Managed AWS MCP server
scripts/               # Reserved for future tooling
```

The `config/` directory is mounted read-only. LiteLLM loads `config/config.yaml`, which includes `credentials.yaml`, `models.yaml`, and `mcp.yaml`. The routing and guardrails files remain unused placeholders. Models reference named credentials; secret values come from the container environment.

## 1. Prepare the environment

Install Docker with Compose v2 or newer and OpenSSL. For remote testing, also install and authenticate ngrok. Local Python, uv, and Make are not required to run this stack.

Create `.env` only if it does not already exist:

```bash
cp -n .env.example .env
chmod 600 .env
```

Fill in every empty value using the instructions in [.env.example](.env.example). Compose rejects missing or empty required values, including `JINA_AI_API_KEY`:

| Variables | Source |
| --- | --- |
| `LITELLM_MASTER_KEY`, `LITELLM_SALT_KEY` | Generate separate random values |
| `POSTGRES_USER`, `POSTGRES_DB`, `POSTGRES_PASSWORD` | Choose database settings and generate a password |
| `DATABASE_URL` | Construct from the PostgreSQL settings |
| `REDIS_PASSWORD` | Generate a separate random password |
| `MICROSOFT_CLIENT_ID`, `MICROSOFT_TENANT`, `MICROSOFT_CLIENT_SECRET` | Microsoft Entra app registration |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION_NAME` | AWS identity for Bedrock and AWS MCP; use `us-east-1` for this setup |
| `JINA_AI_API_KEY` | Jina AI API console |
| `PROXY_BASE_URL` | Browser-facing URL, without a trailing slash or path |

Compose reads `.env` and passes each service its declared variables. Keep real credentials out of Git. Preserve `LITELLM_SALT_KEY` with database backups: changing it prevents decryption of existing encrypted credentials.

### Temporary AWS credentials

The current Compose configuration passes an AWS access key and secret key. If these are temporary credentials, also set `AWS_SESSION_TOKEN` in `.env`, pass it through the LiteLLM service's `environment` in `compose.yaml`, and add `aws_session_token: os.environ/AWS_SESSION_TOKEN` to both the Bedrock credential values and the AWS MCP entry. Refresh all temporary credential values together and recreate the gateway container.

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

## Providers and models

Standard AWS Bedrock and Jina AI credentials are configured. Named credentials in [config/credentials.yaml](config/credentials.yaml) reference environment variables; model routes in [config/models.yaml](config/models.yaml) reference those credentials.

| Provider | Public model name | Purpose |
| --- | --- | --- |
| AWS Bedrock | `nova-2-lite` | Chat |
| AWS Bedrock | `nova-multimodal-embeddings` | Multimodal embeddings |
| AWS Bedrock | `gpt-5.6-sol` | Chat |
| AWS Bedrock | `gpt-5.6-terra` | Chat |
| AWS Bedrock | `gpt-5.6-luna` | Chat |
| AWS Bedrock | `claude-haiku-5-5` | Chat |
| AWS Bedrock | `claude-sonnet-5-5` | Chat |
| AWS Bedrock | `claude-opus-5-5` | Chat |
| AWS Bedrock | `cohere-embed-v4` | Embeddings |
| AWS Bedrock | `cohere-rerank-v3.5` | Reranking |
| Jina AI | `jina-embeddings-v5-text-small` | Text embeddings |
| Jina AI | `jina-reranker-v3.5` | Reranking |

Nova 2 Lite, GPT-5.6, and Claude 5.5 use US inference profiles and default to a 4,096-token output limit. Configure a supported US AWS region such as `us-east-1`. Nova and Cohere embeddings and Cohere reranking use regional model IDs. Cohere reranking requires a foundation-model ARN; this configuration pins its ARN and region to `us-east-1`. All 10 AWS model routes use the `bedrock/` prefix and the `aws-bedrock` credential; the two Jina routes use `jina_ai/`.

Use the public model names with `/v1/chat/completions`, `/v1/embeddings`, or `/v1/rerank`, depending on the model's purpose. Listing a model does not verify provider permissions, quotas, or successful inference.

Bedrock Mantle is not configured. Direct OpenAI, Anthropic, Cohere, and OpenRouter are not configured. Speech models are not configured.

## Model access groups

Each model has one `model_info.access_groups` entry in `config/models.yaml`. These are independent permission groups, not an inheritance hierarchy.

| Group | Public model names |
| --- | --- |
| `standard_models` | `nova-2-lite`, `gpt-5.6-luna`, `claude-haiku-5-5` |
| `reasoning_models` | `gpt-5.6-terra`, `claude-sonnet-5-5` |
| `premium_models` | `gpt-5.6-sol`, `claude-opus-5-5` |
| `embedding_models` | `nova-multimodal-embeddings`, `cohere-embed-v4`, `jina-embeddings-v5-text-small` |
| `reranker_models` | `cohere-rerank-v3.5`, `jina-reranker-v3.5` |

Grant the intended group names through a team's or virtual key's Models setting, or its `models` field in the management API. For example, a team can receive:

```json
{
  "team_alias": "developers",
  "models": ["standard_models", "embedding_models", "reranker_models"]
}
```

This is an example `/team/new` request body, not a YAML section. Team membership and key assignments are stored in the database and have not been provisioned by these files. Entra app roles control dashboard privileges; assigning an Entra role does not assign these model groups.

Group labels alone do not restrict existing unrestricted keys or teams. Review direct model grants, wildcard grants, and group grants together. Test access using a restricted virtual key, rather than the master key: an allowed model should succeed, and a model outside its permissions should be rejected.

See [Model Access Groups](https://docs.litellm.ai/docs/proxy/model_access_groups) and [LLM access](https://docs.litellm.ai/docs/proxy/access_management/llm).

## MCP servers

The managed AWS MCP server is configured in [config/mcp.yaml](config/mcp.yaml) at `https://aws-mcp.us-east-1.api.aws/mcp`. LiteLLM signs requests with SigV4 using the existing AWS environment credentials, service name `aws-mcp`, and signing region `us-east-1`.

This is a shared AWS identity, not the dashboard user's Microsoft identity. AWS tool operations are limited by that identity's IAM permissions. Bedrock permissions alone do not grant access to unrelated AWS services. Assign MCP access to the intended teams or keys separately from model access groups.

If using temporary AWS credentials, also pass `AWS_SESSION_TOKEN` through Compose and add `aws_session_token: os.environ/AWS_SESSION_TOKEN` to the MCP server entry. Configuration does not establish that the remote connection or tool execution has succeeded.

GitHub MCP is a useful optional addition for repositories, issues, and pull requests, but is not configured yet. It needs GitHub OAuth or an appropriately scoped access token.

References: [AWS MCP setup](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/getting-started-aws-mcp-server.html), [LiteLLM SigV4 configuration](https://docs.litellm.ai/docs/mcp_aws_sigv4), [GitHub MCP authentication](https://github.com/github/github-mcp-server/blob/main/docs/host-integration.md).

## Verification status

The configuration has been checked for valid YAML, unique model names, credential references, provider routing, and the MCP include. The five model groups cover all 12 models. These checks do not establish successful provider authentication or tool execution.

After filling `.env` and deploying the current files:

1. Confirm all three services are healthy and Microsoft SSO works.
2. Verify the model list contains all 12 public names.
3. Make a small request to each model using the appropriate endpoint; these calls can incur provider charges.
4. Assign groups to teams or keys and verify both allowed and denied requests.
5. Verify AWS MCP tool discovery and an intended read-only tool call before using operations that change AWS resources.

The current model routes, group permissions, and AWS MCP connection have not been verified end to end. Previous successful deployment checks used an earlier model configuration.

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

With this LiteLLM version, first-start checks can also log `relation does not exist` for `_prisma_migrations` and reporting views before creating them. These messages can occur during initial schema setup; check that initialization finishes and the services become healthy.

If missing-relation errors continue after initialization, or LiteLLM fails its health check, inspect the LiteLLM logs for migration failures. Do not assume persistent errors are harmless or delete the database as the first troubleshooting step.

## References

- [LiteLLM Docker quickstart](https://docs.litellm.ai/docs/proxy/docker_quick_start)
- [LiteLLM configuration](https://docs.litellm.ai/docs/proxy/configs)
- [LiteLLM SSO permissions](https://docs.litellm.ai/docs/proxy/access_management/sso_permissions)
- [LiteLLM production and Redis guidance](https://docs.litellm.ai/docs/proxy/prod)
