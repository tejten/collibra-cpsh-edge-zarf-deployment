# CPSH 2025.10 FIPS + Air-gapped Edge on Single-node k3s (Self-signed Lab)

This repository documents a lab setup for:
- CPSH 2025.10 with FIPS on one VM.
- Collibra Edge (Zarf package) on a separate single-node k3s VM.
- CPSH using a self-signed certificate.

## Important
- This is a lab/POC runbook.
- For production, use CA-signed certificates and supported target architecture.
- Self-signed certificate handling is intentionally included because it affects Edge health and connection flows.

## Architecture
- VM1: CPSH (DGC/API on `5443`, admin console on `5404`)
- VM2: Edge (k3s + Zarf + Edge workloads)

## Runbooks
1. CPSH install and FIPS setup: [`CPSH_INSTALL.md`](./CPSH_INSTALL.md)
2. Edge install (Zarf + k3s + self-signed cert fixes): [`EDGE_INSTALL.md`](./EDGE_INSTALL.md)

## Quick order of execution
1. Complete all steps in [`CPSH_INSTALL.md`](./CPSH_INSTALL.md)
2. Complete all steps in [`EDGE_INSTALL.md`](./EDGE_INSTALL.md)

## References
- CPSH install and FIPS requirements: <https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_install-cpsh.htm>
- CPSH Extended Capabilities install: <https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_cpsh-install.htm>
- Zarf-based CPSH Edge install: <https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_cpsh-edge-extend-cap.htm>
- Delete Edge site: <https://productresources.collibra.com/docs/collibra/latest/Content/Edge/EdgeSiteManagement/ta_delete-edge-site.htm>

## License

This project is licensed under the MIT License. 

## Credits

Original material by Tej Tenmattam (Version 01, February 25, 2026).