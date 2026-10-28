# Crawler Backend

FastAPI service that crawls websites and turns them into searchable knowledge bases, with chat, quizzes, and learning paths on top. Real-time crawl progress is pushed over Socket.IO.

**Stack:** FastAPI · MongoDB · Redis · Qdrant (vector store) · Groq LLMs via LangChain · HuggingFace sentence-transformers embeddings

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) with Docker Compose (needed for both options below)
- Python 3.11 (only for running the API outside Docker; see `.python-version`)
- A [Groq API key](https://console.groq.com/keys)

## 1. Configure environment

```bash
cp .env.example .env
```

Then edit `.env` and set at least:

| Variable | Description |
| --- | --- |
| `MONGODB_URL` | Your MongoDB connection string (e.g. a MongoDB Atlas `mongodb+srv://...` URL). Compose does not run MongoDB |
| `GROQ_API_KEY` | Your Groq API key (required for chat/quiz generation) |
| `SECRET_KEY` | Secret used to sign JWTs; use a long random string |

The Redis and Qdrant defaults in `.env.example` point at the local containers and work as-is.

## 2a. Run everything with Docker Compose (recommended)

```bash
docker compose up --build
```

This starts Redis (`6379`), Qdrant (`6333`), and the backend on **http://localhost:3000**. The backend connects to the MongoDB in `MONGODB_URL` from `.env`; compose only overrides the Redis and Qdrant URLs so the backend reaches those containers by service name.

> If `MONGODB_URL` points at a MongoDB on your own machine, use `mongodb://host.docker.internal:27017` instead of `localhost` — inside the container, `localhost` is the container itself. On Linux this also needs `extra_hosts: ["host.docker.internal:host-gateway"]` on the backend service, and `mongod` must listen on more than `127.0.0.1`.

Stop with `Ctrl+C`, or `docker compose down` (add `-v` to also delete the Qdrant data volume).

## 2b. Run the API locally (for development)

Start only the backing services in Docker:

```bash
docker compose up -d redis qdrant
```

Create a virtual environment and install dependencies:

```bash
python3.11 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Run the server with auto-reload:

```bash
uvicorn main:app --reload --port 3000
```

The first request that needs embeddings downloads the sentence-transformers model, so it may be slow.

## 3. Verify it's running

- Swagger UI: http://localhost:3000/docs
- ReDoc: http://localhost:3000/redoc

Register a user with `POST /users/register`, then log in with `POST /users/login` and pass the returned token as `Authorization: Bearer <token>` on the protected routes (`/scrape`, `/chat`, `/collections`, `/quizzes`).

## Optional settings

| Variable | Default | Description |
| --- | --- | --- |
| `LLM_MODEL` | `llama-3.3-70b-versatile` | Groq model used for generation |
| `CRAWL_ENGINE` | `http` | `http` (lightweight fetcher) or `crawl4ai` (headless browser; run `crawl4ai-setup` once to install Playwright browsers) |
| `CRAWL_RUN_TIMEOUT_SECONDS` | `300` | Max time for a single crawl job |
| `CRAWL_PAGE_TIMEOUT_MS` | `45000` | Per-page fetch timeout |
| `QDRANT_API_KEY` | _(empty)_ | Needed only for a hosted/secured Qdrant |
| `REDIS_URL` | — | Leave empty to run without Redis |

## Project layout

```
main.py              FastAPI app, routers, Socket.IO mount
app/routes/          HTTP routes (users, scrape, chat, collections, quizzes)
app/crawler.py       Website crawling
app/vector_store.py  Qdrant embeddings + search
app/llm_service.py   Groq/LangChain LLM calls
app/*_store.py       MongoDB persistence
app/models/          Pydantic models
```

## Deployment

Pushes to `main` run CI (`.github/workflows/backend-ci.yml`) and, on success, deploy to EC2 by running `docker compose up -d --build` on the server.
