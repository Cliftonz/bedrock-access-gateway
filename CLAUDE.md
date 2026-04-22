# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

OpenAI-compatible REST proxy in front of Amazon Bedrock. FastAPI app translates OpenAI `/chat/completions`, `/embeddings`, and `/models` calls into Bedrock Converse / InvokeModel calls, returning OpenAI-shaped responses (including SSE streaming). Ships as a container for AWS Lambda (behind API Gateway with response streaming) or ECS/Fargate (behind ALB).

## Common commands

Run locally (from `src/`):

```bash
API_KEY=dev-key AWS_REGION=us-west-2 uvicorn api.app:app --host 0.0.0.0 --port 8000
```

Base URL is then `http://localhost:8000/api/v1`. Auth requires `Authorization: Bearer $API_KEY`.

Docker Compose (builds `Dockerfile_ecs`, maps host 8000 → container 8080, mounts `~/.aws`):

```bash
OPENAI_API_KEY=dev-key docker-compose up --build
```

Build images directly:

```bash
# Lambda image (uses AWS Lambda Web Adapter for SSE over API Gateway)
docker build -f src/Dockerfile -t bedrock-proxy-api:lambda src/
# ECS/Fargate image (plain uvicorn, non-root user, HEALTHCHECK)
docker build -f src/Dockerfile_ecs -t bedrock-proxy-api:ecs src/
```

Push both to ECR via `scripts/push-to-ecr.sh` (interactive; builds `linux/arm64` with `--provenance=false --sbom=false` — required for Lambda, see issue #206).

Lint / format (ruff, `ruff.toml`, py312, line 120):

```bash
ruff check src/
ruff format src/
```

No test suite currently lives in the repo.

## Architecture

Entry point is `src/api/app.py` — a FastAPI `app` with `Mangum(app)` exported as `handler` for Lambda. Routers mount under `API_ROUTE_PREFIX` (default `/api/v1`):

- `routers/chat.py` → `POST /chat/completions` (stream + non-stream)
- `routers/embeddings.py` → `POST /embeddings`
- `routers/model.py` → `GET /models`, `GET /models/{id}`
- `GET /health` is unauthenticated; everything under the routers depends on `api.auth.api_key_auth`.

All request/response shapes live in `api/schema.py` (OpenAI-shaped Pydantic models). Abstract interfaces `BaseChatModel` / `BaseEmbeddingsModel` are in `api/models/base.py`. The only concrete implementation is `api/models/bedrock.py` — this is where OpenAI → Bedrock translation happens (message/tool conversion, streaming adapter, cross-region & application inference profile resolution, prompt caching, reasoning / interleaved thinking for Claude 4/4.5 and DeepSeek R1). New model families or behaviors almost always land here.

Config is env-driven in `api/setting.py`: `API_ROUTE_PREFIX`, `AWS_REGION`, `DEFAULT_MODEL`, `DEFAULT_EMBEDDING_MODEL`, `ENABLE_CROSS_REGION_INFERENCE`, `ENABLE_APPLICATION_INFERENCE_PROFILES`, `ENABLE_PROMPT_CACHING`, `DEBUG`. Plus `ALLOWED_ORIGINS` (CORS, in `app.py`) and auth inputs in `api/auth.py`: one of `API_KEY_SECRET_ARN` (Secrets Manager, preferred), `API_KEY_PARAM_NAME` (SSM, legacy), or `API_KEY` (plain env). App fails fast on import if none is set.

The two Dockerfiles are not interchangeable: `Dockerfile` copies the Lambda Web Adapter into `/opt/extensions` and clears `ENTRYPOINT` so uvicorn runs under the Lambda runtime; `Dockerfile_ecs` is a plain Python 3.13-slim image with a non-root user and HEALTHCHECK. Both preload the tiktoken `cl100k_base` encoding at build time (issue #118) — keep that step when editing either file.

CloudFormation in `deployment/` matches the two deploy modes: `BedrockProxy.template` (API Gateway + Lambda, pairs with `Dockerfile`) and `BedrockProxyFargate.template` (ALB + Fargate, pairs with `Dockerfile_ecs`). Both take `ApiKeySecretArn` and `ContainerImageUri` parameters.

## Gotchas

- When bumping or adding models with temperature/top_p quirks, update `TEMPERATURE_TOPP_CONFLICT_MODELS` in `api/models/bedrock.py` (see commit `e5f7d3d`).
- `additionalModelRequestFields` must be merged, not overwritten, when composing Bedrock requests (see commit `6ae73c0`).
- Streaming responses with reasoning tokens have model-specific event shapes — look at existing branches in `bedrock.py` before adding a new one.
- Lambda images must stay Docker V2 Schema 2 (no OCI attestations) or they won't deploy — preserve `--provenance=false --sbom=false` on any build that targets Lambda.
