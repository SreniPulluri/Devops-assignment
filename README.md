# DevOps Assessment: FastAPI + Celery + Redis on AWS

A FastAPI backend, a Celery worker and Redis, containerised with Docker, runnable locally with one
Compose command, and deployed to AWS (ECS Fargate + ElastiCache) with Terraform and GitHub Actions.

![architecture](docs/architecture.png)

| Deliverable | Where |
|---|---|
| Dockerfiles (env-var configuration, non-root) | `backend/Dockerfile`, `backend/Dockerfile.worker` |
| Local stack (API + worker + Redis) | `docker-compose.yml`, `.env.example` |
| Local run guide (build, start, logs, debug) | [`instructions.md`](instructions.md) |
| Infrastructure as Code | `terraform/` |
| CI/CD | `.github/workflows/ci-cd.yml` |
| Decisions and challenges | [`journey.md`](journey.md) |
| Architecture diagram | `docs/architecture.png` (+ Mermaid in `docs/architecture.md`) |
| Frontend | `frontend/index.html` |

## The application
| Component | File | Role |
|---|---|---|
| Backend | `backend/main.py` | `POST /notify/` enqueues a Celery task; `GET /task_status/{task_id}` returns status/result; `/health`, `/ready` for probes |
| Worker | `backend/worker.py` | Celery task `tasks.notify` (simulated job with progress updates) |
| Redis | managed service | Celery broker **and** result backend |
| Frontend | `frontend/index.html` | Triggers a task, polls status every second, shows live status + result; served by the backend at `/` |

## Quick start (local)
```bash
cp .env.example .env
docker compose up --build        # http://localhost:8000
```
Full details (logs, debugging, scaling workers, clean-up): [`instructions.md`](instructions.md).

## Deploy (summary)
Push to `main` → GitHub Actions runs tests and `terraform validate`, creates the ECR repos, builds and pushes
both images tagged with the commit SHA, applies Terraform with that tag, waits for ECS to stabilise and
smoke-tests the live URL (enqueue a task, poll until `SUCCESS`). One-time AWS setup (state bucket, OIDC role,
repo variables) is in [`instructions.md`](instructions.md#deploying-to-aws).

## Configuration
Everything is environment-driven; nothing is hardcoded.

| Variable | Used by | Default |
|---|---|---|
| `REDIS_URL` | backend, worker | `redis://localhost:6379/0` (`rediss://` enables TLS verification) |
| `CELERY_BROKER_URL`, `CELERY_RESULT_BACKEND` | backend, worker | fall back to `REDIS_URL` |
| `TASK_DURATION_SECONDS` | worker | `5` |
| `CELERY_CONCURRENCY` | worker | `2` |
| `CORS_ORIGINS` | backend | empty (same-origin frontend needs none) |
| `FRONTEND_DIR`, `PORT`, `LOG_LEVEL` | backend | set in the image |
