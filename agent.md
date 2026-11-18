# Software Engineering & DevOps (AI310B) - Mid-Semester Examination 2
## Complete Implementation Guide & Execution Log

---

## 1. Executive Summary & Architecture Overview

This project containerizes, networks, and automates the deployment of **TaskFlow**, a secure full-stack task management application.

### Architecture Components:
- **Frontend**: React (Vite) Single Page Application served via a lightweight Nginx web server on Alpine Linux. Nginx acts as both a static file server and a reverse proxy for `/api/` and `/health` requests to the backend service.
- **Backend**: Express.js REST API running on Node.js 20 Alpine Linux. Connects to PostgreSQL for relational persistence and Redis for query caching and session acceleration.
- **PostgreSQL**: PostgreSQL 16 Alpine container with a named volume `postgres_data` mounted to `/var/lib/postgresql/data` for persistent storage and initialization schema mounted at `/docker-entrypoint-initdb.d/init.sql`.
- **Redis**: Redis 7 Alpine container running with AOF (`--appendonly yes`) for fast caching of user task lists.
- **Custom Network**: Bridge network named `taskflow` enabling inter-container communication via service names (`postgres`, `redis`, `backend`) without exposing internal database ports to the host machine.
- **CI/CD Automation**: GitHub Actions pipeline triggering on push to `main` branch, building optimized multi-stage Docker images, tagging with `v1` and `latest`, pushing to Docker Hub, and deploying via `docker-compose.deploy.yml`.

---

## 2. Step-by-Step Task Breakdown

### Task 1: Optimized Dockerfiles
#### A. Frontend Dockerfile (`frontend/Dockerfile`)
- **Stage 1 (Build)**:
  - Base Image: `node:20-alpine` (lightweight, secure).
  - Copies `package*.json` and executes `npm install`.
  - Copies source files and runs `npm run build` to generate production assets in `/app/dist`.
- **Stage 2 (Production)**:
  - Base Image: `nginx:1.27-alpine`.
  - Copies custom `nginx.conf` to `/etc/nginx/conf.d/default.conf` to handle SPA routing (`try_files`) and reverse proxying to `http://backend:3000`.
  - Copies only static output `/app/dist` from the build stage to `/usr/share/nginx/html`.
  - Exposes port `80`.
  - Includes a container HEALTHCHECK using `wget` against `http://127.0.0.1/`.
  - Passes `hadolint` Dockerfile linter checks with zero errors.

#### B. Backend Dockerfile (`backend/Dockerfile`)
- **Stage 1 (Dependencies)**:
  - Base Image: `node:20-alpine`.
  - Copies `package*.json` and runs `npm install --omit=dev` to prevent installing development packages into production.
- **Stage 2 (Production)**:
  - Base Image: `node:20-alpine`.
  - Sets `ENV NODE_ENV=production`.
  - Copies production `node_modules` from dependencies stage.
  - Copies source code (`src/`).
  - Implements the non-root user `USER node` for enhanced security.
  - Exposes port `3000`.
  - Starts the application with `CMD ["npm", "start"]`.
  - Passes `hadolint` Dockerfile linter checks with zero errors.

---

### Task 2: Docker Compose – Networking and Health Checks (`docker-compose.yml`)
- **Services**:
  1. `postgres`: Image `postgres:16-alpine`, health check using `pg_isready -U taskflow_user -d taskflow`, isolated from host (no published ports).
  2. `redis`: Image `redis:7-alpine`, health check using `redis-cli ping`, isolated from host (no published ports).
  3. `backend`: Built from `./backend`, health check probing `http://127.0.0.1:3000/health`, depends on `postgres` and `redis` with `condition: service_healthy`, port `3000:3000`.
  4. `frontend`: Built from `./frontend`, depends on `backend` with `condition: service_healthy`, port `8080:80`.
- **Network**: Custom bridge network `taskflow`.
- **Volume**: Named persistent volume `postgres_data` and cache persistence `redis_data`.

---

### Task 3: Docker Hub Publishing
- Image naming format:
  - `YOUR_DOCKERHUB_USERNAME/frontend:v1` & `:latest`
  - `YOUR_DOCKERHUB_USERNAME/backend:v1` & `:latest`
- Built and tagged locally and via automated CI/CD pipeline.

---

### Task 4: Image-Based Deployment (`docker-compose.deploy.yml`)
- Deploys the stack purely using pre-built images from Docker Hub (`${DOCKERHUB_USERNAME}/frontend:latest` and `${DOCKERHUB_USERNAME}/backend:latest`).
- Contains no `build:` context directives.
- Retains identical health checks, dependencies, custom networking, and persistent storage.

---

### Task 5: Networking & Persistence Verification
- **Test 1 - Frontend to Backend**: Nginx reverse proxy routes `/api/*` and `/health` to backend on port 3000.
- **Test 2 - Backend to PostgreSQL**: Backend successfully connects to PostgreSQL using service name `postgres` and executes queries.
- **Test 3 - Backend to Redis**: Backend connects to Redis using service name `redis` and caches task results (`source: 'redis-cache'`).
- **Test 4 - Database Persistence**: Removing the `postgres` container and recreating it retains all created users and tasks due to named volume `postgres_data`.
- **Test 5 - Redis Restart**: Restarting the Redis container restores cache without application crash.
- **Test 6 - Health Checks**: `docker compose ps` reports `(healthy)` status on postgres, redis, and backend.

---

### Task 6: GitHub Actions CI/CD Pipeline (`.github/workflows/cd.yml`)
- Workflow triggers automatically on `push` to `main` branch.
- Validates secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `POSTGRES_PASSWORD`.
- Builds and pushes images with `v1` and `latest` tags.
- Executes automated image-based deployment using `docker-compose.deploy.yml`.
- Verifies endpoints and executes API test suite (`npm test`).

---

## 3. Viva Preparation & Concepts

1. **Why use multi-stage builds?**
   - Multi-stage builds significantly reduce final image size (from ~1GB down to ~25MB for Nginx) and attack surface by excluding Node.js, build tools, source code, and npm caches from the production image.
2. **Why avoid exposing ports for database and cache to host?**
   - Security best practice (Principle of Least Privilege). Services inside the custom network can communicate securely using Docker DNS (`postgres:5432`, `redis:6379`), preventing external unauthorized access from the host network.
3. **What is the difference between `depends_on` and health conditions?**
   - A basic `depends_on` only waits for container startup, not readiness. Using `condition: service_healthy` ensures dependent services (like the backend) only start once PostgreSQL and Redis are actually ready to accept connections.
4. **How does PostgreSQL persistence work?**
   - Docker volumes store data on the host filesystem managed by Docker (`/var/lib/docker/volumes/...`). Container deletion does not remove named volumes unless `-v` is explicitly passed.
