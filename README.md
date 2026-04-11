# CPSH 2025.10 FIPS + Air-gapped Edge on Single-node k3s

This repository documents a lab setup for:
- CPSH 2025.10 with FIPS on one VM.
- Collibra Edge (Zarf package) on a separate single-node k3s VM.
- Multiple certificate paths for Edge deployment.
- Offline preparation for customer air-gapped environments.

## Important
- This is a lab/POC runbook.
- For production, use CA-signed certificates and supported target architecture.
- Certificate handling matters for Edge health, site registration, and connection flows.
- For all Edge-to-CPSH API calls, use the DGC HTTPS endpoint on `5443`. Do not use the CPSH Admin Console port `5404` for `PLATFORM_ID`.

## Architecture
- VM1: CPSH (DGC/API on `5443`, admin console on `5404`)
- VM2: Edge (k3s + Zarf + Edge workloads)

## Runbooks
1. CPSH install and FIPS setup: [`CPSH_INSTALL.md`](./CPSH_INSTALL.md)
2. CPSH install with customer-provided certificates (`BCFKS` for FIPS): [`CPSH_CUSTOM_CERT.md`](./CPSH_CUSTOM_CERT.md)
3. Edge install (Zarf + k3s + self-signed cert fixes): [`EDGE_INSTALL.md`](./EDGE_INSTALL.md)
4. Edge install with customer-provided certificates: [`EDGE_CUSTOM_CERT.md`](./EDGE_CUSTOM_CERT.md)
5. Customer offline prep for `k3s` + `zarf` + package staging: [`CUSTOMER_AIRGAP_PREP_K3S_ZARF.md`](./CUSTOMER_AIRGAP_PREP_K3S_ZARF.md)
6. AWS 5-VM CPSH 2026.03 runbook: [`CPSH_2026_03_5VM_AWS_RUNBOOK.md`](./CPSH_2026_03_5VM_AWS_RUNBOOK.md)

## Quick order of execution
1. For a customer air-gapped deployment, start with [`CUSTOMER_AIRGAP_PREP_K3S_ZARF.md`](./CUSTOMER_AIRGAP_PREP_K3S_ZARF.md) and stage every binary, RPM, init package, and Zarf package before the install window.
2. For a standard CPSH FIPS install, use [`CPSH_INSTALL.md`](./CPSH_INSTALL.md)
3. For CPSH with customer-provided certificates, use [`CPSH_CUSTOM_CERT.md`](./CPSH_CUSTOM_CERT.md)
4. For self-signed CPSH cert labs, use [`EDGE_INSTALL.md`](./EDGE_INSTALL.md)
5. For customer/private CA cert environments, use [`EDGE_CUSTOM_CERT.md`](./EDGE_CUSTOM_CERT.md)
6. For the AWS 5-VM CPSH 2026.03 pattern, use [`CPSH_2026_03_5VM_AWS_RUNBOOK.md`](./CPSH_2026_03_5VM_AWS_RUNBOOK.md)

## Customer Air-Gap Prep
- Use [`CUSTOMER_AIRGAP_PREP_K3S_ZARF.md`](./CUSTOMER_AIRGAP_PREP_K3S_ZARF.md) when the target servers will not have Internet access.
- That checklist covers:
  - offline `k3s` binaries and air-gap image tarballs
  - offline `zarf` CLI and matching `zarf-init` package
  - optional SELinux RPM staging for RHEL-family hosts
  - checksum verification and bundle transfer
  - offline deployment package staging
- Pin exact versions in advance. Do not use `latest` in production prep.

## Certificate Paths
- Use [`CPSH_CUSTOM_CERT.md`](./CPSH_CUSTOM_CERT.md) when DGC and Console should present customer-provided certificates instead of the default CPSH certificates. This runbook uses `BCFKS` keystores for the FIPS-enabled CPSH services.
- Use [`EDGE_INSTALL.md`](./EDGE_INSTALL.md) when CPSH is running with a self-signed certificate and you need the trust workaround flow.
- Use [`EDGE_CUSTOM_CERT.md`](./EDGE_CUSTOM_CERT.md) when CPSH is already configured with customer-provided certificates and the Edge host should trust the customer CA chain directly.

## References
- CPSH install and FIPS requirements: <https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_install-cpsh.htm>
- CPSH Extended Capabilities install: <https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_cpsh-install.htm>
- Zarf-based CPSH Edge install: <https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_cpsh-edge-extend-cap.htm>
- Delete Edge site: <https://productresources.collibra.com/docs/collibra/latest/Content/Edge/EdgeSiteManagement/ta_delete-edge-site.htm>
