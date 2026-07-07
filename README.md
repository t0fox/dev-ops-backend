# DevOps Test Task

Web application available to the user through **nginx**, running in Docker
containers. The user only ever talks to nginx; nginx proxies requests to a
backend that is not exposed to the host.

## Architecture

    curl (host) --> nginx:80 (published) --HTTP--> backend:8080 (docker-net only)
    network: app-net (bridge)

## Tech stack

- Python 3.12 (http.server, standard library only)
- nginx 1.27 (alpine, official image)
- Docker / Docker Compose

## Project structure

    .
    ├── .github/
    │   └── workflows/
    │       └── ci.yml
    ├── backend/
    │   ├── Dockerfile
    │   └── app.py
    ├── nginx/
    │   └── nginx.conf
    ├── docker-compose.yml
    ├── .env.example
    ├── .gitignore
    └── README.md

## How to run

    cp .env.example .env
    docker compose up --build

## How to verify

    curl http://localhost
    # -> Hello from Effective Mobile!

Backend is not reachable from the host (by design):

    curl http://localhost:8080
    # -> connection refused

But backend is reachable from inside the Docker network:

    docker compose exec nginx wget -qO- http://backend:8080
    # -> Hello from Effective Mobile!

Unknown paths return 404:

    curl -i http://localhost/unknown
    # -> HTTP/1.1 404 Not Found

## Useful commands

    docker compose ps
    docker compose logs -f
    docker compose logs -f nginx
    docker compose logs -f backend
    docker compose down

## CI

On every push / pull request to `main`, GitHub Actions builds the stack, waits for it to answer, and runs a smoke test: `curl http://localhost` must return the expected message and unknown paths must return 404.

## How it works (nginx -> backend)

1. Request hits nginx on port 80.
2. nginx matches location / and proxies to the backend upstream (backend:8080).
3. backend returns "Hello from Effective Mobile!".
4. nginx forwards standard headers (Host, X-Real-IP, X-Forwarded-For, X-Forwarded-Proto).

## Notes / decisions

- Backend runs as a non-root user with a pinned slim base image.
- No multi-stage build: pure stdlib app, no build step, single slim stage is already minimal.
- nginx config is mounted read-only and replaces the default site.
- .env.example is committed as a template; real .env files are local and git-ignored.
- container_name is used because the task asks for clear container names; in scalable Compose setups it is often better to avoid fixed container names.
