# CPSH 2025.10 FIPS Runbooks + Air-gapped Edge

This repository documents a lab setup for:
- CPSH 2025.10 with FIPS on one VM.
- Collibra Edge (Zarf package) on a separate single-node k3s VM.
- Multi-node CPSH installation patterns for production-style environments.
- Multiple certificate paths for Edge deployment.
- Offline preparation for customer air-gapped environments.
- CPSH and Edge upgrade workflows for moving from CPSH 2025.10 to CPSH 2026.03.

## Important
- This is a lab/POC runbook.
- For production, use CA-signed certificates and supported target architecture.
- Certificate handling matters for Edge health, site registration, and connection flows.
- For all Edge-to-CPSH API calls, use the DGC HTTPS endpoint on `5443`. Do not use the CPSH Admin Console port `5404` for `PLATFORM_ID`.

## Architecture
- Single-node lab pattern:
  - VM1: CPSH (DGC/API on `5443`, admin console on `5404`)
  - VM2: Edge (k3s + Zarf + Edge workloads)
- Multi-node production patterns:
  - separate Console
  - separate DGC + Search
  - repository primary
  - repository standby
  - optional Edge node

## Runbooks
1. CPSH install and FIPS setup: [`CPSH_INSTALL.md`](./CPSH_INSTALL.md)
2. CPSH install with customer-provided certificates (`BCFKS` for FIPS): [`CPSH_CUSTOM_CERT.md`](./CPSH_CUSTOM_CERT.md)
3. Edge install (Zarf + k3s + self-signed cert fixes): [`EDGE_INSTALL.md`](./EDGE_INSTALL.md)
4. Edge install with customer-provided certificates: [`EDGE_CUSTOM_CERT.md`](./EDGE_CUSTOM_CERT.md)
5. Customer offline prep for `k3s` + `zarf` + package staging: [`CUSTOMER_AIRGAP_PREP_K3S_ZARF.md`](./CUSTOMER_AIRGAP_PREP_K3S_ZARF.md)
6. Multi-node CPSH runbook with sizing, architecture, and AWS-specific appendices: [`CPSH_MULTI_NODE_INSTALL.md`](./CPSH_MULTI_NODE_INSTALL.md)
7. Multi-node CPSH runbook for customer managed environments with customer certificates and offline Edge flow: [`CPSH_MULTI_NODE_CUSTOMER_INSTALL.md`](./CPSH_MULTI_NODE_CUSTOMER_INSTALL.md)
8. CPSH 2025.10 to 2026.03 upgrade runbook, including PostgreSQL 14 to 17 transition and Edge Zarf upgrade: `./cpsh-2025.10-to-2026.03-upgrade-runbook.md`

## Quick order of execution
1. For a customer air-gapped deployment, start with [`CUSTOMER_AIRGAP_PREP_K3S_ZARF.md`](./CUSTOMER_AIRGAP_PREP_K3S_ZARF.md) and stage every binary, RPM, init package, and Zarf package before the install window.
2. For a standard single-node CPSH FIPS install, use [`CPSH_INSTALL.md`](./CPSH_INSTALL.md)
3. For single-node CPSH with customer-provided certificates, use [`CPSH_CUSTOM_CERT.md`](./CPSH_CUSTOM_CERT.md)
4. For a multi-node CPSH deployment with production sizing and architecture guidance, use [`CPSH_MULTI_NODE_INSTALL.md`](./CPSH_MULTI_NODE_INSTALL.md)
5. For a multi-node customer environment with internal DNS, customer certificates, and offline Edge flow, use [`CPSH_MULTI_NODE_CUSTOMER_INSTALL.md`](./CPSH_MULTI_NODE_CUSTOMER_INSTALL.md)
6. For CPSH 2025.10 to 2026.03 upgrades, including PostgreSQL 14 to 17 transition and Edge package upgrade, use `/Users/ttenmattam/Documents/f1511-cpsh-edge-zarf/cpsh-2025.10-to-2026.03-upgrade-runbook.md`
7. For self-signed CPSH cert labs, use [`EDGE_INSTALL.md`](./EDGE_INSTALL.md)
8. For customer/private CA cert environments, use [`EDGE_CUSTOM_CERT.md`](./EDGE_CUSTOM_CERT.md)

## Customer Air-Gap Prep
- Use [`CUSTOMER_AIRGAP_PREP_K3S_ZARF.md`](./CUSTOMER_AIRGAP_PREP_K3S_ZARF.md) when the target servers will not have Internet access.
- That checklist covers:
  - offline `k3s` binaries and air-gap image tarballs
  - offline `zarf` CLI and matching `zarf-init` package
  - optional SELinux RPM staging for RHEL-family hosts
  - checksum verification and bundle transfer
  - offline deployment package staging
- Pin exact versions in advance. Do not use `latest` in production prep.

## Multi-Node Runbooks
- Use [`CPSH_MULTI_NODE_INSTALL.md`](./CPSH_MULTI_NODE_INSTALL.md) when you want a production style multi-node CPSH runbook with sizing, architecture, port guidance, repository validation, and an AWS-focused appendix.
- Use [`CPSH_MULTI_NODE_CUSTOMER_INSTALL.md`](./CPSH_MULTI_NODE_CUSTOMER_INSTALL.md) when the customer environment uses internal DNS, customer issued certificates, and an offline Edge deployment path.
- Both multi-node runbooks use generic hostnames and placeholders so they can be adapted safely to a new environment.

## Upgrade Runbooks
- Use `/Users/ttenmattam/Documents/f1511-cpsh-edge-zarf/cpsh-2025.10-to-2026.03-upgrade-runbook.md` when upgrading CPSH from 2025.10 to 2026.03.
- That upgrade runbook covers the PostgreSQL 14 to PostgreSQL 17 prerequisite, CPSH node upgrade order, post-upgrade validation, and Edge upgrade through the CPSH Extended Capabilities Zarf package.
- Keep the Edge package version aligned with the Collibra compatibility matrix for CPSH 2026.03 rather than assuming the newest Edge package is supported.

## Certificate Paths
- Use [`CPSH_CUSTOM_CERT.md`](./CPSH_CUSTOM_CERT.md) when DGC and Console should present customer-provided certificates instead of the default CPSH certificates. This runbook uses `BCFKS` keystores for the FIPS-enabled CPSH services.
- Use [`EDGE_INSTALL.md`](./EDGE_INSTALL.md) when CPSH is running with a self-signed certificate and you need the trust workaround flow.
- Use [`EDGE_CUSTOM_CERT.md`](./EDGE_CUSTOM_CERT.md) when CPSH is already configured with customer provided certificates and the Edge host should trust the customer CA chain directly.

## References
- CPSH install and FIPS requirements: <https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_install-cpsh.htm>
- CPSH Extended Capabilities install: <https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_cpsh-install.htm>
- Zarf-based CPSH Edge install: <https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_cpsh-edge-extend-cap.htm>
- Delete Edge site: <https://productresources.collibra.com/docs/collibra/latest/Content/Edge/EdgeSiteManagement/ta_delete-edge-site.htm>
