# Cloudflare Infrastructure: Protected Quick Tunnel Operations

## 1. Overview & Current Status (CP-002 Validated)

The **Protected Cloudflare Quick Tunnel** provides an on-demand, zero-trust external ingress boundary for previewing containerized workloads without requiring public cloud provisioning, custom domain DNS configuration, or open inbound firewall ports.

This mechanism was empirically proven and validated during **CP-002**.

---

## 2. Prerequisites & Tooling Requirements

1. **Local Docker Workload:**
   - The preview application container must be running locally.
   - The HTTP port must be bound strictly to host loopback (`127.0.0.1:8080`).
   - The origin must not be published across LAN or public host interfaces (`0.0.0.0`).
2. **`cloudflared` CLI Utility:**
   - Must be installed on the host system (Windows x64 / Linux).
   - Version requirement: Use a current `cloudflared` release that supports the `--allowed-mail` option (CP-002 was validated with `cloudflared 2026.9.3`).
   - Must support the `--allowed-mail` flag (`cloudflared tunnel --help`).
   - No Cloudflare account, login (`cloudflared tunnel login`), `cert.pem`, or API tokens are required.

---

## 3. Protected Quick Tunnel Operations

### 3.1 Starting a Protected Quick Tunnel

To initiate an outbound tunnel pointing to the local loopback origin with email access restrictions:

```bash
# PowerShell / Bash
cloudflared tunnel --url http://127.0.0.1:8080 --allowed-mail reviewer@example.com
```

To allow multiple reviewers, repeat the flag or provide comma-separated addresses:

```bash
cloudflared tunnel --url http://127.0.0.1:8080 --allowed-mail reviewer1@example.com,reviewer2@example.com
```

### 3.2 Expected Startup Output

Upon execution, `cloudflared` negotiates an outbound QUIC/HTTP2 tunnel connection to the nearest Cloudflare Edge location and outputs a confirmation banner:

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

1. **Request Interception:** An external reviewer navigates to `https://<random-words>.trycloudflare.com`. Cloudflare Access intercepts the connection at the edge before any traffic reaches `cloudflared`.
2. **Identity Challenge:** Cloudflare displays an Access login screen prompting the reviewer for their email address.
3. **Allowlist Verification:**
   - If the entered address matches `--allowed-mail`, Cloudflare generates and emails a temporary cryptographic 6-digit One-Time PIN (OTP).
   - If the entered address is not allowlisted, access is rejected immediately at the edge (`broker assertion identity is not authorized`), and no application bytes are exposed.
4. **Session Token Grant:** Upon entering the valid PIN, Cloudflare sets an authenticated session cookie and proxies requests over the secure tunnel directly to `http://127.0.0.1:8080`.

### 3.4 Stopping the Tunnel

To terminate external access, simply stop the `cloudflared` process (`Ctrl+C` or task cancellation):
- The external `*.trycloudflare.com` URL immediately stops routing (returns `HTTP 502 Bad Gateway`).
- The local Docker workload remains completely unaffected and running locally on `http://127.0.0.1:8080`.

---

## 4. Key Properties & Operational Limitations

### 4.1 Proven Security Properties
- **Zero Inbound Attack Surface:** The host firewall does not permit inbound public web traffic. All communication is established over an outbound connection to Cloudflare.
- **Strict Edge Filtering:** Only authenticated visitors with verified emails can access preview containers.
- **Local Isolation:** Docker remains bound strictly to `127.0.0.1`.

### 4.2 Known Limitations
- **Ephemeral Hostnames:** Every time `cloudflared tunnel` is restarted or recreated, a new random `*.trycloudflare.com` hostname is generated. URLs cannot be reused across disconnected sessions.
- **Session Lifespan:** Access terminates the instant the local `cloudflared` process stops.
- **No Service Level Agreement (SLA):** Quick Tunnels are intended for short, scheduled stakeholder review sessions rather than 24/7 continuous staging hosting.

---

## 5. Long-Term Architecture Transition

For long-term and multi-tenant staging (Version III Portal):
- **Remote Host:** The workload and tunnel will run on a persistent remote Linux compute instance (e.g., Google Cloud Compute Engine or low-cost VPS) in CP-003+.
- **Named Persistent Tunnels:** Transition from Quick Tunnels to named Cloudflare Tunnels bound to a custom branded domain (e.g. `preview.clientdomain.com`).
- **Access Policies:** Centrally managed Access policies through Cloudflare Zero Trust dashboard with group-based access rules.
