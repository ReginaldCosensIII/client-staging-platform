# Project Overview: Client Staging Platform

## 1. Business Need & Context

During the rebuild of the United States Antenna Products, LLC (USAP) website, a critical operational requirement emerged: authorized client stakeholders need external, secure access to review interactive staging and preview builds without exposing unfinished work to the general public or search engines.

Traditionally, staging environments often suffer from either:
1. Being restricted to local or corporate internal networks (preventing convenient client review); or
2. Being exposed on unprotected public URLs or relying on basic auth / security-through-obscurity (exposing draft intellectual property, customer data, and branding).

The **Client Staging Platform** resolves this challenge by creating a secure, isolated staging runtime governed by external zero-trust identity verification.

---

## 2. Platform Separation & Repository Boundary

This repository (`client-staging-platform`) is an independent platform infrastructure project.

- **Isolation from USAP Source:** This repository does NOT contain the USAP website codebase, database, or application secrets. The client website source remains in its dedicated repository (`usap-website`).
- **Isolation from CES Dev:** The existing internal CES Development environment remains completely untouched and undisturbed.
- **Provider & Framework Neutrality:** The platform is conceived as a generic staging host. Although USAP is the immediate first tenant, the platform architecture does not bake in USAP-specific dependencies or assumptions.

---

## 3. Product Vision: Version III Staging Portal

Over the long term, this project forms the foundational infrastructure for the **Version III Client Preview / Staging Portal**.

The system is designed to host containerizable applications across diverse technology stacks:
- ASP.NET Core;
- Static HTML/CSS/JavaScript;
- React / Vite;
- Node.js;
- Python / FastAPI / Django;
- WordPress / CMS workloads.

By decoupling the containerized application packaging boundary from the access and routing layers, Version III can serve multiple client preview projects efficiently.

---

## 4. Branding Strategy

To allow swift delivery for the USAP milestone while protecting future multi-client reuse:
- **Provisional Brand:** CES branding may be provisionally configured for the first USAP preview milestone.
- **Future Reusability:** Version III branding (or custom tenant branding) will be configurable via environment settings without requiring application redesign or refactoring.
- **Branding Separation:** Platform authentication and review chrome remain configuration-driven.

---

## 5. Overarching MVP Success Condition

The overarching success condition for the platform MVP is:

> **An authorized client stakeholder can open a secure HTTPS staging URL from a normal browser, authenticate using an approved email address and one-time code (OTP), and review the actual client application running on a remote preview host, while unauthorized visitors cannot access it and the origin server exposes no public application ports.**

---

## 6. Checkpoint Development Progression

The project strictly follows a phased checkpoint model to validate each layer before proceeding to the next:

| Checkpoint | Scope | Status |
| :--- | :--- | :--- |
| **CP-001** | **Repository & Local Development Foundation** — Lean repository structure, Git discipline, Docker Compose baseline, and framework-neutral static test workload. | **Completed** |
| **CP-001R1**| **Localhost Binding Hardening** — Explicit loopback (`127.0.0.1`) host binding repair to eliminate LAN exposure. | **Completed** |
| **CP-002** | **Protected Local Cloudflare Quick Tunnel Proof** — Evaluation and testing of `cloudflared` Quick Tunnel with Cloudflare Access email allowlisting/OTP on local host. | **Completed / Validated** |
| **CP-002R1**| **cloudflared Installation & Version Normalization** — Binary path deduplication and documentation version floor correction. | **Completed** |
| **CP-003** | **Remote Preview Host Provisioning** — Provisioning of remote Linux host (DigitalOcean Droplet `client-staging-01`, Ubuntu 24.04 LTS, SSH hardening, swap, Docker, cloudflared). | **Completed / Validated** |
| **CP-004** | **Remote Protected Preview Infrastructure Proof** — Remote validation of loopback origin, Cloud Firewall, direct-IP negative proof, protected Quick Tunnel OTP authentication, and clean teardown. | **Completed / Validated** |
| **CP-004R1**| **Remote Host Documentation & Infrastructure Baseline Reconciliation** — Align repository documentation with proven DigitalOcean remote host baseline while maintaining provider-neutral architecture. | **Completed** |
| **CP-004R2**| **Remote Preview Documentation Accuracy Repair** — Correct authorized reviewer acceptance evidence, Docker execution wording, OTP language, and roadmap certainty. | **Current Checkpoint** |
| **CP-005** | **USAP Client Workload Integration** — Packaging and deploying the USAP client web application preview onto the remote staging host. | Planned Next |
| **Future** | **Stable Domain & Named Tunnels (Candidate Direction)** — A later production-style evolution may use a named Cloudflare Tunnel, stable custom hostname, and persistent background service management if justified by project needs. | Deferred |

---

## 7. Current Architecture & Provider Implementation

### 7.1 Proven Remote Architecture
The platform has proven the end-to-end remote preview workflow:
1. **Remote Preview Host:** Dedicated Linux compute instance with inbound traffic restricted by cloud firewall strictly to SSH (TCP 22). No public application ports (80, 443, 8080) are open.
2. **Private Origin Isolation:** The preview workload runs inside Docker (validated via loopback-published `docker run`), published strictly to the host loopback interface (`127.0.0.1:8080`). Direct requests to the host public IP time out.
3. **Protected Quick Tunnel:** An on-demand `cloudflared` process creates an outbound encrypted tunnel to the Cloudflare Edge using `--allowed-mail`.
4. **Edge Zero-Trust Challenge:** Cloudflare Access intercepts incoming HTTPS requests, challenging visitors for an allowlisted email and Cloudflare Access one-time PIN (OTP). Unauthorized users receive a 403 Forbidden response and never reach the origin.
5. **Independent Teardown:** Terminating the tunnel process immediately revokes external access (edge returns Cloudflare 530) while the private application origin remains healthy.

### 7.2 Current Provider Implementation
- **Remote Host Provider:** DigitalOcean (Project: `client-staging-platform`, Region: NYC1, Droplet: `client-staging-01`).
- **Provider Portability:** The platform maintains a provider-neutral abstraction. The current reference deployment runs on a 1 vCPU / 1 GB RAM DigitalOcean Droplet ($6/month tier). The architecture retains complete portability to Google Cloud Compute Engine, other cloud VPS providers, or on-premises servers without modifying container configurations.

### 7.3 Scope Exclusions for Current Platform Baseline
- **No Client Application Source:** This repository does not store or build the USAP website source code. USAP integration occurs in CP-005 via container deployment.
- **No Permanent Named Tunnels or Custom DNS:** A later production-style evolution may use a named Cloudflare Tunnel, stable custom hostname, and persistent background service management if justified by project needs. These remain deferred during current MVP phases.
- **No Platform Database:** Staging authentication is handled entirely at the edge; no database is deployed or required.
- **No Background System Service for Tunnel:** The MVP relies on manually initiated Quick Tunnels for scheduled stakeholder review windows.
