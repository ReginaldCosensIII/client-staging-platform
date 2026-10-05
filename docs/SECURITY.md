# Security Policy & Architecture: Client Staging Platform

## 1. Security Overview

The primary objective of the Client Staging Platform is to enable client stakeholders to review in-progress software builds without risking unauthorized exposure of client intellectual property, internal assets, or staging data.

Security is designed into the platform architecture from the ground up through a zero-trust, edge-authenticated model backed by strict host-level and network-level isolation.

> [!NOTE]
> **CP-004 Security Status (Validated on Remote Host):**
> The external zero-trust boundary and host isolation model were empirically proven on the remote preview host (`client-staging-01`):
>
> **Proven Security Properties:**
> - The application origin is strictly restricted to host loopback (`127.0.0.1:8080`); no external network interfaces or public IP ports are bound.
> - The cloud firewall (`client-staging-ssh-only`) blocks all inbound traffic except TCP 22 (SSH). Public web ports (80, 443, 8080) are closed at the cloud perimeter.
> - Direct external HTTP requests to `http://<DROPLET_PUBLIC_IP>:8080` time out, confirming the origin cannot be reached directly.
> - External access flows exclusively through Cloudflare edge proxy via an outbound encrypted tunnel.
> - Pre-exposure authentication is enforced: unauthenticated visitors are challenged with an email One-Time PIN (OTP).
> - Only pre-approved emails can complete authentication; non-allowlisted emails are rejected with zero access to preview content (`HTTP request authorization failed before origin selection`).
> - Terminating the `cloudflared` process immediately removes external ingress (Cloudflare returns 530 at edge).
>
> **Known Operational Limitations:**
> - Quick Tunnels are intended for short testing/review windows; they provide no SLA or uptime guarantee.
> - Hostnames are ephemeral (`*.trycloudflare.com`) and change each time a tunnel is recreated.
> - Availability depends on the remote `cloudflared` process remaining active. Permanent staging will eventually migrate to named tunnels on stable domains.

---

## 2. Core Security Principles

### 2.1 Deny-by-Default Access
All external access to staging endpoints is denied by default. No request is routed to any staging application container unless the user has been explicitly authenticated and authorized against an approved reviewer allowlist.

### 2.2 Pre-Exposure Edge Authentication
Authentication occurs at the Cloudflare Edge *before* any request traffic reaches the staging host. Unauthorized or unauthenticated visitors are blocked at the perimeter and never make a network connection to the backend preview container.

### 2.3 Strict HTTPS Everywhere
All external transit is strictly encrypted via HTTPS. Insecure HTTP connections are disallowed or automatically upgraded at the edge.

### 2.4 Least-Privilege Network Inbound Surface & Cloud Firewall
The remote host does not expose public inbound web ports (ports 80, 443, 8080, 5000, 5001, etc., are closed). The DigitalOcean Cloud Firewall (`client-staging-ssh-only`) permits inbound traffic strictly on TCP 22 (SSH). All preview traffic arrives via an outbound-only encrypted tunnel connection (`cloudflared`) to the Cloudflare Edge network.

### 2.5 Remote Host Administrative Security & SSH Hardening
The remote preview host adheres to a hardened administrative posture:
- **Dedicated Non-Root User:** Routine operations and administration are performed by a non-root account: `stagingadmin`.
- **SSH Public-Key Authentication Only:** SSH password authentication is disabled (`PasswordAuthentication no`). Public-key cryptography (`PubkeyAuthentication yes`) is mandatory.
- **Root SSH Prohibited:** Direct root login via SSH is disabled (`PermitRootLogin no`). Tested post-hardening root SSH login attempts fail with `Permission denied (publickey)`.
- **Controlled Privilege Elevation:** Administrative actions requiring root privileges use `sudo`.
- **Docker Group Root-Equivalent Implications:** The `stagingadmin` account is a member of the `docker` group, allowing container management without `sudo`. **Security Note:** Membership in the `docker` group effectively grants root-equivalent control over the host filesystem and kernel. This is acceptable for the trusted administrative account, but must never be granted to low-privilege or untrusted accounts.

### 2.6 Defense-in-Depth vs. Access Control
- `noindex, nofollow` meta headers and `robots.txt` disallow directives are included in preview containers to discourage search engine indexing.
- However, search engine directives are **defense-in-depth only** and are never treated as access control mechanisms. True access control is enforced via identity validation.

### 2.7 No Security-Through-Obscurity
Random or unguessable URLs (e.g., secret GUID paths) are insufficient for protecting client assets. All preview links require identity-backed authorization regardless of URL obscurity.

### 2.8 No Anonymous Public Preview
Client staging previews will never be left accessible to anonymous visitors without an active, explicit decision and protective boundaries.

### 2.9 Prevention of Alternate Public Origins & Loopback Binding
Containers must never be bound to all interfaces (`0.0.0.0`) or exposed across local LAN interfaces without perimeter protection. In local development and remote hosts alike, container ports bind strictly to host loopback (`127.0.0.1`) or private Docker bridge networks, ensuring services are reachable only via local loopback or through the authorized `cloudflared` tunnel agent.

> [!IMPORTANT]
> **Core Architecture Requirement:**
> A protected Quick Tunnel must **never** be used as a substitute for origin isolation. The application origin must remain non-public independently of Cloudflare authentication. Even if the tunnel is active or misconfigured, the host's loopback-only binding and cloud firewall prevent direct internet exposure.

---

## 3. Identity & Authentication Architecture

Rather than building a bespoke identity stack (which introduces database management, credential storage risks, and password reset flows), authentication is delegated to **Cloudflare Access Zero Trust**:

- **Mechanism:** Email-based allowlist with One-Time PIN (OTP).
- **Workflow:**
  1. Reviewer navigates to preview URL (`https://*.trycloudflare.com`).
  2. Cloudflare Access intercepts request and prompts for the user's business email.
  3. If the email matches `--allowed-mail <APPROVED_REVIEWER_EMAIL>`, Cloudflare sends a secure temporary cryptographic 6-digit PIN to their inbox.
  4. Upon entering the correct PIN, Cloudflare issues an authenticated session token (via secure, HTTP-only cookie).
  5. Subsequent requests pass through the tunnel directly to `http://127.0.0.1:8080`.
- **Unauthorized Visitors:** Non-allowlisted email addresses (`<UNAPPROVED_EMAIL>`) receive an immediate `403 Forbidden` rejection at the edge (`broker assertion identity is not authorized`). `cloudflared` records `HTTP request authorization failed before origin selection`, and zero application bytes or assets are exposed.
- **Revocation:** Removing a user from the `--allowed-mail` parameter or restarting the tunnel immediately invalidates active reviewer access.

---

## 4. Secrets Management & Repository Hygiene

1. **Zero Secrets in Git:** No passwords, personal access tokens, API secrets, cloud credentials, or SSH private keys are ever committed to source control.
2. **Key Hygiene:** Workstation private SSH keys and administrator passwords must never be stored in repository files. Host public keys should not be unnecessarily identifying.
3. **Identifier Sanitization:** Actual reviewer emails and live temporary Quick Tunnel URLs must not be hard-coded into generic documentation. Use placeholders:
   - `<DROPLET_PUBLIC_IP>`
   - `<APPROVED_REVIEWER_EMAIL>`
   - `<UNAPPROVED_EMAIL>`
4. **Environment File Discipline:** `.env` and `.env.local` files are strictly excluded via `.gitignore`. Only `.env.example` containing non-sensitive template keys is tracked.
5. **Pre-Commit Verification:** Every checkpoint requires explicit inspection (`git diff --check`, `git status`, secret scanning) before committing changes to any branch.

---

## 5. MVP Scope vs. Future Production Security

| Security Dimension | Current MVP Baseline (CP-004) | Future Platform Evolution (CP-006+) |
| :--- | :--- | :--- |
| **Ingress Pattern** | Ephemeral Protected Quick Tunnel | Persistent Named Tunnel with custom domain |
| **Domain & Certs** | Dynamic `*.trycloudflare.com` / Cloudflare certs | Branded domain (`preview.example.com`) / Managed TLS |
| **Identity Source** | CLI `--allowed-mail` flag | Cloudflare Zero Trust IdP / Azure AD / Google Workspace |
| **Tunnel Process** | Manually run `cloudflared` process | Background `systemd` daemon (`cloudflared.service`) |
| **Origin Isolation** | Loopback binding (`127.0.0.1`) + Cloud Firewall | Docker internal network + loopback + firewall |
| **Firewall Ingress** | Inbound TCP 22 (SSH) only | Inbound TCP 22 (SSH) only (or Cloudflare Tunnel SSH) |
