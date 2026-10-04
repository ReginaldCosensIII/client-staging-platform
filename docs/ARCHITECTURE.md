# Platform Architecture: Client Staging Platform

## 1. Architectural Principles

The Client Staging Platform adheres to ten governing architectural principles:

1. **Framework Neutrality:** The platform packaging boundary is container-based (Docker). Any workload that can run inside an OCI-compliant container (ASP.NET Core, Node, static HTML, Python, etc.) can be hosted without requiring platform code modifications.
2. **Provider-Neutral Preview Host:** The Preview Host abstraction represents any Docker-capable Linux VM or compute instance. While Google Cloud Compute Engine is evaluated as the temporary remote MVP host, no proprietary GCP APIs or vendor locks are introduced.
3. **Containerized Client Workloads:** All previewed applications run as isolated containers behind controlled internal port bindings or private Docker networks.
4. **Application / Infrastructure Separation:** Infrastructure management, tunneling, and authentication exist independently of client application source repositories.
5. **Authentication Precedes Exposure:** No client application bytes or assets are transmitted to any requestor before identity verification succeeds at the edge.
6. **No Database Without Justified Need:** Staging portals typically do not require dedicated transactional state storage for access control. A zero-trust edge layer eliminates the operational overhead and attack surface of a custom database.
7. **Zero-Trust Over Custom Identity:** Avoid implementing bespoke user tables, password hashes, and session cookies. Cloudflare Access handles email allowlisting, OTP challenge generation, and session lifecycle.
8. **Configurable Branding:** Visual presentation (brand names, logos, colors, review instructions) is driven by configuration, allowing seamless transitions between provisional client branding (CES) and platform branding (Version III).
9. **Replaceable Remote Hosting:** Host migration (e.g., from local workstation to Google Cloud, and later to a low-cost dedicated VPS) requires only transferring the container definitions and starting the tunnel.
10. **Client Tenant Separation:** USAP is the inaugural staging workload, not the architecture itself. The platform remains a general-purpose staging utility.

---

## 2. Architecture Evolution Progression

### 2.1 CP-001: Local Development & Container Baseline (Current)

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

### 2.2 CP-002: Protected Local Cloudflare Quick Tunnel Proof

Validating the external zero-trust boundary without deploying cloud infrastructure. A local `cloudflared` process creates an outbound tunnel to Cloudflare Edge.

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
               | Outbound Secure Tunnel
               v
+----------------------------+
| Local Workstation / Host   |
|   +----------------------+ |
|   | cloudflared daemon   | |
|   +----------------------+ |
|              | HTTP:8080   |
|              v             |
|   +----------------------+ |
|   | preview-test (80)    | |
|   +----------------------+ |
+----------------------------+
```

---

### 2.3 Later Remote MVP: Temporary Cloud Host

Moving the validated container workload and `cloudflared` agent to a lightweight remote VM (e.g., Google Cloud Compute Engine e2-micro/small).

```text
+----------------------------+
| External Client Stakeholder|
+----------------------------+
               |
               | HTTPS (Access Policy Protected)
               v
+----------------------------+
| Cloudflare Zero Trust Edge |
+----------------------------+
               |
               | Secure Encrypted Tunnel (Outbound Only)
               v
+------------------------------------------+
| Remote Preview Host (Google Cloud VM)    |
|   +------------------------------------+ |
|   | cloudflared agent                  | |
|   +------------------------------------+ |
|                     | Internal Network   |
|                     v                    |
|   +------------------------------------+ |
|   | Client Application Container(s)    | |
|   | (e.g. USAP Website Preview)        | |
|   +------------------------------------+ |
+------------------------------------------+
```

*Note: In this topology, the remote VM requires NO inbound public firewall ports open. All traffic flows through the outbound tunnel established by `cloudflared`.*

---

### 2.4 Later Stable Platform: Multi-Tenant Named Tunnels & Custom Domain

The long-term Version III architecture hosting multiple containerized staging previews behind a custom branded domain and named persistent tunnels.

```text
+-----------------------------------+
| External Reviewers & Stakeholders |
+-----------------------------------+
                  |
                  | HTTPS: preview.example.com
                  v
+-----------------------------------+
| Cloudflare Access                 |
|  - Tenant-based Access Policies   |
|  - Custom Identity Providers / OTP|
+-----------------------------------+
                  |
                  | Persistent Named Tunnel
                  v
+-----------------------------------------------------+
| Dedicated Staging Host (VPS or Managed Compute)     |
|   +-----------------------------------------------+ |
|   | cloudflared daemon                            | |
|   +-----------------------------------------------+ |
|          |                  |                  |    |
|          v                  v                  v    |
|   +-------------+    +-------------+    +---------+ |
|   | Client A    |    | Client B    |    | Version | |
|   | (USAP Site) |    | (Client App)|    | III Hub | |
|   +-------------+    +-------------+    +---------+ |
+-----------------------------------------------------+
```

---

## 3. Component Details & Network Isolation

- **Client Container Isolation:** Client preview containers run on an internal Docker bridge network (`staging-network`).
- **No Direct Inbound Exposure:** Host ports are never bound to `0.0.0.0` or open LAN interfaces. The local test workload binds strictly to host loopback (`127.0.0.1`), and in CP-002 `cloudflared` routes directly to this local loopback origin.
- **Portability:** Moving from one provider to another is accomplished solely by starting Docker and the tunnel configuration on the target host.
