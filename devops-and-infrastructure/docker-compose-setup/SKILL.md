---
name: docker-compose-setup
description: Set up and orchestrate multi-container Docker applications using docker-compose, including service configuration, networking, volumes, and environment management.
license: MIT
metadata:
  author: awesome-ai-agent-skills
  version: 1.0.0
---

# Docker Compose Setup

This skill enables the agent to design and configure multi-container application stacks using Docker Compose. The agent can orchestrate services including web servers, databases, caches, background workers, and reverse proxies with proper networking, volume management, health checks, and environment-specific overrides. The agent understands both development and production configurations and can generate compose files that follow Docker best practices.

## Workflow

1. **Analyze the Application Stack:** The agent reviews the project's architecture to identify all required services and their dependencies. This includes the primary application container, databases (PostgreSQL, MySQL, MongoDB), caches (Redis, Memcached), message queues (RabbitMQ, Kafka), background workers, and reverse proxies (Nginx, Traefik). The agent maps inter-service dependencies to determine startup order and health check requirements.

2. **Define Services and Images:** For each service, the agent specifies the Docker image or build context, exposed ports, environment variables, and resource constraints. Application services typically use a `build` directive pointing to a local Dockerfile, while infrastructure services use official images with pinned version tags. The agent avoids using `latest` tags in production to ensure reproducible deployments.

3. **Configure Networking and Service Discovery:** The agent creates named Docker networks to isolate traffic between service tiers (e.g., a `frontend` network for the proxy and app, a `backend` network for the app and database). Services communicate using their compose service names as DNS hostnames, eliminating the need for hardcoded IP addresses.

4. **Set Up Volumes and Persistence:** The agent defines named volumes for data that must persist across container restarts, such as database storage and file uploads. For development, bind mounts map the host source code into containers to enable hot reloading. The agent ensures that volume permissions and ownership are configured correctly for the container's runtime user.

5. **Add Health Checks and Dependency Ordering:** The agent configures health checks for critical services so that dependent services wait until their dependencies are truly ready, not just started. This prevents common issues like an application container crashing because the database has started but is not yet accepting connections. The `depends_on` directive with `condition: service_healthy` enforces correct startup order.

6. **Create Environment-Specific Overrides:** The agent generates a base `docker-compose.yml` for shared configuration and an override file (`docker-compose.override.yml` for development, `docker-compose.prod.yml` for production) to customize settings per environment. Development overrides include bind mounts, debug ports, and verbose logging, while production overrides include resource limits, restart policies, and optimized logging drivers.

## Supported Technologies

- **Compose Versions:** Docker Compose V2 (integrated Docker CLI plugin)
- **Application Runtimes:** Node.js, Python, Ruby, Go, Java, PHP, .NET
- **Databases:** PostgreSQL, MySQL, MariaDB, MongoDB, Redis, Elasticsearch
- **Reverse Proxies:** Nginx, Traefik, Caddy, HAProxy
- **Message Queues:** RabbitMQ, Kafka, NATS
- **Monitoring:** Prometheus, Grafana, cAdvisor

## Usage

Provide the agent with a description of your application stack, including the services needed, their relationships, and whether the setup is for development or production.

**Example prompt:**

```
Create a docker-compose setup for my Node.js app with:
- PostgreSQL database with persistent storage
- Redis for session caching
- Nginx reverse proxy with SSL termination
- A Celery-like background worker process
- Development setup with hot reloading
```

## Examples

### Example 1: Full Stack Web Application (Node.js + PostgreSQL + Redis + Nginx)

```yaml
services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      app:
        condition: service_healthy
    networks:
      - frontend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://appuser:${DB_PASSWORD}@postgres:5432/myapp
      REDIS_URL: redis://redis:6379
      SESSION_SECRET: ${SESSION_SECRET}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - frontend
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://localhost:3000/health').then(r => r.ok ? process.exit(0) : process.exit(1))"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s

  worker:
    build:
      context: .
      dockerfile: Dockerfile
    command: node worker.js
    environment:
      DATABASE_URL: postgres://appuser:${DB_PASSWORD}@postgres:5432/myapp
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

volumes:
  postgres_data:
  redis_data:

networks:
  frontend:
  backend:
```

### Example 2: Development vs. Production Overrides

**docker-compose.yml** (base, shared configuration):

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgres://appuser:${DB_PASSWORD}@postgres:5432/myapp
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - backend

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:

networks:
  backend:
```

**docker-compose.override.yml** (development — automatically loaded):

```yaml
services:
  app:
    build:
      target: development
    ports:
      - "3000:3000"
      - "9229:9229"   # Node.js debug port
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      NODE_ENV: development
      DEBUG: "app:*"
    command: npm run dev

  postgres:
    ports:
      - "5432:5432"   # Expose DB to host for local tooling
```

**docker-compose.prod.yml** (production — used with `-f`):

```yaml
services:
  app:
    build:
      target: production
    restart: unless-stopped
    environment:
      NODE_ENV: production
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  postgres:
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
```

Run production with: `docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d`

## Best Practices

- **Use named volumes for persistent data:** Never rely on container-internal storage for databases or uploads. Named volumes survive container recreation and can be backed up independently.
- **Pin image versions:** Always specify explicit image tags (e.g., `postgres:16-alpine`) instead of `latest` to avoid unexpected breaking changes when images are updated.
- **Implement health checks on every service:** Health checks enable Docker to detect unhealthy containers and support `depends_on` with `condition: service_healthy` for reliable startup ordering. Without them, dependent services may start before their dependencies are ready.
- **Use `.env` files for secrets:** Store sensitive values like database passwords and API keys in a `.env` file that is excluded from version control via `.gitignore`. Reference variables in compose with `${VARIABLE}` syntax.
- **Separate networks by tier:** Use distinct networks to isolate traffic. A reverse proxy should not be able to reach the database directly. This limits the blast radius of a compromised container.
- **Keep compose files DRY with overrides:** Use the base + override pattern instead of duplicating entire compose files for each environment. This ensures that shared service definitions stay in sync.

## Edge Cases

- **Port conflicts on the host:** If the host port is already in use (e.g., another service on port 5432), Docker Compose will fail to bind. Use unique host ports or set `ports: []` for services that only need container-to-container communication via the Docker network.
- **Volume permission mismatches:** Containers running as non-root may fail to write to mounted volumes if the host directory has restrictive permissions. Use `user:` directives in the compose file or `chown` in the Dockerfile entrypoint to align UID/GID.
- **Build cache invalidation:** Changing a `COPY` or `ADD` instruction early in the Dockerfile invalidates the cache for all subsequent layers. Order Dockerfile instructions from least to most frequently changed (dependencies before source code) to maximize cache reuse.
- **Container DNS resolution during startup:** A service may resolve a dependency's DNS name before the dependency container is assigned an IP, causing connection failures. Health checks with `depends_on` conditions prevent this, but application-level retry logic is still recommended.
- **Orphaned containers and volumes:** Running `docker compose up` after removing a service from the compose file leaves orphan containers running. Use `docker compose up --remove-orphans` and periodically run `docker volume prune` to clean up unused volumes.
