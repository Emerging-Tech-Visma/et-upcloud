# Provision Server Playbook

## Step 1: Create the Cloud Server

```bash
upctl server create \
  --hostname "{project}-prod" \
  --zone "{zone}" \
  --plan "{plan}" \
  --os "Ubuntu Server 24.04 LTS (Noble Numbat)" \
  --os-storage-size 50 \
  --ssh-keys ~/.ssh/id_ed25519.pub \
  --enable-firewall \
  --enable-metadata \
  --network family=IPv4,type=public \
  --network family=IPv6,type=public \
  --wait
```

## Step 2: Get Server IP

```bash
SERVER_IP=$(upctl server show "{project}-prod" -o json | jq -r '.ip_addresses[] | select(.family=="IPv4" and .access=="public") | .address')
echo "Server IP: ${SERVER_IP}"
```

## Step 3: Configure Firewall

```bash
# Allow SSH
upctl server firewall create "{project}-prod" \
  --direction in \
  --action accept \
  --family IPv4 \
  --protocol tcp \
  --destination-port-start 22 \
  --destination-port-end 22

# Allow HTTP
upctl server firewall create "{project}-prod" \
  --direction in \
  --action accept \
  --family IPv4 \
  --protocol tcp \
  --destination-port-start 80 \
  --destination-port-end 80

# Allow HTTPS
upctl server firewall create "{project}-prod" \
  --direction in \
  --action accept \
  --family IPv4 \
  --protocol tcp \
  --destination-port-start 443 \
  --destination-port-end 443
```

## Step 4: Install Docker + Caddy + Infisical CLI

SSH into the server and run:

```bash
ssh root@${SERVER_IP} 'bash -s' <<'INSTALL_SCRIPT'
set -euo pipefail

# Update system
apt-get update && apt-get upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sh
systemctl enable docker
systemctl start docker

# Install Docker Compose plugin
apt-get install -y docker-compose-plugin

# Install Caddy
apt-get install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt-get update
apt-get install -y caddy

# Install Infisical CLI
curl -1sLf 'https://dl.cloudsmith.io/public/infisical/infisical-cli/setup.deb.sh' | bash
apt-get install -y infisical

# Create app directory
mkdir -p /opt/apps

echo "Installation complete"
INSTALL_SCRIPT
```

## Step 5: Verify

```bash
ssh root@${SERVER_IP} 'docker --version && caddy version && infisical --version'
```

## Notes

- The `--wait` flag blocks until the server reaches `started` state
- Firewall is enabled at creation; rules must be added or all traffic is blocked
- The install script is idempotent — safe to run multiple times
