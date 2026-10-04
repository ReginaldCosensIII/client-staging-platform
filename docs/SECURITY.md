# Security Policy & Architecture: Client Staging Platform

## 1. Security Overview

The primary objective of the Client Staging Platform is to enable client stakeholders to review in-progress software builds without risking unauthorized exposure of client intellectual property, internal assets, or staging data.

Security is designed into the platform architecture from the ground up through a zero-trust, edge-authenticated model.

> [!NOTE]
> **CP-002 Security Status (Validated):**
> The external zero-trust boundary was empirically proven via a protected Cloudflare Quick Tunnel using email allowlisting and One-Time PIN (OTP) authentication.
>
> **Proven Security Properties:**
> - The origin is strictly restricted to host loopback (`127.0.0.1:8080`); no external or LAN ports are exposed.
> - External access flows exclusively through Cloudflare edge proxy via outbound encrypted tunnel.
> - Pre-exposure authentication is enforced: unauthenticated visitors are challenged with email OTP.
> - Only pre-approved emails can complete authentication; non-allowlisted emails are rejected with zero access to preview content.
> - Terminating the `cloudflared` process immediately removes external ingress (502 Bad Gateway at edge).
>
> **Known Operational Limitations:**
> - Quick Tunnels are intended for short testing/review windows; they provide no SLA or uptime guarantee.
> - Hostnames are ephemeral (`*.trycloudflare.com`) and change each time a tunnel is recreated.
> - Availability depends on the local `cloudflared` process remaining active. Permanent staging will eventually migrate to named tunnels on stable domains.

---

## 2. Core Security Principles

### 2.1 Deny-by-Default Access
All external access to staging endpoints is denied by default. No request is routed to any staging application container unless the user has been explicitly authenticated and authorized against an approved reviewer allowlist.

### 2.2 Pre-Exposure Edge Authentication
Authentication occurs at the Cloudflare Edge *before* any request traffic reaches the staging host. Unauthorized or unauthenticated visitors are blocked at the perimeter and never make a network connection to the backend preview container.

### 2.3 Strict HTTPS Everywhere
All external transit is strictly encrypted via HTTPS. Insecure HTTP connections are disallowed or automatically upgraded.

### 2.4 Least-Privilege Network Inbound Surface
In remote staging deployments, the host does not expose public inbound ports (e.g., port 80/443 open to the public internet). Instead, an outbound-only tunnel agent (`cloudflared`) connects to the Cloudflare Edge network, dramatically reducing host attack surface.

### 2.5 Defense-in-Depth vs. Access Control
- `noindex, nofollow` meta headers and `robots.txt` disallow directives are included in preview containers to discourage search engine crawling.
- However, search engine directives are **defense-in-depth only** and are never treated as access control mechanisms. True access control is enforced via identity validation.

### 2.6 No Security-Through-Obscurity
Random or unguessable URLs (e.g., secret GUID paths) are insufficient for protecting client assets. All preview links require identity-backed authorization regardless of URL obscurity.

### 2.7 No Anonymous Public Preview
Client staging previews will never be left accessible to anonymous visitors without an active, explicit decision and protective boundaries.

### 2.8 Prevention of Alternate Public Origins & Loopback Binding
Containers must never be bound to all interfaces (`0.0.0.0`) or exposed across local LAN interfaces without perimeter protection. In local development and remote hosts alike, container ports bind strictly to host loopback (`127.0.0.1`) or private Docker bridge networks, ensuring services are reachable only via local loopback or through the authorized `cloudflared` tunnel agent.

---

## 3. Identity & Authentication Architecture

Rather than building a bespoke identity stack (which introduces database management, credential storage risks, and password reset flows), authentication is delegated to **Cloudflare Access Zero Trust**:

- **Mechanism:** Email-based allowlist with One-Time Pin (OTP).
- **Workflow:**
  1. Reviewer navigates to preview URL.
  2. Cloudflare Access intercepts request and prompts for the user's business email.
  3. If the email matches the project's pre-approved reviewer list, Cloudflare sends a secure temporary cryptographic PIN to their inbox.
  4. Upon entering the correct PIN, Cloudflare issues an authenticated session token (via secure, HTTP-only cookie).
  5. Subsequent requests pass through the tunnel directly to the staging application.
- **Revocation:** Removing a user from the allowlist immediately terminates access across all active sessions.

---

## 4. Secrets Management & Repository Hygiene

1. **Zero Secrets in Git:** No passwords, personal access tokens, API secrets, cloud credentials, or private keys are ever committed to source control.
2. **Environment File Discipline:** `.env` and `.env.local` files are strictly excluded via `.gitignore`. Only `.env.example` containing non-sensitive template keys is tracked.
3. **Pre-Commit Verification:** Every checkpoint requires explicit inspection (`git status`, secret scanning) before committing changes to any branch.
