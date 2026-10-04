# Deployment Strategy: Client Staging Platform

## 1. Multi-Tier Deployment Lifecycle

The deployment model distinguishes between four distinct operational tiers:

```text
Tier 1: Local Development (CP-001)
   ↓
Tier 2: Protected Quick Tunnel Proof (CP-002)
   ↓
Tier 3: Remote MVP Host (CP-003+)
   ↓
Tier 4: Stable Multi-Tenant Platform (Long-Term)
```

---

## 2. Detailed Tier Specifications

### Tier 1: Local Development (CP-001)
- **Host:** Local developer workstation (Windows 11 / WSL2).
- **Orchestration:** Docker Compose (`docker-compose.yml`).
- **Network Scope:** Loopback (`http://localhost:8080`).
- **Purpose:** Verifies container image build, container lifecycle, port binding, and base HTML rendering.
- **Access Boundary:** Local host only.

### Tier 2: Protected Quick Tunnel Proof (CP-002)
- **Host:** Local developer workstation.
- **Routing:** Ad-hoc Cloudflare Quick Tunnel (`cloudflared tunnel --url http://localhost:8080`).
- **Domain:** Ephemeral hostname on `*.trycloudflare.com`.
- **Access Boundary:** Cloudflare Access zero-trust application with email OTP allowlist.
- **Purpose:** Proves the external authentication boundary and remote connectivity without incurring cloud hosting costs or provisioning server infrastructure.

### Tier 3: Remote MVP Host (CP-003+)
- **Host:** Ephemeral cloud virtual machine (evaluating Google Cloud Compute Engine e2-micro/small).
- **Orchestration:** Docker and Docker Compose on remote Linux VM.
- **Routing:** `cloudflared` daemon running alongside workload container.
- **Firewall Profile:** Outbound-only connectivity; zero inbound ports exposed to the public internet (SSH access restricted via Google Cloud IAP or secure key).
- **Access Boundary:** Cloudflare Access zero-trust policy.
- **Target Workload:** The actual preview build of the client application (e.g. USAP website).
- **Purpose:** Allows stakeholders to review the staging site 24/7 without requiring the developer workstation to remain running.

### Tier 4: Stable Platform / Custom Domain (Future)
- **Host:** Portable Linux VPS (e.g., low-cost dedicated VPS, Linode/Akamai, DigitalOcean, Hetzner, or Google Cloud).
- **Domain:** Custom branded domain (e.g., `preview.example.com`).
- **Routing:** Named, persistent Cloudflare Tunnels managed via Cloudflare Dashboard or declarative configuration.
- **Access Boundary:** Multi-project access groups, per-client allowlists, and persistent review sessions.
- **Purpose:** Reusable, long-term staging portal for Version III client engagements.

---

## 3. Host Portability & Provider Neutrality

The platform deliberately avoids vendor-proprietary platform-as-a-service (PaaS) dependencies:
- **No Cloud-Specific APIs:** The host only requires standard OCI container tooling (`docker`, `docker compose`) and outbound HTTPS network connectivity.
- **Replaceable Infrastructure:** Transitioning from Google Cloud Compute Engine to any other Linux VPS provider involves:
  1. Provisioning a basic Linux instance (Ubuntu/Debian/Rocky).
  2. Installing Docker and `cloudflared`.
  3. Checking out the repository and running `docker compose up -d`.
  4. Launching the tunnel connector.
- Google Cloud is treated strictly as an immediate candidate for temporary staging compute, not a permanent architectural requirement.
