# CPSH 2025.10 FIPS Install Runbook

Run this on the CPSH VM (`RHEL 9.x`) as `root`.

## 1. Prerequisites
- CPSH installer file available on the VM (`dgc-linux-2025.10.x.sh`).
- Inbound security group rules:
  - `22/tcp` from admin IP
  - `5404/tcp` from admin IP
  - `5443/tcp` from admin IP and Edge VM private IP/SG

## 2. Enable FIPS mode

```bash
fips-mode-setup --check || true
fips-mode-setup --enable
reboot
```

After reboot:

```bash
fips-mode-setup --check
cat /proc/sys/crypto/fips_enabled
```

Expected: enabled + `1`.

## 3. Install CPSH

```bash
chmod +x ./dgc-linux-2025.10.391.sh
./dgc-linux-2025.10.391.sh -- --fips
```

During install, keep a record of:
- CPSH admin credentials
- Repository credentials/passwords prompted by installer

## 4. Validate CPSH Edge Management API

```bash
export PLATFORM_ID="https://<cpsh-host>:5443"
export PLATFORM_USER="Admin"
read -s -p "PLATFORM_PASSWORD: " PLATFORM_PASSWORD; echo

curl -k -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
  "${PLATFORM_ID}/edge/api/rest/v2/internal/info"
```

Expected: HTTP 200 and JSON payload.

## 5. (Optional) TLS localhost port mapping workaround
Use this only if you later see connection-creation errors from CPSH logs like:
`This combination of host and port requires TLS`

```bash
dnf install -y nftables
systemctl enable --now nftables

nft add table ip nat 2>/dev/null || true
nft 'add chain ip nat output { type nat hook output priority 0; }' 2>/dev/null || true
nft list chain ip nat output | grep -q 'ip daddr 127.0.0.1 tcp dport 5443 redirect to 4400' || \
  nft add rule ip nat output ip daddr 127.0.0.1 tcp dport 5443 redirect to 4400
```

## 6. Verify UIs
- DGC/UI: `https://<cpsh-host>:5443`
- Admin console: `https://<cpsh-host>:5404`

For this lab these URLs may show browser warnings due to self-signed cert.
