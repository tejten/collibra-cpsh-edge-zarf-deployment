# CPSH 2025.10 FIPS Runbook (Customer-provided Certificates)

This runbook covers a CPSH 2025.10 FIPS deployment where you replace the default CPSH web certificates with customer-provided certificates.

For a FIPS-enabled CPSH deployment, the HTTPS keystore flow should use `BCFKS`, not a plain `JKS` keystore.

It is based on:
- the Collibra SSL configuration model for DGC and Console
- the attached internal reference pointing to:
  - `/opt/collibra_data/dgc/config/server.json`
  - `/opt/collibra_data/console/config/server.json`

## Scope

Use this runbook when:
- CPSH is already installed or you are installing CPSH now
- the customer has provided certificate material for CPSH access
- you want DGC and Console to present customer certificates instead of default certificates

This runbook assumes:
- CPSH host OS is `RHEL 9.x`
- FIPS is enabled
- DGC will be exposed on `5443`
- Console will be exposed on `5404`

## 1. Inputs

You need:
- server certificate for DGC hostname
- private key for DGC hostname
- server certificate for Console hostname, if different
- private key for Console hostname, if different
- full CA chain bundle
- passwords for generated `PKCS12` / `BCFKS` keystores
- certificate alias names
- resolved Bouncy Castle FIPS provider jar path
- resolved Java security override file path used for FIPS keystore operations

Example placeholders:

```bash
export CERT_DIR="/root/certs"
export DGC_HOST="<dgc-host-fqdn>"
export CONSOLE_HOST="<console-host-fqdn>"

export DGC_CERT="${CERT_DIR}/<dgc-host>.crt"
export DGC_KEY="${CERT_DIR}/<dgc-host>.key"

export CONSOLE_CERT="${CERT_DIR}/<console-host>.crt"
export CONSOLE_KEY="${CERT_DIR}/<console-host>.key"

export CA_BUNDLE="${CERT_DIR}/<customer-ca-bundle>.pem"

export DGC_ALIAS="dgc"
export CONSOLE_ALIAS="console"
export DGC_KS_PASS="<dgc-keystore-password>"
export CONSOLE_KS_PASS="<console-keystore-password>"
```

If the same certificate covers both DGC and Console hostnames through SANs or wildcard coverage, you can reuse one certificate and one keystore for both services. If not, create separate keystores.

## 2. Enable or verify FIPS mode

Run as `root`:

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

Expected: enabled and `1`.

## 3. Install CPSH if not already installed

If CPSH is not installed yet:

```bash
chmod +x ./dgc-linux-2025.10.391.sh
./dgc-linux-2025.10.391.sh -- --fips
```

If CPSH is already installed, continue to the certificate steps below.

## 4. Resolve FIPS provider paths

Before building the `BCFKS` keystores, resolve the installed FIPS provider jars and Java security override file.

Run as `root`:

```bash
FIPS_PROVIDERPATH="$(printf "%s:" /opt/collibra/security/fips/libs/*.jar)"
FIPS_PROVIDERPATH="${FIPS_PROVIDERPATH%:}"

FIPS_JAVA_SECURITY="$(ls /opt/collibra/security/fips/configs/*.security | head -1)"

echo "FIPS_PROVIDERPATH=${FIPS_PROVIDERPATH}"
echo "FIPS_JAVA_SECURITY=${FIPS_JAVA_SECURITY}"

test -n "${FIPS_PROVIDERPATH}"
test -n "${FIPS_JAVA_SECURITY}"
test -f "${FIPS_JAVA_SECURITY}"
```

Expected:
- `FIPS_PROVIDERPATH` expands to a colon-separated list of jars under `/opt/collibra/security/fips/libs`
- `FIPS_JAVA_SECURITY` resolves to a `.security` file under `/opt/collibra/security/fips/configs`

## 5. Validate certificate and key pairs

Run these before building keystores:

```bash
openssl x509 -in "${DGC_CERT}" -noout -subject -issuer -dates
openssl x509 -in "${DGC_CERT}" -noout -modulus | openssl md5
openssl rsa  -in "${DGC_KEY}"  -noout -modulus | openssl md5

openssl verify -CAfile "${CA_BUNDLE}" "${DGC_CERT}"
```

If Console uses a separate certificate:

```bash
openssl x509 -in "${CONSOLE_CERT}" -noout -subject -issuer -dates
openssl x509 -in "${CONSOLE_CERT}" -noout -modulus | openssl md5
openssl rsa  -in "${CONSOLE_KEY}"  -noout -modulus | openssl md5

openssl verify -CAfile "${CA_BUNDLE}" "${CONSOLE_CERT}"
```

The MD5 hashes of certificate modulus and key modulus must match for each pair.

## 6. Build PKCS12 source keystores and BCFKS HTTPS keystores

Create security directories:

```bash
mkdir -p /opt/collibra_data/dgc/security
mkdir -p /opt/collibra_data/console/security
mkdir -p /opt/collibra_data/dgc/tmp
mkdir -p /opt/collibra_data/console/tmp
```

### 5.1 DGC keystore

```bash
openssl pkcs12 -export \
  -in "${DGC_CERT}" \
  -inkey "${DGC_KEY}" \
  -certfile "${CA_BUNDLE}" \
  -name "${DGC_ALIAS}" \
  -out "${CERT_DIR}/dgc-source.p12" \
  -passout pass:"${DGC_KS_PASS}"

/opt/collibra/jre/bin/keytool -importkeystore -noprompt \
  -srckeystore "${CERT_DIR}/dgc-source.p12" \
  -srcstoretype PKCS12 \
  -srcstorepass "${DGC_KS_PASS}" \
  -srcalias "${DGC_ALIAS}" \
  -destkeystore /opt/collibra_data/dgc/security/dgc-https.bcfks \
  -deststoretype BCFKS \
  -deststorepass "${DGC_KS_PASS}" \
  -destkeypass "${DGC_KS_PASS}" \
  -destalias "${DGC_ALIAS}" \
  -providerclass org.bouncycastle.jcajce.provider.BouncyCastleFipsProvider \
  -providername BCFIPS \
  -providerpath "${FIPS_PROVIDERPATH}" \
  -J-Djava.security.properties="${FIPS_JAVA_SECURITY}" \
  -J-Djava.io.tmpdir=/opt/collibra_data/dgc/tmp \
  -J-Dorg.bouncycastle.native.loader.install_dir=/opt/collibra_data/dgc/tmp
```

### 5.2 Console keystore

If Console uses the same certificate, copy or reuse the DGC keystore. If Console uses a separate certificate, build a second keystore:

```bash
openssl pkcs12 -export \
  -in "${CONSOLE_CERT}" \
  -inkey "${CONSOLE_KEY}" \
  -certfile "${CA_BUNDLE}" \
  -name "${CONSOLE_ALIAS}" \
  -out "${CERT_DIR}/console-source.p12" \
  -passout pass:"${CONSOLE_KS_PASS}"

/opt/collibra/jre/bin/keytool -importkeystore -noprompt \
  -srckeystore "${CERT_DIR}/console-source.p12" \
  -srcstoretype PKCS12 \
  -srcstorepass "${CONSOLE_KS_PASS}" \
  -srcalias "${CONSOLE_ALIAS}" \
  -destkeystore /opt/collibra_data/console/security/console-https.bcfks \
  -deststoretype BCFKS \
  -deststorepass "${CONSOLE_KS_PASS}" \
  -destkeypass "${CONSOLE_KS_PASS}" \
  -destalias "${CONSOLE_ALIAS}" \
  -providerclass org.bouncycastle.jcajce.provider.BouncyCastleFipsProvider \
  -providername BCFIPS \
  -providerpath "${FIPS_PROVIDERPATH}" \
  -J-Djava.security.properties="${FIPS_JAVA_SECURITY}" \
  -J-Djava.io.tmpdir=/opt/collibra_data/console/tmp \
  -J-Dorg.bouncycastle.native.loader.install_dir=/opt/collibra_data/console/tmp
```

Verify keystores:

```bash
/opt/collibra/jre/bin/keytool -list -v \
  -keystore /opt/collibra_data/dgc/security/dgc-https.bcfks \
  -storetype BCFKS \
  -storepass "${DGC_KS_PASS}" \
  -providerclass org.bouncycastle.jcajce.provider.BouncyCastleFipsProvider \
  -providername BCFIPS \
  -providerpath "${FIPS_PROVIDERPATH}" \
  -J-Djava.security.properties="${FIPS_JAVA_SECURITY}"

/opt/collibra/jre/bin/keytool -list -v \
  -keystore /opt/collibra_data/console/security/console-https.bcfks \
  -storetype BCFKS \
  -storepass "${CONSOLE_KS_PASS}" \
  -providerclass org.bouncycastle.jcajce.provider.BouncyCastleFipsProvider \
  -providername BCFIPS \
  -providerpath "${FIPS_PROVIDERPATH}" \
  -J-Djava.security.properties="${FIPS_JAVA_SECURITY}"
```

If Console should reuse the DGC certificate, you can copy the generated files and adjust the alias/password values in `server.json` accordingly.

## 7. Configure DGC HTTPS connector

Edit `/opt/collibra_data/dgc/config/server.json` and set or update the `httpsConnector` block.

Example:

```json
"httpsConnector": {
  "port": 5443,
  "keyAlias": "dgc",
  "keyPass": "<dgc-keystore-password>",
  "keystorePass": "<dgc-keystore-password>",
  "keystoreFile": "/opt/collibra_data/dgc/security/dgc-https.bcfks"
}
```

Recommended:
- keep `port` as `5443` if that is already your DGC HTTPS port
- if you want to prevent plain HTTP access, set the `address` value to `127.0.0.1` for the non-SSL listener as documented by Collibra

## 8. Configure Console HTTPS connector

Edit `/opt/collibra_data/console/config/server.json` and set or update the `httpsConnector` block.

Example:

```json
"httpsConnector": {
  "port": 5404,
  "keyAlias": "console",
  "keyPass": "<console-keystore-password>",
  "keystorePass": "<console-keystore-password>",
  "keystoreFile": "/opt/collibra_data/console/security/console-https.bcfks"
}
```

Recommended:
- keep `port` as `5404` if that is already your Console HTTPS port
- if you want to prevent plain HTTP access, set the non-SSL `address` to `127.0.0.1`

## 9. Restart CPSH services

Use Collibra Console to restart the environment, or restart from the host if that is your standard operating path.

After restart, validate both endpoints.

## 10. Validate certificates and service access

### 10.1 DGC

```bash
openssl s_client -connect "${DGC_HOST}:5443" -servername "${DGC_HOST}" </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

### 10.2 Console

```bash
openssl s_client -connect "${CONSOLE_HOST}:5404" -servername "${CONSOLE_HOST}" </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

### 10.3 API check

```bash
curl -u "Admin:<admin-password>" "https://${DGC_HOST}:5443/edge/api/rest/v2/internal/info"
```

Expected: HTTP `200` with JSON.

## 11. Update dependent clients and trust stores

After CPSH presents a customer-provided certificate:
- browsers must trust the issuing CA chain
- Edge nodes must trust the issuing CA chain
- any automation calling CPSH over HTTPS must trust the issuing CA chain

For Edge, use the customer/private CA Edge runbook:
- [`EDGE_CUSTOM_CERT.md`](./EDGE_CUSTOM_CERT.md)

## 12. Common problems

- Browser still shows old certificate:
  - restart completed only partially, or a reverse proxy/load balancer is still presenting an old cert
- `curl` fails with trust error:
  - client does not trust the customer CA bundle yet
- DGC or Console fails to start after `server.json` change:
  - wrong `keystoreFile`
  - wrong alias
  - wrong keystore password or key password
  - wrong `FIPS_PROVIDERPATH` or `FIPS_JAVA_SECURITY` during keystore creation
  - `BCFKS` keystore was not created successfully and the file is empty or missing
  - malformed JSON in `server.json`
- Certificate hostname mismatch:
  - certificate SAN does not include the hostname users are actually using

## References

- Configure SSL to access CPSH DGC: https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_configure-ssl-cpsh.htm
- Configure SSL to access Collibra Console: https://productresources.collibra.com/docs/collibra/latest/Content/Installation/ta_configure-ssl-console.htm
