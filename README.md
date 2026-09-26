# Accounts Service

[![CI Build](https://github.com/shivansh07adi-cloud/accounts-service/actions/workflows/ci-build.yaml/badge.svg)](https://github.com/shivansh07adi-cloud/accounts-service/actions/workflows/ci-build.yaml)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python 3.9](https://img.shields.io/badge/Python-3.9-green.svg)](https://www.python.org/)
[![Flask](https://img.shields.io/badge/Flask-2.1-000000.svg)](https://flask.palletsprojects.com/)
[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy?repo=https://github.com/shivansh07adi-cloud/accounts-service)

A RESTful microservice for managing customer **Accounts**, built with Flask and PostgreSQL, containerized with Docker, deployed to Kubernetes/OpenShift, and shipped through a full CI/CD pipeline (GitHub Actions for CI, Tekton for CD).

**Live demo:** [Frontend](https://accounts-service-eight.vercel.app/) (Vercel) · [API](https://accounts-service-2pc7.onrender.com) (Render) — try `GET /health` or `GET /accounts` directly. (First request to the API may take ~30–60s to wake the free instance.)

**Frontend:** a small, plain console UI lives in [`frontend/`](frontend/) and is deployed separately on Vercel — see [`frontend/README.md`](frontend/README.md) for its own details.

It's a small, focused service — but it's wired up the way a real production microservice would be: input validation, security headers, structured logging, automated tests with coverage reporting, a Dockerized build, Kubernetes manifests, and a Tekton pipeline that lints, tests, builds and deploys it automatically.

---

## Table of contents

- [Deployment topology](#deployment-topology)
- [Architecture](#architecture)
- [API reference](#api-reference)
- [Data model](#data-model)
- [Request lifecycle](#request-lifecycle)
- [Frontend data flow](#frontend-data-flow)
- [CI/CD pipeline](#cicd-pipeline)
- [Project layout](#project-layout)
- [Getting started](#getting-started)
- [Running the tests](#running-the-tests)
- [Running with Docker](#running-with-docker)
- [Deploying to Kubernetes](#deploying-to-kubernetes)
- [Local Kubernetes + Tekton development](#local-kubernetes--tekton-development)
- [License](#license)

---

## Deployment topology

The frontend and backend are deployed independently, on two different free-tier platforms, and talk to each other only over the public internet via `fetch()` — there's no shared build step or hidden coupling between them.

```mermaid
flowchart LR
    User(["Person in a browser"])

    subgraph VercelHost["Vercel — static hosting"]
        FE["frontend/index.html<br/>plain HTML + CSS + JS, no build step<br/>accounts-service-eight.vercel.app"]
    end

    subgraph RenderHost["Render — Docker web service"]
        API["Flask API<br/>gunicorn in a container<br/>accounts-service-2pc7.onrender.com"]
        PG[("PostgreSQL<br/>accounts-db (Render-managed)")]
        API -- SQLAlchemy --> PG
    end

    User -- "loads the page" --> FE
    FE -- "fetch() over HTTPS, CORS-enabled" --> API
```

Both sides redeploy independently and automatically: a push to `main` triggers a new Render build for the API, and Vercel rebuilds the static site whenever `frontend/` changes — there's nothing to wire up by hand after the initial setup in [Deploying to Render](#deploying-to-render-free-live-url-for-a-portfoliodemo) and [`frontend/README.md`](frontend/README.md).

---

## Architecture

```mermaid
flowchart LR
    Client([Client / curl / Postman])

    subgraph Service["Accounts Service (Flask)"]
        direction TB
        MW["Security middleware<br/>Flask-Talisman + Flask-Cors"]
        Routes["REST routes<br/>service/routes.py"]
        Model["Account model<br/>service/models.py"]
        MW --> Routes --> Model
    end

    DB[(PostgreSQL)]

    Client -- HTTPS --> MW
    Model -- SQLAlchemy --> DB

    subgraph Infra["Container / Cluster"]
        direction TB
        Docker["Docker image<br/>gunicorn + Flask app"]
        K8s["Kubernetes Deployment<br/>3 replicas + Service"]
        Docker --> K8s
    end

    Service -. packaged as .-> Docker
```

The service follows a **Model–View–Controller** style split: all persistence and business logic lives in `service/models.py`, all REST routing lives in `service/routes.py`, and `service/common/` holds cross-cutting concerns (HTTP status constants, error handlers, logging setup, CLI commands).

## API reference

| Method | Endpoint | Description | Success |
|---|---|---|---|
| `GET` | `/` | Service info | `200` |
| `GET` | `/health` | Health check (used by Kubernetes probes) | `200` |
| `POST` | `/accounts` | Create an account | `201` + `Location` header |
| `GET` | `/accounts` | List all accounts | `200` |
| `GET` | `/accounts/{id}` | Read one account | `200` |
| `PUT` | `/accounts/{id}` | Update an account | `200` |
| `DELETE` | `/accounts/{id}` | Delete an account | `204` |

Every write endpoint validates its JSON body and returns a `400`/`415` with a clear message on bad or missing data; unknown IDs return a `404`.

**Example**

```bash
http POST :8080/accounts name="Jane Doe" email="jane@example.com" address="123 Main St"
```

```json
{
  "id": 1,
  "name": "Jane Doe",
  "email": "jane@example.com",
  "address": "123 Main St",
  "phone_number": null,
  "date_joined": "2026-09-26"
}
```

## Data model

| Field | Type | Required |
|---|---|---|
| `id` | Integer | auto-generated |
| `name` | String(64) | Yes |
| `email` | String(64) | Yes |
| `address` | String(256) | Yes |
| `phone_number` | String(32) | No |
| `date_joined` | Date | auto-defaults to today |

## Request lifecycle

```mermaid
sequenceDiagram
    participant C as Client
    participant T as Talisman/CORS
    participant R as routes.py
    participant M as models.py (Account)
    participant DB as PostgreSQL

    C->>T: POST /accounts (JSON)
    T->>T: enforce HTTPS + security headers
    T->>R: forward request
    R->>R: check_content_type("application/json")
    R->>M: Account().deserialize(body)
    M-->>R: validated Account (or DataValidationError)
    R->>M: account.create()
    M->>DB: INSERT
    DB-->>M: new id
    M-->>R: account.serialize()
    R-->>C: 201 Created + Location: /accounts/{id}
```

If `deserialize()` raises a `DataValidationError` (missing/invalid field), the global error handler in `service/common/error_handlers.py` turns it into a `400 Bad Request` with a descriptive message instead of a stack trace.

## Frontend data flow

The frontend (`frontend/index.html`) is a single self-contained page — no framework, no build step, no bundler. It holds no state beyond what's currently on screen: every action re-reads from or writes to the live API, so the page and the database can never drift out of sync.

```mermaid
flowchart TD
    Load["Page loads"] --> Poll["Poll GET /health every 20s"]
    Poll -->|"200 OK"| Fetch["GET /accounts"]
    Poll -->|"unreachable"| Quiet["Show a quiet status dot<br/>(no blocking error banner)<br/>keep polling"]
    Quiet --> Poll
    Fetch --> Render["Render the People list"]

    Render --> Action{"Person adds, edits,<br/>or deletes a record"}
    Action -->|"Add person"| Create["POST /accounts"]
    Action -->|"Save"| Update["PUT /accounts/:id"]
    Action -->|"Delete"| Delete["DELETE /accounts/:id"]

    Create --> Refresh["Re-fetch GET /accounts"]
    Update --> Refresh
    Delete --> Refresh
    Refresh --> Render
```

Two deliberate UX decisions worth calling out:

- **The API base URL is just an editable field**, tucked behind a collapsed "Connect to a different API" disclosure — the same page can be pointed at `localhost` during development or at the live Render URL, with no rebuild.
- **A failed health check never blocks the page.** The status dot goes quiet (not red/alarming) and polling continues in the background — appropriate for a free-tier backend that legitimately sleeps and wakes on its own.

## CI/CD pipeline

```mermaid
flowchart TD
    subgraph CI["Continuous Integration — GitHub Actions (.github/workflows/ci-build.yaml)"]
        direction LR
        A[Checkout] --> B[Install deps] --> C[flake8 lint] --> D["pytest + coverage<br/>(against a real Postgres service container)"]
    end

    subgraph CD["Continuous Delivery — Tekton (deploy/tekton/)"]
        direction LR
        E[git-clone] --> F[flake8 task] --> G["pytest task<br/>(sqlite)"]
        F --> H[buildah: build & push image]
        G --> H
        H --> I["oc apply -f deploy/<br/>rolling update on OpenShift"]
    end

    CI -. every push / PR to main .-> CI
    CD -. triggered manually or via a PipelineRun .-> CD
```

Every push and pull request against `main` runs the GitHub Actions workflow (lint + full test suite against a live Postgres service container). The Tekton pipeline in `deploy/tekton/` is the deployment path: it clones the repo, lints, runs the test suite, builds the container image with **Buildah**, substitutes the built image tag into `deploy/deployment.yaml`, and applies the manifests with `oc apply`.

## Project layout

```text
├── service                 <- application package
│   ├── common/              <- HTTP status codes, error handlers, logging, CLI commands
│   ├── config.py            <- configuration (reads from environment variables)
│   ├── models.py            <- the Account model + persistence layer
│   └── routes.py            <- REST API routes
├── tests                    <- unit tests (pytest)
│   ├── factories.py          <- test data factories (factory_boy)
│   ├── test_models.py        <- model-layer tests
│   ├── test_routes.py        <- API tests
│   └── test_cli_commands.py  <- CLI command tests
├── deploy                   <- Kubernetes manifests + Tekton CI/CD pipeline
│   ├── deployment.yaml
│   ├── service.yaml
│   └── tekton/               <- Pipeline, Tasks, and PVC for Tekton-based CD
├── .github/workflows        <- GitHub Actions CI
├── Dockerfile                <- production container image
├── Procfile                  <- process definition for gunicorn/Heroku-style platforms
├── Makefile                  <- day-to-day developer commands (`make help`)
└── requirements.txt
```

## Getting started

Requires **Python 3.9** and **Docker** (for the local Postgres container).

```bash
git clone https://github.com/shivansh07adi-cloud/accounts-service.git
cd accounts-service

python3 -m venv ~/venv
source ~/venv/bin/activate

make install     # installs dependencies
make db          # starts PostgreSQL in Docker
make run         # starts the Flask app with honcho (reads .env / Procfile)
```

The service listens on `http://localhost:8080` (via `honcho`/`Procfile`) and reads its database connection from environment variables — see `service/config.py`:

| Variable | Default |
|---|---|
| `DATABASE_URI` | built from the pieces below if not set |
| `DATABASE_USER` | `postgres` |
| `DATABASE_PASSWORD` | `postgres` |
| `DATABASE_NAME` | `postgres` |
| `DATABASE_HOST` | `localhost` |
| `SECRET_KEY` | a dev default — **override this in any real deployment** |

## Running the tests

```bash
make tests
```

This runs `pytest` with coverage against the `service` package (`pytest -v --cov=service --cov-report=term-missing`). The suite covers the model layer, every route (including error paths — 404s, bad content types, malformed bodies), the security headers added by Talisman, and the Flask CLI commands. Current coverage is **94%+**.

To point the tests at a different database (e.g. SQLite for a quick local run with no Docker):

```bash
DATABASE_URI="sqlite:///test.db" pytest -v
```

## Running with Docker

```bash
make build                      # docker build --rm -t accounts:1.0 .
docker run --rm -p 8080:8080 \
  -e DATABASE_URI="postgresql://postgres:postgres@host.docker.internal:5432/postgres" \
  accounts:1.0
```

The image is based on `python:3.9-slim`, runs as a non-root user, and starts `gunicorn` bound to `0.0.0.0:8080`.

## Deploying to Render (free, live URL for a portfolio/demo)

The repo includes a [`render.yaml`](render.yaml) Blueprint, so you don't need to configure anything by hand:

1. Push this repo to your own GitHub account (see [Getting started](#getting-started)).
2. Go to [dashboard.render.com](https://dashboard.render.com) → **New** → **Blueprint**, and connect the repo.
3. Render reads `render.yaml`, provisions a free PostgreSQL database and a free web service built from the `Dockerfile`, and wires the database credentials into the service automatically. No manual environment-variable setup needed.
4. Click **Apply**. The first build takes a few minutes; after that you get a live `https://accounts-service-xxxx.onrender.com` URL.

**What "free" means here:** the web service is genuinely free forever, but it sleeps after 15 minutes of no traffic (the next request takes 30–60 seconds to wake it up), and Render's free Postgres database expires 30 days after creation. That's fine for a portfolio/demo link — just re-apply the Blueprint (or recreate the database) if it expires. If you want a database that doesn't expire, create a free one on [Neon](https://neon.tech) or [Supabase](https://supabase.com) instead and set `DATABASE_URI` on the web service directly to that connection string, skipping the `databases:` section of the Blueprint.

## Deploying to Kubernetes

The manifests in `deploy/` expect a `postgresql` Secret (keys `database-name`, `database-user`, `database-password`) already present in the target namespace, and a `postgresql` Service reachable at the default Postgres port.

```bash
# after building and pushing your image to a registry the cluster can reach:
sed -i "s|IMAGE_NAME_HERE|<your-registry>/accounts:1.0|g" deploy/deployment.yaml
kubectl apply -f deploy/
kubectl get pods -l app=accounts
```

`deploy/deployment.yaml` runs 3 replicas behind a `ClusterIP` Service (`deploy/service.yaml`) on port 8080. The Tekton pipeline performs this same image substitution and apply automatically as part of `cd-pipeline`.

## Local Kubernetes + Tekton development

To exercise the full Tekton pipeline on your own machine (outside of any specific cloud lab), you'll need [Docker Desktop](https://www.docker.com/products/docker-desktop) and, for the complete dev container experience, [VS Code](https://code.visualstudio.com) with the [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension.

```bash
make cluster        # spin up a local K3D Kubernetes cluster + registry
make tekton          # install Tekton Pipelines, Triggers, and the Dashboard
make clustertasks    # install the openshift-client and buildah ClusterTasks
```

From there you can apply `deploy/tekton/tasks.yaml` and `deploy/tekton/pipeline.yaml`, create a `PipelineRun`, and watch it lint, test, build, and deploy the service — the same pipeline that would run against a real cluster.

## License

Licensed under the Apache License 2.0 — see [LICENSE](LICENSE).
