# Platform Architecture: Client Staging Platform

## 1. Architectural Principles

The Client Staging Platform adheres to ten governing architectural principles:

1. **Framework Neutrality:** The platform packaging boundary is container-based (Docker). Any workload that can run inside an OCI-compliant container (ASP.NET Core, Node, static HTML, Python, etc.) can be hosted without requiring platform code modifications.
2. **Provider-Neutral Preview Host:** The Preview Host abstraction represents any Docker-capable Linux VM or compute instance. While a DigitalOcean Droplet (`client-staging-01`) serves as the current proven reference implementation, and Google Cloud Compute Engine remains a candidate for future evaluation, no proprietary cloud APIs or vendor locks are introduced into the platform architecture.
3. **Containerized Client Workloads:** All previewed applications run as isolated containers behind controlled internal port bindings or private Docker networks.
4. **Application / Infrastructure Separation:** Infrastructure management, tunneling, and authentication exist independently of client application source repositories.
5. **Authentication Precedes Exposure:** No client application bytes or assets are transmitted to any requestor before identity verification succeeds at the edge.
6. **No Database Without Justified Need:** Staging portals typically do not require dedicated transactional state storage for access control. A zero-trust edge layer eliminates the operational overhead and attack surface of a custom database.
7. **Zero-Trust Over Custom Identity:** Avoid implementing bespoke user tables, password hashes, and session cookies. Cloudflare Access handles email allowlisting, OTP challenge generation, and session lifecycle.
8. **Configurable Branding:** Visual presentation (brand names, logos, colors, review instructions) is driven by configuration, allowing seamless transitions between provisional client branding (CES) and platform branding (Version III).
9. **Replaceable Remote Hosting:** Host migration (e.g., from local workstation to DigitalOcean, and potentially later to Google Cloud or another VPS provider) requires only transferring the container definitions and starting the tunnel.
10. **Client Tenant Separation:** USAP is the inaugural staging workload, not the architecture itself. The platform remains a general-purpose staging utility.

---

## 2. Architecture Evolution Progression

### 2.1 CP-001: Local Development & Container Baseline (Validated)

Local developer workstation hosting a static verification container via Docker Compose.

```text
+-------------------------+
| Local Browser (Dev PC)  |
+-------------------------+
            |
            | HTTP: http://127.0.0.1:8080 (Loopback only)
            v
+-------------------------+
| Local Docker Host       |
|  +--------------------+ |
|  | preview-test (80)  | |
|  | (nginx:alpine)     | |
|  +--------------------+ |
+-------------------------+
```

---

### 2.2 CP-002: Protected Local Cloudflare Quick Tunnel Proof (Validated)

Proven external zero-trust ingress without public cloud infrastructure or open inbound firewall ports. A local `cloudflared` process creates an outbound-only QUIC/HTTP2 tunnel connection to Cloudflare Edge.

```text
+----------------------------+
| External Reviewer Browser  |
+----------------------------+
               |
               | HTTPS (Temporary *.trycloudflare.com)
               v
+----------------------------+
| Cloudflare Zero Trust Edge |
|  - Email OTP Allowlist     |
|  - Access Policy Shield    |
+----------------------------+
               |
               | Outbound Secure Tunnel (QUIC / TLS)
               v
+------------------------------------+
| Local Workstation / Host           |
|   +------------------------------+ |
|   | cloudflared agent            | |
|   +------------------------------+ |
|                  | HTTP: 127.0.0.1:8080 (Loopback only)
|                  v                 |
|   +------------------------------+ |
|   | preview-test (80)            | |
|   +------------------------------+ |
+------------------------------------+
```

---

### 2.3 CP-003 & CP-004: Proven Remote Preview Host Architecture (Validated)

The proven remote staging architecture deploys the container workload onto an isolated Remote Preview Host (currently implemented on DigitalOcean Droplet `client-staging-01`).

External review traffic enters strictly through Cloudflare Zero Trust Edge via an outbound tunnel. Direct public access to the server's application ports is completely blocked by the cloud firewall and loopback-only binding.

```text
===================================================================================
PROVEN REMOTE PREVIEW ARCHITECTURE (CP-003 / CP-004)
===================================================================================

[ External Authorized Reviewer ]
               │
               │ HTTPS (Temporary *.trycloudflare.com)
               ▼
+─────────────────────────────────────────────────────────────+
│ Cloudflare Zero Trust Edge                                  │
│  - Email Allowlist Verification                             │
│  - One-Time PIN (OTP) Challenge & Session Token             │
│  - Unauthorized emails rejected at edge (HTTP 403 Forbidden) │
+─────────────────────────────────────────────────────────────+
               │
               │ Outbound-Only Encrypted Tunnel (QUIC / TLS)
               ▼
+─────────────────────────────────────────────────────────────────────────────────+
│ Remote Preview Host: client-staging-01 (DigitalOcean Droplet, Ubuntu 24.04 LTS) │
│                                                                                 │
│   Cloud Firewall (client-staging-ssh-only):                                     │
│   - Inbound TCP 22 (SSH) allowed for stagingadmin (public-key only)             │
│   - ALL INBOUND APPLICATION PORTS (80, 443, 8080, etc.) PROHIBITED & BLOCKED    │
│   - Outbound traffic allowed for tunnel and package management                  │
│                                                                                 │
│   cloudflared (v2026.9.3)                                                       │
│          │                                                                      │
│          │ HTTP: http://127.0.0.1:8080 (Host Loopback Only)                     │
│          ▼                                                                      │
│   Docker Engine (v29.8.2)                                                       │
│   ┌───────────────────────────────────────────────────────────┐                 │
│   │ preview-test container (nginx:alpine, port 80)            │                 │
│   │ Launched via: docker run -d -p 127.0.0.1:8080:80 ...      │                 │
│   │ (Future: USAP Client Application Container in CP-005)     │                 │
│   └───────────────────────────────────────────────────────────┘                 │
+─────────────────────────────────────────────────────────────────────────────────+

[ Direct External Request ]
               │
               │ Direct HTTP to http://<DROPLET_PUBLIC_IP>:8080
               ▼
+─────────────────────────────────────────────────────────────+
│ Cloud Firewall & Host Loopback Binding                      │
│ ➔ RESULT: Connection Timed Out (Access Blocked)             │
+─────────────────────────────────────────────────────────────+
```

**Key Validated Properties:**
- **Zero Inbound App Exposure:** The host firewall permits ONLY inbound SSH (TCP 22). No web traffic ports (80, 443, 8080) are open to the internet.
- **Strict Loopback Binding:** The container engine publishes ports strictly to `127.0.0.1:8080`. Even within the host, external network adapters cannot reach the container.
- **Edge Authentication Precedes Access:** Cloudflare Access challenges all visitors for allowlisted email and Cloudflare Access one-time PIN (OTP) before proxying traffic.
- **Direct-Origin Negative Proof:** Direct requests to `http://<DROPLET_PUBLIC_IP>:8080` time out, proving origin isolation independently of Cloudflare.
- **Independent Teardown:** Terminating `cloudflared` (`Ctrl+C`) immediately severs external ingress (edge returns Cloudflare 530) while the local Docker origin remains healthy and running.

---

### 2.4 Future Platform Evolution: Named Tunnels & Custom Domains (Candidate Direction)

A later production-style evolution may use a named Cloudflare Tunnel, stable custom hostname, and persistent background service management if justified by project needs. The diagram below illustrates this potential long-term direction:

```text
+───────────────────────────────────+
│ External Reviewers & Stakeholders │
+───────────────────────────────────+
                  │
                  │ HTTPS: preview.example.com
                  ▼
+───────────────────────────────────+
│ Cloudflare Access Zero Trust      │
│  - Tenant-based Access Policies   │
│  - Custom Identity Providers / OTP│
+───────────────────────────────────+
                  │
                  │ Persistent Named Tunnel (cloudflared daemon)
                  ▼
+─────────────────────────────────────────────────────+
│ Dedicated Staging Host (VPS or Managed Compute)     │
│   +───────────────────────────────────────────────+ │
│   │ cloudflared.service (systemd managed)         │ │
│   +───────────────────────────────────────────────+ │
│          │                  │                  │    │
│          ▼                  ▼                  ▼    │
│   +─────────────+    +─────────────+    +─────────+ │
│   │ Tenant A    │    │ Tenant B    │    │ Version │ │
│   │ (USAP Site) │    │ (Client App)│    │ III Hub │ │
│   +─────────────+    +─────────────+    +─────────+ │
+─────────────────────────────────────────────────────+
```

---

## 3. Component Details & Security Boundaries

### 3.1 Host Boundary (Remote Preview Host)
- **Current Implementation:** DigitalOcean Droplet `client-staging-01` (NYC1 region, 1 vCPU, 1 GB RAM, 25 GB SSD).
- **Swap Resilience:** Configured with a persistent 1.0 GiB swapfile (`/swapfile`, configured in `/etc/fstab`) to provide a small memory buffer on the 1 GB instance without replacing physical RAM.
- **Operating System:** Ubuntu 24.04.5 LTS x64 (Kernel 6.8.0-146-generic). Standardized on the LTS release channel.
- **Administrative Access:** Non-root account `stagingadmin` with `sudo` privileges. SSH public-key authentication enforced; root SSH login disabled (`PermitRootLogin no`); password authentication disabled (`PasswordAuthentication no`).

### 3.2 Network & Cloud Firewall Boundary
- **DigitalOcean Cloud Firewall:** Profile `client-staging-ssh-only` restricts all inbound network traffic strictly to TCP 22 (SSH).
- **No Public Web Ingress:** Inbound ports 80, 443, 8080, 5000, 5001, etc., are closed at the cloud perimeter.
- **Outbound Connectivity:** Unrestricted outbound access allows the host to connect outbound to Cloudflare Edge servers (QUIC/UDP 7844 or HTTPS/TCP 443) and APT repositories.

### 3.3 Docker Container Boundary
- **Runtime:** Docker Engine 29.8.2 and Docker Compose plugin 5.6.0 installed from official Docker APT repositories. The remote proof workload was launched using a direct `docker run` command with loopback binding (`-p 127.0.0.1:8080:80`), with Compose supported across the platform.
- **Port Binding:** Container ports bind strictly to host loopback (`127.0.0.1:8080:80`). Binding to `0.0.0.0` is strictly prohibited.
- **Docker Privilege Implications:** Administrative membership in the `docker` group grants root-equivalent control over the host. This privilege is restricted to the trusted `stagingadmin` administrative account.

### 3.4 Ingress & Access Boundary (Protected Quick Tunnel)
- **Tooling:** Official `cloudflared` binary (v2026.9.3) at `/usr/bin/cloudflared` (symlinked from `/usr/local/bin/cloudflared`).
- **Access Policy:** Initiated with `--allowed-mail <APPROVED_REVIEWER_EMAIL>`. Cloudflare Access enforces email validation and Cloudflare Access one-time PIN (OTP) issuance.
- **Observed Non-Blocking Warnings:** `cloudflared` logged non-blocking warnings regarding ICMP ping group permissions and QUIC receive buffer size. These did not affect tunnel registration, QUIC transport, or OTP authentication.
- **Session Lifespan & Ephemerality:** Hostnames on `*.trycloudflare.com` are temporary and change upon each tunnel instantiation. Stopping `cloudflared` revokes external routing immediately.

### 3.5 Origin Isolation Independence
A protected Quick Tunnel must **never** be used as a substitute for origin isolation. The application origin must remain non-public independently of Cloudflare authentication. The combination of cloud firewall restrictions and loopback binding ensures defense-in-depth even if tunnel configuration is modified.

### 3.6 Provider Portability & Future Migrations
The Remote Preview Host architecture is completely decoupled from DigitalOcean-specific features. The identical deployment pattern (Linux, SSH hardening, swap, Docker, loopback binding, and `cloudflared`) can be instantiated on Google Cloud Compute Engine, another VPS provider, or an on-premises host whenever appropriate.
