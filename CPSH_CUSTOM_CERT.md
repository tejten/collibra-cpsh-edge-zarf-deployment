# CPSH 2025.10 FIPS Runbook (Customer-provided Certificates)

This runbook covers a CPSH 2025.10 FIPS deployment where you replace the default CPSH web certificates with customer-provided certificates.

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
- passwords for generated `PKCS12` / `JKS` keystores
- certificate alias names

Example placeholders:

```bash
export DGC_HOST="<dgc-host-fqdn>"
export CONSOLE_HOST="<console-host-fqdn>"

export DGC_CERT="/root/certs/<dgc-host>.crt"
export DGC_KEY="/root/certs/<dgc-host>.key"

export CONSOLE_CERT="/root/certs/<console-host>.crt"
export CONSOLE_KEY="/root/certs/<console-host>.key"

export CA_BUNDLE="/root/certs/<customer-ca-bundle>.pem"

export DGC_ALIAS="dgc"
export CONSOLE_ALIAS="console"
export DGC_P12_PASS="<dgc-p12-password>"
export CONSOLE_P12_PASS="<console-p12-password>"
export DGC_JKS_PASS="<dgc-jks-password>"
export CONSOLE_JKS_PASS="<console-jks-password>"
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

## 4. Validate certificate and key pairs

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

## 5. Build PKCS12 and JKS keystores

Create security directories:

```bash
mkdir -p /opt/collibra_data/dgc/security
mkdir -p /opt/collibra_data/console/security
```

### 5.1 DGC keystore

```bash
openssl pkcs12 -export \
  -in "${DGC_CERT}" \
  -inkey "${DGC_KEY}" \
  -certfile "${CA_BUNDLE}" \
  -name "${DGC_ALIAS}" \
  -out /opt/collibra_data/dgc/security/dgc.p12 \
  -passout pass:"${DGC_P12_PASS}"

keytool -importkeystore \
  -deststorepass "${DGC_JKS_PASS}" \
  -destkeypass "${DGC_JKS_PASS}" \
  -destkeystore /opt/collibra_data/dgc/security/dgc.jks \
  -srckeystore /opt/collibra_data/dgc/security/dgc.p12 \
  -srcstoretype PKCS12 \
  -srcstorepass "${DGC_P12_PASS}" \
  -alias "${DGC_ALIAS}"
```

### 5.2 Console keystore

If Console uses the same certificate, copy or reuse the DGC keystore. If Console uses a separate certificate, build a second keystore:

```bash
openssl pkcs12 -export \
  -in "${CONSOLE_CERT}" \
  -inkey "${CONSOLE_KEY}" \
  -certfile "${CA_BUNDLE}" \
  -name "${CONSOLE_ALIAS}" \
  -out /opt/collibra_data/console/security/console.p12 \
  -passout pass:"${CONSOLE_P12_PASS}"

keytool -importkeystore \
  -deststorepass "${CONSOLE_JKS_PASS}" \
  -destkeypass "${CONSOLE_JKS_PASS}" \
  -destkeystore /opt/collibra_data/console/security/console.jks \
  -srckeystore /opt/collibra_data/console/security/console.p12 \
  -srcstoretype PKCS12 \
  -srcstorepass "${CONSOLE_P12_PASS}" \
  -alias "${CONSOLE_ALIAS}"
```

Verify keystores:

```bash
keytool -list -v -keystore /opt/collibra_data/dgc/security/dgc.jks -storepass "${DGC_JKS_PASS}"
keytool -list -v -keystore /opt/collibra_data/console/security/console.jks -storepass "${CONSOLE_JKS_PASS}"
```

## 6. Configure DGC HTTPS connector

Edit `/opt/collibra_data/dgc/config/server.json` and set or update the `httpsConnector` block.

Example:

```json
"httpsConnector": {
  "port": 5443,
  "keyAlias": "dgc",
  "keyPass": "<dgc-jks-password>",
  "keystorePass": "<dgc-jks-password>",
  "keystoreFile": "/opt/collibra_data/dgc/security/dgc.jks"
}
```

Recommended:
- keep `port` as `5443` if that is already your DGC HTTPS port
- if you want to prevent plain HTTP access, set the `address` value to `127.0.0.1` for the non-SSL listener as documented by Collibra

## 7. Configure Console HTTPS connector

Edit `/opt/collibra_data/console/config/server.json` and set or update the `httpsConnector` block.

Example:

```json
"httpsConnector": {
  "port": 5404,
  "keyAlias": "console",
  "keyPass": "<console-jks-password>",
  "keystorePass": "<console-jks-password>",
  "keystoreFile": "/opt/collibra_data/console/security/console.jks"
}
```

Recommended:
- keep `port` as `5404` if that is already your Console HTTPS port
- if you want to prevent plain HTTP access, set the non-SSL `address` to `127.0.0.1`

## 8. Restart CPSH services

Use Collibra Console to restart the environment, or restart from the host if that is your standard operating path.

After restart, validate both endpoints.

## 9. Validate certificates and service access

### 9.1 DGC

```bash
openssl s_client -connect "${DGC_HOST}:5443" -servername "${DGC_HOST}" </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

### 9.2 Console

```bash
openssl s_client -connect "${CONSOLE_HOST}:5404" -servername "${CONSOLE_HOST}" </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

### 9.3 API check

```bash
curl -u "Admin:<admin-password>" "https://${DGC_HOST}:5443/edge/api/rest/v2/internal/info"
```

Expected: HTTP `200` with JSON.

## 10. Update dependent clients and trust stores

After CPSH presents a customer-provided certificate:
- browsers must trust the issuing CA chain
- Edge nodes must trust the issuing CA chain
- any automation calling CPSH over HTTPS must trust the issuing CA chain

For Edge, use the customer/private CA Edge runbook:
- [`EDGE_CUSTOM_CERT.md`](./EDGE_CUSTOM_CERT.md)

## 11. Common problems

- Browser still shows old certificate:
  - restart completed only partially, or a reverse proxy/load balancer is still presenting an old cert
- `curl` fails with trust error:
  - client does not trust the customer CA bundle yet
- DGC or Console fails to start after `server.json` change:
  - wrong `keystoreFile`
  - wrong alias
  - wrong keystore password or key password
  - malformed JSON in `server.json`
- Certificate hostname mismatch:
  - certificate SAN does not include the hostname users are actually using

## References

- Configure SSL to access CPSH DGC: https://productresources.collibra.com/docs/cpsh/latest/Content/Installation/CPSH/ta_configure-ssl-cpsh.htm
- Configure SSL to access Collibra Console: https://productresources.collibra.com/docs/collibra/latest/Content/Installation/ta_configure-ssl-console.htm
