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

> **An authorized client stakeholder can open a secure HTTPS staging URL from a normal browser, authenticate using an approved email address and one-time code (OTP), and review the actual client application while unauthorized visitors cannot access it.**

---

## 6. Checkpoint Development Progression

The project strictly follows a phased checkpoint model to validate each layer before proceeding to the next:

| Checkpoint | Scope | Status |
| :--- | :--- | :--- |
| **CP-001** | **Repository & Local Development Foundation** — Lean repository structure, Git discipline, Docker Compose baseline, and framework-neutral static test workload. | **Active / Implemented** |
| **CP-002** | **Protected Cloudflare Quick Tunnel Proof** — Evaluation and testing of `cloudflared` Quick Tunnel with Cloudflare Access email allowlisting/OTP. | Planned |
| **CP-003** | **Remote MVP Host Deployment** — Provisioning and container execution on a temporary remote host (evaluating Google Cloud Compute Engine or equivalent). | Planned |
| **CP-004** | **USAP Client Workload Integration** — Packaging and deploying the USAP preview workload onto the staging host. | Planned |
| **CP-005+**| **Stable Domain & Named Tunnels** — Transitioning to custom branded domain and persistent Cloudflare Tunnels if needed. | Deferred |

---

## 7. Current CP-001 Scope & Exclusions

### In Scope for CP-001
- Clean, provider-neutral repository structure.
- Docker Compose configuration and framework-neutral static container workload.
- Comprehensive foundational architecture, security, deployment, and branding documentation.
- Rigorous local lifecycle validation (`build`, `up -d`, `ps`, HTTP response, `down`, recreation).

### Explicit Exclusions for CP-001
- No Cloudflare account, tunnel, or access configuration.
- No Google Cloud provisioning, billing, or resource creation.
- No public DNS or custom domain changes.
- No custom authentication, user tables, or passwords.
- No platform database (no EF Core, PostgreSQL, etc.).
- No client application source code imported or modified.
- No automated deployment pipelines or CI/CD actions.
