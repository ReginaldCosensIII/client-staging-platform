# Cloudflare Infrastructure: Protected Quick Tunnel Operations

## 1. Overview & Current Status (CP-004 Validated on Remote Host)

The **Protected Cloudflare Quick Tunnel** provides an on-demand, zero-trust external ingress boundary for previewing containerized workloads without requiring public cloud provisioning, custom domain DNS configuration, or open inbound firewall ports.

This mechanism was empirically proven and validated on the local workstation in **CP-002** and subsequently on the remote preview host (`client-staging-01`) in **CP-004**.

---

## 2. Prerequisites & Tooling Requirements

1. **Private Origin Workload:**
   - The preview application container must be running inside Docker.
   - The HTTP port must be bound strictly to host loopback (`127.0.0.1:8080`).
   - The origin must not be published across LAN or public host interfaces (`0.0.0.0`).
2. **`cloudflared` CLI Utility:**
   - Installed on both local development workstations and the Remote Preview Host.
   - **Remote Host Version:** Validated with `cloudflared 2026.9.3` (built 2026-09-24T08:31 UTC) installed via Cloudflare's official Debian/Ubuntu APT repository (`/usr/bin/cloudflared`, symlinked to `/usr/local/bin/cloudflared`).
   - **Version Requirement:** Use a current `cloudflared` release that supports the `--allowed-mail` option.
   - Must support the `--allowed-mail` flag (`cloudflared tunnel --help`).
   - No Cloudflare account, login (`cloudflared tunnel login`), `cert.pem`, or API tokens are required for Quick Tunnels.

---

## 3. Protected Quick Tunnel Operations

### 3.1 Starting a Protected Quick Tunnel

To initiate an outbound tunnel pointing to the local loopback origin with email access restrictions:

```bash
cloudflared tunnel --url http://127.0.0.1:8080 --allowed-mail <APPROVED_REVIEWER_EMAIL>
```

To allow multiple reviewers, repeat the flag or provide comma-separated addresses:

```bash
cloudflared tunnel --url http://127.0.0.1:8080 --allowed-mail reviewer1@example.com,reviewer2@example.com
```

### 3.2 Connectivity Pre-Checks & Expected Startup Output

Upon execution on the remote host, `cloudflared` automatically executes connectivity pre-checks:
- DNS resolution of Cloudflare edge endpoints;
- QUIC / UDP connectivity (using UDP port 7844);
- HTTP/2 / TCP fallback connectivity;
- Cloudflare edge API reachability.

Once pre-checks succeed, `cloudflared` selects QUIC as its primary transport and outputs the operational confirmation banner:

```text
+------------------------------------------------------------------------------------------------------+
|  Your protected quick Tunnel has been created! Visit it at (it may take some time to be reachable):  |
|  https://<random-words>.trycloudflare.com                                                            |
|  Authentication: One-Time PIN (using Cloudflare Access)                                              |
|  Allowed recipients: 1 address                                                                       |
|  Local origin: http://127.0.0.1:8080                                                                 |
+------------------------------------------------------------------------------------------------------+
```

### 3.3 Access & Authentication Workflow (Email OTP)

1. **Request Interception:** An external reviewer navigates to `https://<random-words>.trycloudflare.com`. Cloudflare Access intercepts the connection at the edge before any traffic reaches `cloudflared` or the remote host.
2. **Identity Challenge:** Cloudflare displays an Access login screen prompting the reviewer for their email address.
3. **Authorized Visitor Flow:**
   - Reviewer enters an allowlisted email (`<APPROVED_REVIEWER_EMAIL>`).
   - Cloudflare generates and emails a Cloudflare Access one-time PIN (OTP).
   - Upon submitting the correct PIN, Cloudflare sets an authenticated session cookie and proxies requests over the secure tunnel directly to the origin, successfully reaching the protected nginx origin page (`http://127.0.0.1:8080`).
4. **Unauthorized Visitor Flow:**
   - Visitor enters a non-allowlisted email (`<UNAPPROVED_EMAIL>`).
   - Cloudflare Access rejects the attempt immediately at the edge with an HTTP `403 Forbidden` (`broker assertion identity is not authorized`).
   - `cloudflared` logs `HTTP request authorization failed before origin selection` with an unauthorized identity condition.
   - Zero application bytes or origin traffic reach the visitor.

### 3.4 Stopping the Tunnel

To terminate external access, simply stop the foreground `cloudflared` process (`Ctrl+C`):
- The external `*.trycloudflare.com` URL immediately stops routing (returns Cloudflare `HTTP 530`).
- The local Docker workload remains completely unaffected and running locally on `http://127.0.0.1:8080`.

---

## 4. Key Properties, Observed Warnings & Operational Limitations

### 4.1 Proven Security Properties
- **Zero Inbound Attack Surface:** The host cloud firewall permits only SSH (TCP 22). No inbound web ports (80, 443, 8080) are open.
- **Strict Edge Filtering:** Only authenticated visitors with verified emails can access preview containers.
- **Local Isolation:** Docker remains bound strictly to `127.0.0.1:8080`. Direct requests to `http://<DROPLET_PUBLIC_IP>:8080` time out.

### 4.2 Observed Non-Blocking Warnings
During remote execution, `cloudflared` logged two non-blocking warnings:
1. `ICMP proxy disabled`: The user GID was outside the configured ping group range. Because ICMP routing is unnecessary for HTTP/HTTPS tunnels, this warning is benign.
2. `QUIC receive buffer size`: Warning that the kernel UDP receive buffer could not be increased to the ideal requested size.
Neither warning hindered tunnel registration, QUIC connectivity, OTP generation, or preview serving.

### 4.3 Operational Limitations
- **Ephemeral Hostnames:** Every time `cloudflared tunnel` is restarted or recreated, a new random `*.trycloudflare.com` hostname is generated. URLs cannot be reused across disconnected sessions.
- **Session Lifespan:** Access terminates the instant the `cloudflared` process stops. No background systemd service (`cloudflared.service`) is installed for the MVP.
- **No Service Level Agreement (SLA):** Quick Tunnels are intended for short, scheduled stakeholder review sessions rather than 24/7 continuous production staging.

---

## 5. Candidate Future Evolution: Named Tunnels & Custom Domains

A later production-style evolution may use a named Cloudflare Tunnel, stable custom hostname, and persistent background service management if justified by project needs.

Candidate future directions include:
- **Named Persistent Tunnels:** Transitioning from Quick Tunnels to pre-created, named Cloudflare Tunnels managed via Cloudflare Zero Trust.
- **Custom Branded Domain:** Routing traffic through a permanent custom hostname (e.g., `preview.example.com`).
- **Persistent Service:** Managing `cloudflared` via a systemd background daemon (`cloudflared.service`) with auto-restart on reboot.
- **Centralized Access Policies:** Centrally managed Access policies through Cloudflare Zero Trust dashboard with group-based access rules.
