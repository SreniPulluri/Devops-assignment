# Developer instructions

## Prerequisites
Docker Engine 24+ with the Compose plugin (`docker compose version`). Nothing else is needed to run the stack.

## Run the stack locally
```bash
cp .env.example .env                 # optional: defaults work without it

docker compose build                 # 1. build both images (backend + worker)
docker compose up -d                 # 2. start redis, backend, worker
docker compose ps                    # 3. wait until all services are "healthy"
```
Open **http://localhost:8000**, click **Run task**, and watch the status go PENDING → PROGRESS → SUCCESS.
(One command alternative: `docker compose up --build`.)

API from the command line:
```bash
curl -s -X POST localhost:8000/notify/ -H 'content-type: application/json' -d '{"message":"hello"}'
# {"task_id":"3f0c…","status":"PENDING"}
curl -s localhost:8000/task_status/<task_id>
curl -s localhost:8000/health ; curl -s localhost:8000/ready
```

## View logs
```bash
docker compose logs -f                    # everything
docker compose logs -f worker             # only the Celery worker (shows task received / succeeded)
docker compose logs --since 5m backend
```

## Debug
| Symptom | Check |
|---|---|
| Task stays `PENDING` | Worker not running or can't reach Redis: `docker compose ps`, `docker compose logs worker`. Unknown task ids are also `PENDING` by Celery design. |
| `503 task queue unavailable` | Redis down/unhealthy: `docker compose logs redis`; `curl localhost:8000/ready`. |
| Code change not picked up | `docker compose up -d --build backend worker` (images are immutable; no bind mounts). |
| Port 8000 in use | Set `API_PORT=8080` in `.env`. |

```bash
docker compose exec backend sh                                  # shell in the API container
docker compose exec redis redis-cli                             # inspect the broker (KEYS *, LLEN celery)
docker compose exec worker celery -A worker.celery_app inspect active
docker compose exec worker celery -A worker.celery_app inspect stats
docker compose up -d --scale worker=3                           # more workers; tasks spread across them
docker stats                                                    # CPU / memory per container
```

## Tests
```bash
python -m venv .venv && . .venv/bin/activate
pip install -r backend/requirements-dev.txt
python -m pytest backend/tests -q
```

## Stop / clean up
```bash
docker compose down            # stop and remove containers
docker compose down -v --rmi local   # also remove volumes and locally built images
```

---

## Deploying to AWS

### One-time setup
1. **State bucket** (versioned, private):
   ```bash
   aws s3api create-bucket --bucket <unique-state-bucket> --region ap-south-1 \
     --create-bucket-configuration LocationConstraint=ap-south-1
   aws s3api put-bucket-versioning --bucket <unique-state-bucket> --versioning-configuration Status=Enabled
   ```
2. **GitHub → AWS trust (OIDC, no static keys).** Add the GitHub OIDC identity provider
   (`token.actions.githubusercontent.com`, audience `sts.amazonaws.com`) and create an IAM role with this trust policy:
   ```json
   {"Version":"2012-10-17","Statement":[{"Effect":"Allow",
     "Principal":{"Federated":"arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"},
     "Action":"sts:AssumeRoleWithWebIdentity",
     "Condition":{"StringEquals":{"token.actions.githubusercontent.com:aud":"sts.amazonaws.com"},
                  "StringLike":{"token.actions.githubusercontent.com:sub":"repo:<OWNER>/<REPO>:*"}}}]}
   ```
   Attach permissions to create the resources in `terraform/` (VPC, ECS, ECR, ELB, ElastiCache, IAM roles,
   CloudWatch, SNS, Application Auto Scaling) plus read/write on the state bucket. For an assessment
   `PowerUserAccess` + `IAMFullAccess` is the quick route; scope it down for real use.
3. **GitHub repository settings → Variables:** `AWS_ROLE_ARN`, `TF_STATE_BUCKET`, optionally `AWS_REGION`.
   Create an environment named `dev` (add required reviewers there if you want a manual approval gate).

### Deploy
Push to `main` (or run the workflow manually). The pipeline: test → `terraform validate` → create ECR → build and
push `backend` and `worker` images tagged with the commit SHA → `terraform apply` → wait for ECS stable → smoke test.
The URL is printed in the job log as `app_url`.

### Manual deploy from a laptop
```bash
cd terraform
terraform init -backend-config="bucket=<state-bucket>" -backend-config="key=devops-assessment/dev.tfstate" -backend-config="region=ap-south-1"
terraform apply -target=aws_ecr_repository.repo -var image_tag=manual1      # registries first
# docker login to ECR, then build and push both images with tag manual1
terraform apply -var image_tag=manual1
terraform output app_url
```

### Operate
```bash
aws logs tail /ecs/devops-assessment-dev/backend --follow        # API logs
aws logs tail /ecs/devops-assessment-dev/worker  --follow        # Celery logs
aws ecs describe-services --cluster devops-assessment-dev --services backend worker \
  --query 'services[].{n:serviceName,running:runningCount,desired:desiredCount,events:events[0].message}'
```
CloudWatch alarms (ALB 5xx, unhealthy targets, CPU, Redis CPU, Celery task errors) publish to an SNS topic;
set `alarm_email` to get notified. **Rollback:** the ECS deployment circuit breaker rolls back failed deploys
automatically; to go back manually, re-run the workflow on an earlier commit.

### Tear down
```bash
cd terraform && terraform destroy -var image_tag=any
```
(The NAT gateway and ElastiCache node bill by the hour, so destroy when you are done.)
