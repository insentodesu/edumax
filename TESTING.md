# Manual Testing Guide (Docker)

This document describes how to test the backend, chat bot, and (optionally) frontend using Docker. Mini app UI is still outside the main testing scope, but containers will be built so that the full stack is available.

## 1. Prerequisites

- Docker Engine 20.10+ and Docker Compose v2 (`docker compose` command).
- Repository cloned to your workstation.
- Valid Max bot token (`BOT_TOKEN`) so the chat bot can connect to Max Messenger.

## 2. Environment Variables (`.env` next to `docker-compose.yml`)

Docker Compose automatically loads variables from `.env` that lives beside `docker-compose.yml`. Create/update `edumax/.env` with at least:

```env
# Backend
SECRET_KEY=supersecret
API_V1_PREFIX=/api/v1
DATABASE_URL=sqlite:////data/app.db
STATIC_ROOT=/data/static
STATIC_DIR=/data/static
STATIC_URL=/static
BOT_NOTIFY_BASE_URL=http://bot:8080
BOT_NOTIFY_TOKEN=my-bot-secret          # shared with the bot HTTP API
BOT_DEFAULT_SENDER_MAX_ID=1

# YooKassa (keep empty if you do not test payments)
YOOKASSA_SHOP_ID=
YOOKASSA_SECRET_KEY=
YOOKASSA_TEST_MODE=true

# Bot service
BOT_TOKEN=<paste Max token>
HTTP_BACKEND_TOKEN=my-bot-secret        # must match BOT_NOTIFY_TOKEN
LOG_LEVEL=debug
BACKEND_API_BASE_URL=http://backend:8000/api/v1

# Frontend build arg (not mandatory for tests but used during build)
VITE_API_TOKEN=
VITE_API_BASE_URL=http://localhost:8000/api/v1
```

With this setup you no longer need a separate `backend/.env`; containers read everything from the root `.env`.

## 3. Launch via Helper Script

```
chmod +x scripts/test-backend-bot.sh
./scripts/test-backend-bot.sh
```

> Need to rebuild images after changing build-time envs (like `VITE_API_BASE_URL`)? Run `WITH_REBUILD=1 ./scripts/test-backend-bot.sh` or execute the shortcut `./scripts/test-backend-bot-rebuild.sh`.

What the script does:

- Starts `backend`, `bot`, and `frontend` services (`docker compose up -d backend bot frontend`).
- Waits until backend `/health`, bot `/healthz`, and the frontend root page respond.
- Runs `bash ./init_db.sh` inside the backend container to apply schema and seed data (skip with `SKIP_SEED=1 ./scripts/test-backend-bot.sh`).
- Prints health responses and container status.

Useful overrides:

- `COMPOSE_CMD="docker compose -p edumax-dev" ./scripts/test-backend-bot.sh`
- `SERVICES="backend bot" ./scripts/test-backend-bot.sh` (if frontend is not needed temporarily)
- `BACKEND_HEALTH_URL="http://localhost:18000/health" ./scripts/test-backend-bot.sh`
- `FRONTEND_HEALTH_URL="http://localhost:5000" ./scripts/test-backend-bot.sh`

## 4. Manual Docker Commands

```bash
docker compose up --build backend bot frontend

# After the containers are up
docker compose exec backend bash -lc "bash ./init_db.sh"
```

`init_db.sh` creates the schema and loads demo data (universities, students, schedules, events, requests, payments, broadcasts, etc.).

## 5. Backend Checks

All endpoints live under `http://localhost:8000/api/v1/...`. Use `Authorization: Bearer <token>` for protected calls.

1. Health:
   ```bash
   curl -s http://localhost:8000/health
   ```
2. Swagger:
   visit `http://localhost:8000/docs` and inspect available endpoints.
3. Obtain token (example for `max_id=1001` seeded in DB):
   ```bash
   TOKEN=$(curl -s "http://localhost:8000/api/v1/auth/login-by-max-id?max_id=1001" | jq -r .access_token)
   ```
4. Schedule:
   ```bash
   curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/schedule/today
   ```
5. Requests:
   ```bash
   curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/requests/my
   ```
6. Payments:
   ```bash
   curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/payments/balance
   curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/payments
   ```
7. Broadcasts (STAFF/ADMIN):
   ```bash
   curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/broadcasts/my
   ```

Watch `docker compose logs backend` for errors (should be clean: no 5xx responses during these calls).

## 6. Bot Checks

1. HTTP API:
   ```bash
   curl -s http://localhost:8080/healthz
   ```
2. Messenger interaction:
   - Chat with the bot in Max Messenger and send `/start`.
   - Verify that the main menu is shown and actions (schedule, requests, payments) respond.
3. Logs:
   ```bash
   docker compose logs -f bot
   ```
   You should see entries about incoming messages/callbacks.

## 7. Backend → Bot Integration

1. Ensure `BOT_NOTIFY_TOKEN` equals `HTTP_BACKEND_TOKEN`.
2. Sign in to backend as an admin/staff user (use `max_id` from seed data).
3. Create a broadcast:
   ```bash
   curl -s -H "Authorization: Bearer $TOKEN" \
        -H "Content-Type: application/json" \
        -d '{"title":"Smoke test","message":"Hello from backend","group_id":null,"faculty_id":null}' \
        http://localhost:8000/api/v1/broadcasts
   ```
   Bot logs should contain POST `/notify/bulk`, and students with `max_id` receive the message in Max Messenger.
4. Tuition reminder:
   ```bash
   curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
        -X POST http://localhost:8000/api/v1/payments/tuition/remind/1001
   ```
   Bot triggers `/notify/payment/tuition/1001` and sends the reminder.

If backend responds with HTTP 502, the bot request probably failed (token mismatch or bot unavailable).

## 8. Frontend (Optional)

The frontend container serves the built SPA on `http://localhost:4173`. Although not the focus, you can open the page to ensure assets load. Authentication flows rely on the backend API exposed at `http://localhost:8000`.

## 9. Useful Commands

```bash
# Logs
docker compose logs -f backend
docker compose logs -f bot
docker compose logs -f frontend

# Container status
docker compose ps backend bot frontend

# Re-run DB seed
docker compose exec backend bash -lc "bash ./init_db.sh"

# Stop everything
docker compose down
```

Follow this checklist to exercise backend APIs, bot flows, and verify that all Docker services (including the frontend) remain healthy.***

## 10. Backend Helper Scripts

Some backend endpoints (broadcasts, manual payments, student creation) do not have a UI. Use the Python helpers under `scripts/backend-tools/` to trigger them quickly — see `scripts/backend-tools/README.md` for detailed instructions and remember that the canonical reference for all payloads is still the Swagger UI (`http://localhost:8000/docs`).
