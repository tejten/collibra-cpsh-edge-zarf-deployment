# Edge Install Runbook (Zarf on Single-node k3s, Self-signed CPSH Cert)

Run this on the Edge VM (`RHEL 9.x`) as `root`.

## 1. Variables

```bash
export CPSH_HOST="<cpsh-host-or-dns>"
export PLATFORM_ID="https://${CPSH_HOST}:5443"
export PLATFORM_USER="Admin"
export SITE_NAME="tej-edge-site-$(date +%Y%m%d-%H%M%S)"
read -s -p "PLATFORM_PASSWORD: " PLATFORM_PASSWORD; echo
```

## 2. Install tools + Zarf CLI

```bash

dnf install -y jq curl openssl

# Keep SELinux enforcing; add contexts used by k3s/local-path-provisioner
sudo mkdir -p /var/lib/rancher/k3s /opt/local-path-provisioner
sudo semanage fcontext -a -t container_file_t "/var/lib/rancher(/.*)?" || true
sudo semanage fcontext -a -t container_file_t "/opt/local-path-provisioner(/.*)?" || true
sudo restorecon -Rv /var/lib/rancher /opt/local-path-provisioner || true

# Lab networking prep
sudo systemctl disable --now nm-cloud-setup.service || true
sudo systemctl stop firewalld || true

# Kernel networking
echo br_netfilter | sudo tee /etc/modules-load.d/br_netfilter.conf
sudo modprobe br_netfilter
echo 'net.bridge.bridge-nf-call-iptables=1' | sudo tee /etc/sysctl.d/99-k3s.conf
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.d/99-k3s.conf
sudo sysctl --system

# ---- Install zarf ----

ZARF_VERSION="v0.73.0"

rm -f /tmp/zarf /usr/local/bin/zarf

curl -sL "https://github.com/zarf-dev/zarf/releases/download/${ZARF_VERSION}/zarf_${ZARF_VERSION}_Linux_amd64" \
  -o /tmp/zarf

ls -lh /tmp/zarf
file /tmp/zarf
chmod +x /tmp/zarf
install -m 0755 /tmp/zarf /usr/local/bin/zarf

/usr/local/bin/zarf version

# ---- Install k3s ----
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER:$USER" ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG="$HOME/.kube/config"
kubectl get nodes
```

## 3. Extract Edge package from Collibra bundle

```bash
PKG_VER="1.43.7"
EDGE_PKG="/tmp/zarf-package-cpsh-edge-amd64-${PKG_VER}.tar.zst"
echo "$EDGE_PKG"
test -f "$EDGE_PKG"
```

## 4. Initialize Zarf + k3s

```bash
zarf init
```

Recommended answers:
- Pull init package: `Yes`
- Deploy init package: `Yes`
- Deploy `k3s`: `Yes`
- Deploy `git-server`: `No`

Verify:

```bash
kubectl get nodes
```

## 5. Create required PriorityClasses

```bash
cat <<'EOF2' | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: platform
value: 1000000
globalDefault: false
description: "Collibra Platform Critical Components"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: application
value: 10000
globalDefault: false
description: "Collibra Application Workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: job
value: 100000
globalDefault: false
description: "Collibra Job Workloads"
EOF2

kubectl get priorityclass
```

## 6. Precreate namespace + trust secret for self-signed CPSH cert

```bash
kubectl create namespace collibra-edge --dry-run=client -o yaml | kubectl apply -f -

openssl s_client -showcerts -connect "${CPSH_HOST}:5443" -servername "${CPSH_HOST}" </dev/null 2>/dev/null \
| sed -n '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/p' > /tmp/cpsh-ca.pem

kubectl -n collibra-edge create secret generic edge-ca.pem \
  --from-file=ca.pem=/tmp/cpsh-ca.pem \
  --dry-run=client -o yaml | kubectl apply -f -
```

## 7. Temporary curl wrapper for package pre-check (self-signed lab workaround)

```bash
mkdir -p /tmp/zarf-lab-bin
cat > /tmp/zarf-lab-bin/curl <<'EOF2'
#!/usr/bin/env bash
exec /usr/bin/curl -k "$@"
EOF2
chmod +x /tmp/zarf-lab-bin/curl

export ORIG_PATH="$PATH"
export PATH="/tmp/zarf-lab-bin:$PATH"
hash -r
```

## 8. Deploy Edge package

```bash
zarf package deploy "$EDGE_PKG" \
  --confirm \
  --set-variables PLATFORM_ID="${PLATFORM_ID}" \
  --set-variables PLATFORM_USER="${PLATFORM_USER}" \
  --set-variables PLATFORM_PASSWORD="${PLATFORM_PASSWORD}" \
  --set-variables SITE_NAME="${SITE_NAME}" || true
```

`|| true` is intentional for this lab flow because package verify can fail before site fully converges.

Restore path after deploy:

```bash
export PATH="$ORIG_PATH"
hash -r
rm -rf /tmp/zarf-lab-bin
```

## 9. Critical post-deploy fix: reapply site credentials + CA toggle

### 9.1 Fetch `SITE_ID` and generated Edge credentials

```bash
SITE_ID="$(curl -sk -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
  "${PLATFORM_ID}/edge/api/rest/v2/sites" \
  | jq -r --arg n "$SITE_NAME" '.[] | select(.name==$n) | .id' | head -n1)"

echo "SITE_ID=${SITE_ID}"
test -n "$SITE_ID"

CREDS_JSON="$(curl -sk -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" -X POST \
  "${PLATFORM_ID}/edge/api/rest/v2/sites/${SITE_ID}/installerProperties")"

EDGE_USER="$(echo "$CREDS_JSON" | jq -r '.dicInfo.basic.username')"
EDGE_PASS="$(echo "$CREDS_JSON" | jq -r '.dicInfo.basic.password')"
```

### 9.2 Recreate `dgc-secret`

```bash
printf '%s' "$EDGE_PASS" > /tmp/edge-pass.txt

kubectl -n collibra-edge create secret generic dgc-secret \
  --from-literal=collibra.edge.platform.endpoint="${PLATFORM_ID}" \
  --from-literal=collibra.edge.platform.user="${EDGE_USER}" \
  --from-file=collibra.edge.platform.password=/tmp/edge-pass.txt \
  --dry-run=client -o yaml | kubectl apply -f -

rm -f /tmp/edge-pass.txt
```

### 9.3 Force custom CA on controller config and restart deployments

```bash
kubectl -n collibra-edge patch configmap collibra-edge-controller-config \
  --type merge \
  -p '{"data":{"collibra.edge.controller.proxy-custom-ca-enabled":"true"}}' || true

kubectl -n collibra-edge rollout restart deployment/edge-controller
kubectl -n collibra-edge rollout restart deployment/collibra-edge-cd
kubectl -n collibra-edge rollout restart deployment/edge-proxy
kubectl -n collibra-edge rollout restart deployment/collibra-edge-session-manager
kubectl -n collibra-edge rollout restart deployment/collibra-edge-objects-server

for d in edge-controller collibra-edge-cd edge-proxy collibra-edge-session-manager collibra-edge-objects-server; do
  kubectl -n collibra-edge rollout status deployment/$d --timeout=300s
done
```

## 10. Validate healthy state

```bash
kubectl get pods -n collibra-edge

kubectl -n collibra-edge logs deployment/edge-controller --since=10m | egrep -i "x509|unknown authority|error" || true
kubectl -n collibra-edge logs deployment/edge-proxy --since=10m | egrep -i "x509|unknown authority|error" || true
```

Check site status:

```bash
for i in {1..40}; do
  RESP="$(curl -sk -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
    "${PLATFORM_ID}/edge/api/rest/v2/sites/${SITE_ID}")"
  echo "$RESP" | jq '{name,status,version}'
  STATUS="$(echo "$RESP" | jq -r '.status')"
  [ "$STATUS" = "HEALTHY" ] && break
  sleep 30
done
```

## 11. Delete site and reset for reinstall

Delete site in CPSH:

```bash
curl -sk -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
  -X DELETE "${PLATFORM_ID}/edge/api/rest/v2/sites/${SITE_ID}" \
  -o /tmp/delete-site.json -w "HTTP %{http_code}\n"
```

Expected: `HTTP 204`.

Remove package from old Edge VM:

```bash
ZARF_VAR_KEEP_CPSH_EDGE_SITE=true \
ZARF_VAR_SKIP_BACKUP_EDGE_SITE=true \
zarf package remove cpsh-edge --confirm
```

Then provision a new Edge VM and repeat this runbook.

## 12. Common errors
- `x509: certificate signed by unknown authority`: missing CA secret or custom CA toggle.
- Site `PENDING`: bad credentials in `dgc-secret` or cert trust not applied.
- `This combination of host and port requires TLS`: apply nftables workaround on CPSH VM.
