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
| `NGINX_*`, `BACKEND_HOST`, `FRONTEND_HOST`, `PORT` | nginx templates (`nginx/templates/*.template`, rendered by envsubst) |
| `BROKER_*`, `BROKERS` | kafka-init / healthcheck, backend |
| `NODE_ENV`, `BIND_HOST`, `PORT`, `DEBUG_PORT`, `ELASTICSEARCH_*`, `API_INTERNAL_URL`, `PUBLIC_API_URL` | backend / frontend |
| `HOST_*`, `IMAGE_*`, `REGISTRY`, `*_TAG`, `COMPOSE_*` | compose (published ports, image names) |

Don't name non-broker variables `KAFKA_*` — the Kafka container would treat them as broker
settings (that's why the backend reads `BROKERS`).

Note: with one shared file every container can see every secret. If that becomes a concern,
split secrets per service later (e.g. Docker secrets).

## First run

```bash
cp .env.example .env
```

Elasticsearch needs `vm.max_map_count >= 262144` on the host:

```bash
sudo sysctl -w vm.max_map_count=262144
```

## Development

Builds `../backend` and `../frontend` with their `dev` target, source bind-mounted, hot reload:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build
```

(or uncomment `COMPOSE_FILE=docker-compose.yml:docker-compose.dev.yml` in `.env` and just run `docker compose up --build`)

| URL | What |
|---|---|
| `http://localhost:${HOST_HTTP_PORT}` | site via nginx (use this one) |
| `http://localhost:${HOST_HTTP_PORT}/api/health` | backend health: Postgres / Redis / Kafka / ES |
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
```

Only nginx publishes a port. On the server set `NODE_ENV=production` in `.env`. Before going live: enable Elasticsearch security, set real
`POSTGRES_PASSWORD` / `REDIS_PASSWORD`, add TLS in nginx.

## Notes
- The app Dockerfiles contain no `ENV`/`ARG` either, and nothing is baked in at build time:
  the browser gets `PUBLIC_API_URL` (`/api`) at runtime, SSR uses `API_INTERNAL_URL`.
- Kafka: containers use `kafka:9092`, apps on the host use `localhost:29092`.
- Kafka is pinned to 3.9.x: Kafka 4.x drops old protocol versions that KafkaJS still relies on.
