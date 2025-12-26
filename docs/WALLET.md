# Pool Wallet Management

This guide covers managing the Litecoin wallet created during pool setup.

## Where is My Wallet?

The wallet was created during setup and stored securely:

| Item | Location |
|------|----------|
| Wallet files | `/var/lib/litecoin/wallets/pool-wallet/` |
| Config (address, credentials) | `/etc/litecoin/pool-wallet.conf` |
| Daemon config | `/etc/litecoin/litecoin.conf` |

## View Wallet Details

```bash
# View address and wallet info
sudo cat /etc/litecoin/pool-wallet.conf
```

**Important:** This file contains sensitive information. Anyone with access can potentially control your funds.

## How the Pool Wallet Works

```
Miners submit shares → Pool finds block → Block reward → Your Pool Wallet
                                              ↓
                              Pool fee (your cut) stays in wallet
                              Miner payouts sent from wallet
```

The pool wallet:
1. **Receives** block rewards when the pool finds blocks
2. **Holds** your pool fee percentage  
3. **Sends** payouts to miners (configured in pool software)

## Litecoin Wallet Types

Litecoin Core supports several wallet types:

| Type | Description | Use Case |
|------|-------------|----------|
| **Legacy** | Original format (starts with L or 3) | Maximum compatibility |
| **Bech32** | SegWit native (starts with ltc1) | Lower fees, recommended |
| **Descriptor** | New format in Core 0.21+ | Advanced features |

The setup script creates a descriptor wallet by default for best forward compatibility.

## Accessing Your Wallet

### Check Wallet Balance

Once your node is synced:

```bash
# Check if node is synced first
litecoin-cli -conf=/etc/litecoin/litecoin.conf getblockchaininfo | jq '{blocks, headers, sync: (if .blocks == .headers then "SYNCED" else "SYNCING" end)}'

# Check wallet balance
litecoin-cli -conf=/etc/litecoin/litecoin.conf getbalance
```

### View Wallet Addresses

```bash
# List all addresses
litecoin-cli -conf=/etc/litecoin/litecoin.conf listreceivedbyaddress 0 true

# Get new receiving address
litecoin-cli -conf=/etc/litecoin/litecoin.conf getnewaddress "" "bech32"
```

### View Transaction History

```bash
# List recent transactions
litecoin-cli -conf=/etc/litecoin/litecoin.conf listtransactions "*" 20

# Get transaction details
litecoin-cli -conf=/etc/litecoin/litecoin.conf gettransaction "txid"
```

## Sending Litecoin

### Basic Send

```bash
# Send to address
litecoin-cli -conf=/etc/litecoin/litecoin.conf sendtoaddress "ltc1q..." 1.0

# Send with specific fee rate (sat/vB)
litecoin-cli -conf=/etc/litecoin/litecoin.conf sendtoaddress "ltc1q..." 1.0 "" "" false true 1 "UNSET" null false 10
```

### Send Entire Balance

```bash
# Sweep wallet to new address
litecoin-cli -conf=/etc/litecoin/litecoin.conf sendall '["ltc1qYOURADDRESS"]'
```

### Batch Payouts (for pools)

```bash
# Send to multiple addresses in one transaction
litecoin-cli -conf=/etc/litecoin/litecoin.conf sendmany "" '{"ltc1qaddr1": 0.5, "ltc1qaddr2": 1.0, "ltc1qaddr3": 0.25}'
```

## Wallet Backup

### Export Wallet Descriptor

For descriptor wallets (default in new setups):

```bash
# Export wallet descriptors (KEEP THIS SAFE!)
litecoin-cli -conf=/etc/litecoin/litecoin.conf listdescriptors true
```

Save this output securely - it contains everything needed to restore your wallet.

### Backup Wallet Files

```bash
# Backup entire wallet directory
sudo cp -r /var/lib/litecoin/wallets/pool-wallet /secure-backup/litecoin-wallet-backup-$(date +%Y%m%d)

# Or use the built-in backup command
litecoin-cli -conf=/etc/litecoin/litecoin.conf backupwallet "/secure-backup/wallet-$(date +%Y%m%d).dat"
```

### What to Backup

| Item | How to Backup | Recovery Method |
|------|---------------|-----------------|
| Wallet files | Copy `/var/lib/litecoin/wallets/` | Replace directory |
| Descriptors | `listdescriptors true` | `importdescriptors` |
| Individual keys | `dumpprivkey ADDRESS` | `importprivkey` |

## Wallet Restoration

### From Wallet Files

```bash
# Stop daemon
sudo systemctl stop litecoind

# Restore wallet directory
sudo cp -r /secure-backup/litecoin-wallet-backup /var/lib/litecoin/wallets/pool-wallet
sudo chown -R litecoin:litecoin /var/lib/litecoin/wallets

# Start daemon
sudo systemctl start litecoind
```

### From Descriptors

```bash
# Create new empty wallet
litecoin-cli -conf=/etc/litecoin/litecoin.conf createwallet "restored-wallet" false false "" false true

# Import descriptors (from your backup)
litecoin-cli -conf=/etc/litecoin/litecoin.conf importdescriptors '[{"desc":"wpkh([fingerprint/84h/2h/0h]xpub.../0/*)#checksum","timestamp":"now","active":true,"internal":false}]'

# Rescan blockchain for transactions
litecoin-cli -conf=/etc/litecoin/litecoin.conf rescanblockchain
```

## Security Best Practices

### 1. Backup Immediately After Setup

```bash
# Right after wallet creation, export descriptors
litecoin-cli -conf=/etc/litecoin/litecoin.conf listdescriptors true > ~/litecoin-wallet-descriptors-$(date +%Y%m%d).json

# Store this file securely (encrypted USB, safe deposit box, etc.)
```

### 2. Restrict Config File Access

The setup script already does this, but verify:

```bash
# Check permissions (should be 600 = owner read/write only)
ls -la /etc/litecoin/pool-wallet.conf
ls -la /etc/litecoin/litecoin.conf

# Fix if needed
sudo chmod 600 /etc/litecoin/pool-wallet.conf
sudo chmod 600 /etc/litecoin/litecoin.conf
sudo chown litecoin:litecoin /etc/litecoin/*.conf
```

### 3. Wallet Encryption

Encrypt your wallet with a passphrase:

```bash
# Encrypt wallet (you'll be prompted for passphrase)
litecoin-cli -conf=/etc/litecoin/litecoin.conf encryptwallet "your-strong-passphrase"

# Note: This requires a daemon restart
# After encryption, you must unlock to send:
litecoin-cli -conf=/etc/litecoin/litecoin.conf walletpassphrase "your-passphrase" 60
```

### 4. Separate Withdrawal Wallet

For larger operations, consider:
1. Pool wallet receives rewards
2. Periodically transfer to a "cold" wallet you control elsewhere
3. Keep only operational funds in the pool wallet

```bash
# Automated sweep to cold storage (example cron job)
# 0 0 * * 0 /usr/local/bin/litecoin-cli -conf=/etc/litecoin/litecoin.conf sendtoaddress "ltc1q_cold_wallet" $(litecoin-cli -conf=/etc/litecoin/litecoin.conf getbalance) "" "" true
```

## Wallet Commands Reference

| Command | Description |
|---------|-------------|
| `getbalance` | Show confirmed balance |
| `getunconfirmedbalance` | Show unconfirmed balance |
| `getnewaddress` | Generate new receiving address |
| `listreceivedbyaddress` | List addresses with received amounts |
| `listtransactions` | Show transaction history |
| `sendtoaddress ADDR AMT` | Send LTC to address |
| `sendmany "" {addrs}` | Send to multiple addresses |
| `sendall [addrs]` | Send entire balance |
| `backupwallet FILE` | Backup wallet to file |
| `listdescriptors` | Show wallet descriptors |
| `listwallets` | Show loaded wallets |
| `loadwallet NAME` | Load a wallet |
| `unloadwallet NAME` | Unload a wallet |
| `encryptwallet PASS` | Encrypt wallet |
| `walletpassphrase PASS SEC` | Unlock wallet for SEC seconds |
| `walletlock` | Lock encrypted wallet |

## Address Types

Litecoin supports multiple address formats:

| Format | Prefix | Example | Command |
|--------|--------|---------|---------|
| Legacy | L or M | `LcR3E...` | `getnewaddress "" "legacy"` |
| P2SH-SegWit | M | `MSc2q...` | `getnewaddress "" "p2sh-segwit"` |
| Bech32 | ltc1q | `ltc1qxy...` | `getnewaddress "" "bech32"` |
| Bech32m | ltc1p | `ltc1pxy...` | `getnewaddress "" "bech32m"` |

**Recommendation:** Use Bech32 (ltc1q) for lowest fees and best compatibility with modern wallets.

## Troubleshooting

### "Wallet not found" Error

```bash
# List available wallets
litecoin-cli -conf=/etc/litecoin/litecoin.conf listwallets

# Load wallet if not loaded
litecoin-cli -conf=/etc/litecoin/litecoin.conf loadwallet "pool-wallet"
```

### "Wallet is not connected to daemon"

Your node isn't synced or isn't running:

```bash
# Check node status
sudo systemctl status litecoind

# Check sync progress
litecoin-cli -conf=/etc/litecoin/litecoin.conf getblockchaininfo | jq '.blocks, .headers'
```

### "Error opening wallet"

Possible causes:
- Wallet is already open in another process
- Corrupted wallet file
- Wrong wallet path

```bash
# Check for multiple processes
pgrep -a litecoind

# Verify wallet exists
ls -la /var/lib/litecoin/wallets/
```

### Wallet Shows 0 Balance After Sync

If your node just synced, transactions may not be visible yet:

```bash
# Rescan blockchain for wallet transactions
litecoin-cli -conf=/etc/litecoin/litecoin.conf rescanblockchain

# Or rescan from specific height
litecoin-cli -conf=/etc/litecoin/litecoin.conf rescanblockchain 2000000
```

### Transaction Stuck/Unconfirmed

```bash
# Check transaction status
litecoin-cli -conf=/etc/litecoin/litecoin.conf gettransaction "txid"

# Bump fee (if RBF was enabled)
litecoin-cli -conf=/etc/litecoin/litecoin.conf bumpfee "txid"

# Or abandon and retry
litecoin-cli -conf=/etc/litecoin/litecoin.conf abandontransaction "txid"
```

## Mining Pool Integration

### Wallet Address for Pool Software

```bash
# Get the primary receiving address
PRIMARY_ADDR=$(litecoin-cli -conf=/etc/litecoin/litecoin.conf getnewaddress "pool-rewards" "bech32")
echo "Pool Rewards Address: $PRIMARY_ADDR"

# Add to pool software configuration
```

### Checking Block Rewards

```bash
# List coinbase (mining) transactions
litecoin-cli -conf=/etc/litecoin/litecoin.conf listtransactions "*" 100 | jq '[.[] | select(.category == "generate" or .category == "immature")]'
```

### Pool Fee Revenue Estimation

| Pool Hashrate | Est. Blocks/Month | 1% Fee Income |
|---------------|-------------------|---------------|
| 100 GH/s | ~0.5 | ~0.06 LTC |
| 1 TH/s | ~5 | ~0.6 LTC |
| 10 TH/s | ~50 | ~6 LTC |
| 100 TH/s | ~500 | ~60 LTC |

*Varies significantly with network difficulty and luck*

## Next Steps

Once your node is synced:
1. Verify wallet address matches pool software config
2. Test a small transaction
3. Set up wallet backup automation
4. Configure pool software payout settings
5. Monitor wallet balance for block rewards

