# Deployment Guide

This guide covers various deployment methods for `ethdo`, a command-line tool for managing common tasks in Ethereum 2.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Deployment Methods](#deployment-methods)
  - [Docker Deployment](#docker-deployment)
  - [Binary Deployment](#binary-deployment)
  - [Building from Source](#building-from-source)
- [Configuration](#configuration)
- [Production Considerations](#production-considerations)
- [Networking](#networking)
- [Security](#security)
- [Monitoring](#monitoring)

## Prerequisites

Before deploying `ethdo`, ensure you have:

- Access to an Ethereum 2 beacon node with REST API enabled
- Appropriate permissions for the deployment environment
- Network connectivity to the beacon node

## Deployment Methods

### Docker Deployment

Docker is the recommended deployment method for production environments as it provides isolation and consistency.

#### Using Pre-built Images

Pull the latest image from Docker Hub:

```bash
docker pull wealdtech/ethdo:latest
```

Or use a specific version:

```bash
docker pull wealdtech/ethdo:v1.38.0
```

#### Building Custom Image

Build the Docker image locally:

```bash
docker build -t ethdo:custom .
```

For multi-architecture builds:

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t ethdo:custom .
```

#### Running the Container

Basic usage:

```bash
docker run -it wealdtech/ethdo:latest --help
```

With host network access (for local beacon node):

```bash
docker run --network=host wealdtech/ethdo:latest chain status --connection=http://localhost:5052
```

With custom network:

```bash
# Create a shared network
docker network create eth2

# Run your beacon node with --network=eth2
# Then run ethdo with the same network
docker run --network=eth2 wealdtech/ethdo:latest chain status --connection=http://beacon-node:5052
```

With persistent volumes for wallets:

```bash
docker run -v /path/to/wallets:/wallets \
  -e ETHDO_BASE_DIR=/wallets \
  wealdtech/ethdo:latest wallet list
```

#### Docker Compose

Create a `docker-compose.yml` file:

```yaml
version: '3.8'
services:
  ethdo:
    image: wealdtech/ethdo:latest
    networks:
      - eth2
    volumes:
      - ./wallets:/wallets
    environment:
      - ETHDO_BASE_DIR=/wallets
      - ETHDO_CONNECTION=http://beacon-node:5052
    command: ["--help"]

networks:
  eth2:
    external: true
```

Run with:

```bash
docker-compose run ethdo chain status
```

### Binary Deployment

#### Download Pre-built Binaries

Download the latest release from the [releases page](https://github.com/wealdtech/ethdo/releases):

```bash
# Linux AMD64
wget https://github.com/wealdtech/ethdo/releases/download/v1.38.0/ethdo-1.38.0-linux-amd64.tar.gz
tar -xzf ethdo-1.38.0-linux-amd64.tar.gz
sudo mv ethdo /usr/local/bin/
sudo chmod +x /usr/local/bin/ethdo
```

#### System Service Deployment

For running as a systemd service (for scheduled tasks):

Create `/etc/systemd/system/ethdo-task.service`:

```ini
[Unit]
Description=Ethdo Task Runner
After=network.target

[Service]
Type=oneshot
User=ethdo
Group=ethdo
WorkingDirectory=/home/ethdo
Environment="ETHDO_CONNECTION=http://localhost:5052"
Environment="ETHDO_BASE_DIR=/home/ethdo/wallets"
ExecStart=/usr/local/bin/ethdo chain status
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Create a timer `/etc/systemd/system/ethdo-task.timer`:

```ini
[Unit]
Description=Run Ethdo Task Periodically

[Timer]
OnCalendar=hourly
Persistent=true

[Install]
WantedBy=timers.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable ethdo-task.timer
sudo systemctl start ethdo-task.timer
```

### Building from Source

#### Using Go

```bash
# Install Go 1.20 or later
go version

# Install ethdo
go install github.com/wealdtech/ethdo@latest

# The binary will be in $GOPATH/bin or ~/go/bin
```

#### Manual Build

```bash
# Clone the repository
git clone https://github.com/wealdtech/ethdo.git
cd ethdo

# Download dependencies
go mod download

# Build
go build -o ethdo

# Install
sudo mv ethdo /usr/local/bin/
```

## Configuration

### Configuration File

Create a configuration file in your home directory (`.ethdo.json` or `.ethdo.yml`):

**JSON format** (`.ethdo.json`):

```json
{
  "connection": "http://localhost:5052",
  "base-dir": "/home/ethdo/wallets",
  "verbose": false,
  "timeout": "30s"
}
```

**YAML format** (`.ethdo.yml`):

```yaml
connection: http://localhost:5052
base-dir: /home/ethdo/wallets
verbose: false
timeout: 30s
```

Custom configuration file location:

```bash
ethdo --config=/etc/ethdo/config.json chain status
```

### Environment Variables

All configuration options can be set via environment variables with the `ETHDO_` prefix:

```bash
export ETHDO_CONNECTION=http://localhost:5052
export ETHDO_BASE_DIR=/home/ethdo/wallets
export ETHDO_PASSPHRASE="my secure passphrase"
export ETHDO_VERBOSE=true
```

Priority order (highest to lowest):
1. Command-line arguments
2. Environment variables
3. Configuration file
4. Default values

### Common Configuration Options

- `connection`: Beacon node REST API endpoint (e.g., `http://localhost:5052`)
- `base-dir`: Directory for wallet storage
- `passphrase`: Account passphrase (use environment variable, not config file)
- `storepassphrase`: Store passphrase for encrypted stores
- `timeout`: Timeout for beacon node connections
- `verbose`: Enable verbose output
- `quiet`: Suppress all output except errors

## Production Considerations

### File Permissions

Ensure proper permissions on wallet directories:

```bash
# Create dedicated user
sudo useradd -r -s /bin/false ethdo

# Set up directories
sudo mkdir -p /var/lib/ethdo/wallets
sudo chown -R ethdo:ethdo /var/lib/ethdo
sudo chmod 700 /var/lib/ethdo/wallets
```

### Backup Strategy

Implement a backup strategy for wallets:

```bash
# Backup wallets
tar -czf ethdo-wallets-$(date +%Y%m%d).tar.gz /var/lib/ethdo/wallets

# Encrypt backup
gpg -c ethdo-wallets-$(date +%Y%m%d).tar.gz

# Store securely off-site
```

### Secret Management

**Never** store passphrases in configuration files or version control. Use:

- Environment variables (for temporary use)
- Secret management systems (HashiCorp Vault, AWS Secrets Manager, etc.)
- Secure key management services

Example with HashiCorp Vault:

```bash
# Retrieve passphrase from Vault
export ETHDO_PASSPHRASE=$(vault kv get -field=passphrase secret/ethdo/account)

# Run ethdo command
ethdo account info --account="wallet/account"
```

### Resource Requirements

Minimum requirements:
- CPU: 1 core
- RAM: 512 MB
- Disk: 100 MB (plus wallet storage)
- Network: Stable connection to beacon node

## Networking

### Beacon Node Connection

Configure your beacon node to allow connections:

#### Lighthouse

```bash
lighthouse bn --http --http-address 0.0.0.0 --http-port 5052
```

#### Nimbus

```bash
nimbus_beacon_node --rest --rest-address=0.0.0.0 --rest-port=5052
```

#### Prysm

```bash
prysm --grpc-gateway-host=0.0.0.0 --grpc-gateway-port=3500 --grpc-max-msg-size=268435456
```

#### Teku

```bash
teku --rest-api-enabled=true --rest-api-interface=0.0.0.0 --rest-api-port=5051
```

#### Lodestar

```bash
lodestar beacon --rest --rest.address 0.0.0.0 --rest.port 9596
```

### Firewall Configuration

If using a remote beacon node, ensure appropriate firewall rules:

```bash
# Allow outbound connections to beacon node
sudo ufw allow out to <beacon-node-ip> port 5052

# If running beacon node locally
sudo ufw allow 5052/tcp
```

### TLS/SSL

For production deployments, use TLS for beacon node connections:

```bash
# Use HTTPS connection
ethdo chain status --connection=https://beacon-node.example.com:5052
```

Consider using a reverse proxy like nginx for TLS termination:

```nginx
server {
    listen 443 ssl;
    server_name beacon-node.example.com;

    ssl_certificate /etc/ssl/certs/beacon-node.crt;
    ssl_certificate_key /etc/ssl/private/beacon-node.key;

    location / {
        proxy_pass http://localhost:5052;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Security

### Best Practices

1. **Minimize Exposure**: Run `ethdo` on the same server as the beacon node when possible
2. **Use Read-Only Access**: Configure beacon node with read-only API access for `ethdo`
3. **Encrypt Wallets**: Always use encrypted wallet stores with strong passphrases
4. **Regular Updates**: Keep `ethdo` and beacon node software up to date
5. **Audit Logs**: Monitor access logs and command history
6. **Network Segmentation**: Use private networks or VPNs for remote access

### Passphrase Guidelines

Follow these guidelines for secure passphrases:

- Minimum length: 12 characters
- Use mix of uppercase, lowercase, numbers, and symbols
- Avoid dictionary words and common patterns
- Use a password manager
- Never reuse passphrases

### Secure Deployment Checklist

- [ ] Wallets stored with encrypted file system
- [ ] Proper file permissions (700 for wallet directories)
- [ ] Passphrases managed via secure secret management
- [ ] TLS enabled for remote beacon node connections
- [ ] Firewall rules configured
- [ ] Regular backups implemented
- [ ] Audit logging enabled
- [ ] Minimal user privileges
- [ ] Regular security updates

## Monitoring

### Health Checks

Monitor beacon node connectivity:

```bash
# Check connection
ethdo chain status --connection=http://localhost:5052

# Check specific validator
ethdo validator info --validator=<pubkey> --connection=http://localhost:5052
```

### Logging

Configure logging for production:

```bash
# With systemd, view logs
journalctl -u ethdo-task -f

# Docker logging
docker logs -f ethdo-container

# File logging
ethdo --verbose chain status > /var/log/ethdo/status.log 2>&1
```

### Alerting

Set up monitoring and alerting:

```bash
# Example health check script
#!/bin/bash
if ! ethdo chain status --connection=http://localhost:5052 > /dev/null 2>&1; then
    echo "Beacon node connection failed" | mail -s "Ethdo Alert" admin@example.com
fi
```

### Prometheus Integration

For beacon nodes with Prometheus metrics:

```bash
# Monitor beacon node metrics
curl http://localhost:5054/metrics
```

## Troubleshooting

### Common Issues

**Connection Refused**:
- Verify beacon node is running
- Check firewall rules
- Confirm REST API is enabled on beacon node

**Authentication Failed**:
- Verify passphrase is correct
- Check environment variables
- Ensure wallet exists

**Timeout Errors**:
- Increase timeout: `--timeout=60s`
- Check network connectivity
- Verify beacon node is synced

For more troubleshooting help, see [troubleshooting.md](troubleshooting.md).

## Additional Resources

- [Usage Guide](usage.md)
- [How-To Guide](howto.md)
- [Troubleshooting](troubleshooting.md)
- [GitHub Repository](https://github.com/wealdtech/ethdo)
- [Ethereum 2 Beacon API](https://ethereum.github.io/beacon-APIs/)

## Support

For issues and questions:
- GitHub Issues: https://github.com/wealdtech/ethdo/issues
- Documentation: https://github.com/wealdtech/ethdo/tree/master/docs
