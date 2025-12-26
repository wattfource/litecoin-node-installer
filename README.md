# Litecoin Node Setup for Debian 13

Interactive setup script for deploying Litecoin nodes on Debian 13, supporting both personal use and mining pool backends.

## Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 4GB | 8GB |
| Disk (Full) | 150GB SSD | 250GB SSD |
| Disk (Pruned) | 20GB | 50GB |
| CPU | 2 cores | 4 cores |
| OS | Debian 13 (Trixie) | |

> **Note:** Mining pools require SSD storage. See [VM Requirements](docs/VM-REQUIREMENTS.md) for details.

## Quick Start

### One-Liner Install

```bash
sudo apt update && sudo apt install -y git curl && rm -rf /tmp/litecoin-setup && git clone https://github.com/wattfource/litecoin-node-installer.git /tmp/litecoin-setup && cd /tmp/litecoin-setup && chmod +x setup-litecoin.sh && sudo ./setup-litecoin.sh
```

### Manual Install

```bash
git clone https://github.com/wattfource/litecoin-node-installer.git
cd litecoin-node-installer
sudo ./setup-litecoin.sh
```

## Setup Options

The interactive wizard guides you through:

| Step | Options |
|------|---------|
| **Node Type** | Standard (personal) or Mining Pool Backend |
| **Blockchain** | Full (~120GB) or Pruned (~4GB) |
| **Network** | RPC binding, firewall rules |
| **Pool Wallet** | Create new or use existing (pool mode) |

## After Installation

```bash
# Check status
sudo systemctl status litecoind

# View sync progress
litecoin-cli -conf=/etc/litecoin/litecoin.conf getblockchaininfo

# View logs
sudo journalctl -u litecoind -f

# Re-run setup (update/reconfigure)
sudo ./setup-litecoin.sh
```

## Port Forwarding

| Port | Purpose | Forward? |
|------|---------|----------|
| **9333** | P2P network | **Yes** - required |
| 9332 | RPC | No (localhost only for pools) |
| 28332/28333 | ZMQ | No (localhost only) |

See [Firewall Guide](docs/FIREWALL.md) for router configuration.

## Uninstall

### Complete Removal (One-Liner)

Downloads a fresh uninstall script and removes **everything** installed by the setup script:

```bash
rm -rf /tmp/litecoin-setup && git clone https://github.com/wattfource/litecoin-node-installer.git /tmp/litecoin-setup && sudo /tmp/litecoin-setup/uninstall-litecoin.sh --force && rm -rf /tmp/litecoin-setup
```

### Interactive Uninstall

For more control over what gets removed:

```bash
# Download fresh and run interactively
rm -rf /tmp/litecoin-setup && git clone https://github.com/wattfource/litecoin-node-installer.git /tmp/litecoin-setup && sudo /tmp/litecoin-setup/uninstall-litecoin.sh
```

### Uninstall Options

```bash
# Keep blockchain data (faster reinstall)
sudo ./uninstall-litecoin.sh --keep-blockchain

# Keep wallet files only
sudo ./uninstall-litecoin.sh --keep-wallets

# Silent complete removal (no prompts)
sudo ./uninstall-litecoin.sh --force --quiet
```

## File Locations

| Path | Description |
|------|-------------|
| `/opt/litecoin/` | Binaries |
| `/var/lib/litecoin/` | Blockchain data |
| `/etc/litecoin/litecoin.conf` | Configuration |
| `/var/log/litecoin/` | Logs |

## Documentation

| Guide | Description |
|-------|-------------|
| [Mining Pool Setup](docs/MINING-POOL.md) | Pool-specific configuration and RPC methods |
| [VM Requirements](docs/VM-REQUIREMENTS.md) | Detailed specifications and cloud providers |
| [Wallet Management](docs/WALLET.md) | Wallet CLI, backups, security |
| [Troubleshooting](docs/TROUBLESHOOTING.md) | Common issues and fixes |
| [Firewall Setup](docs/FIREWALL.md) | Port forwarding for different routers |
| [Lessons Learned](docs/LESSONS-LEARNED.md) | Build issues, fixes, and best practices for Debian 13 |

## Resources

- [Litecoin Website](https://litecoin.org/)
- [Litecoin GitHub](https://github.com/litecoin-project/litecoin)

## License

MIT License
