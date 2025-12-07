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
