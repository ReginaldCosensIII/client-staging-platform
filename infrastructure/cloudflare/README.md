# Cloudflare Infrastructure Specification

## 1. Scope & Status (CP-001 vs. CP-002)

> [!IMPORTANT]
> **Documentation Only for CP-001:**
> During CP-001, **no Cloudflare resources, tunnels, or accounts are created or executed**. This document serves exclusively as the forward-looking specification for **CP-002: Protected Local Cloudflare Quick Tunnel Proof**.

---

## 2. CP-002 Planned Scope & Evaluation

In CP-002, the implementation team will evaluate and test the edge zero-trust boundary against the local test workload:

1. **`cloudflared` CLI Utility:** Install and verify the `cloudflared` daemon on the local workstation.
2. **Ad-Hoc Quick Tunnel:** Create a secure outbound connection from the local workstation to Cloudflare edge pointing to `http://localhost:8080`.
3. **Ephemeral Public URL:** Route through a temporary `*.trycloudflare.com` domain without modifying production DNS records.
4. **Cloudflare Zero Trust Access Policy:**
   - Configure a Cloudflare Access application protecting the tunnel URL.
   - Enforce an approved reviewer email allowlist.
   - Challenge incoming visitors with an email One-Time Pin (OTP).
5. **External Verification:** Verify from an external device/browser (e.g. mobile phone on cellular network) that:
   - Unauthenticated visitors are blocked at the Cloudflare login challenge;
   - An authorized email address receives an OTP and can enter;
   - Once authenticated, the reviewer sees the containerized preview test page.

---

## 3. Explicit CP-001 Non-Actions

To maintain strict checkpoint boundaries, the following actions are explicitly prohibited and omitted in CP-001:
- Do NOT start or run `cloudflared` tunnels.
- Do NOT create or configure Cloudflare accounts or API tokens.
- Do NOT configure Cloudflare Zero Trust Access applications.
- Do NOT input or store client reviewer email addresses.
- Do NOT create DNS records, CNAMEs, or domain routings.
- Do NOT associate any custom domains or production certificates.
