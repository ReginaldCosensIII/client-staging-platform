# Deployment Strategy & Operations Runbook: Client Staging Platform

## 1. Multi-Tier Deployment Lifecycle

The deployment model distinguishes between four distinct operational tiers:

```text
Tier 1: Local Development Foundation (CP-001 / CP-001R1)
   ↓
Tier 2: Protected Local Quick Tunnel Proof (CP-002 / CP-002R1)
   ↓
Tier 3: Remote Preview Host & Infrastructure Proof (CP-003 / CP-004 — Current Proven Baseline)
   ↓
Tier 4: Stable Multi-Tenant Platform & Custom Domains (CP-006+ — Long-Term Future)
```

- **Tier 1 (Local Development):** Local workstation running containerized workload via Docker Compose bound strictly to IPv4 loopback (`127.0.0.1:8080`).
- **Tier 2 (Protected Local Tunnel):** Workstation loopback origin exposed externally via an ad-hoc protected Quick Tunnel (`cloudflared tunnel --allowed-mail`).
- **Tier 3 (Remote Preview Host):** Remote Linux compute instance (currently reference-implemented on DigitalOcean Droplet `client-staging-01`) hosting containerized preview workloads behind cloud firewall and loopback origin, exposed to reviewers via protected Quick Tunnel.
- **Tier 4 (Stable Multi-Tenant Platform):** Long-term architecture featuring persistent named Cloudflare Tunnels, custom branded domains (`preview.example.com`), and systemd-managed daemons.

---

## 2. Remote Preview Host Prerequisites

The current proven remote staging environment requires the following baseline:

- **Host Infrastructure:** Remote Preview Host (current reference implementation: DigitalOcean Droplet `client-staging-01`, NYC1 region, 1 vCPU, 1 GB RAM, 25 GB SSD).
- **Swapfile:** 1.0 GiB persistent swapfile (`/swapfile`, configured in `/etc/fstab`) providing memory resilience.
- **Operating System:** Ubuntu 24.04.5 LTS x64 (Kernel 6.8.0-146-generic). Standardized on the LTS release channel (no `do-release-upgrade`).
- **Administrative Account:** `stagingadmin` with `sudo` and `docker` group membership. SSH public-key authentication enforced; root SSH login disabled (`PermitRootLogin no`); password authentication disabled (`PasswordAuthentication no`).
- **Cloud Firewall:** Inbound traffic restricted strictly to TCP 22 (SSH). Inbound application ports (80, 443, 8080, etc.) are closed at the cloud perimeter. Outbound traffic is unrestricted.
- **Docker Engine:** Version 29.8.2 and Docker Compose plugin 5.6.0 installed from official Docker APT repositories. Docker service active and enabled.
- **cloudflared:** Version 2026.9.3 installed from Cloudflare's official package repository (`/usr/bin/cloudflared`, symlinked to `/usr/local/bin/cloudflared`).

---

## 3. Remote Staging Operations Runbook

This runbook documents the step-by-step procedure to execute an authorized client review session on the Remote Preview Host.

### Step 1: Connect to the Remote Host

Establish an administrative SSH session as `stagingadmin` using your configured SSH key:

```bash
ssh stagingadmin@<DROPLET_PUBLIC_IP>
```

*(Note: Direct root login is blocked by policy; connecting as `root` will fail with `Permission denied (publickey)`).*

---

### Step 2: Start the Private Origin Workload

Navigate to the platform workspace directory and start the preview container in detached mode:

```bash
cd /path/to/client-staging-platform
docker compose up -d
```

*(Note: The current test workload is `examples/preview-test` using `nginx:alpine` to prove infrastructure. In CP-005, this will be replaced with or accompanied by the packaged USAP client application).*

---

### Step 3: Validate Private Origin Isolation

Verify that the workload is running, published strictly to the host loopback interface, and listening on port 8080:

```bash
# 1. Verify container status and port mapping
docker compose ps
# Expected output shows: 127.0.0.1:8080->80/tcp

# 2. Inspect local TCP socket listeners
sudo ss -ltnp | grep 8080
# Expected output confirms listener on 127.0.0.1:8080 (NOT 0.0.0.0:8080)

# 3. Test local loopback HTTP response
curl -I http://127.0.0.1:8080
# Expected output: HTTP/1.1 200 OK
```

---

### Step 4: Perform Direct-Origin Negative Validation

From an **external workstation or device** (outside the Droplet), attempt to connect directly to the Droplet's public IP on port 8080:

```bash
# Run from external machine:
curl -I --connect-timeout 5 http://<DROPLET_PUBLIC_IP>:8080
```

- **Expected Result:** The connection **times out**.
- **Security Confirmation:** Confirms that the combination of DigitalOcean Cloud Firewall (`client-staging-ssh-only`) and Docker loopback-only binding prevents direct internet access to the application origin.

---

### Step 5: Start the Protected Quick Tunnel

Launch `cloudflared` to establish an outbound encrypted tunnel to Cloudflare Edge, specifying the loopback origin and approved reviewer email address(es):

```bash
cloudflared tunnel \
  --url http://127.0.0.1:8080 \
  --allowed-mail <APPROVED_REVIEWER_EMAIL>
```

To authorize multiple reviewers, provide a comma-separated list:

```bash
cloudflared tunnel \
  --url http://127.0.0.1:8080 \
  --allowed-mail reviewer1@example.com,reviewer2@example.com
```

#### Observed Startup Output:
- Cloudflare executes connectivity pre-checks (DNS, QUIC/UDP, HTTP/2/TCP, and Cloudflare API).
- Cloudflare selects QUIC as the primary transport protocol.
- Startup logs confirm:
  ```text
  Authentication: One-Time PIN (using Cloudflare Access)
  Allowed recipients: 1 address
  Local origin: http://127.0.0.1:8080
  Your quick Tunnel has been created! Visit it at:
  https://<random-words>.trycloudflare.com
  ```
- *(Note on Non-Blocking Warnings: Warnings regarding ICMP ping group permissions and QUIC buffer sizes may appear. These are non-blocking and do not impair tunnel connectivity or authentication).*

---

### Step 6: Reviewer Validation

#### A. Authorized Reviewer Flow:
1. Reviewer navigates to `https://<random-words>.trycloudflare.com`.
2. Cloudflare Access login screen displays, requesting an email address.
3. Reviewer enters `<APPROVED_REVIEWER_EMAIL>`.
4. Cloudflare sends a 6-digit cryptographic One-Time PIN (OTP) to the user's inbox.
5. Reviewer enters the OTP; Cloudflare sets an authenticated session cookie and routes requests through the tunnel.
6. The preview application renders successfully in the reviewer's browser.

#### B. Unauthorized Reviewer Flow:
1. Visitor navigates to `https://<random-words>.trycloudflare.com`.
2. Visitor enters an unauthorized email address (`<UNAPPROVED_EMAIL>`).
3. Cloudflare Access denies access at the edge with an HTTP `403 Forbidden` (`broker assertion identity is not authorized`).
4. `cloudflared` on the host records: `HTTP request authorization failed before origin selection`.
5. Zero application bytes or origin traffic are exposed to the unauthorized visitor.

---

### Step 7: Review Session Teardown

Once the review session is complete, terminate external access by stopping the foreground `cloudflared` process:

```text
Press Ctrl+C in the terminal running cloudflared
```

#### Post-Shutdown Ingress Validation:
- Immediately query the temporary URL from an external browser: Cloudflare returns an error / `HTTP 530`.
- Verify the local origin on the host:
  ```bash
  curl -I http://127.0.0.1:8080
  ```
  Returns `HTTP/1.1 200 OK`. The workload remains healthy, proving external access can be severed independently of container execution.

---

### Step 8: Container Teardown

When the preview workload is no longer needed:

```bash
docker compose down
```

Verify that no containers remain running:

```bash
docker compose ps
```

---

## 4. Operational Characteristics & Quick Tunnel Limitations

- **Ephemeral Hostnames:** Quick Tunnel URLs (`https://*.trycloudflare.com`) are generated dynamically upon each process launch. Terminating and restarting `cloudflared` generates a new random URL.
- **Session Lifespan:** External review is available only while the `cloudflared` command is actively running in a terminal or managed screen/tmux session. No persistent `cloudflared.service` is installed for the MVP.
- **No Service Level Agreement (SLA):** Quick Tunnels are designed for on-demand stakeholder testing and review windows. Permanent hosting will transition to named tunnels in CP-006+.

---

## 5. Host Cost, Billing & Lifecycle Management

- **Compute Tier:** DigitalOcean Basic Droplet (Regular Intel/AMD, 1 vCPU, 1 GB RAM, 25 GB SSD).
- **Advertised Pricing:** At CP-003 provisioning time, the selected Basic Droplet was advertised at a maximum of $6/month and approximately $0.009/hour.
- **Billing Lifecycle Rules:**
  - **DigitalOcean bills for provisioned Droplets regardless of power state.**
  - **Powering off or shutting down the operating system (`sudo poweroff`) DOES NOT stop Droplet billing.** Compute, RAM, and storage allocations remain reserved.
  - **Destroying the Droplet via the DigitalOcean control panel or API is required to cease compute billing.**
  - **Development Strategy:** Given the nominal cost (~$0.20/day), the host may remain provisioned during active staging development and testing cycles, avoiding the operational overhead of daily reprovisioning.

---

## 6. Provider Portability & Future Migrations

The platform architecture enforces strict provider neutrality under the **Remote Preview Host** abstraction:
- DigitalOcean Droplet `client-staging-01` is the current proven reference host.
- The platform relies exclusively on standard Linux facilities: standard Ubuntu packages, Docker Engine, loopback socket binding, and `cloudflared`.
- The workload can be seamlessly migrated to **Google Cloud Compute Engine** (e.g., when planned GCP credits/trials become available), another VPS provider, or an on-premises Linux server simply by:
  1. Provisioning a standard Ubuntu 24.04 instance.
  2. Applying SSH key hardening and firewall rules (inbound TCP 22 only).
  3. Installing Docker and `cloudflared`.
  4. Launching the container workload and protected tunnel.
- No vendor-proprietary APIs, managed databases, or cloud-specific ingress controllers are used.
