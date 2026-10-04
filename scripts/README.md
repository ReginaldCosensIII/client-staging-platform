# Automation & Deployment Scripts

## 1. Overview & Directory Policy

This directory is reserved for repeatable setup, maintenance, and deployment scripts when justified by operational need in future checkpoints.

In adherence to lean architectural discipline:
- **No Premature Automation:** No wrapper scripts (e.g. bash or PowerShell scripts merely wrapping standard `docker compose` commands) are added during CP-001. Standard CLI commands are documented directly in the root `README.md` and `infrastructure/docker/README.md`.
- **Future Candidate Scripts:**
  - Remote host initialization script (Docker & `cloudflared` bootstrap for Linux VMs in CP-003+).
  - Workload staging refresh helper scripts.
