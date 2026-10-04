# Client Staging Platform

> **Working / Product Concept:** Version III Client Preview & Staging Portal
> **Repository:** `client-staging-platform`
> **Current Checkpoint:** CP-001 — Repository & Local Development Foundation

---

## 1. Overview & Purpose

The **Client Staging Platform** is a reusable, provider-neutral software and infrastructure project designed to host secure, private staging and preview deployments of client web applications.

Its immediate initial use case is providing authorized stakeholders of the United States Antenna Products, LLC (USAP) website rebuild with secure external review access. Over the long term, the platform is designed to serve as a general-purpose staging boundary for containerized workloads across different frameworks (ASP.NET Core, static sites, React/Vite, Node, Python, etc.) under the Version III product vision.

---

## 2. Current Status & MVP Scope

- **Current Status:** Foundation established (CP-001). A lightweight, framework-neutral static container workload verifies local Docker packaging, container execution, networking, and service lifecycle.
- **Current MVP Objective:** Validate local development discipline, Docker Compose operations, and baseline repository documentation.
- **Upcoming Phases:**
  - **CP-002:** Protected Cloudflare Quick Tunnel proof with email one-time pin (OTP) allowlisting.
  - **Later Checkpoints:** Remote MVP preview hosting (evaluating Google Cloud Compute Engine or comparable Docker-capable host) and packaging the actual client application.

> [!IMPORTANT]
> **Boundary Notes:**
> - **Separation from Client Code:** This repository is strictly decoupled from client application repositories (including `usap-website`). No client application code or CES Dev infrastructure is modified during CP-001.
> - **Cloudflare Integration:** Tunnel and access policies belong strictly to CP-002 and subsequent checkpoints.
> - **Remote Infrastructure:** Google Cloud Compute Engine is an evaluated temporary remote MVP host candidate, but is not a hard-coded architectural dependency. No remote cloud resources are provisioned in CP-001.

---

## 3. Prerequisites

To run and validate the local development environment, the host machine requires:

- **Git:** >= 2.40
- **Docker:** Engine >= 24.0 (Docker Desktop or Linux Docker Engine)
- **Docker Compose:** v2 or v5 plugin (`docker compose`)

*(Note: `cloudflared` is an optional prerequisite reserved for CP-002; .NET SDK is not required for running the framework-neutral static test workload).*

---

## 4. Local Quick Start

### 4.1 Configuration (Optional)

Copy the example configuration to `.env` if custom port or container naming is desired:

```bash
cp .env.example .env
```

Default settings map port `8080` on host loopback (`127.0.0.1`) to container port `80`, ensuring the service is isolated from local network (LAN) and external interfaces.

### 4.2 Build and Start the Workload

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

### 4.3 Expected Local URL

Once running, navigate to:

```text
http://localhost:8080
```

The browser will display the **Client Staging Platform — Infrastructure Test** confirmation card indicating status: `Running` and environment: `Local`.

### 4.4 Stopping the Workload

To tear down the running container and its network:

```bash
docker compose down
```

---

## 5. Documentation Directory

Detailed architectural decisions, security boundaries, and roadmaps are organized under `docs/` and infrastructure directories:

- [Project Overview](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/PROJECT_OVERVIEW.md) — Business context, scope, exclusions, and phased milestones.
- [Architecture](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/ARCHITECTURE.md) — Request flow progressions, host neutrality, and technical principles.
- [Security](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/SECURITY.md) — Deny-by-default access, secrets hygiene, and identity boundaries.
- [Deployment](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/DEPLOYMENT.md) — Local vs. Quick Tunnel vs. remote MVP host progression.
- [Branding](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/BRANDING.md) — Configuration-driven branding (CES provisional / Version III reusable).
- [Decision Log](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/DECISION_LOG.md) — Architectural Decision Records (ADRs).
- [Work Log](file:///C:/Users/Regin/Source/repos/client-staging-platform/docs/WORK_LOG.md) — Chronological checkpoint implementation log.
- [Docker Infrastructure](file:///C:/Users/Regin/Source/repos/client-staging-platform/infrastructure/docker/README.md) — Docker container baseline and Compose specs.
- [Cloudflare Infrastructure](file:///C:/Users/Regin/Source/repos/client-staging-platform/infrastructure/cloudflare/README.md) — CP-002 forward-looking specifications.
- [Scripts](file:///C:/Users/Regin/Source/repos/client-staging-platform/scripts/README.md) — Scripts directory policy.
