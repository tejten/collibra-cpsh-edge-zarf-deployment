# CPSH 2025.10 FIPS + Air-gapped Edge on Single-node k3s

This repository documents a lab setup for:
- CPSH 2025.10 with FIPS on one VM.
- Collibra Edge (Zarf package) on a separate single-node k3s VM.
- Multiple certificate paths for Edge deployment.

## Important
- This is a lab/POC runbook.
- For production, use CA-signed certificates and supported target architecture.
- Certificate handling matters for Edge health, site registration, and connection flows.

## Architecture
- VM1: CPSH (DGC/API on `5443`, admin console on `5404`)
- VM2: Edge (k3s + Zarf + Edge workloads)

## Runbooks
1. CPSH install and FIPS setup: [`CPSH_INSTALL.md`](./CPSH_INSTALL.md)
2. CPSH install with customer-provided certificates: [`CPSH_CUSTOM_CERT.md`](./CPSH_CUSTOM_CERT.md)
3. Edge install (Zarf + k3s + self-signed cert fixes): [`EDGE_INSTALL.md`](./EDGE_INSTALL.md)
4. Edge install with customer-provided certificates: [`EDGE_CUSTOM_CERT.md`](./EDGE_CUSTOM_CERT.md)

## Quick order of execution
1. For a standard CPSH FIPS install, use [`CPSH_INSTALL.md`](./CPSH_INSTALL.md)
2. For CPSH with customer-provided certificates, use [`CPSH_CUSTOM_CERT.md`](./CPSH_CUSTOM_CERT.md)
3. For self-signed CPSH cert labs, use [`EDGE_INSTALL.md`](./EDGE_INSTALL.md)
4. For customer/private CA cert environments, use [`EDGE_CUSTOM_CERT.md`](./EDGE_CUSTOM_CERT.md)

## Certificate Paths
- Use [`CPSH_CUSTOM_CERT.md`](./CPSH_CUSTOM_CERT.md) when DGC and Console should present customer-provided certificates instead of the default CPSH certificates.
- Use [`EDGE_INSTALL.md`](./EDGE_INSTALL.md) when CPSH is running with a self-signed certificate and you need the trust workaround flow.
- Use [`EDGE_CUSTOM_CERT.md`](./EDGE_CUSTOM_CERT.md) when CPSH is already configured with customer-provided certificates and the Edge host should trust the customer CA chain directly.

## References
- CPSH install and FIPS requirements: <https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_install-cpsh.htm>
- CPSH Extended Capabilities install: <https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_cpsh-install.htm>
- Zarf-based CPSH Edge install: <https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_cpsh-edge-extend-cap.htm>
- Delete Edge site: <https://productresources.collibra.com/docs/collibra/latest/Content/Edge/EdgeSiteManagement/ta_delete-edge-site.htm>
