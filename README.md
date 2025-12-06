# Messaging App - Local Setup and Real-time Messaging Guide

This README explains how to run the Messaging App locally (Docker Compose), apply database migrations, and test real-time messaging (WebSockets). It is written for developers who may be new to the project or to Docker/Channels. Follow each section in order.

## Quick summary
- Start everything with Docker Compose (Postgres, Redis, auth service, messaging service, Traefik):

  ```bash
  docker compose build
  docker compose up -d
  ```

- The messaging service runs a migration step on startup (entrypoint waits for DB and runs `manage.py migrate`). If you prefer to run migrations manually, the commands are shown below.
- Test REST endpoints with curl or Postman. Test WebSockets with Postman (WebSocket requests) or wscat (CLI).

## Prerequisites
- Git
- Docker Engine and Docker Compose (v2) installed and configured
- (Optional) Node.js + npm to install `wscat` for quick WebSocket CLI tests

On Linux, install Docker and start the Docker daemon before continuing.

## Repository layout (important locations)
- `messaging_services/` — messaging Django project and chat app
  - `messaging_services/chat/consumers.py` — WebSocket consumers (real-time messaging logic)
  - `messaging_services/chat/routing.py` — WebSocket URL patterns
  - `messaging_services/chat/middleware.py` — JWT auth for WebSockets (uses `token` query param or `Authorization` header)
  - `messaging_services/chat/models.py` — ChatRoom and Message models
  - `messaging_services/Dockerfile` and `messaging_services/entrypoint.sh` — container image and startup script (runs migrations)
- `Tutoring_API/` — the auth service (issue tokens, user profiles)
- `docker-compose.yaml` — orchestration for local development
- `shared/configs/env/` — environment variable files used by compose for different services

Read the files above if you want to understand internals. You do not need to change them for a normal local setup.

## Environment variables
The compose setup uses env files in `shared/configs/env/`. Confirm these contain correct database credentials, JWT settings, and service URLs. Key variables used by messaging service include:
- `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` — Postgres connection
- `REDIS_HOST`, `REDIS_PORT` — Redis for Channels
- `AUTH_API_URL` — URL for your auth service (default: `http://authservice:8080`)
- `JWT_SECRET`, `JWT_ALGORITHM`, `JWT_PUBLIC_KEY` — JWT verification settings

If you need to change values, edit the corresponding env file and re-run `docker compose up -d`.

## Build & run (first time)
1. Build images (uses Dockerfile in each service):

```bash
docker compose build
```

2. Start all services in detached mode:

```bash
docker compose up -d
```

3. Check logs to verify startup and migrations:

```bash
docker compose logs -f messaging_service
docker compose logs -f authservice
docker compose logs -f redis
```

The messaging service image includes an `entrypoint.sh` that waits for the database to be ready and runs `python manage.py migrate --noinput` before starting Daphne (ASGI server). If migrations fail, check the messaging service logs for errors and DB credentials.

## Manual migrations (if needed)
If you prefer to run migrations manually or if automatic migrations fail, run:

```bash
docker compose exec messaging_service python manage.py makemigrations chat
docker compose exec messaging_service python manage.py migrate
```

If the database is not yet ready and you see connection errors, wait a few seconds and retry. You can also run a one-shot command from the host:

```bash
docker compose run --rm messaging_service python manage.py migrate
```

## Testing the REST API (create or get room)
Use curl or Postman to call the create-room endpoint. You need a valid JWT (issued by the auth service) in the `Authorization: Bearer <token>` header.

Example (replace `<TOKEN>`):

```bash
curl -i -X POST "http://localhost:8000/api/chat/rooms/" \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"participant_a":"1","participant_b":"2"}'
```

Response example:

```json
{"room_id":1,"created":false,"participant_a":"1","participant_b":"2","created_at":"2025-11-17T04:33:47.222756Z"}
```

`created:false` means the endpoint returned an existing room; `created:true` would indicate a newly created room.

## Testing WebSockets (real-time messaging)
The WebSocket endpoint is:

```
ws://localhost:8000/ws/chat/<room_id>/?token=<JWT>
```

Authentication options:
- Pass token as a query parameter `?token=<JWT>` (supported by the project's middleware).
- Or pass `Authorization: Bearer <JWT>` in the WebSocket initial headers (Postman allows adding initial headers).

Message formats (JSON text frames):
- Send a chat message:

```json
{"type":"message", "content":"Hello from A"}
```

Expected broadcast message received by clients in the room (example):

```json
{"id": 5, "room": 1, "sender_id": "1", "content": "Hello from A", "timestamp": "2025-11-18T12:34:56.789Z"}
```

- Typing notifications:

```json
{"type":"typing","is_typing":true}
```

Recipients will receive:

```json
{"type":"typing","user_id":"1","is_typing":true}
```

- Presence updates are broadcast automatically on connect/disconnect:

```json
{"type":"presence","user_id":"2","status":"online"}
```

### Test WebSocket with Postman
1. Open Postman and create a new WebSocket request.
2. Set URL to `ws://localhost:8000/ws/chat/<room_id>/?token=<JWT>`.
3. (Optional) Add header: `Authorization: Bearer <JWT>` instead of query param.
4. Connect. You should see a presence event and be able to send/receive messages using the JSON format above.

### Test WebSocket with wscat (CLI)
1. Install `wscat`:

```bash
npm install -g wscat
```

2. Connect two terminals (one per user):

```bash
wscat -c "ws://localhost:8000/ws/chat/1/?token=<JWT_USER_1>"
wscat -c "ws://localhost:8000/ws/chat/1/?token=<JWT_USER_2>"
```

3. Type JSON messages and press Enter. You should see messages mirrored between clients.

## Including sender display name in real-time messages
The project currently sends `sender_id` in message payloads. Two common approaches to include a sender name:

- **Option A (recommended)**: include the username/display name in live payloads only (no DB changes). The consumer fetches a profile once per WebSocket connection (or uses the username from the JWT) and includes `sender_username` in broadcasted messages.
- **Option B**: denormalize and store `sender_username` on the `Message` model (requires migration). This preserves historical names even if users rename later.

See `messaging_services/chat/consumers.py` for where to add `sender_username` to the outgoing payload. Option A is quick and works well in most development setups.

## Troubleshooting
- **Error: `relation "chat_chatroom" does not exist`** — means migrations haven't been applied. Run the migration steps above.
- **Error: `Invalid HTTP_HOST header` or `DisallowedHost`** — ensure service hostnames are valid (avoid underscores) and `ALLOWED_HOSTS` in the auth service contain the internal Docker hostname (we use `authservice`).
- **Docker: `Network ... Resource is still in use` when running `docker compose down`** — run `docker network inspect <name>` to see attached containers, remove leftover containers, or run `docker network prune` to remove unused networks. Prune is safe: Docker will recreate networks when you `docker compose up`.
- **WebSocket disconnects immediately after connecting** — check `messaging_service` logs. Common causes:
  - Middleware failed to decode token (no `scope['auth_user']`) and consumer closes the connection.
  - Consumer rejects the user because they are not a participant in the room (check `ChatRoom` participants and token `user_id`).
  - Redis/channel layer errors — ensure Redis is running and the `REDIS_HOST`/`REDIS_PORT` are correct.

## Useful commands
- Show containers & status:

```bash
docker compose ps
```

- View logs (streaming):

```bash
docker compose logs -f messaging_service
docker compose logs -f authservice
docker compose logs -f redis
```

- Run a management command in the messaging container:

```bash
docker compose exec messaging_service python manage.py <command>
```

- Stop all services:

```bash
docker compose down
```


---

