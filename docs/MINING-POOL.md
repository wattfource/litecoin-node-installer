# Mining Pool Configuration

Guide for setting up a Litecoin node as a mining pool backend.

## Overview

When you select **Mining Pool Backend** during setup, the script configures:

- `txindex=1` - Transaction index for pool lookups
- `rpcbind=127.0.0.1` - RPC on localhost only (secure)
- `maxconnections=256` - Higher connection limits
- Block notification options (RPC polling, ZMQ, or blocknotify)

## Pool Software Connection

```
RPC Endpoint:  http://127.0.0.1:9332
RPC User:      (configured during setup)
RPC Password:  (configured during setup)
```

View your credentials:

```bash
grep -E "rpcuser|rpcpassword" /etc/litecoin/litecoin.conf
```

## Block Notification Methods

### Option 1: RPC Polling (Recommended)

Most compatible method. Pool software polls for new blocks:

```bash
# Poll every 1-2 seconds
litecoin-cli -conf=/etc/litecoin/litecoin.conf getbestblockhash
```

No additional configuration needed.

### Option 2: ZMQ Notifications

Instant push notifications (may have compatibility issues):

```
ZMQ Hashblock: tcp://127.0.0.1:28332
ZMQ Rawblock:  tcp://127.0.0.1:28333
```

If ZMQ causes problems, disable it:

```bash
sudo sed -i 's/^zmqpub/#zmqpub/' /etc/litecoin/litecoin.conf
sudo systemctl restart litecoind
```

### Option 3: blocknotify Script

Runs a command on each new block:

```ini
# In litecoin.conf
blocknotify=curl -s http://localhost:8000/newblock/%s
```

## Key RPC Methods

| Method | Description |
|--------|-------------|
| `getblocktemplate` | Get work for miners |
| `submitblock` | Submit found blocks |
| `getbestblockhash` | Check for new blocks |
| `getblockchaininfo` | Node sync status |
| `validateaddress` | Validate miner addresses |
| `getblock` | Get block by hash |
| `getblockhash` | Get hash by height |

## Test Commands

```bash
# Check node sync status
litecoin-cli -conf=/etc/litecoin/litecoin.conf getblockchaininfo

# Get block template for mining
litecoin-cli -conf=/etc/litecoin/litecoin.conf getblocktemplate '{"rules":["segwit"]}'

# Check current best block
litecoin-cli -conf=/etc/litecoin/litecoin.conf getbestblockhash

# Get mining info
litecoin-cli -conf=/etc/litecoin/litecoin.conf getmininginfo

# Validate an address
litecoin-cli -conf=/etc/litecoin/litecoin.conf validateaddress "ltc1q..."
```

## Pool Wallet

View wallet configuration:

```bash
sudo cat /etc/litecoin/pool-wallet.conf
```

See [Wallet Management](WALLET.md) for wallet operations.

## Example Configuration

```ini
# /etc/litecoin/litecoin.conf (Mining Pool Mode)

datadir=/var/lib/litecoin
listen=1
port=9333

# RPC (localhost only)
server=1
rpcuser=poolrpc
rpcpassword=your-secure-password
rpcbind=127.0.0.1
rpcport=9332
rpcallowip=127.0.0.1

# Mining Pool Settings
txindex=1
maxconnections=256

# ZMQ (optional)
# zmqpubhashblock=tcp://127.0.0.1:28332
# zmqpubrawblock=tcp://127.0.0.1:28333

# Performance
dbcache=450
par=4

# Security
upnp=0
```

## Recommended Pool Software

- **[NOMP](https://github.com/zone117x/node-open-mining-portal)** - Node.js Open Mining Portal
- **[open-litecoin-pool](https://github.com/nicehash/open-litecoin-pool)** - Go-based stratum server
- **[MPOS](https://github.com/MPOS/php-mpos)** - PHP Mining Portal

## Multi-Coin Architecture

```
┌─────────────────────────────────────┐
│       Pool Frontend + Database      │
└──────────────────┬──────────────────┘
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│   LTC   │  │   BTC   │  │   XMR   │
│ Stratum │  │ Stratum │  │ Stratum │
│ + Node  │  │ + Node  │  │ + Node  │
└─────────┘  └─────────┘  └─────────┘
```

Each coin needs its own node and stratum server.

## Algorithm Info

- **Algorithm:** Scrypt
- **Block Time:** ~2.5 minutes
- **Merged Mining:** Compatible with Dogecoin

## Troubleshooting

### Node not synced

Pool won't work until fully synced:

```bash
litecoin-cli -conf=/etc/litecoin/litecoin.conf getblockchaininfo | grep -E "blocks|headers|verificationprogress"
```

### RPC connection refused

1. Check service is running: `sudo systemctl status litecoind`
2. Check config: `grep rpc /etc/litecoin/litecoin.conf`
3. Check port: `sudo ss -tlnp | grep 9332`

### getblocktemplate fails

Ensure node is fully synced and txindex is enabled:

```bash
grep txindex /etc/litecoin/litecoin.conf
# Should show: txindex=1
```

