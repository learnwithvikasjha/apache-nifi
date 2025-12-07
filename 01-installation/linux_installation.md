# Installing Apache NiFi on Ubuntu

```
#!/usr/bin/env bash
set -e

# --- Config ---
NIFI_VERSION="2.6.0"
NIFI_ARCHIVE="nifi-${NIFI_VERSION}-bin.zip"
NIFI_DOWNLOAD_URL="https://downloads.apache.org/nifi/${NIFI_VERSION}/${NIFI_ARCHIVE}"
NIFI_BASE_DIR="/opt"
NIFI_DIR="${NIFI_BASE_DIR}/nifi"
NIFI_USER="nifi"
NIFI_GROUP="nifi"
SYSTEMD_SERVICE="/etc/systemd/system/nifi.service"

echo "==> Updating apt and installing dependencies (Java 21, wget, unzip)..."
sudo apt update -y
sudo apt install -y openjdk-21-jdk wget unzip

echo "==> Ensuring base dir ${NIFI_BASE_DIR} exists..."
sudo mkdir -p "${NIFI_BASE_DIR}"
cd "${NIFI_BASE_DIR}"

if [ ! -f "${NIFI_ARCHIVE}" ]; then
  echo "==> Downloading Apache NiFi ${NIFI_VERSION}..."
  sudo wget "${NIFI_DOWNLOAD_URL}"
else
  echo "==> Archive ${NIFI_ARCHIVE} already exists, skipping download."
fi

if [ -d "${NIFI_DIR}" ]; then
  echo "==> Directory ${NIFI_DIR} already exists, skipping extract."
else
  echo "==> Extracting NiFi..."
  sudo unzip -q "${NIFI_ARCHIVE}"
  sudo mv "nifi-${NIFI_VERSION}" "${NIFI_DIR}"
fi

echo "==> Ensuring group ${NIFI_GROUP} exists..."
if getent group "${NIFI_GROUP}" > /dev/null 2>&1; then
  echo "Group ${NIFI_GROUP} already exists."
else
  sudo groupadd "${NIFI_GROUP}"
fi

echo "==> Ensuring user ${NIFI_USER} exists..."
if id "${NIFI_USER}" >/dev/null 2>&1; then
  echo "User ${NIFI_USER} already exists."
else
  sudo useradd -r -g "${NIFI_GROUP}" -d "${NIFI_DIR}" -s /bin/bash "${NIFI_USER}"
fi

echo "==> Setting ownership on ${NIFI_DIR}..."
sudo chown -R "${NIFI_USER}:${NIFI_GROUP}" "${NIFI_DIR}"

echo "==> Writing systemd service to ${SYSTEMD_SERVICE}..."
sudo bash -c "cat > ${SYSTEMD_SERVICE}" <<EOF
[Unit]
Description=Apache NiFi ${NIFI_VERSION}
After=network.target

[Service]
Type=forking
User=${NIFI_USER}
Group=${NIFI_GROUP}
ExecStart=${NIFI_DIR}/bin/nifi.sh start
ExecStop=${NIFI_DIR}/bin/nifi.sh stop
Restart=on-abort
LimitNOFILE=50000

[Install]
WantedBy=multi-user.target
EOF

echo "==> Reloading systemd daemon..."
sudo systemctl daemon-reload

echo "==> Enabling NiFi service..."
sudo systemctl enable nifi

echo "==> Starting (or restarting) NiFi..."
sudo systemctl restart nifi || sudo systemctl start nifi

echo "==> NiFi service status:"
sudo systemctl --no-pager status nifi || true

SERVER_IP=$(hostname -I | awk '{print $1}')

echo "============================================================"
echo "Apache NiFi ${NIFI_VERSION} install/update completed."
if [ -n "${SERVER_IP}" ]; then
  echo "NiFi uses HTTPS by default in 2.x."
  echo "Try opening (you'll likely see a browser warning for the self-signed cert):"
  echo "    https://${SERVER_IP}:8443/nifi"
else
  echo "NiFi uses HTTPS by default in 2.x."
  echo "Access NiFi at: https://<your-server-ip>:8443/nifi"
fi
echo "============================================================"
```

# Configure NiFI TLS 
```
#!/usr/bin/env bash
set -euo pipefail

# === CONFIG ===
NIFI_CONF_DIR="/opt/nifi/conf"
NIFI_IP="31.97.202.180"
CERT_PASS="${CERT_PASS:-ChangeMe123!}"   # override by EXPORT CERT_PASS=... if you want

KEYSTORE_FILE="${NIFI_CONF_DIR}/keystore_ip.p12"
TRUSTSTORE_FILE="${NIFI_CONF_DIR}/truststore_ip.p12"
CERT_PEM="${NIFI_CONF_DIR}/nifi-cert.pem"
NIFI_PROPS="${NIFI_CONF_DIR}/nifi.properties"

echo "==> Using NiFi conf dir: ${NIFI_CONF_DIR}"
echo "==> Using IP: ${NIFI_IP}"
echo "==> Using certificate password (CERT_PASS): [hidden]"

cd "${NIFI_CONF_DIR}"

echo "==> Backing up existing config and TLS files (if present)..."
cp "${NIFI_PROPS}" "${NIFI_PROPS}.bak.$(date +%s)"
cp keystore.p12 keystore.p12.bak.$(date +%s) 2>/dev/null || true
cp truststore.p12 truststore.p12.bak.$(date +%s) 2>/dev/null || true

echo "==> Generating new PKCS12 keystore with SAN = ip:${NIFI_IP}, dns:localhost ..."
keytool -genkeypair \
  -alias nifi \
  -keyalg RSA \
  -keysize 4096 \
  -storetype PKCS12 \
  -keystore "${KEYSTORE_FILE}" \
  -storepass "${CERT_PASS}" \
  -keypass "${CERT_PASS}" \
  -validity 3650 \
  -dname "CN=${NIFI_IP}, OU=NiFi, O=MyOrg, L=City, S=State, C=IN" \
  -ext "san=ip:${NIFI_IP},dns:localhost"

echo "==> Exporting certificate from keystore..."
keytool -exportcert \
  -alias nifi \
  -keystore "${KEYSTORE_FILE}" \
  -storepass "${CERT_PASS}" \
  -rfc \
  -file "${CERT_PEM}"

echo "==> Creating truststore from exported certificate..."
keytool -importcert \
  -alias nifi \
  -file "${CERT_PEM}" \
  -keystore "${TRUSTSTORE_FILE}" \
  -storetype PKCS12 \
  -storepass "${CERT_PASS}" \
  -noprompt

echo "==> Updating nifi.properties with HTTPS + TLS settings..."

prop_set() {
  local key="$1"
  local value="$2"
  local file="$3"

  if grep -qE "^${key}=" "${file}"; then
    # Replace existing line
    sed -i "s|^${key}=.*|${key}=${value}|" "${file}"
  else
    # Append new line
    echo "${key}=${value}" >> "${file}"
  fi
}

# Web HTTPS
prop_set "nifi.web.https.host" "${NIFI_IP}" "${NIFI_PROPS}"
prop_set "nifi.web.https.port" "8443" "${NIFI_PROPS}"

# Keystore
prop_set "nifi.security.keystore" "${KEYSTORE_FILE}" "${NIFI_PROPS}"
prop_set "nifi.security.keystoreType" "PKCS12" "${NIFI_PROPS}"
prop_set "nifi.security.keystorePasswd" "${CERT_PASS}" "${NIFI_PROPS}"
prop_set "nifi.security.keyPasswd" "${CERT_PASS}" "${NIFI_PROPS}"

# Truststore
prop_set "nifi.security.truststore" "${TRUSTSTORE_FILE}" "${NIFI_PROPS}"
prop_set "nifi.security.truststoreType" "PKCS12" "${NIFI_PROPS}"
prop_set "nifi.security.truststorePasswd" "${CERT_PASS}" "${NIFI_PROPS}"

echo "==> Done updating nifi.properties."

echo
echo "============================================================"
echo "NiFi TLS configuration complete."
echo "Keystore:   ${KEYSTORE_FILE}"
echo "Truststore: ${TRUSTSTORE_FILE}"
echo
echo "Now restart NiFi:"
echo "  sudo systemctl restart nifi"
echo
echo "Then access NiFi at:"
echo "  https://${NIFI_IP}:8443/nifi"
echo "============================================================"

```

# Restart NiFi
```
sudo systemctl restart nifi
```

# Removing the Keystore if required

```
sudo rm -f /opt/nifi/conf/keystore_ip.p12 \
           /opt/nifi/conf/truststore_ip.p12 \
           /opt/nifi/conf/nifi-cert.pem
```

# Finding NiFi Username/Password from Logs
```
grep -E "Generated (Username|Password)" /opt/nifi/logs/nifi-app.log
```

# Changing NiFi Username/Password
```
sudo /opt/nifi/bin/nifi.sh set-single-user-credentials admin 'admin@admin@'
```
