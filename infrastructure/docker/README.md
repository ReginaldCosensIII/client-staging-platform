# Docker Infrastructure Baseline

## 1. Role of Docker in Platform Architecture

Docker forms the foundational execution boundary for the Client Staging Platform. It fulfills three critical architectural requirements:

1. **Framework Neutrality:** Docker encapsulates any application runtime (ASP.NET Core, static sites, Node.js, Python, PHP/WordPress) within standard OCI containers, freeing the hosting platform from language-specific dependency management.
2. **Environment Reproducibility:** Containers behave identically whether running on a local development PC (Windows/WSL2) or a remote cloud VM (Linux).
3. **Internal Network Isolation:** Workloads run inside private Docker bridge networks, isolated from direct host-level exposure.

---

## 2. CP-001 Test Workload vs. Future Client Workloads

- **Current CP-001 Test Workload (`preview-test`):**
  - Located in `examples/preview-test/`.
  - Framework-neutral static container using `nginx:alpine`.
  - Serves a lightweight HTML confirmation card displaying platform status and environment.
  - Exists solely to prove image compilation, port binding, HTTP serving, container restartability, and future tunnel endpoints.
- **Future Client Workloads (e.g. USAP Website):**
  - Will replace or run alongside the test workload in later checkpoints.
  - Will package the actual client web application and its runtime dependencies.
  - Decoupled from the platform repository source control.

---

## 3. Docker Compose Operations

All local container management is executed via standard Docker Compose commands:

```bash
# 1. Validate docker-compose.yml syntax and variable substitution
docker compose config

# 2. Build or rebuild container images
docker compose build

# 3. Start services in the background (detached mode)
docker compose up -d

# 4. Inspect container process status and port mappings
docker compose ps

# 5. Tail container stdout/stderr logs
docker compose logs -f preview-test

# 6. Stop and remove containers and networks
docker compose down
```

---

## 4. Portability to Remote Preview Hosts

The Compose specification (`docker-compose.yml`) avoids host-specific absolute filesystem paths or platform-dependent assumptions:
- Context paths are relative (`./examples/preview-test`).
- Port mappings and container names use fallback environment variables (`${PREVIEW_LOCAL_PORT:-8080}`).
- Sane restart policy (`unless-stopped`) ensures auto-recovery.
- Identical Compose commands deploy and manage the containers on any remote Linux VM.
