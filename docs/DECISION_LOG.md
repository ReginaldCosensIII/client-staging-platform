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
