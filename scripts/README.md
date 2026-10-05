# Automation & Deployment Scripts

## 1. Overview & Directory Policy

This directory is reserved for repeatable setup, maintenance, and deployment scripts when justified by operational need in future checkpoints.

In adherence to lean architectural discipline:
- **No Premature Automation:** Provisioning and hardening of the Remote Preview Host during CP-003 and CP-004 were intentionally executed manually as an infrastructure proof. No fragile wrapper scripts were introduced merely to wrap standard package managers or `docker compose` commands.
- **Future Candidate Scripts:**
  - Repeatable host bootstrap automation (e.g., cloud-init or shell provisioning for multi-host scaling in future phases).
  - Workload staging refresh helper scripts.
