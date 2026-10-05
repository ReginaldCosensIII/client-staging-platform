# Client Staging Platform

> **Working / Product Concept:** Version III Client Preview & Staging Portal
> **Repository:** `client-staging-platform`
> **Current Checkpoint:** CP-004R1 — Remote Host Documentation & Infrastructure Baseline Reconciliation

---

## 1. Overview & Purpose

The **Client Staging Platform** is a reusable, provider-neutral software and infrastructure project designed to host secure, private staging and preview deployments of client web applications.

Its immediate initial use case is providing authorized stakeholders of the United States Antenna Products, LLC (USAP) website rebuild with secure external review access. Over the long term, the platform is designed to serve as a general-purpose staging boundary for containerized workloads across different frameworks (ASP.NET Core, static sites, React/Vite, Node, Python, etc.) under the Version III product vision.

---

## 2. Current Status & MVP Scope

- **Current Status:**
  - **CP-001 / CP-001R1:** Local Development Foundation and loopback binding completed and validated.
  - **CP-002 / CP-002R1:** Protected Local Cloudflare Quick Tunnel Proof completed and validated.
  - **CP-003:** Remote Preview Host Provisioning (DigitalOcean Droplet `client-staging-01`, Ubuntu 24.04 LTS, SSH hardening, Docker, cloudflared) completed and validated.
  - **CP-004:** Remote Protected Preview Infrastructure Proof (loopback-only origin, direct-IP negative test, email OTP allowlisting, tunnel teardown) completed and validated.
  - **CP-004R1 (Current):** Remote Host Documentation & Infrastructure Baseline Reconciliation.
- **Current MVP Objective:** Provide an on-demand, zero-trust external review pathway for containerized client previews running on an isolated Remote Preview Host, protected by Cloudflare Access email One-Time PIN (OTP) authentication.
- **Next Milestone:**
  - **CP-005:** USAP Preview Deployment Integration (packaging and deploying the USAP client web application preview onto the remote staging host).
- **Future Platform Milestones:**
  - **CP-006+:** Named Cloudflare Tunnels, custom branded domain routing (`preview.example.com`), and persistent systemd service management.

> [!IMPORTANT]
> **Architectural & Boundary Notes:**
> - **Provider-Neutral Concept:** The platform architecture defines a generic **Remote Preview Host**. The current reference implementation is deployed on a low-cost DigitalOcean Droplet (`client-staging-01`), but the design retains complete portability to Google Cloud Compute Engine, other VPS providers, or local workstations.
> - **Separation from Client Code:** This repository is strictly decoupled from client application repositories (including `usap-website`). Client applications are deployed as isolated containers.
> - **Ephemeral Quick Tunnel Ingress:** The `*.trycloudflare.com` URL is an ad-hoc, temporary testing/review ingress intended for coordinated stakeholder sessions; no permanent public endpoint or custom domain is active for the MVP.
> - **Non-Public Origin Isolation:** The remote origin binds exclusively to loopback (`127.0.0.1:8080`) behind a cloud firewall permitting SSH only. Direct public access to the application origin is strictly prohibited and empirically verified as unreachable.

---

## 3. Platform Architecture & Ingress Flow

```text
External Client Reviewer
          │
          ▼ HTTPS (*.trycloudflare.com)
+─────────────────────────────────────────────+
│ Cloudflare Zero Trust Edge                  │
│  - Email Allowlist Verification             │
│  - One-Time PIN (OTP) Challenge             │
+─────────────────────────────────────────────+
          │
          ▼ Outbound Encrypted Tunnel (QUIC / TLS)
+─────────────────────────────────────────────────────────────+
│ Remote Preview Host (DigitalOcean Droplet / Ubuntu 24.04)    │
│  Firewall: Inbound TCP 22 (SSH) only (All app ports closed) │
│                                                             │
│   cloudflared (2026.9.3)                                    │
│          │                                                  │
│          ▼ HTTP (Host Loopback: 127.0.0.1:8080)             │
│   Docker Container Origin                                   │
│   (preview-test / future client preview)                    │
+─────────────────────────────────────────────────────────────+
```

---

## 4. Prerequisites

### Local Development Workstation
- **Git:** >= 2.40
- **Docker:** Engine >= 24.0 (Docker Desktop or Linux Docker Engine)
- **Docker Compose:** v2 or v5 plugin (`docker compose`)
- **cloudflared:** Use a current `cloudflared` release that supports the `--allowed-mail` option (validated with `cloudflared 2026.9.3`).

### Remote Preview Host Baseline (DigitalOcean Reference)
- **Operating System:** Ubuntu 24.04 LTS x64 (Linux Kernel 6.8+)
- **Compute:** 1 vCPU, 1 GB RAM, 25 GB SSD (with 1.0 GiB swapfile)
- **Firewall:** DigitalOcean Cloud Firewall (`client-staging-ssh-only`: Inbound TCP 22 only)
- **Administrative Account:** `stagingadmin` (SSH public-key auth only; root login and password auth disabled; `sudo` & `docker` group membership)
- **Docker Engine:** 29.8.2 (Official Docker APT repository)
- **Docker Compose Plugin:** 5.6.0
- **cloudflared:** 2026.9.3 (Official Cloudflare package repository)

---

## 5. Local Quick Start

### 5.1 Configuration (Optional)

Copy the example configuration to `.env` if custom port or container naming is desired:

```bash
cp .env.example .env
```

Default settings map port `8080` on host loopback (`127.0.0.1`) to container port `80`, ensuring the service is isolated from local network (LAN) and external interfaces.

### 5.2 Build and Start the Workload

```bash
# Validate Compose configuration
docker compose config

# Build the test workload image
docker compose build

# Start the container in detached mode
docker compose up -d

# Verify container status
docker compose ps

# Inspect logs
docker compose logs -f preview-test
```

### 5.3 Expected Local URL

Once running, navigate to:

```text
http://localhost:8080
```

The browser will display the **Client Staging Platform — Infrastructure Test** confirmation card indicating status: `Running` and environment: `Local`.
Only the host loopback interface (`127.0.0.1`) can reach this port.

### 5.4 Protected Quick Tunnel (External Preview Proof)

To securely expose the local loopback origin to an external reviewer using Cloudflare Access email allowlisting:

```bash
cloudflared tunnel --url http://127.0.0.1:8080 --allowed-mail reviewer@example.com
```

Cloudflare generates an ephemeral `https://*.trycloudflare.com` URL protected by email One-Time PIN (OTP). Unapproved visitors are blocked at the Cloudflare edge.

### 5.5 Stopping the Workload

To stop the tunnel, terminate the `cloudflared` process (`Ctrl+C`).
To tear down the running container and its network:

```bash
docker compose down
```

---

## 6. Documentation Directory

Detailed architectural decisions, security boundaries, operational runbooks, and baselines are organized under `docs/` and infrastructure directories:

- [Project Overview](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/PROJECT_OVERVIEW.md) — Business context, scope, exclusions, and phased milestones.
- [Architecture](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/ARCHITECTURE.md) — Request flow progressions, host neutrality, and technical principles.
- [Remote Host Baseline](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/REMOTE_HOST_BASELINE.md) — Technical baseline and specifications for the provisioned DigitalOcean Droplet.
- [Security](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/SECURITY.md) — Deny-by-default access, SSH hardening, secrets hygiene, and identity boundaries.
- [Deployment](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/DEPLOYMENT.md) — Operational runbook for local and remote preview workflows.
- [Branding](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/BRANDING.md) — Configuration-driven branding (CES provisional / Version III reusable).
- [Decision Log](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/DECISION_LOG.md) — Architectural Decision Records (ADRs).
- [Work Log](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/WORK_LOG.md) — Chronological checkpoint implementation log.
- [Docker Infrastructure](file:///C:/Users/Regin/Source/repos/client-staging-platform/infrastructure/docker/README.md) — Docker container baseline and Compose specs.
- [Cloudflare Infrastructure](file:///C:/Users/Regin/Source/repos/client-staging-platform/infrastructure/cloudflare/README.md) — Protected Quick Tunnel operations guide and specifications.
- [Scripts](file:///C:/Users/Regin/Source/repos/client-staging-platform/scripts/README.md) — Scripts directory policy.
