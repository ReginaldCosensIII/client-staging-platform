# Remote Preview Host Baseline: DigitalOcean Reference Host

This document records the concrete technical baseline of the Remote Preview Host provisioned and validated during **CP-003** and **CP-004**.

The platform architecture defines a provider-neutral **Remote Preview Host** abstraction. The host specifications below document the current reference implementation on DigitalOcean.

---

## 1. Cloud Provider & Project Classification

| Parameter | Specification | Notes |
| :--- | :--- | :--- |
| **Cloud Provider** | DigitalOcean | Current reference infrastructure provider |
| **Project Name** | `client-staging-platform` | Dedicated platform project |
| **Environment Tag** | `Staging` | DigitalOcean metadata classification |
| **Project Purpose** | `Operational / Developer tooling` | DigitalOcean metadata classification |
| **Resource Tags** | `client-staging`, `staging` | Applied to Droplet and associated resources |

---

## 2. Compute Instance (Droplet) Specifications

| Parameter | Specification | Notes |
| :--- | :--- | :--- |
| **Hostname** | `client-staging-01` | Host identification |
| **Data Center / Region** | `NYC1` (New York 1) | Selected because the $6 Basic plan was unavailable in Richmond at provisioning time; not architecturally required |
| **Plan / Tier** | Basic / Shared CPU / Regular Intel or AMD | Cost-optimized development tier |
| **vCPU** | 1 vCPU | Sufficient for containerized preview workloads |
| **Physical Memory (RAM)** | 1 GB | Augmented with 1.0 GiB persistent swapfile |
| **Storage** | 25 GB SSD | Primary disk partition |
| **Network Transfer** | 1000 GB / month | Outbound transfer allowance |
| **Public IPv4** | Enabled (1 address) | Inbound filtered strictly by Cloud Firewall |
| **Public IPv6** | Disabled | Simplified network surface |
| **Monitoring** | Enabled | DigitalOcean Improved Metrics and Monitoring agent |
| **Automated Backups** | Disabled | Staging host is ephemeral and recreation-capable |
| **Additional Block Storage**| None | Workloads use local ephemeral container storage |
| **Managed Database** | None | Edge zero-trust authentication eliminates database need |
| **Startup Script (Cloud-Init)**| None | Provisioned and hardened interactively |

---

## 3. Cost & Billing Lifecycle

- **Pricing Baseline:** At CP-003 provisioning time, the selected Basic Droplet was advertised at a maximum of **$6/month** and approximately **$0.009/hour**.
- **Billing Mechanics:**
  - DigitalOcean bills continuously for provisioned Droplets regardless of power state.
  - Shutting down the OS (`sudo poweroff`) or powering off the Droplet **does not** stop billing, as compute, RAM, and IP allocations remain reserved.
  - **Destroying** the Droplet via the DigitalOcean control panel or API is required to cease billing.
  - Given the low cost (~$0.20/day), the host is kept active during ongoing development cycles to avoid daily reprovisioning friction.

---

## 4. Operating System & Kernel Baseline

- **Distribution:** Ubuntu 24.04.5 LTS x64 (Noble Numbat)
- **Kernel Version:** `6.8.0-146-generic`
- **Package Status:** 0 pending package updates following CP-003 baseline patch cycle.
- **LTS Channel Policy:** The operating system suggested upgrading to non-LTS releases during updates; the platform intentionally remains on the Ubuntu 24.04 LTS channel. Running `do-release-upgrade` is not recommended.

---

## 5. Memory & Swap Configuration

The 1 GB instance originally provisioned with zero swap space. A persistent swapfile was created and validated:

- **Path:** `/swapfile`
- **Size:** 1.0 GiB
- **Permissions:** `0600` (root-readable only)
- **Persistence:** Configured in `/etc/fstab`:
  ```text
  /swapfile none swap sw 0 0
  ```
- **Reboot Verification:** Successfully survived host reboot.
- **Architectural Purpose:** Provides a modest buffer against out-of-memory (OOM) process termination during container image builds or traffic spikes on the 1 GB VM. It does not replace physical RAM and does not imply suitability for unlimited memory-intensive workloads.

---

## 6. Administrative Security & SSH Posture

- **Administrative User:** `stagingadmin`
- **Group Memberships:** `sudo`, `docker`
- **Privilege Elevation:** Via `sudo` with administrative credentials.
- **Authentication Method:** SSH public-key authentication exclusively (`PubkeyAuthentication yes`).
- **Hardening Rules:**
  - Password authentication disabled: `PasswordAuthentication no`.
  - Root SSH login disabled: `PermitRootLogin no`.
  - Root SSH rejection verified: tested post-hardening SSH connection attempts as `root` fail with `Permission denied (publickey)`.
- **Docker Privilege Implications:** Membership in the `docker` group grants root-equivalent control over the host. This privilege is restricted to `stagingadmin` and must not be extended to low-privilege users.

---

## 7. Cloud Firewall Baseline

- **Firewall Name:** `client-staging-ssh-only`
- **Inbound Rules:**
  - **TCP 22 (SSH):** Allowed from any IPv4 address (access gated by SSH public-key cryptography).
  - **All Web / Application Ports:** 80, 443, 8080, 5000, 5001, etc., are **closed/blocked**.
- **Outbound Rules:**
  - All outbound traffic is permitted (enabling `cloudflared` outbound QUIC/HTTP2 connections to Cloudflare Edge and APT package repository updates).
- **Host Firewall (UFW):** Inactive / not configured during CP-003/CP-004; perimeter filtering is enforced at the cloud network layer via DigitalOcean Cloud Firewall.

---

## 8. Container Runtime (Docker) Baseline

Installed from Docker's official Ubuntu APT repository (`download.docker.com`):

- **Docker Engine:** `29.8.2`
- **Docker Compose Plugin:** `5.6.0` (`docker compose`)
- **containerd:** `2.3.6`
- **Docker Buildx:** Installed and active
- **Service Status:** `docker.service` enabled and active (running)
- **Execution Rights:** Non-root execution verified via `docker ps` and `docker run --rm hello-world` under `stagingadmin`.

---

## 9. Cloudflare Tunnel (`cloudflared`) Baseline

Installed from Cloudflare's official Debian/Ubuntu repository (`pkg.cloudflare.com`):

- **Version:** `cloudflared 2026.9.3` (built 2026-09-24T08:31 UTC)
- **Binary Path:** `/usr/bin/cloudflared`
- **Symlink:** `/usr/local/bin/cloudflared -> /usr/bin/cloudflared`
- **Feature Verification:** `--allowed-mail` flag confirmed supported for email-restricted protected Quick Tunnels.
- **Service Mode:** No background `systemd` service (`cloudflared.service`) is installed. The MVP uses foreground, manually started processes for on-demand review sessions.
- **Observed Non-Blocking Warnings:**
  - *ICMP Proxy:* Warning indicating user GID outside ping group range. ICMP routing is unused; HTTP/HTTPS tunneling is unaffected.
  - *QUIC Buffer:* Warning indicating UDP receive buffer size limit. QUIC transport negotiates cleanly and operates reliably.

---

## 10. Origin Isolation & Network Verification (CP-004 Proven)

- **Workload Launch:** Launched directly via `docker run -d --name preview-test --restart no -p 127.0.0.1:8080:80 nginx:alpine`.
- **Loopback Origin Publication:** Container ports bind strictly to `127.0.0.1:8080:80`.
- **Local Listener:** Socket verified on `127.0.0.1:8080` (no listener on `0.0.0.0` or public interface).
- **Direct-Origin Negative Test:** Direct external HTTP requests to `http://<DROPLET_PUBLIC_IP>:8080` time out.
- **Tunnel Ingress Model:**
  ```text
  Reviewer -> Cloudflare Edge (Email OTP) -> Outbound QUIC Tunnel -> cloudflared -> http://127.0.0.1:8080 -> nginx container
  ```
- **Authorized Reviewer Proof:** Verified that allowlisted email submission and OTP entry successfully routed through the tunnel to the protected nginx origin page (`Welcome to nginx!`).
- **Teardown Independence:** Stopping `cloudflared` drops external access immediately (returns Cloudflare 530) while the origin container remains running and healthy on `127.0.0.1:8080`.

---

## 11. Provider Portability

While this host is provisioned on DigitalOcean, the entire software stack (Ubuntu, Docker, Compose, `cloudflared`, loopback binding) is strictly standard and portable. The exact same configuration can be deployed onto:
- **Google Cloud Compute Engine** (e2-micro / e2-small VM);
- **Other VPS Providers** (Linode/Akamai, Hetzner, Vultr);
- **Local Workstations** (Windows/WSL2, macOS, Linux).
