# Running Concourse Locally with Docker Compose

This guide walks you through setting up a local Concourse CI instance for testing these ZAP scan pipelines.

## Prerequisites

- Docker and Docker Compose installed
- `fly` CLI installed ([download](https://concourse-ci.org/fly.html))

## Step 1: Start Concourse

Create a `docker-compose.yml` file (or use the one from the Concourse docs):

```yaml
version: '3'

services:
  concourse-db:
    image: postgres:15
    environment:
      POSTGRES_DB: concourse
      POSTGRES_USER: concourse_user
      POSTGRES_PASSWORD: concourse_pass
    volumes:
      - pgdata:/var/lib/postgresql/data

  concourse-web:
    image: concourse/concourse:7.11
    command: web
    depends_on:
      - concourse-db
    ports:
      - "8080:8080"
    environment:
      CONCOURSE_SESSION_SIGNING_KEY: /keys/session_signing_key
      CONCOURSE_TSA_HOST_KEY: /keys/tsa_host_key
      CONCOURSE_TSA_AUTHORIZED_KEYS: /keys/authorized_worker_keys
      CONCOURSE_POSTGRES_HOST: concourse-db
      CONCOURSE_POSTGRES_USER: concourse_user
      CONCOURSE_POSTGRES_PASSWORD: concourse_pass
      CONCOURSE_POSTGRES_DATABASE: concourse
      CONCOURSE_EXTERNAL_URL: http://localhost:8080
      CONCOURSE_ADD_LOCAL_USER: admin:admin
      CONCOURSE_MAIN_TEAM_LOCAL_USER: admin
    volumes:
      - ./keys:/keys

  concourse-worker:
    image: concourse/concourse:7.11
    command: worker
    privileged: true
    depends_on:
      - concourse-web
    environment:
      CONCOURSE_TSA_HOST: concourse-web:2222
      CONCOURSE_TSA_PUBLIC_KEY: /keys/tsa_host_key.pub
      CONCOURSE_TSA_WORKER_PRIVATE_KEY: /keys/worker_key
      CONCOURSE_GARDEN_DNS_SERVER: 8.8.8.8
    volumes:
      - ./keys:/keys

volumes:
  pgdata:
```

## Step 2: Generate Keys

```bash
mkdir -p keys

ssh-keygen -t rsa -b 4096 -f keys/tsa_host_key -N ''
ssh-keygen -t rsa -b 4096 -f keys/worker_key -N ''
ssh-keygen -t rsa -b 4096 -f keys/session_signing_key -N ''

cp keys/worker_key.pub keys/authorized_worker_keys
```

## Step 3: Start the Cluster

```bash
docker compose up -d
```

Wait about 30 seconds for Concourse to initialize.

## Step 4: Log In with fly

```bash
fly -t local login -c http://localhost:8080 -u admin -p admin
```

## Step 5: Set the ZAP Pipeline

```bash
fly -t local set-pipeline \
  -p zap-scan \
  -c pipelines/zap-scan.yml \
  -l examples/params-baseline.yml
```

## Step 6: Run a Scan

```bash
fly -t local unpause-pipeline -p zap-scan
fly -t local trigger-job -j zap-scan/baseline-scan --watch
```

## Step 7: Tear Down

```bash
docker compose down -v
rm -rf keys
```

## Notes

- The local Concourse worker needs network access to reach your target URL. If scanning a local application, ensure it is accessible from within the Docker network.
- For scanning applications running on the host machine, use `host.docker.internal` (macOS/Windows) or the host's IP address (Linux).
- The worker needs to be able to pull the ZAP Docker image. Ensure internet access is available or pre-pull the image.
