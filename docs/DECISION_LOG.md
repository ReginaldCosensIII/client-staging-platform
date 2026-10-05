# Architecture Decision Log (ADR)

This log documents all major architectural and operational decisions accepted for the Client Staging Platform.

---

### ADR-001: Dedicated Repository for Platform Infrastructure
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** Staging infrastructure could theoretically be embedded inside the client project repo (`usap-website`).
- **Decision:** Create an independent, dedicated repository named `client-staging-platform` (`https://github.com/ReginaldCosensIII/client-staging-platform.git`).
- **Consequences:** Clean separation of concerns. Protects client code from infrastructure churn and enables platform reuse across multiple client projects under Version III.

---

### ADR-002: Docker as Primary Workload Packaging Boundary
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** The platform must support heterogeneous application stacks (ASP.NET Core, static sites, Node, React, Python).
- **Decision:** Standardize on Docker / OCI container images as the universal deployment and packaging boundary.
- **Consequences:** The platform runtime remains framework-agnostic. Any containerizable application can be previewed without modifying the hosting layer.

---

### ADR-003: Provider-Neutral Preview Host Abstraction
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** Early infrastructure choices could easily lead to proprietary public cloud lock-in (e.g. Google Cloud Run or AWS ECS proprietary configurations).
- **Decision:** Define the "Preview Host" as an agnostic Linux instance running standard Docker and `cloudflared`.
- **Consequences:** The platform can run on a local workstation, Google Cloud Compute Engine, or a low-cost VPS without architectural rewrites.

---

### ADR-004: No Database for Platform MVP
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** Traditional staging platforms frequently incorporate databases for user accounts, sessions, and state.
- **Decision:** Prohibit any platform database (PostgreSQL, SQL Server, SQLite, EF Core) for the initial MVP.
- **Consequences:** Greatly reduces security attack surface, backup/restore overhead, and operational maintenance. Access control is managed at the network edge.

---

### ADR-005: Cloudflare Zero Trust Edge Authentication Over Custom Identity
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** Client stakeholders require simple, secure authentication to view staging builds.
- **Decision:** Use Cloudflare Access email allowlisting with One-Time Pin (OTP) instead of building custom user registrations, password hashing, and login screens.
- **Consequences:** Eliminates credential theft risks, password reset flows, and session database maintenance. Security is enforced before traffic reaches the host.

---

### ADR-006: Evaluation of Protected Quick Tunnel for CP-002
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** Need a fast, low-friction proof of external access and zero-trust authentication before provisioning remote servers.
- **Decision:** Plan for CP-002 to test an ephemeral Cloudflare Quick Tunnel (`cloudflared tunnel`) paired with Cloudflare Access.
- **Consequences:** Validates the complete reviewer user experience locally without incurring cloud hosting costs.

---

### ADR-007: Acceptance of Ephemeral Hostname for Initial External Proof
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** Setting up custom domain DNS and certificates requires domain management overhead.
- **Decision:** Accept temporary `*.trycloudflare.com` hostnames for early checkpoint validation and initial MVP testing.
- **Consequences:** Fast iteration and zero DNS mutation on production domains during early milestones.

---

### ADR-008: Deferral of Custom Domains and Named Cloudflare Tunnels
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** Custom domains (e.g., `preview.example.com`) and named persistent tunnels add setup complexity.
- **Decision:** Defer custom domains and named tunnels to later checkpoints once the foundational workflow is proven.
- **Consequences:** Reduces CP-001/CP-002 scope to essential mechanics.

---

### ADR-009: Google Cloud Compute Engine Preferred for Temporary Remote MVP
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** A remote host is required so stakeholders can review staging without the developer's local PC running 24/7.
- **Decision:** Prefer Google Cloud Compute Engine (e2-micro/small VM) as the temporary remote MVP host candidate, while deferring actual GCP provisioning to CP-003+.
- **Consequences:** Fast spin-up and teardown in existing cloud accounts; zero cloud provisioning in CP-001.

---

### ADR-010: Remote Hosting Must Remain Replaceable
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** Long-term hosting costs should remain minimal.
- **Decision:** Explicitly mandate that Google Cloud or any other cloud provider must remain replaceable with an inexpensive VPS or other Docker host at any time.
- **Consequences:** Prevents coupling to cloud-specific services or proprietary networking features.

---

### ADR-011: Configurable Branding (Provisional CES / Long-Term Version III)
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** The immediate client engagement expects CES presentation, while the platform asset is intended for Version III.
- **Decision:** Treat all branding elements (names, logos, messages, accents) as configuration parameters rather than hard-coded templates.
- **Consequences:** Dual-use capability without code branching or redesign.

---

### ADR-012: Non-Disruption of Existing CES Development Environment
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** Active development is underway on other systems within CES infrastructure.
- **Decision:** The Client Staging Platform must not alter, bind to, or disrupt existing CES Dev instances or configurations.
- **Consequences:** Absolute isolation of environments and zero risk of cross-project downtime.

---

### ADR-013: Staging Platform Proof Precedes Client Application Integration
- **Date:** 2026-10-04
- **Status:** Accepted
- **Context:** It is tempting to immediately try to containerize the USAP website and configure staging concurrently.
- **Decision:** Prove the infrastructure, Docker Compose lifecycle, and tunnel mechanics first using a trivial static test container before integrating the real client application.
- **Consequences:** Decouples infrastructure debugging from application-level debugging, preventing confusing failure modes.

---

### ADR-014: Protected Cloudflare Quick Tunnel Validated for Initial MVP Ingress
- **Date:** 2026-10-04
- **Status:** Accepted (Validated in CP-002)
- **Context:** An external ingress mechanism is required to allow remote client stakeholders to review preview builds securely without public cloud provisioning or DNS complexity.
- **Decision:** Validate and adopt Protected Cloudflare Quick Tunnels (`cloudflared tunnel --allowed-mail ...`) as the initial testing and MVP ingress pattern.
- **Rationale:**
  - $0 operational cost for temporary staging access.
  - Requires no Cloudflare account or API credentials.
  - Requires no custom domain or DNS modifications during testing phases.
  - Enforces edge email allowlisting with One-Time PIN (OTP) verification.
  - Suitable for scheduled, human-in-the-loop stakeholder review sessions.
  - Explicitly recognized as non-permanent: not accepted as permanent production/staging ingress.
- **Consequences:** Provides an immediate, zero-cost, zero-trust review workflow. Permanent custom domains and persistent named tunnels remain deferred to future multi-tenant platform milestones.

---

### ADR-015: Strict Host Loopback Binding Preserved Across Ingress Layers
- **Date:** 2026-10-04
- **Status:** Accepted (Validated in CP-001R1 & CP-002)
- **Context:** Workload container ports could inadvertently be published across `0.0.0.0` or local network interfaces when adding tunnel connectors.
- **Decision:** Enforce that container ports bind strictly to host loopback (`127.0.0.1:8080`) on both local workstations and future remote preview hosts.
- **Consequences:** Prevents accidental LAN or direct public IP exposure. All external traffic must transit through the authorized edge authentication layer.

---

### ADR-016: DigitalOcean Selected as Initial Remote Preview Host Implementation
- **Date:** 2026-10-05
- **Status:** Accepted (Implemented in CP-003)
- **Context:** A remote compute instance was needed to validate the remote preview architecture. Google Cloud Compute Engine had been initially evaluated, but required account activation/credit steps.
- **Decision:** Select DigitalOcean as the initial infrastructure provider for the Remote Preview Host (`client-staging-platform` project, `client-staging-01` Droplet).
- **Rationale:**
  - Extremely low compute cost ($6/month, ~$0.009/hour).
  - Immediate availability with zero setup friction.
  - Complete isolation from CES internal infrastructure.
  - Provides practical, hands-on Linux VPS and container management experience.
  - Retains complete provider neutrality; Google Cloud Compute Engine evaluation remains deferred and fully viable for future migration.
- **Consequences:** Proves remote staging without high cloud overhead. Architecture remains provider-neutral and portable.

---

### ADR-017: NYC1 Region Selected Due to Plan Availability
- **Date:** 2026-10-05
- **Status:** Accepted (Implemented in CP-003)
- **Context:** Droplets can be provisioned in multiple data centers. The project initially considered Richmond or other nearby regions.
- **Decision:** Provision the staging Droplet in DigitalOcean's `NYC1` (New York 1) region.
- **Rationale:** The bundled $6 Basic plan (1 vCPU, 1 GB RAM, 25 GB SSD) was unavailable in Richmond at provisioning time. NYC1 offered immediate inventory at the target price point.
- **Consequences:** NYC1 is adopted as a tactical deployment choice; it is not an architectural constraint.

---

### ADR-018: 1 GB Basic Droplet Selected for Remote Preview Infrastructure Proof
- **Date:** 2026-10-05
- **Status:** Accepted (Implemented in CP-003)
- **Context:** Need to select Droplet compute specifications for CP-003 and CP-004 proofs.
- **Decision:** Provision the minimum viable Basic Droplet (1 vCPU, 1 GB RAM, 25 GB SSD, $6/month).
- **Rationale:**
  - Sizing is entirely adequate to prove OS hardening, Docker runtime, and `cloudflared` Quick Tunnel ingress.
  - Minimizes financial commitment during early experimental development.
  - Resource consumption (CPU/RAM) can be empirically measured under actual client workloads in CP-005 before committing to vertical resizing.
- **Consequences:** Requires swap configuration to guard against OOM spikes; sizing sufficiency will be re-evaluated when hosting client application workloads.

---

### ADR-019: Remote Host Administrative User & SSH Key Hardening
- **Date:** 2026-10-05
- **Status:** Accepted (Implemented in CP-003)
- **Context:** Default cloud VPS instances often allow root login or password authentication, posing an immediate security vulnerability.
- **Decision:** Create a dedicated non-root administrative account (`stagingadmin`), enforce SSH public-key authentication (`PubkeyAuthentication yes`), disable SSH password authentication (`PasswordAuthentication no`), and explicitly disable root SSH login (`PermitRootLogin no`).
- **Rationale:**
  - Industry-standard least-privilege administrative baseline.
  - Eliminates SSH password brute-force attack vectors.
  - Verified post-hardening: root SSH login fails with `Permission denied (publickey)`.
- **Consequences:** Requires private key management on administrator workstations; keys must never be stored in source control.

---

### ADR-020: DigitalOcean Cloud Firewall Restricts Inbound to SSH Only
- **Date:** 2026-10-05
- **Status:** Accepted (Implemented in CP-003)
- **Context:** The remote Droplet is assigned a public IPv4 address. Exposing application ports (80, 443, 8080) directly to the internet violates zero-trust principles.
- **Decision:** Apply DigitalOcean Cloud Firewall (`client-staging-ssh-only`) permitting inbound traffic strictly on TCP port 22 (SSH). All web and application ports are blocked at the perimeter.
- **Rationale:**
  - External preview traffic transits exclusively via the outbound encrypted tunnel (`cloudflared`).
  - Perimeter firewall blocks automated port scanners and unauthorized direct connections.
  - Negative test confirmed: direct external requests to `http://<DROPLET_PUBLIC_IP>:8080` time out.
- **Consequences:** No public web ingress ports are ever opened on the host.

---

### ADR-021: 1.0 GiB Persistent Swapfile for Host Memory Resilience
- **Date:** 2026-10-05
- **Status:** Accepted (Implemented in CP-003)
- **Context:** The 1 GB RAM Basic Droplet was provisioned with zero swap space by default, creating vulnerability to sudden OOM process kills during Docker builds or package updates.
- **Decision:** Configure a 1.0 GiB persistent swapfile (`/swapfile`), secured with `0600` permissions and mounted via `/etc/fstab`.
- **Rationale:**
  - Provides a safety buffer to absorb short-lived memory spikes without crashing the kernel or terminating critical processes.
  - Empirically verified to survive host reboot.
- **Consequences:** Improves host resilience; does not substitute for physical RAM if application workloads consistently exceed 1 GB.

---

### ADR-022: Official Docker Ubuntu APT Repository for Container Runtime
- **Date:** 2026-10-05
- **Status:** Accepted (Implemented in CP-003)
- **Context:** Docker can be installed via Ubuntu default repositories (`docker.io`), snap packages, or Docker's official upstream APT repository.
- **Decision:** Install Docker Engine (v29.8.2) and the Docker Compose plugin (v5.6.0) from Docker's official Ubuntu repository (`download.docker.com`).
- **Rationale:**
  - Ensures access to current container runtime features, security patches, and the native `docker compose` V2/V5 CLI plugin.
  - Avoids outdated distribution packages or snap confinement quirks.
- **Consequences:** Standardized modern container tooling matching developer workstation environments.

---

### ADR-023: Manual Protected Quick Tunnel Execution for Remote MVP
- **Date:** 2026-10-05
- **Status:** Accepted (Validated in CP-004)
- **Context:** Ingress for stakeholder preview sessions can be run continuously as a background service or manually launched on-demand.
- **Decision:** Operate the Protected Cloudflare Quick Tunnel (`cloudflared tunnel --allowed-mail ...`) manually in foreground/terminal sessions for the MVP phase.
- **Rationale:**
  - Aligns with coordinated, scheduled stakeholder review windows.
  - Clean separation: stopping the tunnel process (`Ctrl+C`) immediately revokes all external access while leaving the workload running privately.
  - Zero ongoing background ingress exposure when review sessions are not actively in progress.
- **Consequences:** Requires manual execution by the administrator to start review sessions; persistent background services remain deferred to CP-006+.

---

### ADR-024: Deferral of Persistent cloudflared Service Daemon
- **Date:** 2026-10-05
- **Status:** Accepted (Decided in CP-004)
- **Context:** `cloudflared` can be registered as a systemd service (`cloudflared service install`).
- **Decision:** Do NOT install or enable a persistent systemd service for `cloudflared` during the current MVP.
- **Rationale:**
  - Quick Tunnels are ephemeral and generate dynamic URLs upon restart. An auto-restarting Quick Tunnel daemon would generate new, untracked URLs without administrator knowledge.
  - A persistent system service is only appropriate when paired with named tunnels and stable custom domains in CP-006+.
- **Consequences:** Preserves operator awareness and intentional session management.

---

### ADR-025: Deferral of Custom Domains and Named Persistent Tunnels
- **Date:** 2026-10-05
- **Status:** Accepted (Reaffirmed in CP-004)
- **Context:** Staging could use permanent branded DNS names (e.g., `preview.example.com`) and named Cloudflare Tunnels requiring Cloudflare account authentication.
- **Decision:** Retain the decision to defer custom domains and named tunnels to CP-006+.
- **Rationale:**
  - The protected Quick Tunnel completely proves external stakeholder access and OTP authentication with zero DNS overhead and $0 Cloudflare costs.
  - Keeps CP-004 and CP-005 focused on host provisioning and client application integration.
- **Consequences:** Quick Tunnel hostnames remain ephemeral for early testing.

---

### ADR-026: Origin Isolation Enforced Independently of Cloudflare Ingress
- **Date:** 2026-10-05
- **Status:** Accepted (Validated in CP-004)
- **Context:** An external tunnel with edge authentication might encourage lax origin port security.
- **Decision:** Formally mandate that origin isolation (loopback-only binding + perimeter firewall) must remain strictly enforced independently of Cloudflare authentication.
- **Rationale:**
  - Defense-in-depth: If the tunnel configuration is altered or edge policies change, the origin remains unreachable from the public internet.
  - Direct negative test proved that `http://<DROPLET_PUBLIC_IP>:8080` times out from external networks.
- **Consequences:** Cloudflare Access is never treated as a substitute for host-level network perimeter defense.
