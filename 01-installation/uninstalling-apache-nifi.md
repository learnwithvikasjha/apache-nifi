# Uninstalling Apache NiFi

```
#!/usr/bin/env bash
set -euo pipefail

SERVICE_NAME="nifi"
NIFI_DIR="/opt/nifi"
NIFI_CONF_DIR="${NIFI_DIR}/conf"
NIFI_USER="nifi"
NIFI_GROUP="nifi"
SYSTEMD_UNIT="/etc/systemd/system/${SERVICE_NAME}.service"

echo "==> Stopping NiFi service (if running)..."
if systemctl list-units --type=service | grep -q "^${SERVICE_NAME}.service"; then
  sudo systemctl stop "${SERVICE_NAME}" || true
  sudo systemctl disable "${SERVICE_NAME}" || true
else
  echo "NiFi service not found in systemd list, skipping stop/disable."
fi

echo "==> Removing systemd unit (if present)..."
if [ -f "${SYSTEMD_UNIT}" ]; then
  sudo rm -f "${SYSTEMD_UNIT}"
  sudo systemctl daemon-reload
  echo "Removed ${SYSTEMD_UNIT} and reloaded systemd."
else
  echo "No ${SYSTEMD_UNIT} found, skipping."
fi

echo "==> Killing any remaining NiFi JVM processes (if any)..."
# Best-effort cleanup
if pgrep -f "org.apache.nifi.NiFi" >/dev/null 2>&1; then
  sudo pkill -f "org.apache.nifi.NiFi" || true
fi

echo "==> Removing NiFi TLS certs we created (if present)..."
if [ -d "${NIFI_CONF_DIR}" ]; then
  sudo rm -f \
    "${NIFI_CONF_DIR}/keystore_ip.p12" \
    "${NIFI_CONF_DIR}/truststore_ip.p12" \
    "${NIFI_CONF_DIR}/nifi-cert.pem" \
    "${NIFI_CONF_DIR}/nifi-keystore.p12" \
    "${NIFI_CONF_DIR}/nifi-truststore.p12" \
    "${NIFI_CONF_DIR}"/nifi.properties.bak* 2>/dev/null || true
else
  echo "NiFi conf dir ${NIFI_CONF_DIR} not found, skipping cert removal."
fi

echo "==> Removing NiFi installation directory ${NIFI_DIR}..."
if [ -d "${NIFI_DIR}" ]; then
  sudo rm -rf "${NIFI_DIR}"
  echo "Removed ${NIFI_DIR}."
else
  echo "NiFi directory ${NIFI_DIR} not found, skipping."
fi

echo "==> Removing NiFi user and group (if present)..."
if id "${NIFI_USER}" >/dev/null 2>&1; then
  # -r removes the home directory if it exists
  sudo userdel -r "${NIFI_USER}" || true
  echo "Removed user ${NIFI_USER}."
else
  echo "User ${NIFI_USER} does not exist, skipping."
fi

if getent group "${NIFI_GROUP}" >/dev/null 2>&1; then
  sudo groupdel "${NIFI_GROUP}" || true
  echo "Removed group ${NIFI_GROUP}."
else
  echo "Group ${NIFI_GROUP} does not exist, skipping."
fi

echo
echo "============================================================"
echo "Apache NiFi and related certs have been removed (as far as this script knows)."
echo "Java, Nginx, and any external data directories (e.g. /efs/...) were NOT touched."
echo "============================================================"

```
