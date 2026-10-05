# Docker Infrastructure Baseline

## 1. Role of Docker in Platform Architecture

Docker forms the foundational execution and packaging boundary for the Client Staging Platform. It fulfills three critical architectural requirements across both local development workstations and Remote Preview Hosts:

1. **Framework Neutrality:** Docker encapsulates any application runtime (ASP.NET Core, static sites, Node.js, Python, PHP/WordPress) within standard OCI containers, freeing the hosting platform from language-specific dependency management.
2. **Environment Reproducibility:** Containers behave identically whether running on a local development PC (Windows/WSL2) or a remote cloud VM (e.g., Ubuntu 24.04 LTS on DigitalOcean or Google Cloud).
3. **Internal Network & Origin Isolation:** Workloads run inside private Docker bridge networks and publish strictly to the host loopback interface (`127.0.0.1`), isolated from direct host-level or LAN exposure.

---

## 2. Remote Preview Host Docker Baseline (CP-003 / CP-004 Validated)

The Remote Preview Host runs the official Docker distribution installed via Docker's Ubuntu APT repository (`download.docker.com`):

- **Docker Engine:** `29.8.2`
- **Docker Compose Plugin:** `5.6.0` (`docker compose`)
- **containerd:** `2.3.6`
- **Docker Buildx:** Installed and active
- **Service Status:** `docker.service` enabled and active (running)
- **Administrative Access:** Managed by `stagingadmin` without requiring `sudo` via membership in the `docker` group.

> [!WARNING]
> **Docker Group Security Note:**
> Membership in the `docker` group grants access to the Docker daemon socket (`/var/run/docker.sock`), which effectively conveys root-equivalent privileges over the host filesystem and kernel. This configuration is acceptable for the dedicated, trusted administrative account (`stagingadmin`), but must **never** be granted to low-privilege or untrusted user accounts.

---

## 3. Strict Loopback Binding Requirement

> [!IMPORTANT]
> **Prohibition of Public Docker Publishing:**
> Container ports must **never** be published across all host network interfaces (`0.0.0.0`) or exposed directly to public internet interfaces.
>
> In `docker-compose.yml`, port specifications must explicitly designate the IPv4 loopback address:
> ```yaml
> ports:
>   - "127.0.0.1:${PREVIEW_LOCAL_PORT:-8080}:80"
> ```
>
> **Rationale:**
> A protected Cloudflare Quick Tunnel is not an excuse for loose origin security. Binding exclusively to `127.0.0.1` ensures that the container origin cannot be reached directly by external requests even if firewall rules change.

---

## 4. Workload Progression: Test vs. Client Preview

- **Current Infrastructure Test Workload (`preview-test`):**
  - Located in `examples/preview-test/`.
  - Framework-neutral static container using `nginx:alpine`.
  - Serves a lightweight HTML confirmation card displaying platform status and environment.
  - Used in CP-001, CP-002, CP-003, and CP-004 to empirically validate container lifecycle, port binding, HTTP serving, restartability, and tunnel endpoints.
- **Client Application Workloads (CP-005: USAP Website):**
  - In CP-005, the platform will package and deploy the actual USAP client application preview container.
  - Client application source code remains decoupled in its dedicated repository (`usap-website`).
  - The client workload container will bind to loopback port 8080 (or an assigned loopback port), seamlessly substituting or augmenting the test workload behind the `cloudflared` tunnel.

---

## 5. Standard Docker Operations

All container management on both local and remote hosts is executed via standard Docker Compose commands:

```bash
# 1. Validate docker-compose.yml syntax and variable substitution
docker compose config

# 2. Build or rebuild container images
docker compose build

# 3. Start services in the background (detached mode)
docker compose up -d

# 4. Inspect container process status and port mappings
docker compose ps
# Expected: 127.0.0.1:8080->80/tcp

# 5. Tail container stdout/stderr logs
docker compose logs -f preview-test

# 6. Stop and remove containers and networks
docker compose down
```
