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
├── models.yaml        # Empty placeholder
├── routing.yaml       # Empty placeholder
├── guardrails.yaml    # Empty placeholder
└── mcp.yaml           # Empty placeholder
scripts/               # Reserved for future tooling
```

The `config/` directory is mounted read-only. Only `config/config.yaml` is loaded; sibling YAML files are not automatically merged. Models are currently unconfigured (`model_list: []`), so dashboard login can be tested before adding providers.

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

## Everyday commands

| Command | Action |
| --- | --- |
| `docker compose ps` | Show service status |
| `docker compose logs -f --tail=100` | Follow recent service logs |
| `docker compose logs -f --tail=100 litellm` | Follow gateway logs |
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
