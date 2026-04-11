# Customer Air-Gap Prep Checklist (k3s + Zarf + Edge Packages)

Use this document to prepare all required binaries and packages **before** the production install window.

This is intended for a customer environment where:
- the production servers will not have Internet access
- `k3s` will be installed offline
- `zarf` will be installed offline
- one or more Zarf packages will be deployed offline

The goal is to avoid live commands such as:

```bash
curl -sfL https://get.k3s.io | sh -
curl -sL "https://github.com/zarf-dev/zarf/releases/..."
```

Instead, download everything in advance on a connected workstation, verify it, and transfer it into the air-gapped environment.

## 1. Decide and Pin Exact Versions

Before the customer downloads anything, pin the exact versions you validated in your lab.

Required:
- `K3S_VERSION`
- `ZARF_VERSION`
- the exact Zarf package versions you plan to deploy

Example placeholders:

```bash
export ARCH="amd64"
export K3S_VERSION="<validated-k3s-version>"
export ZARF_VERSION="<validated-zarf-version>"
```

Notes:
- do not use `latest`
- use the exact same versions you tested successfully
- keep the same versions across bundle creation and bundle deployment

## 2. What the Customer Must Download Ahead of Time

### 2.1 Required for offline k3s install

Download all of the following:

1. `k3s` binary
2. `k3s-airgap-images-<arch>.tar.zst`
3. `install.sh` from `https://get.k3s.io`
4. `sha256sum-<arch>.txt` for the pinned K3s release

### 2.2 Required for RHEL / Rocky / Alma hosts with SELinux enabled

If the target hosts are RHEL-family and SELinux is enabled, also pre-download:

1. `k3s-selinux` RPM matching the OS major version
2. any required OS RPM dependencies if the host cannot access internal repos:
   - `container-selinux`
   - `policycoreutils`
   - `selinux-policy`
   - distro-specific dependencies as needed

If the customer already has an internal RPM mirror available from the prod network, they may use that instead of pre-downloading these RPMs manually.

### 2.3 Required for offline Zarf install

Download:

1. `zarf` binary for Linux `amd64`
2. Zarf `checksums.txt`
3. matching `zarf-init-*.tar.zst` package if you will run `zarf init`

### 2.4 Required application packages

Download every Zarf package you plan to deploy, for example:

1. `zarf-package-cpsh-edge-amd64-<version>.tar.zst`
2. any CPSH or supporting package tarballs used in your deployment flow

## 3. Recommended Download Bundle Layout

Create a single portable bundle on an Internet-connected machine:

```text
airgap-bundle/
  k3s/
    install.sh
    k3s
    k3s-airgap-images-amd64.tar.zst
    sha256sum-amd64.txt
  rpms/
    k3s-selinux-....rpm
    container-selinux-....rpm
    policycoreutils-....rpm
    selinux-policy-....rpm
  zarf/
    zarf
    checksums.txt
    zarf-init-....tar.zst
  packages/
    zarf-package-cpsh-edge-amd64-....tar.zst
    ...
  SHA256SUMS
```

## 4. Internet-Connected Download Script

Run this on a connected Linux or macOS workstation.

Update the pinned versions first.

```bash
set -euo pipefail

export ARCH="amd64"
export K3S_VERSION="<validated-k3s-version>"
export ZARF_VERSION="<validated-zarf-version>"

mkdir -p airgap-bundle/{k3s,zarf,packages,rpms}

# GitHub release URLs for k3s require the '+' to be URL-encoded.
K3S_VERSION_URL="$(printf '%s' "$K3S_VERSION" | sed 's/+/%2B/g')"

# ---- K3s artifacts ----
curl -fL "https://github.com/k3s-io/k3s/releases/download/${K3S_VERSION_URL}/k3s" \
  -o "airgap-bundle/k3s/k3s"

curl -fL "https://github.com/k3s-io/k3s/releases/download/${K3S_VERSION_URL}/k3s-airgap-images-${ARCH}.tar.zst" \
  -o "airgap-bundle/k3s/k3s-airgap-images-${ARCH}.tar.zst"

curl -fL "https://github.com/k3s-io/k3s/releases/download/${K3S_VERSION_URL}/sha256sum-${ARCH}.txt" \
  -o "airgap-bundle/k3s/sha256sum-${ARCH}.txt"

curl -fL "https://get.k3s.io" \
  -o "airgap-bundle/k3s/install.sh"

chmod +x airgap-bundle/k3s/k3s airgap-bundle/k3s/install.sh

# ---- Optional RHEL-family SELinux RPM ----
# Replace the example URL/version with the exact RPM appropriate for the target OS.
# Example:
# curl -fL "https://github.com/k3s-io/k3s-selinux/releases/download/<rpm-version>/k3s-selinux-<rpm-version>.el9.noarch.rpm" \
#   -o "airgap-bundle/rpms/k3s-selinux-<rpm-version>.el9.noarch.rpm"

# ---- Zarf CLI ----
curl -fL "https://github.com/zarf-dev/zarf/releases/download/${ZARF_VERSION}/zarf_${ZARF_VERSION}_Linux_${ARCH}" \
  -o "airgap-bundle/zarf/zarf"

curl -fL "https://github.com/zarf-dev/zarf/releases/download/${ZARF_VERSION}/checksums.txt" \
  -o "airgap-bundle/zarf/checksums.txt"

chmod +x airgap-bundle/zarf/zarf

# ---- Download matching Zarf init package ----
(
  cd airgap-bundle/zarf
  ./zarf tools download-init
)

# ---- Copy the actual deployment packages into the bundle ----
# Example:
# cp /path/to/zarf-package-cpsh-edge-amd64-*.tar.zst airgap-bundle/packages/
# cp /path/to/other-required-packages/*.tar.zst airgap-bundle/packages/

# ---- Create a checksum manifest for the whole bundle ----
(
  cd airgap-bundle
  find . -type f -print0 | sort -z | xargs -0 sha256sum > SHA256SUMS
)

tar -C airgap-bundle -czf airgap-bundle.tar.gz .
```

## 5. Transfer Into the Air-Gapped Environment

The customer should copy `airgap-bundle.tar.gz` into the isolated environment using their approved transfer process.

Example destination:

```bash
mkdir -p /root/airgap
cp airgap-bundle.tar.gz /root/airgap/
cd /root/airgap
tar -xzf airgap-bundle.tar.gz
sha256sum -c SHA256SUMS
```

Expected:
- every file reports `OK`

## 6. Offline k3s Install on the Production Host

Run this on the production server as `root`.

### 6.1 Install local binaries and air-gap image tarball

```bash
install -m 0755 /root/airgap/k3s/k3s /usr/local/bin/k3s

mkdir -p /var/lib/rancher/k3s/agent/images
cp /root/airgap/k3s/k3s-airgap-images-amd64.tar.zst \
  /var/lib/rancher/k3s/agent/images/

chmod +x /root/airgap/k3s/install.sh
```

### 6.2 Install SELinux RPM if required

Only do this on RHEL-family systems where SELinux support is needed.

Example:

```bash
dnf install -y /root/airgap/rpms/k3s-selinux-*.rpm
```

If the node has no repo access at all, also install any OS dependency RPMs from the staged `rpms/` directory.

### 6.3 Install k3s without downloading anything

```bash
INSTALL_K3S_SKIP_DOWNLOAD=true \
INSTALL_K3S_SKIP_SELINUX_RPM=true \
/root/airgap/k3s/install.sh
```

Then configure `kubectl` access:

```bash
mkdir -p ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
chown "$USER:$USER" ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG="$HOME/.kube/config"
kubectl get nodes
```

Expected:
- the node shows as `Ready`

## 7. Offline Zarf Install

Install the CLI from the pre-downloaded binary:

```bash
install -m 0755 /root/airgap/zarf/zarf /usr/local/bin/zarf
zarf version
```

## 8. Offline Zarf Initialization

If the target cluster has not been initialized with Zarf yet:

```bash
zarf init /root/airgap/zarf/zarf-init-*.tar.zst --confirm
```

## 9. Offline Zarf Package Deployment

Deploy the staged application package:

```bash
zarf package deploy /root/airgap/packages/zarf-package-cpsh-edge-amd64-*.tar.zst --confirm
```

If your package requires variables, pass them explicitly:

```bash
zarf package deploy /root/airgap/packages/zarf-package-cpsh-edge-amd64-*.tar.zst \
  --confirm \
  --set-variables PLATFORM_ID="<platform-id>" \
  --set-variables PLATFORM_USER="<platform-user>" \
  --set-variables PLATFORM_PASSWORD="<platform-password>" \
  --set-variables SITE_NAME="<site-name>"
```

## 10. Customer Prep Questions to Resolve Before the Install Window

Confirm all of the following:

1. target OS distribution and version
2. target CPU architecture (`amd64` or `arm64`)
3. SELinux enabled or disabled
4. whether an internal RPM mirror is available
5. exact pinned `k3s` version
6. exact pinned `zarf` version
7. exact Zarf package filenames that will be deployed
8. approved file transfer path into the air-gapped zone
9. available free disk space for:
   - K3s images
   - Zarf cache/temp usage
   - application package extraction

## 11. What Not to Do in Production

Do not rely on:

```bash
curl -sfL https://get.k3s.io | sh -
curl -sL https://github.com/...
zarf init
```

without a local init package already staged.

Do not use:
- `latest`
- unpinned package versions
- package downloads directly from prod nodes

## 12. Validation Checklist

Before the install window starts, confirm:

1. `airgap-bundle.tar.gz` exists
2. checksums were generated on the connected machine
3. checksums validate on the target air-gapped host
4. `k3s` binary is present
5. `k3s-airgap-images-*.tar.zst` is present
6. `install.sh` is present
7. `zarf` binary is present
8. `zarf-init-*.tar.zst` is present
9. all deployment package tarballs are present
10. all needed RPMs are present for the target OS

## References

- [K3s Air-Gap Install](https://docs.k3s.io/installation/airgap)
- [K3s Installation](https://docs.k3s.io/installation)
- [K3s Advanced / SELinux Notes](https://docs.k3s.io/advanced)
- [Zarf Install](https://docs.zarf.dev/getting-started/install/)
- [Zarf Init Package](https://docs.zarf.dev/ref/init-package/)
- [Zarf Package Deploy](https://docs.zarf.dev/ref/deploy/)
- [Zarf Upgrade / Init Alignment](https://www.docs.zarf.dev/best-practices/upgrading-zarf/)
