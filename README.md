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

## Next Steps
- Add a database service and orchestrate both with Docker Compose
- Add a CI/CD pipeline to automate build and push to a registry