  # Docker & Kubernetes — Lead .NET Engineer Interview Prep

> Complete reference covering containerization fundamentals through production Kubernetes patterns. Every code sample is a real, working configuration.

---

## Table of Contents

1. [Dockerfile Best Practices](#1-dockerfile-best-practices)
2. [Multi-stage Builds](#2-multi-stage-builds)
3. [Image Layers and Caching](#3-image-layers-and-caching)
4. [Volumes and Networking](#4-volumes-and-networking)
5. [Docker Compose](#5-docker-compose)
6. [Kubernetes Core Concepts](#6-kubernetes-core-concepts)
7. [ConfigMap and Secret](#7-configmap-and-secret)
8. [Liveness and Readiness Probes](#8-liveness-and-readiness-probes)
9. [HPA — Horizontal Pod Autoscaler](#9-hpa--horizontal-pod-autoscaler)
10. [Rolling Updates and Rollback](#10-rolling-updates-and-rollback)
11. [Resource Requests and Limits](#11-resource-requests-and-limits)
12. [Kubernetes Networking](#12-kubernetes-networking)

---

## 1. Dockerfile Best Practices

### Why Layer Order Matters

Docker builds images as a stack of layers. Each instruction (`RUN`, `COPY`, `ADD`) creates a new layer. When a layer changes, **all subsequent layers are invalidated and rebuilt from scratch**.

The golden rule: **put things that change least at the top, things that change most at the bottom**.

```
Layer order (slowest to rebuild → fastest to rebuild):
  Base image        ← almost never changes
  OS packages       ← changes rarely
  Runtime deps      ← changes when project file changes
  Application code  ← changes on every commit
```

### Key Best Practice Rules

| Practice | Why |
|---|---|
| COPY project files before source code | Restores NuGet/npm packages only when dependencies change |
| Multi-stage build | Discard SDK/build tools from final image |
| .dockerignore | Prevent sending large files (bin, obj, .git) to Docker daemon |
| Non-root user | Reduce attack surface if container is compromised |
| HEALTHCHECK | Docker can restart unhealthy containers automatically |
| Pin base image tags | `mcr.microsoft.com/dotnet/aspnet:8.0` not `latest` |
| Minimize RUN layers | Chain commands with `&&` to reduce layer count |

### .dockerignore

```
# .dockerignore
**/bin
**/obj
**/.git
**/.gitignore
**/README.md
**/Dockerfile*
**/.dockerignore
**/node_modules
**/TestResults
**/*.user
**/*.suo
.env
.env.*
```

### HEALTHCHECK Instruction

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

- `--interval`: how often to run the check
- `--timeout`: how long to wait before marking as failed
- `--start-period`: grace period for slow-starting apps
- `--retries`: consecutive failures before marking unhealthy

### Before (Naive) vs After (Optimized)

**BEFORE — rebuilds everything on every code change:**

```dockerfile
# BAD: Copies everything first, NuGet restore runs every time code changes
FROM mcr.microsoft.com/dotnet/sdk:8.0
WORKDIR /app
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /out
ENTRYPOINT ["dotnet", "/out/MyApi.dll"]
```

Problems:
- Single stage — ships 800MB SDK image to production
- No .dockerignore — sends bin/obj to daemon
- Runs as root
- No health check
- NuGet restore re-runs on every code change

**AFTER — optimized multi-stage with proper caching:**

```dockerfile
# GOOD: Optimized multi-stage Dockerfile for ASP.NET Core 8
# ─────────────────────────────────────────────────────────
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Step 1: Copy only project files first (cache NuGet restore)
COPY ["src/MyApi/MyApi.csproj", "src/MyApi/"]
COPY ["src/MyApi.Core/MyApi.Core.csproj", "src/MyApi.Core/"]
COPY ["src/MyApi.Data/MyApi.Data.csproj", "src/MyApi.Data/"]

# Step 2: Restore dependencies (this layer is cached until .csproj changes)
RUN dotnet restore "src/MyApi/MyApi.csproj"

# Step 3: Copy source code and build
COPY . .
WORKDIR "/src/src/MyApi"
RUN dotnet build "MyApi.csproj" -c Release --no-restore -o /app/build

# Stage 2: Publish
FROM build AS publish
RUN dotnet publish "MyApi.csproj" -c Release --no-restore \
    --self-contained false \
    --runtime linux-x64 \
    -o /app/publish

# Stage 3: Runtime image (no SDK — tiny!)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app

# Security: create and use non-root user
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

# Copy published output from publish stage
COPY --from=publish /app/publish .

# Health check via ASP.NET Core health endpoint
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Switch to non-root
USER appuser

# Expose port (documentation only — does not publish)
EXPOSE 8080

# Use exec form (no shell — signals go directly to process)
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

> **Interview Tip:** Be ready to explain why you use exec form (`["dotnet", "MyApi.dll"]`) vs shell form (`dotnet MyApi.dll`). Exec form sends SIGTERM directly to the .NET process, enabling graceful shutdown. Shell form wraps in `/bin/sh -c`, and SIGTERM goes to sh, not dotnet.

---

## 2. Multi-stage Builds

### What It Solves

Without multi-stage builds, the final image includes the full .NET SDK (~800MB). With multi-stage builds, the final image uses only the ASP.NET runtime (~200MB).

```
Single stage:  ~800MB (SDK + source + build artifacts + compiled output)
Multi-stage:   ~200MB (runtime + compiled output only)
Distroless:    ~100MB (runtime only, no shell, no package manager)
```

### Stage Anatomy

```dockerfile
# ── Stage 1: SDK Image ──────────────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
# Full SDK: dotnet CLI, MSBuild, NuGet, compilers
# ~800MB — never shipped to production

# ── Stage 2: Publish ────────────────────────────────────────────────────────
FROM build AS publish
# Uses build output; runs dotnet publish
# Produces self-contained or framework-dependent output

# ── Stage 3: Runtime Image ──────────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
# Only ASP.NET Core runtime — no SDK, no compilers
# ~200MB — this is what runs in production
COPY --from=publish /app/publish .
```

### Complete .NET 8 Web API Dockerfile

```dockerfile
# ─────────────────────────────────────────────────────────────────────────────
# Multi-stage Dockerfile for .NET 8 Web API
# Build:   docker build -t myapi:latest .
# Run:     docker run -p 8080:8080 myapi:latest
# ─────────────────────────────────────────────────────────────────────────────

ARG DOTNET_VERSION=8.0

# ── Stage 1: Restore ─────────────────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS restore
WORKDIR /src

# Copy solution + all project files for dependency restore
COPY ["MyApi.sln", "."]
COPY ["src/MyApi/MyApi.csproj",           "src/MyApi/"]
COPY ["src/MyApi.Application/MyApi.Application.csproj", "src/MyApi.Application/"]
COPY ["src/MyApi.Domain/MyApi.Domain.csproj",           "src/MyApi.Domain/"]
COPY ["src/MyApi.Infrastructure/MyApi.Infrastructure.csproj", "src/MyApi.Infrastructure/"]
COPY ["tests/MyApi.Tests/MyApi.Tests.csproj", "tests/MyApi.Tests/"]

RUN dotnet restore "MyApi.sln"

# ── Stage 2: Build ───────────────────────────────────────────────────────────
FROM restore AS build
COPY . .
RUN dotnet build "MyApi.sln" -c Release --no-restore

# ── Stage 3: Test (optional — skip in prod pipeline with --target publish) ───
FROM build AS test
RUN dotnet test "tests/MyApi.Tests/MyApi.Tests.csproj" \
    -c Release \
    --no-build \
    --logger "trx;LogFileName=results.trx" \
    --results-directory /TestResults

# ── Stage 4: Publish ─────────────────────────────────────────────────────────
FROM build AS publish
RUN dotnet publish "src/MyApi/MyApi.csproj" \
    -c Release \
    --no-build \
    --runtime linux-x64 \
    --self-contained false \
    -o /app/publish \
    /p:UseAppHost=false

# ── Stage 5: Final Runtime Image ─────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION} AS final
WORKDIR /app

# Create non-root user
RUN groupadd --system appgroup \
    && useradd --system --gid appgroup --no-create-home appuser

# Copy published artifacts
COPY --from=publish --chown=appuser:appgroup /app/publish .

# Environment
ENV ASPNETCORE_URLS=http://+:8080 \
    ASPNETCORE_ENVIRONMENT=Production \
    DOTNET_RUNNING_IN_CONTAINER=true

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8080/health/live || exit 1

USER appuser
EXPOSE 8080

ENTRYPOINT ["dotnet", "MyApi.dll"]
```

### Base Image Options

| Image | Size | Use case |
|---|---|---|
| `mcr.microsoft.com/dotnet/sdk:8.0` | ~800MB | Build stage only |
| `mcr.microsoft.com/dotnet/aspnet:8.0` | ~200MB | ASP.NET Core apps |
| `mcr.microsoft.com/dotnet/runtime:8.0` | ~130MB | Console apps / workers |
| `mcr.microsoft.com/dotnet/aspnet:8.0-alpine` | ~120MB | Smaller, musl libc |
| `mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled` | ~100MB | Distroless-style, no shell |

> **Pitfall:** Alpine images use musl libc. Some native libraries (e.g., certain globalization/ICU features) behave differently. Add `ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1` or use debian-based images for full globalization support.

---

## 3. Image Layers and Caching

### How Layers Work

Every Dockerfile instruction that modifies the filesystem creates a new layer. Docker caches these layers by their instruction + content hash.

```
Image ID: sha256:abc123...
  Layer 5: COPY . .                   ← changes every commit
  Layer 4: RUN dotnet restore          ← changes when .csproj changes
  Layer 3: COPY *.csproj .             ← changes when deps change
  Layer 2: RUN apt-get install curl    ← changes rarely
  Layer 1: FROM aspnet:8.0            ← almost never changes
```

When layer 3 is invalidated, layers 4 and 5 must be rebuilt — they cannot use cache.

### Cache Invalidation Rules

```dockerfile
# Instruction 1: Cached as long as the base image hasn't changed
FROM mcr.microsoft.com/dotnet/sdk:8.0

# Instruction 2: Cached as long as the RUN command string is identical
RUN apt-get update && apt-get install -y curl

# Instruction 3: Invalidated when ANY .csproj file changes
COPY **/*.csproj ./

# Instruction 4: Invalidated when instruction 3 is invalidated
RUN dotnet restore

# Instruction 5: Invalidated whenever any source file changes
COPY . .

# Instruction 6: Always invalidated after instruction 5 changes
RUN dotnet publish -c Release -o /out
```

### Optimal Order for Maximum Cache Hits

```
1. FROM base-image                    (cache: very stable)
2. RUN install OS packages            (cache: changes rarely)
3. COPY project/dependency files      (cache: changes when deps change)
4. RUN restore dependencies           (cache: reuses when deps unchanged)
5. COPY source code                   (cache: changes every commit)
6. RUN build/compile                  (cache: invalidated by code changes)
```

### Checking Layer Sizes

```bash
# Inspect layers and sizes
docker history myapi:latest

# Output:
# IMAGE          CREATED       CREATED BY                        SIZE
# abc123...      2 min ago    COPY --from=publish /app/publish  45.2MB
# def456...      2 min ago    ENV ASPNETCORE_URLS=...           0B
# ghi789...      3 min ago    RUN groupadd ...                  4.1kB
# ...

# Detailed inspection
docker inspect myapi:latest | jq '.[0].RootFS.Layers'
```

---

## 4. Volumes and Networking

### Volume Types

| Type | Command | Use case |
|---|---|---|
| Named volume | `docker volume create mydata` | Persistent data managed by Docker |
| Bind mount | `-v /host/path:/container/path` | Dev — mount source code |
| tmpfs mount | `--tmpfs /tmp` | In-memory, ephemeral (secrets, temp files) |

```bash
# Named volume — survives container removal
docker run -v postgres_data:/var/lib/postgresql/data postgres:16

# Bind mount — host directory mirrored into container (dev workflow)
docker run -v $(pwd)/src:/app/src myapi:dev

# tmpfs — in-memory, cleared on restart
docker run --tmpfs /tmp:rw,noexec,nosuid,size=64m myapi:latest
```

### Docker Network Types

| Driver | Description | Use case |
|---|---|---|
| `bridge` | Default; isolated network per container | Single-host development |
| `host` | Shares host network stack | High performance, no NAT |
| `none` | No networking | Fully isolated |
| `overlay` | Multi-host (Swarm/Kubernetes) | Production clustering |
| `macvlan` | Assign MAC address | Legacy apps needing MAC |

```bash
# Create a custom bridge network
docker network create --driver bridge myapp-network

# Connect containers by name (DNS resolution)
docker run -d --name postgres --network myapp-network postgres:16
docker run -d --name myapi   --network myapp-network \
    -e ConnectionStrings__Default="Host=postgres;Database=mydb" \
    myapi:latest
```

### Container DNS Resolution

Containers on the same user-defined network can resolve each other by container name:

```
Container: myapi
  Hostname:  myapi
  DNS query: postgres → 172.18.0.2 (resolved by Docker's embedded DNS)

Container: postgres
  Hostname:  postgres
  Port:      5432
```

> **Pitfall:** The default `bridge` network does NOT support DNS resolution by container name. Always create a user-defined network for service discovery.

### Real Example: docker-compose with App + Postgres + Redis

```yaml
# docker-compose.yml — App + Postgres + Redis
version: "3.9"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: final
    ports:
      - "8080:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ConnectionStrings__Default: "Host=postgres;Port=5432;Database=mydb;Username=appuser;Password=${POSTGRES_PASSWORD}"
      ConnectionStrings__Redis: "redis:6379"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - backend

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - backend

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  backend:
    driver: bridge
```

---

## 5. Docker Compose

### Full Stack .NET App + SQL Server + Redis

```yaml
# docker-compose.yml — Complete .NET 8 stack
version: "3.9"

# Load from .env file automatically
# .env contents:
#   SQL_SA_PASSWORD=YourStr0ngP@ssword!
#   REDIS_PASSWORD=redispassword123
#   JWT_SECRET=super-secret-key-min-32-chars!!

services:

  # ── API Service ──────────────────────────────────────────────────────────
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: final
      args:
        DOTNET_VERSION: "8.0"
    image: myapp/api:${API_VERSION:-latest}
    container_name: myapp_api
    ports:
      - "${API_PORT:-8080}:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: ${ENVIRONMENT:-Development}
      ASPNETCORE_URLS: http://+:8080
      ConnectionStrings__SqlServer: >-
        Server=sqlserver,1433;Database=MyAppDb;
        User Id=sa;Password=${SQL_SA_PASSWORD};
        TrustServerCertificate=true;
      ConnectionStrings__Redis: "redis:6379,password=${REDIS_PASSWORD},abortConnect=false"
      Jwt__Secret: "${JWT_SECRET}"
      Jwt__Issuer: "myapp.local"
      Jwt__Audience: "myapp.local"
    env_file:
      - .env
    depends_on:
      sqlserver:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app_network
    volumes:
      - api_logs:/app/logs
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M

  # ── SQL Server ───────────────────────────────────────────────────────────
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: myapp_sqlserver
    environment:
      ACCEPT_EULA: "Y"
      SA_PASSWORD: "${SQL_SA_PASSWORD}"
      MSSQL_PID: Developer
    ports:
      - "1433:1433"
    volumes:
      - sqlserver_data:/var/opt/mssql/data
      - sqlserver_log:/var/opt/mssql/log
      - ./db/init:/db/init:ro
    healthcheck:
      test:
        - "CMD-SHELL"
        - "/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P ${SQL_SA_PASSWORD} -C -Q 'SELECT 1' || exit 1"
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 30s
    networks:
      - app_network
    restart: unless-stopped

  # ── Redis ────────────────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: myapp_redis
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --appendonly yes
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 5s
    networks:
      - app_network
    restart: unless-stopped

  # ── Seq (structured logging UI) ─────────────────────────────────────────
  seq:
    image: datalust/seq:latest
    container_name: myapp_seq
    environment:
      ACCEPT_EULA: "Y"
    ports:
      - "5341:80"
    volumes:
      - seq_data:/data
    networks:
      - app_network
    restart: unless-stopped

volumes:
  sqlserver_data:
  sqlserver_log:
  redis_data:
  api_logs:
  seq_data:

networks:
  app_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

```bash
# Commands
docker compose up -d              # Start all services in background
docker compose up -d --build      # Rebuild images before starting
docker compose logs -f api        # Tail API logs
docker compose ps                 # Show running services
docker compose exec api bash      # Shell into API container
docker compose down               # Stop and remove containers
docker compose down -v            # Also remove volumes (DESTRUCTIVE)
```

---

## 6. Kubernetes Core Concepts

### Object Hierarchy

```
Cluster
└── Namespace
    ├── Deployment  →  manages →  ReplicaSet  →  manages →  Pod(s)
    ├── Service     →  routes traffic to →  Pod(s)  (via label selector)
    ├── Ingress     →  routes HTTP(S) to →  Service(s)
    ├── ConfigMap   →  provides config to →  Pod(s)
    └── Secret      →  provides secrets to →  Pod(s)
```

### Pod

The smallest deployable unit. Usually contains one container, sometimes a sidecar.

```yaml
# pod.yaml — rarely created directly; use Deployment instead
apiVersion: v1
kind: Pod
metadata:
  name: myapi-pod
  labels:
    app: myapi
spec:
  containers:
    - name: myapi
      image: myapp/api:1.2.3
      ports:
        - containerPort: 8080
```

### Deployment

Manages ReplicaSets, handles rolling updates, and maintains desired state.

```
Deployment
└── ReplicaSet (current)
    ├── Pod 1 (running)
    ├── Pod 2 (running)
    └── Pod 3 (running)
└── ReplicaSet (previous — kept for rollback)
```

### Service Types

| Type | Description | Use case |
|---|---|---|
| `ClusterIP` | Internal cluster IP only | Service-to-service communication |
| `NodePort` | Exposes on each node's IP at a static port | Dev/testing, direct node access |
| `LoadBalancer` | Provisions cloud load balancer | Production external access |
| `ExternalName` | DNS alias for external service | Connect to external DB by name |

### Ingress

Routes external HTTP/HTTPS traffic to services based on hostname and path.

```
Internet → LoadBalancer → Ingress Controller → Ingress Rule → Service → Pod
                          (nginx, traefik)
```

---

## 7. ConfigMap and Secret

### ConfigMap — Non-sensitive Configuration

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapi-config
  namespace: production
data:
  # Key-value pairs
  ASPNETCORE_ENVIRONMENT: "Production"
  ASPNETCORE_URLS: "http://+:8080"
  Logging__LogLevel__Default: "Information"
  Logging__LogLevel__Microsoft: "Warning"
  
  # Multi-line value (appsettings override)
  appsettings.override.json: |
    {
      "FeatureFlags": {
        "EnableNewDashboard": true,
        "MaxPageSize": 100
      },
      "Cache": {
        "DefaultExpirationMinutes": 15
      }
    }
```

### Secret — Sensitive Values

> **Important:** Kubernetes Secrets are base64-encoded, NOT encrypted. Anyone with RBAC access to secrets can decode them. Use External Secrets Operator + AWS Secrets Manager / Azure Key Vault for real security.

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapi-secrets
  namespace: production
type: Opaque
data:
  # Values must be base64-encoded: echo -n 'mypassword' | base64
  ConnectionStrings__SqlServer: U2VydmVyPXNxbHNlcnZlcjt...
  Jwt__Secret: c3VwZXItc2VjcmV0LWtleQ==
  
stringData:
  # stringData accepts plain text (Kubernetes encodes it)
  Redis__Password: "myredispassword"
  ApiKey__External: "sk-abc123def456"
```

### Mounting as Environment Variables

```yaml
# In a Deployment spec.template.spec.containers[]
env:
  # From ConfigMap — individual key
  - name: ASPNETCORE_ENVIRONMENT
    valueFrom:
      configMapKeyRef:
        name: myapi-config
        key: ASPNETCORE_ENVIRONMENT

  # From Secret — individual key
  - name: ConnectionStrings__SqlServer
    valueFrom:
      secretKeyRef:
        name: myapi-secrets
        key: ConnectionStrings__SqlServer

# Or inject all keys from ConfigMap/Secret
envFrom:
  - configMapRef:
      name: myapi-config
  - secretRef:
      name: myapi-secrets
```

### Mounting as Volume Files

```yaml
# Mount ConfigMap as file (useful for appsettings.json override)
volumes:
  - name: config-volume
    configMap:
      name: myapi-config
      items:
        - key: appsettings.override.json
          path: appsettings.override.json

  - name: secrets-volume
    secret:
      secretName: myapi-secrets
      defaultMode: 0400  # read-only, owner only

containers:
  - name: myapi
    volumeMounts:
      - name: config-volume
        mountPath: /app/config
        readOnly: true
      - name: secrets-volume
        mountPath: /app/secrets
        readOnly: true
```

### External Secrets Operator (Production Pattern)

```yaml
# ExternalSecret — syncs from AWS Secrets Manager to Kubernetes Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapi-external-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager-store
    kind: ClusterSecretStore
  target:
    name: myapi-secrets         # Creates this Kubernetes Secret
    creationPolicy: Owner
  data:
    - secretKey: ConnectionStrings__SqlServer
      remoteRef:
        key: production/myapi/database
        property: connection_string
    - secretKey: Jwt__Secret
      remoteRef:
        key: production/myapi/jwt
        property: secret
```

---

## 8. Liveness and Readiness Probes

### Probe Types and Their Purpose

| Probe | Action on failure | When to use |
|---|---|---|
| **Liveness** | Restart the container | App is deadlocked, unresponsive |
| **Readiness** | Remove from Service endpoints | App is loading, warming up, or overloaded |
| **Startup** | Restart if never succeeds | Slow-starting apps (prevents liveness kills during startup) |

```
Timeline for a slow-starting app:

  0s ──── 60s (startup probe active) ──── ready ──── running
            │                               │
            └── if fail → restart           └── liveness + readiness take over
```

### Probe Methods

```yaml
# HTTP probe — most common for web apps
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
    httpHeaders:
      - name: Accept
        value: application/json
  initialDelaySeconds: 10
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1

# TCP probe — for non-HTTP services
readinessProbe:
  tcpSocket:
    port: 1433
  periodSeconds: 10
  failureThreshold: 3

# Exec probe — run a command inside the container
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - "redis-cli ping | grep PONG"
  periodSeconds: 10
```

### Complete Example: ASP.NET Core Health Check as Readiness Probe

```csharp
// Program.cs — Register health checks
builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("SqlServer")!,
        name: "database",
        tags: ["ready"])
    .AddRedis(
        redisConnectionString: builder.Configuration.GetConnectionString("Redis")!,
        name: "redis",
        tags: ["ready"])
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"]);

// Map health endpoints
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

```yaml
# Kubernetes probe configuration matching above endpoints
containers:
  - name: myapi
    image: myapp/api:1.2.3
    
    startupProbe:
      httpGet:
        path: /health/live
        port: 8080
      failureThreshold: 30    # 30 * 10s = 5 min to start
      periodSeconds: 10

    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 0   # startup probe handles delay
      periodSeconds: 15
      timeoutSeconds: 5
      failureThreshold: 3

    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
      successThreshold: 1    # must pass once to become ready
```

> **Pitfall:** Setting `initialDelaySeconds` too low on liveness probes kills pods that are still starting up. Use `startupProbe` for slow apps instead.

---

## 9. HPA — Horizontal Pod Autoscaler

### How HPA Works

```
HPA controller (control loop, 15s default):
  1. Query metrics API for current utilization
  2. Calculate: desiredReplicas = ceil(currentReplicas * currentMetric / targetMetric)
  3. Apply scale (respect min/max bounds)
  4. Wait cooldown period before next scale-down
```

### Standard HPA (CPU/Memory)

```yaml
# hpa.yaml — Scale .NET API on CPU and memory
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapi-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapi-deployment

  minReplicas: 2
  maxReplicas: 20

  metrics:
    # Scale when average CPU > 70%
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

    # Scale when average memory > 80%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # Wait 60s before scaling up again
      policies:
        - type: Percent
          value: 100                    # Double pods at most per period
          periodSeconds: 60
        - type: Pods
          value: 4                      # Or add 4 pods max per period
          periodSeconds: 60
      selectPolicy: Max                 # Use whichever policy adds more pods

    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 min before scaling down
      policies:
        - type: Percent
          value: 25                     # Remove 25% of pods per period
          periodSeconds: 60
      selectPolicy: Min                 # Use whichever policy removes fewer pods
```

### KEDA — Scale on Custom Metrics (Queue Length)

```yaml
# keda-scaledobject.yaml — Scale based on Azure Service Bus queue length
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: myworker-scaledobject
  namespace: production
spec:
  scaleTargetRef:
    name: myworker-deployment
  minReplicaCount: 0          # Scale to zero when queue empty
  maxReplicaCount: 50
  cooldownPeriod: 300         # 5 min after queue empty before scale to 0
  pollingInterval: 15         # Check queue every 15s

  triggers:
    - type: azure-servicebus
      metadata:
        queueName: order-processing-queue
        namespace: myapp-servicebus
        messageCount: "10"      # 1 replica per 10 messages
      authenticationRef:
        name: azure-servicebus-trigger-auth

---
# TriggerAuthentication for the above
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-servicebus-trigger-auth
  namespace: production
spec:
  secretTargetRef:
    - parameter: connection
      name: servicebus-secret
      key: ConnectionString
```

---

## 10. Rolling Updates and Rollback

### Rolling Update Strategy

```yaml
# deployment.yaml — Rolling update configuration
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Allow 1 extra pod during update (4+1=5 max)
      maxUnavailable: 0    # Never go below desired count (zero-downtime)
```

```
Before update (4 replicas, v1):
  [v1] [v1] [v1] [v1]

During update (maxSurge=1, maxUnavailable=0):
  Step 1: [v1] [v1] [v1] [v1] [v2]  ← v2 added (5 total)
  Step 2: [v1] [v1] [v1] [v2]        ← one v1 terminated
  Step 3: [v1] [v1] [v2] [v2]
  Step 4: [v1] [v2] [v2] [v2]
  Step 5: [v2] [v2] [v2] [v2]        ← complete

After update (4 replicas, v2):
  [v2] [v2] [v2] [v2]
```

### Common kubectl Rollout Commands

```bash
# Deploy a new image version
kubectl set image deployment/myapi-deployment myapi=myapp/api:1.3.0 -n production

# Watch rollout progress
kubectl rollout status deployment/myapi-deployment -n production

# View rollout history
kubectl rollout history deployment/myapi-deployment -n production

# View details of a specific revision
kubectl rollout history deployment/myapi-deployment --revision=3 -n production

# Rollback to previous version
kubectl rollout undo deployment/myapi-deployment -n production

# Rollback to specific revision
kubectl rollout undo deployment/myapi-deployment --to-revision=2 -n production

# Pause a rollout (e.g., to investigate)
kubectl rollout pause deployment/myapi-deployment -n production

# Resume paused rollout
kubectl rollout resume deployment/myapi-deployment -n production
```

### Blue/Green Deployment in Kubernetes

```yaml
# blue-deployment.yaml — currently live
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi-blue
  namespace: production
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapi
      version: blue
  template:
    metadata:
      labels:
        app: myapi
        version: blue
    spec:
      containers:
        - name: myapi
          image: myapp/api:1.2.0

---
# green-deployment.yaml — new version (not live yet)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi-green
  namespace: production
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapi
      version: green
  template:
    metadata:
      labels:
        app: myapi
        version: green
    spec:
      containers:
        - name: myapi
          image: myapp/api:1.3.0

---
# service.yaml — switch by changing selector
apiVersion: v1
kind: Service
metadata:
  name: myapi-service
  namespace: production
spec:
  selector:
    app: myapi
    version: blue    # ← Change to "green" to switch traffic
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# Switch traffic from blue to green (instant, atomic)
kubectl patch service myapi-service -n production \
  -p '{"spec":{"selector":{"version":"green"}}}'

# Switch back if issues found
kubectl patch service myapi-service -n production \
  -p '{"spec":{"selector":{"version":"blue"}}}'
```

### Canary Deployment (Traffic Splitting with Ingress)

```yaml
# canary-ingress.yaml — NGINX Ingress canary annotation
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapi-canary
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"    # 20% to canary
spec:
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapi-canary-service    # points to new version
                port:
                  number: 80
```

---

## 11. Resource Requests and Limits

### Requests vs Limits

```yaml
resources:
  requests:
    cpu: "250m"      # 0.25 vCPU — minimum guaranteed (used for scheduling)
    memory: "128Mi"  # 128 MB — minimum guaranteed

  limits:
    cpu: "1000m"     # 1 vCPU — maximum allowed (throttled if exceeded)
    memory: "512Mi"  # 512 MB — maximum allowed (OOMKilled if exceeded)
```

- **requests**: The scheduler uses this to find a node with enough free capacity. The pod is guaranteed this amount.
- **limits**: The runtime enforcement. CPU is throttled (not killed). Memory OOMKills the container.

### CPU Units

```
1 CPU    = 1 vCPU = 1 Core = 1000m (millicores)
500m     = 0.5 vCPU
250m     = 0.25 vCPU
100m     = 0.1 vCPU (minimum recommended)
```

### OOMKilled — What It Means and How to Fix

```bash
# Detect OOMKilled
kubectl describe pod myapi-pod-xyz -n production
# Look for:
# State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137

kubectl get events -n production | grep OOMKilled

# Check memory usage trends before killing
kubectl top pods -n production
```

**Common causes and fixes:**

| Cause | Fix |
|---|---|
| Limit too low | Increase `limits.memory` |
| Memory leak in app | Profile with dotnet-monitor, fix the leak |
| Unexpected traffic spike | Increase limit + add HPA |
| GC not releasing memory | Tune GC with `DOTNET_GCConserveMemory` or `GCHeapHardLimit` |

```bash
# .NET GC environment variables for containers
ENV DOTNET_GCConserveMemory=5        # 0-9, higher = more conservative
ENV DOTNET_GCHeapHardLimit=419430400 # Hard limit: 400MB in bytes
ENV DOTNET_GCHeapHardLimitPercent=75 # Or as percentage of container limit
```

### QoS Classes

Kubernetes assigns QoS class based on requests/limits:

| QoS Class | Condition | Behavior under pressure |
|---|---|---|
| **Guaranteed** | `requests == limits` for all resources | Last to be evicted |
| **Burstable** | Some resources have requests < limits | Evicted after BestEffort |
| **BestEffort** | No requests or limits set | First to be evicted |

```yaml
# Guaranteed QoS — requests == limits
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

# Burstable QoS — limits > requests
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "1000m"
    memory: "512Mi"

# BestEffort QoS — no resources specified (don't do this in production)
# resources: {}
```

---

## 12. Kubernetes Networking

### How Pod Networking Works

```
Pod A (10.1.0.5) → CNI Plugin → Node veth → Bridge/Overlay → Node veth → Pod B (10.1.0.8)

kube-proxy: Maintains iptables/IPVS rules for Service IP → Pod IP translation
CNI (Container Network Interface): Plugin that handles actual packet routing
  Common CNIs: Calico, Flannel, Cilium, WeaveNet
```

### DNS Resolution Pattern

```
Format: <service-name>.<namespace>.svc.cluster.local

Examples:
  myapi-service.production.svc.cluster.local     → 10.96.45.123 (ClusterIP)
  postgres.production.svc.cluster.local           → 10.96.45.124
  redis.production.svc.cluster.local              → 10.96.45.125

Short forms (within same namespace):
  myapi-service                                   → resolves automatically
  postgres                                        → resolves automatically

Cross-namespace:
  myapi-service.production                        → resolves cross-namespace
```

### NetworkPolicy — Pod-to-Pod Firewall

By default, all pods can communicate with all other pods. NetworkPolicy restricts this.

```yaml
# networkpolicy.yaml — Allow only frontend to call backend API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  # Apply this policy to backend pods
  podSelector:
    matchLabels:
      app: myapi
      tier: backend

  policyTypes:
    - Ingress
    - Egress

  ingress:
    # Allow traffic FROM frontend pods only
    - from:
        - podSelector:
            matchLabels:
              app: frontend
              tier: web
      ports:
        - protocol: TCP
          port: 8080

    # Also allow from monitoring namespace (Prometheus scraping)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090

  egress:
    # Allow to SQL Server
    - to:
        - podSelector:
            matchLabels:
              app: sqlserver
      ports:
        - protocol: TCP
          port: 1433

    # Allow to Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379

    # Allow DNS resolution
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

---

## Complete Kubernetes Deployment — Production Example

Full, working manifests for a .NET 8 Web API deployment.

### Namespace

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
```

### ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapi-config
  namespace: production
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  ASPNETCORE_URLS: "http://+:8080"
  Logging__LogLevel__Default: "Information"
  Logging__LogLevel__Microsoft__AspNetCore: "Warning"
  Cache__DefaultExpirationMinutes: "15"
  FeatureFlags__EnableNewDashboard: "true"
```

### Secret

```yaml
# secret.yaml (use External Secrets in real environments)
apiVersion: v1
kind: Secret
metadata:
  name: myapi-secrets
  namespace: production
type: Opaque
stringData:
  ConnectionStrings__SqlServer: "Server=sqlserver.production.svc.cluster.local;Database=MyAppDb;User Id=sa;Password=YourPassword;TrustServerCertificate=true;"
  ConnectionStrings__Redis: "redis.production.svc.cluster.local:6379,password=redispassword"
  Jwt__Secret: "super-secret-jwt-key-minimum-32-chars!!"
```

### Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi-deployment
  namespace: production
  labels:
    app: myapi
    version: "1.3.0"
  annotations:
    kubernetes.io/change-cause: "Release 1.3.0 — new dashboard feature"
spec:
  replicas: 3
  revisionHistoryLimit: 5

  selector:
    matchLabels:
      app: myapi

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  template:
    metadata:
      labels:
        app: myapi
        version: "1.3.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"

    spec:
      serviceAccountName: myapi-sa
      terminationGracePeriodSeconds: 30

      # Pod anti-affinity — spread pods across nodes
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - myapi
                topologyKey: kubernetes.io/hostname

      containers:
        - name: myapi
          image: myapp/api:1.3.0
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP

          # Config from ConfigMap
          envFrom:
            - configMapRef:
                name: myapi-config

          # Secrets as individual env vars
          env:
            - name: ConnectionStrings__SqlServer
              valueFrom:
                secretKeyRef:
                  name: myapi-secrets
                  key: ConnectionStrings__SqlServer
            - name: ConnectionStrings__Redis
              valueFrom:
                secretKeyRef:
                  name: myapi-secrets
                  key: ConnectionStrings__Redis
            - name: Jwt__Secret
              valueFrom:
                secretKeyRef:
                  name: myapi-secrets
                  key: Jwt__Secret
            # Expose pod info to app
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace

          resources:
            requests:
              cpu: "250m"
              memory: "128Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"

          startupProbe:
            httpGet:
              path: /health/live
              port: 8080
            failureThreshold: 30
            periodSeconds: 10

          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
            successThreshold: 1

          lifecycle:
            preStop:
              exec:
                # Delay shutdown to allow load balancer to drain
                command: ["/bin/sh", "-c", "sleep 5"]

          volumeMounts:
            - name: tmp-dir
              mountPath: /tmp

          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL

      volumes:
        - name: tmp-dir
          emptyDir: {}

      securityContext:
        fsGroup: 1000
```

### Services

```yaml
# service-clusterip.yaml — Internal service
apiVersion: v1
kind: Service
metadata:
  name: myapi-service
  namespace: production
  labels:
    app: myapi
spec:
  type: ClusterIP
  selector:
    app: myapi
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP

---
# service-loadbalancer.yaml — External access (use Ingress instead in most cases)
apiVersion: v1
kind: Service
metadata:
  name: myapi-lb
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: myapi
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

### Ingress with TLS

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapi-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    # cert-manager issues TLS certificate automatically
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - api.myapp.com
      secretName: myapp-tls-secret    # cert-manager populates this

  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: myapi-service
                port:
                  number: 80
          - path: /health
            pathType: Prefix
            backend:
              service:
                name: myapi-service
                port:
                  number: 80
```

### HPA

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapi-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapi-deployment
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
```

---

## Interview Q&A

**Q: What is the difference between a liveness and readiness probe?**

A: Liveness answers "is the container alive?" — if it fails, Kubernetes restarts the container. Readiness answers "is the container ready to receive traffic?" — if it fails, the pod is removed from the Service's endpoint list but not restarted. Use liveness for deadlock detection and readiness for dependency checks (database connectivity). Use startup probe for slow-starting apps to avoid liveness killing them during initialization.

**Q: How do you achieve zero-downtime deployments?**

A: Configure `maxUnavailable: 0` in the rolling update strategy so pods are never below desired count. Set `maxSurge: 1` so new pods are added before old ones are terminated. Add readiness probes so traffic only routes to healthy pods. Add a `preStop` hook with a brief sleep to allow the load balancer time to drain connections. Set `terminationGracePeriodSeconds` high enough for in-flight requests to complete.

**Q: What is OOMKilled and how do you fix it?**

A: OOMKilled (exit code 137) means the container exceeded its memory limit and was killed by the OS OOM killer. Fix by: (1) increasing the memory limit, (2) finding and fixing a memory leak with dotnet-monitor or heap dumps, (3) tuning .NET GC with `DOTNET_GCHeapHardLimitPercent`, (4) adding HPA to spread load across more pods.

**Q: How is a Kubernetes Secret different from a ConfigMap?**

A: ConfigMap stores non-sensitive configuration as plain text. Secret stores sensitive data base64-encoded. Importantly, base64 is encoding not encryption — anyone with RBAC access to the Secret can decode it. For real security, integrate with External Secrets Operator + a secrets manager (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault) which encrypts at rest and enforces IAM policies.

**Q: Explain QoS classes in Kubernetes.**

A: Guaranteed (requests == limits) — never evicted unless node fails. Burstable (requests < limits) — evicted after BestEffort if node is under memory pressure. BestEffort (no requests/limits) — evicted first. Production workloads should be at minimum Burstable; stateful workloads and critical services should be Guaranteed.

**Q: How does Docker layer caching work and how do you optimize it?**

A: Each Dockerfile instruction creates a layer identified by a hash of the instruction and its inputs. When a layer changes, all subsequent layers are invalidated. Optimize by putting stable instructions first: base image → OS packages → project files + restore → source code → build. Copy only the project/dependency files first, run restore, then copy all source code. This way, NuGet/npm restore only re-runs when dependencies actually change, not on every code commit.

**Q: What is the difference between `requests` and `limits` for CPU?**

A: `requests` is used by the scheduler to find a node with sufficient capacity — the pod is guaranteed this much CPU. `limits` is the maximum the container can use; if it exceeds this, the kernel throttles it (CPU is compressible — the container slows down, not killed). Memory works differently: if a container exceeds its memory limit, it is OOMKilled (memory is incompressible).