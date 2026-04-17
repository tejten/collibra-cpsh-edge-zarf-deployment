# Collibra CPSH Multi-Node Installation Runbook for Customer Environments

This runbook describes a multi-node Collibra Platform Self-Hosted (CPSH) installation for a customer-managed environment that uses internal DNS, customer-issued certificates, and an optional offline Edge deployment.

It is based on a FIPS-enabled production flow and is written with generic server names, certificate names, passwords, and internal network placeholders.

## Scope

This runbook covers:

- preparation on all Linux hosts
- Console installation
- Repository primary installation
- Repository standby installation
- DGC + Search installation
- customer certificate import for Console HTTPS
- customer certificate import for DGC HTTPS
- environment creation in Console
- repository primary and standby validation
- optional offline Edge installation with a customer CA bundle

This runbook does not cover:

- operating system provisioning
- load balancer implementation
- enterprise monitoring stack configuration
- backup tooling and recovery procedures

## Recommended Node Layout

| Node | Role | Example Internal Hostname |
|---|---|---|
| Node 1 | Collibra Console | `console.prod.example.internal` |
| Node 2 | DGC + Search | `dgc-search.prod.example.internal` |
| Node 3 | Repository primary | `repo-primary.prod.example.internal` |
| Node 4 | Repository standby | `repo-standby.prod.example.internal` |
| Node 5 | Edge optional | `edge.prod.example.internal` |

## Core Ports

| Service | Port | Notes |
|---|---:|---|
| Console HTTP | 4402 | Default Console UI port |
| Console HTTPS | 5404 | Recommended production endpoint |
| Console database | 4420 | Local Console DB port |
| DGC HTTP | 4400 | Default DGC service port |
| DGC HTTPS | 5443 | Recommended production endpoint |
| Agent | 4401 | Used by installed services |
| Repository service | 4403 | Repository endpoint |
| Search REST | 4421 | Search API port |
| Search transport | 4422 | Search transport port |

## Fill In These Values Before You Start

Replace these placeholders throughout the runbook:

| Placeholder | Purpose |
|---|---|
| `<console-fqdn>` | Console hostname |
| `<dgc-search-fqdn>` | DGC + Search hostname |
| `<repo-primary-fqdn>` | Repository primary hostname |
| `<repo-standby-fqdn>` | Repository standby hostname |
| `<dgc-search-routable-ip-or-fqdn>` | Agent address used during DGC installer prompts |
| `<repository-admin-password>` | Repository admin password |
| `<repository-dgc-password>` | Repository DGC password |
| `<console-keystore-password>` | Console HTTPS keystore password |
| `<dgc-keystore-password>` | DGC HTTPS keystore password |
| `<platform-admin-password>` | DGC administrator password used for Edge onboarding |
| `<console-cert-file>` | Customer-issued Console server certificate |
| `<console-key-file>` | Private key for the Console certificate |
| `<console-chain-file>` | CA/intermediate chain for the Console certificate |
| `<dgc-cert-file>` | Customer-issued DGC server certificate |
| `<dgc-key-file>` | Private key for the DGC certificate |
| `<dgc-chain-file>` | CA/intermediate chain for the DGC certificate |
| `<customer-ca-bundle-file>` | CA bundle trusted by Edge and clients |

## Prerequisites

Before starting, confirm the following:

- every node has a stable internal hostname
- internal DNS resolves every CPSH host from every CPSH node
- NTP or chrony is healthy on all nodes
- the installer file `dgc-linux-2025.10.394.sh` is staged on every CPSH node
- PostgreSQL 14 packages are reachable from internal repositories or are pre-staged
- the customer certificate, private key, and CA chain are staged on the Console and DGC nodes
- the customer CA chain is staged for the optional Edge host
- required firewall ports are open between nodes

If you plan to deploy Edge in an air-gapped customer environment, stage the offline bundle ahead of time using the checklist in `CUSTOMER_AIRGAP_PREP_K3S_ZARF.md`.

## Firewall Checklist

Open these paths before installation:

| Source | Destination | Port | Purpose |
|---|---|---:|---|
| Admin workstation or jump host | Console | 4402 or 5404 | Console UI |
| User networks or approved reverse proxy | DGC | 4400 or 5443 | UI and APIs |
| DGC + Search | Repository primary | 4403 | Repository service |
| DGC + Search | Repository standby | 4403 | Repository service |
| Repository primary | Repository standby | 5432 or approved PostgreSQL replication path | DB replication |
| CPSH nodes as required | CPSH nodes as required | 4401 | Agent communication |
| Optional Edge | DGC | 5443 | Edge HTTPS |

## Installation Order

Follow this order:

1. Prepare all hosts
2. Install Console
3. Configure Console HTTPS with customer certificate
4. Install Repository primary
5. Install Repository standby
6. Install DGC + Search
7. Configure DGC HTTPS with customer certificate
8. Create the repository cluster and environment in Console
9. Validate repository primary and standby behavior
10. Install optional Edge only after core CPSH is healthy

## 1. Prepare All Hosts

### Enable FIPS Mode

Run on every CPSH host:

```bash
sudo fips-mode-setup --enable
sudo reboot
```

After reboot:

```bash
sudo fips-mode-setup --check
update-crypto-policies --show
```

Expected:

- FIPS mode is enabled
- the active crypto policy matches your security baseline

### Create the `collibra` User and Host Limits

Run on every CPSH host:

```bash
sudo useradd -m -s /bin/bash collibra

sudo tee /etc/security/limits.d/collibra.conf >/dev/null <<'EOF'
collibra soft nofile 65536
collibra hard nofile 65536
collibra soft nproc 4096
collibra hard nproc 4096
collibra soft fsize unlimited
collibra hard fsize unlimited
collibra soft as unlimited
collibra hard as unlimited
EOF

sudo tee /etc/sysctl.d/99-collibra.conf >/dev/null <<'EOF'
vm.max_map_count = 262144
EOF

sudo sysctl --system
```

### Hostname and DNS Validation

Run on every host:

```bash
hostnamectl set-hostname <fqdn>
hostnamectl
getent hosts <fqdn>
timedatectl status
chronyc tracking || true
```

## STIG RHEL 9.7 Service Registration Workaround

Use this section only if the installer does not register native services on a hardened or STIG'd RHEL 9.7 image.

Typical symptom:

- `sudo /opt/collibra/console/bin/console install` fails because `chkconfig` is not installed
- `sudo /opt/collibra/agent/bin/agent install` fails for the same reason

If the installer-created service registration works in your environment, skip this section.

### Create a Console `systemd` Unit

Run on the Console node:

```bash
cat >/etc/systemd/system/collibra-console.service <<'EOF'
[Unit]
Description=Collibra Console
Wants=network-online.target
After=network-online.target postgresql-14.service

[Service]
Type=simple
User=collibra
Group=collibra
WorkingDirectory=/opt/collibra/console
ExecStart=/opt/collibra/console/bin/console console
ExecStop=/opt/collibra/console/bin/console stop
Restart=on-failure
RestartSec=15
TimeoutStartSec=300
TimeoutStopSec=120
SuccessExitStatus=0 143
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
EOF
```

### Create an Agent `systemd` Unit

Run on each node that has an agent:

```bash
cat >/etc/systemd/system/collibra-agent.service <<'EOF'
[Unit]
Description=Collibra Agent
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=collibra
Group=collibra
WorkingDirectory=/opt/collibra/agent
ExecStart=/opt/collibra/agent/bin/agent console
ExecStop=/opt/collibra/agent/bin/agent stop
Restart=on-failure
RestartSec=15
TimeoutStartSec=300
TimeoutStopSec=120
SuccessExitStatus=0 143
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
EOF
```

### Reload and Enable Services

```bash
systemctl daemon-reload
systemctl enable --now collibra-console
systemctl enable --now collibra-agent
```

Use these enable commands as follows:

- run `systemctl enable --now collibra-console` on the Console node only
- run `systemctl enable --now collibra-agent` on each node that has an agent

### Validate Service Health

```bash
systemctl status collibra-console --no-pager
systemctl status collibra-agent --no-pager
journalctl -u collibra-console -b --no-pager | tail -100
journalctl -u collibra-agent -b --no-pager | tail -100
```

Notes:

- this workaround addresses service registration on hardened builds where `chkconfig` is unavailable
- if the wrapper scripts daemonize instead of staying in the foreground in your environment, revisit the `Type=` setting before the change window

## 2. Install Collibra Console

Node:

- `console.prod.example.internal`

### Install PostgreSQL 14 for Console

```bash
yum clean all && yum update -y
yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-$(rpm -E %{rhel})-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf repolist --all | grep -i pgdg
dnf config-manager --set-disabled pgdg18
dnf config-manager --set-disabled pgdg17
dnf config-manager --set-enabled pgdg14
dnf clean all
dnf -y update
yum -y install postgresql14 postgresql14-server postgresql14-contrib
```

Update the PostgreSQL tmpfiles entry:

```bash
vi /usr/lib/tmpfiles.d/postgresql-14.conf
```

Set:

```text
d /run/postgresql 2777 postgres postgres - -
```

Reboot if needed, then verify:

```bash
stat -c '%a %U %G %n' /run/postgresql
```

Initialize and start PostgreSQL:

```bash
echo 'LC_ALL="en_US.UTF-8"' >> /etc/locale.conf
/usr/pgsql-14/bin/postgresql-14-setup initdb
systemctl enable postgresql-14
systemctl start postgresql-14
```

Edit `/var/lib/pgsql/14/data/postgresql.conf` and set at minimum:

```text
max_connections = 500
shared_buffers = 256MB
```

Restart PostgreSQL:

```bash
systemctl restart postgresql-14.service
```

### Install Console Component

```bash
cd /tmp
chmod a+x dgc-linux-2025.10.394.sh
./dgc-linux-2025.10.394.sh -- --fips
```

Use these installer selections:

- Data Governance Center: `n`
- Repository: `n`
- Jobserver: `n`
- Search: `n`
- Management Console: `Y`
- PostgreSQL 14 path: default `/usr/pgsql-14`
- Management Console port: `4402`
- Management Console database port: `4420`

Validate:

```bash
/opt/collibra/console/bin/console status
```

## 3. Configure Console HTTPS with Customer Certificate

Use the customer-issued Console server certificate, key, and chain bundle instead of a self-signed certificate.

### Stage Certificate Material

Example variables:

```bash
CERT_DIR="/tmp/certs"
CERT_FILE="${CERT_DIR}/<console-cert-file>"
KEY_FILE="${CERT_DIR}/<console-key-file>"
CHAIN_FILE="${CERT_DIR}/<console-chain-file>"
ALIAS="collibra"
KS_PASS='<console-keystore-password>'
```

### Validate Certificate and Key

```bash
openssl x509 -in "${CERT_FILE}" -noout -subject -issuer -dates
openssl x509 -in "${CERT_FILE}" -noout -text | egrep "Signature Algorithm|Public-Key|DNS:"
openssl x509 -noout -modulus -in "${CERT_FILE}" | openssl md5
openssl rsa  -noout -modulus -in "${KEY_FILE}" | openssl md5
```

The modulus checks should match.

### Build a PKCS12 Bundle

```bash
openssl pkcs12 -export \
  -name "${ALIAS}" \
  -inkey "${KEY_FILE}" \
  -in "${CERT_FILE}" \
  -certfile "${CHAIN_FILE}" \
  -out "${CERT_DIR}/console-https-source-fips.p12" \
  -passout pass:"${KS_PASS}"
```

### Resolve FIPS Provider Paths

```bash
FIPS_PROVIDERPATH="$(printf "%s:" /opt/collibra/security/fips/libs/*.jar)"
FIPS_PROVIDERPATH="${FIPS_PROVIDERPATH%:}"
FIPS_JAVA_SECURITY="$(ls /opt/collibra/security/fips/configs/*.security | head -1)"

mkdir -p /opt/collibra_data/console/security
mkdir -p /opt/collibra_data/console/tmp
```

### Import into a Console BCFKS Keystore

```bash
/opt/collibra/jre/bin/keytool -importkeystore -noprompt \
  -srckeystore "${CERT_DIR}/console-https-source-fips.p12" \
  -srcstoretype PKCS12 \
  -srcstorepass "${KS_PASS}" \
  -srcalias "${ALIAS}" \
  -destkeystore /opt/collibra_data/console/security/console-https-fips.bcfks \
  -deststoretype BCFKS \
  -deststorepass "${KS_PASS}" \
  -destkeypass "${KS_PASS}" \
  -destalias "${ALIAS}" \
  -providerclass org.bouncycastle.jcajce.provider.BouncyCastleFipsProvider \
  -providername BCFIPS \
  -providerpath "${FIPS_PROVIDERPATH}" \
  -J-Djava.security.properties="${FIPS_JAVA_SECURITY}" \
  -J-Djava.io.tmpdir=/opt/collibra_data/console/tmp \
  -J-Dorg.bouncycastle.native.loader.install_dir=/opt/collibra_data/console/tmp
```

Set ownership and mode:

```bash
chown collibra:collibra /opt/collibra_data/console/security/console-https-fips.bcfks
chmod 640 /opt/collibra_data/console/security/console-https-fips.bcfks
```

Edit `/opt/collibra_data/console/config/server.json` so the `httpsConnector` block uses the imported keystore:

```json
"httpsConnector" : {
  "port" : 5404,
  "keyAlias" : "collibra",
  "keyPass" : "<console-keystore-password>",
  "keystorePass" : "<console-keystore-password>",
  "keystoreFile" : "/opt/collibra_data/console/security/console-https-fips.bcfks",
  "keystoreType" : "bcfks"
}
```

Restart Console and open the port if your host firewall is enabled:

```bash
systemctl restart collibra-console
firewall-cmd --permanent --add-port=5404/tcp
firewall-cmd --reload
/opt/collibra/console/bin/console restart
```

## 4. Install Repository Primary

Node:

- `repo-primary.prod.example.internal`

### Install PostgreSQL 14

```bash
yum clean all && yum update -y
yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-$(rpm -E %{rhel})-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf repolist --all | grep -i pgdg
dnf config-manager --set-disabled pgdg18
dnf config-manager --set-disabled pgdg17
dnf config-manager --set-enabled pgdg14
dnf clean all
dnf -y update
yum -y install postgresql14 postgresql14-server postgresql14-contrib
```

Update:

```bash
vi /usr/lib/tmpfiles.d/postgresql-14.conf
```

Set:

```text
d /run/postgresql 2777 postgres postgres - -
```

Initialize and start PostgreSQL:

```bash
echo 'LC_ALL="en_US.UTF-8"' >> /etc/locale.conf
/usr/pgsql-14/bin/postgresql-14-setup initdb
systemctl enable postgresql-14
systemctl start postgresql-14
```

Edit `/var/lib/pgsql/14/data/postgresql.conf`:

```text
max_connections = 500
shared_buffers = 256MB
```

Edit `/var/lib/pgsql/14/data/pg_hba.conf` and allow:

- the DGC + Search node
- the repository standby node
- local administrative access

Example:

```text
host    all             all             <dgc-search-private-cidr-or-ip>/32   scram-sha-256
host    all             all             <repo-standby-private-cidr-or-ip>/32 scram-sha-256
```

Restart PostgreSQL and validate:

```bash
systemctl restart postgresql-14.service
systemctl status postgresql-14.service --no-pager
ss -lntp | grep 5432
```

### Install Repository Component

```bash
cd /tmp
chmod a+x dgc-linux-2025.10.394.sh
./dgc-linux-2025.10.394.sh -- --fips
```

Use these installer selections:

- Data Governance Center: `n`
- Repository: `Y`
- Jobserver: `n`
- Search: `n`
- Management Console: `n`
- PostgreSQL 14 path: default `/usr/pgsql-14`
- Agent port: `4401`
- Repository port: `4403`
- Repository admin password: `<repository-admin-password>`
- Repository DGC password: `<repository-dgc-password>`
- Repository memory in MB: `4096`

Validate:

```bash
/opt/collibra/agent/bin/agent status
```

## 5. Install Repository Standby

Node:

- `repo-standby.prod.example.internal`

### Install PostgreSQL 14

Repeat the PostgreSQL installation from the primary node.

Edit `/var/lib/pgsql/14/data/postgresql.conf`:

```text
max_connections = 500
shared_buffers = 256MB
```

Edit `/var/lib/pgsql/14/data/pg_hba.conf` and allow:

- the DGC + Search node
- the repository primary node
- local administrative access

Example:

```text
host    all             all             <dgc-search-private-cidr-or-ip>/32   scram-sha-256
host    all             all             <repo-primary-private-cidr-or-ip>/32 scram-sha-256
```

Restart PostgreSQL and validate:

```bash
systemctl restart postgresql-14.service
systemctl status postgresql-14.service --no-pager
ss -lntp | grep 5432
```

### Install Repository Component

```bash
cd /tmp
chmod a+x dgc-linux-2025.10.394.sh
./dgc-linux-2025.10.394.sh -- --fips
```

Use these installer selections:

- Data Governance Center: `n`
- Repository: `Y`
- Jobserver: `n`
- Search: `n`
- Management Console: `n`
- PostgreSQL 14 path: default `/usr/pgsql-14`
- Agent port: `4401`
- Repository port: `4403`
- Repository admin password: `<repository-admin-password>`
- Repository DGC password: `<repository-dgc-password>`
- Repository memory in MB: `4096`

Validate:

```bash
/opt/collibra/agent/bin/agent status
```

## 6. Install DGC + Search

Node:

- `dgc-search.prod.example.internal`

Important:

- do not leave the Agent address at `localhost`
- use the node’s routable internal hostname or IP during the installer prompts

### Install DGC + Search Components

```bash
cd /tmp
chmod a+x dgc-linux-2025.10.394.sh
./dgc-linux-2025.10.394.sh -- --fips
```

Use these installer selections:

- Data Governance Center: `Y`
- Repository: `n`
- Jobserver: `n`
- Search: `Y`
- Management Console: `n`
- Agent port: `4401`
- Agent address: `<dgc-search-routable-ip-or-fqdn>`
- Search HTTP port: `4421`
- Search transport port: `4422`
- Search memory in MB: `4096`
- Data Governance Center port: `4400`
- Data Governance Center shutdown port: `4430`
- DGC minimum memory in MB: `4096`
- DGC maximum memory in MB: `8192`

Validate:

```bash
/opt/collibra/agent/bin/agent status
```

## 7. Configure DGC HTTPS with Customer Certificate

Use the customer-issued DGC certificate, key, and chain bundle.

### Stage Certificate Material

```bash
CERT_DIR="/tmp/certs"
CERT_FILE="${CERT_DIR}/<dgc-cert-file>"
KEY_FILE="${CERT_DIR}/<dgc-key-file>"
CHAIN_FILE="${CERT_DIR}/<dgc-chain-file>"
ALIAS="collibra"
KS_PASS='<dgc-keystore-password>'
```

### Validate Certificate and Key

```bash
openssl x509 -in "${CERT_FILE}" -noout -subject -issuer -dates
openssl x509 -in "${CERT_FILE}" -noout -text | egrep "Signature Algorithm|Public-Key|DNS:"
openssl x509 -noout -modulus -in "${CERT_FILE}" | openssl md5
openssl rsa  -noout -modulus -in "${KEY_FILE}" | openssl md5
```

### Build a PKCS12 Bundle

```bash
openssl pkcs12 -export \
  -name "${ALIAS}" \
  -inkey "${KEY_FILE}" \
  -in "${CERT_FILE}" \
  -certfile "${CHAIN_FILE}" \
  -out "${CERT_DIR}/dgc-https-source-fips.p12" \
  -passout pass:"${KS_PASS}"
```

### Resolve FIPS Provider Paths

```bash
FIPS_PROVIDERPATH="$(printf "%s:" /opt/collibra/security/fips/libs/*.jar)"
FIPS_PROVIDERPATH="${FIPS_PROVIDERPATH%:}"
FIPS_JAVA_SECURITY="$(ls /opt/collibra/security/fips/configs/*.security | head -1)"

mkdir -p /opt/collibra_data/dgc/security
mkdir -p /opt/collibra_data/dgc/tmp
```

### Import into a DGC BCFKS Keystore

```bash
/opt/collibra/jre/bin/keytool -importkeystore -noprompt \
  -srckeystore "${CERT_DIR}/dgc-https-source-fips.p12" \
  -srcstoretype PKCS12 \
  -srcstorepass "${KS_PASS}" \
  -srcalias "${ALIAS}" \
  -destkeystore /opt/collibra_data/dgc/security/dgc-https-fips.bcfks \
  -deststoretype BCFKS \
  -deststorepass "${KS_PASS}" \
  -destkeypass "${KS_PASS}" \
  -destalias "${ALIAS}" \
  -providerclass org.bouncycastle.jcajce.provider.BouncyCastleFipsProvider \
  -providername BCFIPS \
  -providerpath "${FIPS_PROVIDERPATH}" \
  -J-Djava.security.properties="${FIPS_JAVA_SECURITY}" \
  -J-Djava.io.tmpdir=/opt/collibra_data/dgc/tmp \
  -J-Dorg.bouncycastle.native.loader.install_dir=/opt/collibra_data/dgc/tmp
```

Set ownership and mode:

```bash
chown collibra:collibra /opt/collibra_data/dgc/security/dgc-https-fips.bcfks
chmod 640 /opt/collibra_data/dgc/security/dgc-https-fips.bcfks
```

Edit `/opt/collibra_data/dgc/config/server.json`:

```json
"httpsConnector" : {
  "port" : 5443,
  "keyAlias" : "collibra",
  "keyPass" : "<dgc-keystore-password>",
  "keystorePass" : "<dgc-keystore-password>",
  "keystoreFile" : "/opt/collibra_data/dgc/security/dgc-https-fips.bcfks",
  "keystoreType" : "bcfks"
}
```

Restart the DGC-side services and open the HTTPS port if host firewalld is enabled:

```bash
systemctl restart collibra-agent
firewall-cmd --permanent --add-port=5443/tcp
firewall-cmd --reload
/opt/collibra/agent/bin/agent status
```

Note:

- if your environment uses a separate systemd unit for DGC, restart that unit as well
- validate the actual service unit names in the target environment before the change window

## 8. Create the Environment in Console

In Collibra Console:

1. Add the nodes
2. Create a repository cluster
3. Add the Repository service from `repo-primary.prod.example.internal` as `Master`
4. Add the Repository service from `repo-standby.prod.example.internal` as `Slave`
5. Create a new environment
6. Add services from `dgc-search.prod.example.internal`
7. Add the repository cluster to the environment
8. Start the environment

Recommended names:

- Repository cluster: `repo-cluster-prod`
- Environment: `production`

## 9. Validate Repository Primary and Standby

### Confirm Database Access

```bash
sudo -u postgres /usr/pgsql-14/bin/psql \
  "host=127.0.0.1 port=4403 dbname=postgres user=collibra sslmode=require" \
  -W -c '\l'
```

List schemas:

```bash
sudo -u postgres /usr/pgsql-14/bin/psql \
  "host=127.0.0.1 port=4403 dbname=dgc user=collibra sslmode=require" \
  -W -c '\dn+'
```

List tables:

```bash
sudo -u postgres /usr/pgsql-14/bin/psql \
  "host=127.0.0.1 port=4403 dbname=dgc user=collibra sslmode=require" \
  -W -c '\dt *.*'
```

### Confirm Primary Role

```bash
sudo -u postgres /usr/pgsql-14/bin/psql \
  "host=<repo-primary-private-ip> port=4403 dbname=dgc user=dgc sslmode=require" \
  -W -c "select current_user, current_database(), inet_server_addr(), pg_is_in_recovery();"
```

Expected:

- `pg_is_in_recovery = f`

### Confirm Standby Role

```bash
sudo -u postgres /usr/pgsql-14/bin/psql \
  "host=<repo-standby-private-ip> port=4403 dbname=dgc user=dgc sslmode=require" \
  -W -c "select current_user, current_database(), inet_server_addr(), pg_is_in_recovery();"
```

Expected:

- `pg_is_in_recovery = t`

## 10. Optional Appendix: Offline Edge Installation

Use this section only after the core CPSH environment is healthy.

Before using this section, complete the offline preparation steps in `CUSTOMER_AIRGAP_PREP_K3S_ZARF.md`. That companion document covers how to build or validate the `airgap-bundle.tar.gz` package, how to pre-download the exact `k3s` and `zarf` artifacts, and what to stage before the production install window.

### Stage Required Files

Copy to the Edge host:

- `zarf-package-cpsh-edge-amd64-1.43.15.tar.zst`
- `airgap-bundle.tar.gz`
- the customer CA bundle used by DGC

If `airgap-bundle.tar.gz` is not already staged, stop here and follow `CUSTOMER_AIRGAP_PREP_K3S_ZARF.md` first.

### Set Core Variables

```bash
export DGC_HOST="<dgc-search-fqdn>"
export PLATFORM_ID="https://${DGC_HOST}:5443"
export PLATFORM_USER="Admin"
export PLATFORM_PASSWORD="<platform-admin-password>"
export SITE_NAME="edge-site-$(date +%Y%m%d-%H%M%S)"
export EDGE_HOST="<edge-fqdn>"
```

### Prepare CA Trust on the Edge Host

```bash
mkdir -p /opt/collibra/certs
cp /tmp/certs/<customer-ca-bundle-file> /opt/collibra/certs/ca-bundle.pem
chmod 644 /opt/collibra/certs/ca-bundle.pem

sudo cp /tmp/certs/<customer-ca-bundle-file> /etc/pki/ca-trust/source/anchors/customer-cpsh-ca.pem
sudo update-ca-trust extract
```

Validate DGC TLS without `-k`:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
  "${PLATFORM_ID}/edge/api/rest/v2/internal/info"

curl --cacert /opt/collibra/certs/ca-bundle.pem \
  -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
  "${PLATFORM_ID}/edge/api/rest/v2/internal/info"
```

Expected:

- HTTP `200`

### Install Base Packages

```bash
sudo dnf install -y jq curl openssl
```

Prepare SELinux contexts and kernel networking:

```bash
sudo mkdir -p /var/lib/rancher/k3s /opt/local-path-provisioner
sudo semanage fcontext -a -t container_file_t "/var/lib/rancher(/.*)?" || true
sudo semanage fcontext -a -t container_file_t "/opt/local-path-provisioner(/.*)?" || true
sudo restorecon -Rv /var/lib/rancher /opt/local-path-provisioner || true

echo br_netfilter | sudo tee /etc/modules-load.d/br_netfilter.conf
sudo modprobe br_netfilter
echo 'net.bridge.bridge-nf-call-iptables=1' | sudo tee /etc/sysctl.d/99-k3s.conf
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.d/99-k3s.conf
sudo sysctl --system
```

### Install k3s Offline

This runbook assumes the offline bundle was prepared in advance. If not, use `CUSTOMER_AIRGAP_PREP_K3S_ZARF.md` to generate and validate it before continuing.

```bash
mkdir -p /root/airgap
tar -xzf airgap-bundle.tar.gz -C /root/airgap
cd /root/airgap

sha256sum -c SHA256SUMS

install -m 0755 k3s/k3s /usr/local/bin/k3s
mkdir -p /var/lib/rancher/k3s/agent/images
cp k3s/k3s-airgap-images-amd64.tar.zst /var/lib/rancher/k3s/agent/images/
chmod +x k3s/install.sh

INSTALL_K3S_SKIP_DOWNLOAD=true \
INSTALL_K3S_SKIP_SELINUX_RPM=true \
./k3s/install.sh
```

Configure `kubectl`:

```bash
mkdir -p ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
chown "$USER:$USER" ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG="$HOME/.kube/config"

kubectl get nodes
k3s --version
```

### Install Zarf Offline

The `zarf` binary and matching `zarf-init` package in this section should already come from the pre-staged bundle described in `CUSTOMER_AIRGAP_PREP_K3S_ZARF.md`.

```bash
install -m 0755 zarf/zarf /usr/local/bin/zarf
zarf version
zarf init ./zarf/zarf-init-*.tar.zst --confirm
```

### Create Required PriorityClasses

```bash
cat <<'EOF' | kubectl apply -f -
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
EOF
```

### Precreate Namespace and CA Secret

```bash
kubectl create namespace collibra-edge --dry-run=client -o yaml | kubectl apply -f -

kubectl -n collibra-edge create secret generic edge-ca.pem \
  --from-file=ca.pem=/opt/collibra/certs/ca-bundle.pem \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Deploy the Edge Package

```bash
PKG_VER="1.43.15"
EDGE_PKG="/tmp/zarf-package-cpsh-edge-amd64-${PKG_VER}.tar.zst"
test -f "$EDGE_PKG"

zarf package deploy "$EDGE_PKG" \
  --confirm \
  --set-variables PLATFORM_ID="${PLATFORM_ID}" \
  --set-variables PLATFORM_USER="${PLATFORM_USER}" \
  --set-variables PLATFORM_PASSWORD="${PLATFORM_PASSWORD}" \
  --set-variables SITE_NAME="${SITE_NAME}" || true
```

### Fetch Site ID and Generated Edge Credentials

```bash
SITE_ID="$(curl --cacert /opt/collibra/certs/ca-bundle.pem \
  -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
  "${PLATFORM_ID}/edge/api/rest/v2/sites" \
  | jq -r --arg n "$SITE_NAME" '.[] | select(.name==$n) | .id' | head -n1)"

test -n "$SITE_ID"

CREDS_JSON="$(curl --cacert /opt/collibra/certs/ca-bundle.pem \
  -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
  -X POST "${PLATFORM_ID}/edge/api/rest/v2/sites/${SITE_ID}/installerProperties")"

EDGE_USER="$(echo "$CREDS_JSON" | jq -r '.dicInfo.basic.username')"
EDGE_PASS="$(echo "$CREDS_JSON" | jq -r '.dicInfo.basic.password')"
```

### Recreate `dgc-secret`

```bash
printf '%s' "$EDGE_PASS" > /tmp/edge-pass.txt

kubectl -n collibra-edge create secret generic dgc-secret \
  --from-literal=collibra.edge.platform.endpoint="${PLATFORM_ID}" \
  --from-literal=collibra.edge.platform.user="${EDGE_USER}" \
  --from-file=collibra.edge.platform.password=/tmp/edge-pass.txt \
  --dry-run=client -o yaml | kubectl apply -f -

rm -f /tmp/edge-pass.txt
```

### Force Custom CA Toggle and Restart

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

### Validate Edge

```bash
kubectl get pods -n collibra-edge

kubectl -n collibra-edge logs deployment/edge-controller --since=10m | egrep -i "x509|unknown authority|error" || true
kubectl -n collibra-edge logs deployment/edge-proxy --since=10m | egrep -i "x509|unknown authority|error" || true

curl --cacert /opt/collibra/certs/ca-bundle.pem \
  -u "${PLATFORM_USER}:${PLATFORM_PASSWORD}" \
  "${PLATFORM_ID}/edge/api/rest/v2/sites/${SITE_ID}" \
  | jq '{name,status,installedVersion,versionSyncStatus,lastHeartbeatTimestamp}'
```

Expected final status:

- `HEALTHY`

## Post-Install Notes

- Keep Console and DGC certificates aligned to the exact hostnames clients use.
- Do not reuse cloud-specific hostnames in certificates if the customer environment relies on internal DNS.
- For Edge, use the customer CA bundle consistently in OS trust and the Kubernetes `edge-ca.pem` secret.
- Validate repository primary and standby state before onboarding Edge or integrations.
