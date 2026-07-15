# DevOps Test Task

Web application available to the user through **nginx**, running in Docker
containers. The user only ever talks to nginx; nginx proxies requests to a
backend that is not exposed to the host.

## Architecture

```text
curl (host) → nginx:80 (published) → HTTP → backend:8080 (Docker network only)

network: app-net (bridge)
```

## Tech stack

- Python 3.12 (`http.server`, standard library only)
- nginx 1.27 (Alpine, official image)
- Docker / Docker Compose

## Project structure

```text
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
```

## How to run

Run the commands from the project root — the directory that contains
`docker-compose.yml`.

```bash
cp .env.example .env
docker compose up --build
```

## How to verify

Check the public entry point through nginx:

```bash
curl http://localhost
# -> Hello from Effective Mobile!
```

Backend is not reachable from the host by design:

```bash
curl http://localhost:8080
# -> connection refused
```

Backend is reachable from inside the Docker network:

```bash
docker compose exec nginx wget -qO- http://backend:8080
# -> Hello from Effective Mobile!
```

`docker compose exec` searches for `docker-compose.yml` in the current
directory. Run it from the project root.

Alternative command that works from any directory while containers are running:

```bash
docker exec em-nginx wget -qO- http://backend:8080
# -> Hello from Effective Mobile!
```

Unknown paths return `404`:

```bash
curl -i http://localhost/unknown
# -> HTTP/1.1 404 Not Found
```

## Useful commands

Run these commands from the project root:

```bash
docker compose ps
docker compose logs -f
docker compose logs -f nginx
docker compose logs -f backend
docker compose down
```

## CI

On every push or pull request to `main`, GitHub Actions builds the stack,
waits for it to answer, and runs smoke tests:

- `curl http://localhost` must return `Hello from Effective Mobile!`
- an unknown path must return `404`

## How it works: nginx → backend

1. A request reaches nginx on published port `80`.
2. nginx matches `location /` and proxies the request to the backend upstream
   at `backend:8080`.
3. Backend returns `Hello from Effective Mobile!`.
4. nginx forwards standard headers: `Host`, `X-Real-IP`,
   `X-Forwarded-For`, and `X-Forwarded-Proto`.

## Notes / decisions

- Backend runs as a non-root user with a pinned slim base image.
- No multi-stage build: this is a pure standard-library application with no
  build step, so a single slim stage is already minimal.
- nginx configuration is mounted read-only and replaces the default site.
- `.env.example` is committed as a template; real `.env` files stay local and
  are ignored by Git.
- `container_name` is used because the task asks for clear container names.
  In scalable Compose setups, fixed names are usually better avoided.
