# Docker Compose Exam

A small web application stack built incrementally through 6 challenges using Docker Compose.

## Services

| Service  | Image         | Host Port | Description              |
|----------|---------------|-----------|--------------------------|
| db       | postgres:15   | —         | PostgreSQL database       |
| redis    | redis:7       | —         | Redis cache               |
| backend  | nginx:latest  | 8080      | Backend web server        |
| frontend | nginx:latest  | 3000      | Frontend web server       |

---

## Final `docker-compose.yml`

```yaml
services:
  db:
    image: postgres:15
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    image: nginx:latest
    restart: unless-stopped
    ports:
      - "8080:80"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  frontend:
    image: nginx:latest
    restart: unless-stopped
    ports:
      - "3000:80"
    depends_on:
      - backend

volumes:
  pgdata:
```

---

## Deliverables

### 1. All 4 services running — `docker compose ps`

```
$ docker compose up -d
$ docker compose ps

NAME                            IMAGE         COMMAND                  SERVICE    CREATED          STATUS                    PORTS
docker-compose-exam-backend-1   nginx:latest  "/docker-entrypoint.…"   backend    10 seconds ago   Up 9 seconds              0.0.0.0:8080->80/tcp
docker-compose-exam-db-1        postgres:15   "docker-entrypoint.s…"   db         10 seconds ago   Up 10 seconds (healthy)
docker-compose-exam-frontend-1  nginx:latest  "/docker-entrypoint.…"   frontend   10 seconds ago   Up 9 seconds              0.0.0.0:3000->80/tcp
docker-compose-exam-redis-1     redis:7       "docker-entrypoint.s…"   redis      10 seconds ago   Up 10 seconds (healthy)
```

---

### 2. Database persists data across restarts

The `db` service mounts a named volume `pgdata` at `/var/lib/postgresql/data`:

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data
```

Data is preserved across `docker compose down` + `docker compose up -d` because named volumes are not removed unless explicitly deleted with `docker compose down -v`.

**Verification steps:**

```bash
# Write data
docker exec -it docker-compose-exam-db-1 psql -U appuser -d appdb -c \
  "CREATE TABLE test (id SERIAL, val TEXT); INSERT INTO test (val) VALUES ('hello');"

# Bring stack down and back up
docker compose down
docker compose up -d

# Confirm data is still there
docker exec -it docker-compose-exam-db-1 psql -U appuser -d appdb -c "SELECT * FROM test;"
#  id |  val
# ----+-------
#   1 | hello
```

---

### 3. Services auto-restart after Docker daemon restart

All services have `restart: unless-stopped`, which instructs the Docker daemon to automatically restart containers after it itself restarts (e.g. `systemctl restart docker` or a server reboot), unless the containers were explicitly stopped beforehand.

**Verification steps:**

```bash
# Start in detached mode (required for restart policy to apply)
docker compose up -d

# Restart the Docker daemon
sudo systemctl restart docker

# Wait a few seconds, then check
sleep 5
docker compose ps
# All 4 services should be running again without manual intervention
```

---

## Environment Variables

Database credentials are loaded from a `.env` file (not committed to version control):

```env
POSTGRES_USER=appuser
POSTGRES_PASSWORD=apppassword
POSTGRES_DB=appdb
```
