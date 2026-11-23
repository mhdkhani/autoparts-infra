# autoparts-infra

Docker Compose stack: nginx · NestJS backend · Next.js frontend · PostgreSQL · Kafka (KRaft) · Redis · Elasticsearch.

| Service | Image |
|---|---|
| backend / frontend | built from `node:22-alpine` |
| nginx | `nginx:latest` |
| postgres | `postgres:17` |
| redis | `redis:8-alpine` |
| kafka | `apache/kafka-native:3.9.1` (dev) · `apache/kafka:3.9.1` (prod) |
| elasticsearch | `elasticsearch:8.18.1` |
| kafka-init / media-init | one-shot jobs that chown a fresh named volume (kafka image / nginx image) |

Image names are set by the `IMAGE_*` variables in `.env`.

The apps live in their own repos. Clone all three side by side:

```
workspace/
├── backend/    # NestJS repo   (Dockerfile)
├── frontend/   # Next.js repo  (Dockerfile)
└── infra/      # this repo
```

## Configuration: one `.env` for everything

`.env` is the **only** place for settings. The compose files contain no values and no
defaults: every service (backend, frontend, nginx, postgres, redis, kafka, elasticsearch)
loads `./.env` through `env_file`, image names and ports in compose are `${VAR:?}` reads
from it (a missing variable is an error), and the apps have no defaults in code either.
Values inside `.env` may reference each other, e.g. `BROKERS=${BROKER_HOST}:${BROKER_PORT}`.

| Prefix / vars | Read by |
|---|---|
| `POSTGRES_*`, `PGPORT` | postgres image, backend |
| `REDIS_*` | redis (`--requirepass $REDIS_PASSWORD`), backend |
| `KAFKA_*`, `CLUSTER_ID` | apache/kafka — each `KAFKA_FOO_BAR` becomes `foo.bar` in server.properties |
| `ES_SETTING_*`, `ES_JAVA_OPTS` | elasticsearch — each `ES_SETTING_FOO_BAR` becomes `foo.bar` |
| `NGINX_*`, `BACKEND_HOST`, `FRONTEND_HOST`, `PORT`, `MEDIA_DIR`, `MEDIA_URL` | nginx templates (`nginx/templates/*.template`, rendered by envsubst) |
| `BROKER_*`, `BROKERS` | kafka-init / healthcheck, backend |
| `NODE_ENV`, `BIND_HOST`, `PORT`, `DEBUG_PORT`, `ELASTICSEARCH_*`, `API_INTERNAL_URL`, `PUBLIC_API_URL` | backend / frontend |
| `ADMIN_JWT_*`, `CUSTOMER_JWT_*`, `AUTH_THROTTLE_*`, `ADMIN_LOCK_*` | backend — JWT secrets / lifetimes, login rate limit, admin account lock |
| `COOKIE_SECURE` | frontend — `Secure` flag of its httpOnly cookies (the API itself uses no cookies) |
| `MEDIA_DIR`, `MEDIA_URL`, `MEDIA_UID` | backend (writes uploads), nginx (serves them), media-init (owner) |
| `SEED_ADMIN_PASSWORD` | backend dev seed only |
| `HOST_*`, `IMAGE_*`, `REGISTRY`, `*_TAG`, `COMPOSE_*` | compose (published ports, image names) |

Don't name non-broker variables `KAFKA_*` — the Kafka container would treat them as broker
settings (that's why the backend reads `BROKERS`).

Note: with one shared file every container can see every secret. If that becomes a concern,
split secrets per service later (e.g. Docker secrets).

## Volumes

| Volume | Used by | Holds |
|---|---|---|
| `pg-data` | postgres | the database |
| `redis-data` | redis | cache, revoked tokens (append-only file) |
| `kafka-data` | kafka | topics |
| `es-data` | elasticsearch | search indexes |
| `media-data` | backend (read-write), nginx (read-only) | uploaded files (logo, …) at `MEDIA_DIR`, served as `MEDIA_URL/…` |

`docker compose down` keeps them; `down -v` deletes them. Back up `pg-data` **and** `media-data`.

## nginx
- `/api/*` → backend (the `/api` prefix is kept).
- `MEDIA_URL/*` (`/media/…`) → straight from the `media-data` volume (`nosniff`, 30-day cache).
- everything else → frontend.
- `Host` is forwarded as `$http_host` (with the port), so absolute redirects built by the apps are right.

## First run

```bash
cp .env.example .env      # then set real secrets: openssl rand -hex 32
```

Elasticsearch needs `vm.max_map_count >= 262144` on the host:

```bash
sudo sysctl -w vm.max_map_count=262144
```

After the stack is up, create the tables, then either sample data (dev) or the first admin:

```bash
docker compose exec backend npm run db:migrate
docker compose exec backend npx tsx src/db/seed.ts            # dev only: sample roles + admins
docker compose exec backend npm run admin:create -- --username admin --name "…" --email you@example.com
```

## Development

Builds `../backend` and `../frontend` with their `dev` target, source bind-mounted, hot reload:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build
```

(or uncomment `COMPOSE_FILE=docker-compose.yml:docker-compose.dev.yml` in `.env` and just run `docker compose up --build`)

After adding an npm package to an app, rebuild with `-V` so the anonymous `node_modules` volume is replaced:
`docker compose up -d --build -V backend`.

| URL | What |
|---|---|
| `http://localhost:${HOST_HTTP_PORT}` | site via nginx (use this one) |
| `http://localhost:${HOST_HTTP_PORT}/admin/login` | admin panel |
| `http://localhost:${HOST_HTTP_PORT}/api/health` | backend health (public: only ok / degraded) |
| `localhost:${HOST_BACKEND_PORT}` | backend directly · `${HOST_DEBUG_PORT}` Node inspector |
| `localhost:${HOST_FRONTEND_PORT}` | Next dev server directly |
| `localhost:${HOST_POSTGRES_PORT}` | Postgres |
| `localhost:${HOST_REDIS_PORT}` · `localhost:29092` · `localhost:${HOST_ES_PORT}` | Redis · Kafka · Elasticsearch |

## Production

Each app repo's CI builds its `prod` image and pushes `${REGISTRY}/backend` / `${REGISTRY}/frontend`.
The server only has this repo:

```bash
docker compose pull
docker compose up -d
docker compose exec backend npm run db:migrate
```

Only nginx publishes a port. On the server, in `.env`: `NODE_ENV=production`, `COOKIE_SECURE=true`,
fresh `ADMIN_JWT_SECRET` / `CUSTOMER_JWT_SECRET`, real `POSTGRES_PASSWORD` / `REDIS_PASSWORD`
(`SEED_ADMIN_PASSWORD` isn't needed — the seed refuses to run in production; create the first admin with
`admin:create`). Before going live: enable Elasticsearch security, add TLS in nginx.

## Notes
- The app Dockerfiles contain no `ENV`/`ARG` either, and nothing is baked in at build time:
  the browser gets `PUBLIC_API_URL` (`/api`) at runtime, SSR uses `API_INTERNAL_URL`.
- The API authenticates with `Authorization: Bearer` only; see the backend and frontend READMEs.
- Kafka: containers use `kafka:9092`, apps on the host use `localhost:29092`.
- Kafka is pinned to 3.9.x: Kafka 4.x drops old protocol versions that KafkaJS still relies on.
