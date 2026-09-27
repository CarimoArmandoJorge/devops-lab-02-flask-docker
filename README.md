# DevOps Lab 02 - Containerized Flask Application with Docker

## Objective
Containerize a simple Python web application using Docker, understanding
image layering, build caching, and container networking fundamentals.

## Architecture
```
Browser → EC2 Security Group (port 5000) → Docker container
                                              └── Flask app (host 0.0.0.0)
```

## What I Did
1. Wrote a minimal Flask application (`app.py`) that returns a simple
   response, binding to `0.0.0.0` (required to accept connections from
   outside the container binding to `127.0.0.1` would only accept
   connections from within the container itself).
2. Defined dependencies in `requirements.txt`.
3. Wrote a `Dockerfile` to build a reproducible image.
4. Built the image and ran it as a container, exposing port 5000.
5. Verified the app was reachable from a browser via the EC2 public IP.

## Dockerfile
```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python3", "app.py"]
```

## Commands Used
```bash
docker build -t flask-docker-app .
docker run -d -p 5000:5000 --name flask-app flask-docker-app
docker ps
```

## Key Concept: Docker Layer Caching
`requirements.txt` is copied and installed **before** `app.py` is copied,
even though it might seem more natural to copy everything at once.

Docker caches each instruction as a layer. Application code (`app.py`)
changes frequently during development, while dependencies
(`requirements.txt`) change rarely. By copying dependencies first, Docker
can reuse the cached `pip install` layer whenever only the application code
changes turning a build that could take minutes into one that takes
seconds.

## Key Concept: Container Networking
The app binds to `0.0.0.0:5000` instead of `127.0.0.1:5000`. Binding to
`127.0.0.1` would restrict the server to accept connections only from
inside the container's own network namespace external requests (like a
browser from outside) would never reach it. `0.0.0.0` accepts connections
from any interface, which is required for host-to-container port mapping
(`-p 5000:5000`) to actually work.

## What I Learned
- Building and tagging Docker images
- Running containers in detached mode with port mapping
- Docker layer caching and its impact on build performance
- Container networking basics (bind address vs port mapping)
- `docker ps` for inspecting running containers

## Docker Compose: Multi-Container Orchestration
Evolved the application to connect to a PostgreSQL database, orchestrated
via Docker Compose.

**Environment-based configuration:** database credentials and host are read
from environment variables (`os.environ.get(...)`) instead of being
hardcoded required since each environment (dev/staging/prod) has
different values.

**Compose internal networking:** service names in `docker-compose.yml`
(e.g. `db`) act as resolvable hostnames within Compose's internal network
conceptually similar to DNS (Phase 3), but scoped to the containers defined
in the same Compose project.

## Race Condition: `depends_on` vs Actual Readiness
Initially used `depends_on: - db`, which only guarantees **start order**,
not that the database is actually ready to accept connections. Testing
`/db-check` immediately after `docker compose up` occasionally failed.

**Fix:** added a proper healthcheck to the `db` service using `pg_isready`,
and changed `depends_on` to use `condition: service_healthy`:
```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
  interval: 5s
  timeout: 5s
  retries: 5
```

**Second-order finding:** even with the database healthcheck in place, a
`curl` fired immediately after `docker compose up -d` could still fail with
`Connection reset by peer` this time because the **application container
itself** hadn't finished starting yet, not the database. This demonstrated
that "container running" is not the same as "application ready to serve
traffic" a distinction that matters even more at the orchestration level
(Kubernetes readiness probes address this properly; a fixed `sleep` here
was only used for demonstration, not a production-grade fix).

## What I Learned
- Multi-container orchestration with Docker Compose
- Environment variables for configuration across environments
- Compose's internal DNS-like service resolution
- The difference between process start order and actual service readiness
- Why `depends_on` alone is insufficient without healthchecks

## CI/CD Pipeline (GitHub Actions)
Added a two-stage pipeline (`.github/workflows/ci.yml`) that runs
automatically on every push or pull request to `main`:

1. **test** - installs dependencies and runs automated tests with `pytest`.
2. **build** - builds the Docker image, but only if the `test` job succeeds
   (`needs: test`).

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - run: pytest

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t flask-docker-app .
```

**Test added:** `test_app.py` uses Flask's built-in `test_client()` to
simulate a request to the `/` route without needing a running server,verifying both status code and response content.

**Key concept - `needs:`** ensures the build only happens after tests pass, avoiding wasted CI time (and potential deployment of broken code) if tests
fail.

**Debugging note:** an early pipeline run only showed one job instead of two traced back to an unsaved file in the editor being committed as its older version. A reminder that a file must actually be saved before `git add` picks up the intended changes.

## Publishing to Docker Hub
Extended the pipeline with a third job (`push`) that publishes the built
image to Docker Hub, but only on pushes to `main` (not on pull requests):

```yaml
push:
  needs: build
  runs-on: ubuntu-latest
  if: github.ref == 'refs/heads/main'
  steps:
    - uses: actions/checkout@v4
    - uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    - uses: docker/build-push-action@v6
      with:
        context: .
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/flask-docker-app:latest
```

**Why restrict to `main` only:** publishing on every pull request would push
unreviewed code to a public `:latest` tag, and would unnecessarily expose
registry credentials in untrusted contexts (e.g. external PRs in open
source projects). Build and test should run on PRs; publishing should only
happen after code is merged.

**Debugging the credential setup (real issues hit and fixed):**
1. `Error: Username required` - traced to a typo in the GitHub Secret name
   (`DOCKERHUB_USERNAMI` instead of `DOCKERHUB_USERNAME`); secret names must
   match the workflow reference exactly.
2. `malformed HTTP Authorization header` - the Docker Hub token worked
   correctly when tested locally (`docker login`), which isolated the
   problem to how the secret was pasted into GitHub (likely a stray
   whitespace/newline from copy-paste). Regenerating the token and pasting
   it more carefully resolved it.

**Verified:** image successfully published and publicly available at
`docker pull carimoarmandojorge/flask-docker-app`.