# Edge Install Runbook (Customer-provided Certificates)

This runbook installs Air-gapped Edge on a dedicated Edge server (single-node k3s) against CPSH running with customer-provided certificates.

Important:
- Official Collibra Zarf docs for CPSH extended capabilities list EKS as the supported target for this method.
- The steps below follow the tested single-node k3s lab pattern.
- This runbook does not use `curl -k` or self-signed shortcuts.

## 1. Inputs

Set these values on the Edge server:

```bash
export EDGE_HOST="<edge-host-fqdn>"
export CPSH_HOST="<cpsh-host-fqdn>"
export PLATFORM_ID="https://${CPSH_HOST}:5443"
export PLATFORM_USER="Admin"
export SITE_NAME="<edge-site-name>"
read -s -p "PLATFORM_PASSWORD: " PLATFORM_PASSWORD; echo

# Collibra package versions
export PKG_VER="1.43.7"
export ZARF_VERSION="0.73.0"
```

Certificate files expected:
- Edge host private key: `<edge-host>.key`
- Edge host certificate or PFX bundle: `<edge-host>.crt` or `<edge-host>.pfx`
- CA chain bundle that signs the CPSH certificate: `<customer-ca-bundle>.pem`

## 2. Prepare and validate cert material on Edge server

```bash
mkdir -p /opt/collibra/certs
cp /path/to/<customer-ca-bundle>.pem /opt/collibra/certs/ca-bundle.pem
chmod 644 /opt/collibra/certs/ca-bundle.pem

# Optional: verify edge host key/cert pair if you plan to use it for ingress later
# openssl x509 -in /path/to/<edge-host>.crt -noout -modulus | openssl md5
# openssl rsa  -in /path/to/<edge-host>.key -noout -modulus | openssl md5
```

Validate CPSH TLS from the Edge host without `-k`:

```bash
curl --cacert /opt/collibra/certs/ca-bundle.pem \
  -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
  "${PLATFORM_ID}/edge/api/rest/v2/internal/info"
```

Expected: HTTP `200` and JSON response.

## 3. Install required tools and Zarf CLI

```bash
sudo dnf install -y jq curl tar zstd openssl

curl -L "https://github.com/zarf-dev/zarf/releases/download/${ZARF_VERSION}/zarf_${ZARF_VERSION}_Linux_amd64.tar.gz" \
  -o /tmp/zarf-${ZARF_VERSION}.tar.gz

mkdir -p /tmp/zarf-install
tar -xzf /tmp/zarf-${ZARF_VERSION}.tar.gz -C /tmp/zarf-install
sudo install -m 0755 "$(find /tmp/zarf-install -type f -name zarf | head -n1)" /usr/local/bin/zarf
zarf version
```

## 4. Extract Edge package and initialize k3s

```bash
tar -xzf "/tmp/zarf-packages-amd64-${PKG_VER}.tar.gz" -C /tmp
EDGE_PKG="$(find /tmp -type f -name "zarf-package-cpsh-edge-amd64-${PKG_VER}.tar.zst" | head -n1)"
echo "$EDGE_PKG"
test -f "$EDGE_PKG"

zarf init
```

Use these answers at `zarf init`:
- Pull init package: `Yes`
- Deploy init package: `Yes`
- Deploy `k3s`: `Yes`
- Deploy `git-server`: `No`

Verify:

```bash
kubectl get nodes
```

## 5. Apply required PriorityClasses

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: platform
value: 10000000
globalDefault: false
description: "Platform supporting services level"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: application
value: 100000
globalDefault: false
description: "Application level"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: job
value: 1000
globalDefault: false
description: "Job level"
EOF
```

## 6. Precreate namespace and CA secret for Edge pods

This is the critical certificate step for private CA chains.

```bash
kubectl create namespace collibra-edge --dry-run=client -o yaml | kubectl apply -f -

kubectl -n collibra-edge create secret generic edge-ca.pem \
  --from-file=ca.pem=/opt/collibra/certs/ca-bundle.pem \
  --dry-run=client -o yaml | kubectl apply -f -
```

## 7. Deploy the Edge Zarf package

```bash
zarf package deploy "$EDGE_PKG" \
  --confirm \
  --set-variables PLATFORM_ID="${PLATFORM_ID}" \
  --set-variables PLATFORM_USER="${PLATFORM_USER}" \
  --set-variables PLATFORM_PASSWORD="${PLATFORM_PASSWORD}" \
  --set-variables SITE_NAME="${SITE_NAME}"
```

## 8. Post-deploy stabilization

Run these even if deploy succeeds, to avoid the common `PENDING` state.

### 8.1 Fetch site ID and generated Edge user

```bash
SITE_ID="$(curl --cacert /opt/collibra/certs/ca-bundle.pem \
  -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
  "${PLATFORM_ID}/edge/api/rest/v2/sites" \
  | jq -r --arg n "$SITE_NAME" '.[] | select(.name==$n) | .id' | head -n1)"

echo "SITE_ID=${SITE_ID}"
test -n "$SITE_ID"

CREDS_JSON="$(curl --cacert /opt/collibra/certs/ca-bundle.pem \
  -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
  -X POST "${PLATFORM_ID}/edge/api/rest/v2/sites/${SITE_ID}/installerProperties")"

EDGE_USER="$(echo "$CREDS_JSON" | jq -r '.dicInfo.basic.username')"
EDGE_PASS="$(echo "$CREDS_JSON" | jq -r '.dicInfo.basic.password')"
```

### 8.2 Recreate `dgc-secret`

```bash
printf '%s' "$EDGE_PASS" > /tmp/edge-pass.txt

kubectl -n collibra-edge create secret generic dgc-secret \
  --from-literal=collibra.edge.platform.endpoint="${PLATFORM_ID}" \
  --from-literal=collibra.edge.platform.user="${EDGE_USER}" \
  --from-file=collibra.edge.platform.password=/tmp/edge-pass.txt \
  --dry-run=client -o yaml | kubectl apply -f -

rm -f /tmp/edge-pass.txt
```

### 8.3 Force custom CA toggle and restart

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

## 9. Validate

```bash
kubectl get pods -n collibra-edge

kubectl -n collibra-edge logs deployment/edge-controller --since=10m | egrep -i "x509|unknown authority|error" || true
kubectl -n collibra-edge logs deployment/edge-proxy --since=10m | egrep -i "x509|unknown authority|error" || true

curl --cacert /opt/collibra/certs/ca-bundle.pem \
  -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
  "${PLATFORM_ID}/edge/api/rest/v2/sites/${SITE_ID}" | jq '{name,status,version}'
```

Expected final status: `HEALTHY`.

## 10. Optional: use edge host cert as Kubernetes TLS secret

Only needed if you explicitly expose an HTTPS ingress on the Edge cluster.

```bash
kubectl -n collibra-edge create secret tls edge-host-tls \
  --cert=/path/to/<edge-host>.crt \
  --key=/path/to/<edge-host>.key
```

If you do not expose ingress, this is not required for the core Zarf Edge installation.

## 11. Fast troubleshooting

- If deploy ends in `PENDING`: rerun section 8 and recheck logs.
- If you see `x509: certificate signed by unknown authority`: confirm `edge-ca.pem` exists and section 8.3 was applied.
- If connection creation fails with `This combination of host and port requires TLS`: apply the CPSH-side nftables redirect workaround (`127.0.0.1:5443 -> 4400`) and retry.

## References

- Air-gapped Edge via Zarf (CPSH docs): https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_cpsh-edge-extend-cap.htm
- Edge CLI certificate truststore commands: https://productresources.collibra.com/docs/collibra/latest/Content/Edge/EdgeSiteManagement/co_edge-cli.htm
- Enable Edge for CPSH (tokens/config): https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_enable-edge-cpsh.htm
