# call-center-ai — working notes

FastAPI service (granian) + Redis, Python 3.13 managed by `uv`. Everything below is
what a fresh session needs to know that the manifests do not say.

## Running in the Base44 sandbox

- `docker compose -f docker-compose.base44.yml up -d --build` — `api` on host port 3000
  (granian `--reload`, source bind-mounted) and `cache` (Redis, used as the cache mode).
- The API has **no root route**. When `BASE44_PREVIEW_MODE=1`, `app/main.py` registers
  `/` as a redirect to `/docs` so the preview lands on something useful. Unset → original
  404 behavior.
- `config.yaml` is mandatory at import time (`app/helpers/config.py`) and is gitignored.
  The sandbox mounts `.base44/base44-config.yaml` (placeholders) at `/app/config.yaml`.
  Do not create a `config.yaml` at the repo root.
- Every config value can be overridden by a nested env var using `__`
  (`RootModel` is `BaseSettings`, env has priority over the YAML), e.g.
  `COMMUNICATION_SERVICES__ACCESS_KEY`, `DATABASE__COSMOS_DB__ENDPOINT`.
  Real credentials come from `/run/base44/app.env`.
- `app/helpers/resources.py` resolves templates from `$PWD/app`, so the process must
  run with the repository root as working directory.
- Dependencies are installed at image build time into `/opt/venv`
  (`UV_PROJECT_ENVIRONMENT`) and re-synced from `uv.lock` on container start; the
  `.dockerignore` only lets `app/`, `pyproject.toml`, `uv.lock` and the Base44
  Dockerfile into the build context.

## What works without Azure

The whole call flow is Azure-bound. With the placeholder config:

- `/health/liveness` → 200, `/docs` → 200, `/openapi.json` → 200.
- `/health/readiness` → 503: `cache` and `sms` report `ok`, `store` (Cosmos DB) and
  `ai_search` fail. Expected, not a regression.
- `/report` → 500 and `POST /call` fail: Cosmos DB / Communication Services are
  unreachable. Real data needs Cosmos DB, ACS, Azure OpenAI, AI Search, Speech,
  Translation, Storage Queues and App Configuration credentials.

## Checks

- Lint: `docker compose -f docker-compose.base44.yml exec -T api uv run --frozen ruff check --select I,PL,RUF,UP,ASYNC,A,DTZ,T20,ARG,PERF --ignore RUF012,A005`
- Unit tests (`uv run --frozen pytest tests/*.py`, with `PUBLIC_DOMAIN=dummy`) do not
  collect at this commit: pytest-asyncio raises `MultipleEventLoopsRequestedError` for
  `tests/llm.py`, `tests/search.py`, `tests/store.py`. Pre-existing, unrelated to the
  environment.
- `pyright` cannot run in the dev image (it needs Node/Python 3.13 host bootstrap);
  ruff is the usable static check here.
