# Journey: decisions and challenges

## 1. Containerising
- **One image per process type.** `Dockerfile` (API) and `Dockerfile.worker` (Celery) are built from the same slim Python base but ship only what each needs; the worker image has no web code or frontend. They are built from the repo root so the API image can include `frontend/`.
- **Config via environment only.** The original hardcoded Redis location became `REDIS_URL` (with optional `CELERY_BROKER_URL` / `CELERY_RESULT_BACKEND` overrides). The same image therefore runs unchanged in Compose (`redis://redis:6379/0`) and on AWS (`rediss://<elasticache>:6379/0`).
- **Small, safe images:** `python:3.12-slim`, pinned requirements, `--no-cache-dir`, requirements copied before source for layer caching, non-root user, no `.env` baked in (`.dockerignore`).
- **Health checks:** `/health` is liveness (process up, used by the ALB and ECS), `/ready` checks the broker. I deliberately kept the broker out of the liveness check: a Redis blip should not make the load balancer kill every API task.

## 2. Local stack
Compose runs Redis, API and worker with `depends_on: condition: service_healthy`, so the API never starts before Redis is ready. Redis persistence is off locally because it only holds transient Celery state. The worker has a real health check (`celery inspect ping`) rather than "process exists". `docker compose up --scale worker=3` demonstrates horizontal scaling.

## 3. Cloud architecture and why
| Choice | Reasoning |
|---|---|
| **ECS Fargate** | No servers to patch, per-task billing, first-class ALB integration; the app is just two containers, so Kubernetes would be overhead. |
| **ElastiCache Redis (managed)** | Brief asked for managed over self-hosted. TLS in transit + encryption at rest, reachable only from the app security group. |
| **ALB → API in private subnets** | Only the load balancer is internet-facing; tasks have no public IPs. |
| **Frontend served by the API** | The page and the API share an origin, so no CORS, no extra service/bucket/CDN to deploy. (If the frontend is split out later, `CORS_ORIGINS` and `?api=` already support it.) |
| **Single NAT gateway** | Tasks need outbound access for ECR/CloudWatch. One NAT is the cost-aware dev default; production should use one per AZ (or VPC endpoints). |
| **Empty task role, minimal execution role** | The app calls no AWS APIs, so it gets no permissions. |
| **Autoscaling on CPU** | Simple and robust. For bursty queues, queue-length-based scaling (custom CloudWatch metric) would track load better; noted as a next step. |

## 4. Infrastructure as Code
Terraform, split by concern (`network`, `ecr`, `redis`, `alb`, `ecs`, `monitoring`). State lives in S3 with native locking (`use_lockfile`, Terraform ≥ 1.10) so no DynamoDB table is needed. Everything tunable is a variable; the image tag is a required variable so a deploy is always an explicit, immutable artefact. ECR tags are immutable and set to the git SHA, which makes every running task traceable to a commit.

## 5. CI/CD
- **Pull requests:** tests, `terraform validate`, and a build of both images (no push).
- **Push to main:** create ECR → build/push → `terraform apply` → wait for stability → **smoke test that exercises the real flow** (enqueue a task and poll until `SUCCESS`), which verifies API, Redis and worker together, not just that a port is open.
- **Auth:** GitHub OIDC assumes an IAM role; no long-lived AWS keys in GitHub.
- **Chicken-and-egg:** images can't be pushed before ECR exists, and ECS can't start before images exist. Solved with a targeted first apply (`-target=aws_ecr_repository.repo`) and then a full apply.
- **Safety:** concurrency group prevents overlapping applies; ECS deployment circuit breaker rolls back a bad release.

## 6. Observability
Container logs go to CloudWatch Logs (30-day retention) via `awslogs`; Container Insights is on. Alarms: ALB 5xx, unhealthy targets, ECS CPU, Redis CPU, and a **log metric filter on Celery's "raised unexpected" line**, so background failures alert even when the API looks healthy.

## 7. Challenges and how I handled them
- **TLS to Redis.** ElastiCache with in-transit encryption needs `rediss://`, and Celery refuses a `rediss://` URL without an explicit certificate policy. `worker.py` sets `ssl_cert_reqs=CERT_REQUIRED` for broker and result backend whenever the scheme is `rediss://`, keeping verification on instead of disabling it.
- **Graceful worker shutdown.** On deploys ECS sends SIGTERM; Celery finishes in-flight tasks, so the worker task definition sets `stopTimeout = 60`.
- **`PENDING` ambiguity.** Celery reports unknown task ids as `PENDING`. The task reports `PROGRESS` as it runs and the frontend caps polling (2 minutes) and surfaces errors rather than spinning forever.
- **Terraform vs autoscaling fighting over `desired_count`.** Fixed with `lifecycle.ignore_changes` on the services.

## 8. What I'd do next
HTTPS with a custom domain (the `certificate_arn` variable already enables it), secrets via Secrets Manager once there are any, WAF on the ALB, VPC endpoints instead of NAT, multi-AZ Redis with failover, queue-depth autoscaling, Trivy image scanning in CI, and separate staging/prod workspaces with a manual approval gate.
