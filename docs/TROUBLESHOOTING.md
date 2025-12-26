# Troubleshooting Guide

Common issues and solutions for the Litecoin node setup.

## Quick Diagnostics

Run these commands to check your node's health:

```bash
# Is the service running?
sudo systemctl status litecoind

# Check recent logs
sudo journalctl -u litecoind -n 50

# Check sync progress
litecoin-cli -conf=/etc/litecoin/litecoin.conf getblockchaininfo 2>/dev/null | jq '{blocks, headers, verificationprogress, initialblockdownload}'

# Check peer connections
litecoin-cli -conf=/etc/litecoin/litecoin.conf getnetworkinfo 2>/dev/null | jq '{connections, connections_in, connections_out}'
```

### One-Liner Health Check

```bash
echo "=== Service ===" && systemctl is-active litecoind && echo "=== Sync ===" && litecoin-cli -conf=/etc/litecoin/litecoin.conf getblockchaininfo 2>/dev/null | jq -r '"Blocks: \(.blocks)/\(.headers) (\(.verificationprogress * 100 | floor)%)"' && echo "=== Peers ===" && litecoin-cli -conf=/etc/litecoin/litecoin.conf getnetworkinfo 2>/dev/null | jq -r '"Connections: \(.connections) (in: \(.connections_in), out: \(.connections_out))"'
```

## Common Issues

### Node Won't Start

**Symptom:** Service fails to start or keeps restarting

**Check logs:**
```bash
sudo journalctl -u litecoind -n 100
sudo tail -100 /var/log/litecoin/debug.log
```

**Common causes:**

1. **Port already in use:**
   ```bash
   sudo ss -tlnp | grep -E "9333|9332"
   # Kill any rogue processes
   sudo pkill -9 litecoind
   sudo systemctl start litecoind
   ```

2. **Permission issues:**
   ```bash
   sudo chown -R litecoin:litecoin /var/lib/litecoin
   sudo chown -R litecoin:litecoin /var/log/litecoin
   sudo chown -R litecoin:litecoin /etc/litecoin
   sudo chmod 700 /var/lib/litecoin
   sudo chmod 700 /var/lib/litecoin/wallets
   ```

3. **Disk full:**
   ```bash
   df -h /var/lib/litecoin
   # Need ~120GB for full node, ~4GB for pruned
   ```

4. **Corrupted database:**
   ```bash
   sudo systemctl stop litecoind
   # Backup then remove chainstate (will re-verify from blocks)
   sudo mv /var/lib/litecoin/chainstate /var/lib/litecoin/chainstate.bak
   sudo systemctl start litecoind
   
   # Or for a complete resync (slower but more thorough):
   # sudo rm -rf /var/lib/litecoin/blocks /var/lib/litecoin/chainstate
   ```

5. **Configuration error:**
   ```bash
   # Check config syntax
   sudo cat /etc/litecoin/litecoin.conf
   
   # Look for error messages about config
   sudo journalctl -u litecoind | grep -i "error\|invalid\|failed"
   ```

### Build Fails

**Symptom:** Compilation errors during `make`

**Common causes and fixes:**

1. **Out of memory:**
   ```bash
   # Check memory
   free -h
   
   # Add swap space
   sudo fallocate -l 4G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   
   # Make permanent
   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   
   # Reduce parallel jobs in setup script (edit MAKE_JOBS)
   ```

2. **Missing dependencies:**
   ```bash
   # Re-run the dependency installation
   sudo apt update
   sudo apt install -y build-essential libtool autotools-dev automake \
     pkg-config bsdmainutils python3 libssl-dev libevent-dev \
     libboost-system-dev libboost-filesystem-dev libboost-chrono-dev \
     libboost-test-dev libboost-thread-dev libboost-program-options-dev \
     libminiupnpc-dev libnatpmp-dev libzmq3-dev libsqlite3-dev
   ```

3. **Berkeley DB 4.8 issues:**
   ```bash
   # Check if BDB is installed
   ls -la /usr/local/BerkeleyDB.4.8/
   
   # If missing, the setup script will build it
   # If corrupted, remove and re-run setup:
   sudo rm -rf /usr/local/BerkeleyDB.4.8
   sudo ./setup-litecoin.sh
   ```

4. **Autogen/configure failures:**
   ```bash
   # Clean and retry
   cd /usr/local/src/litecoin
   make distclean 2>/dev/null
   ./autogen.sh
   ./configure --prefix=/opt/litecoin \
     BDB_LIBS="-L/usr/local/BerkeleyDB.4.8/lib -ldb_cxx-4.8" \
     BDB_CFLAGS="-I/usr/local/BerkeleyDB.4.8/include" \
     --with-incompatible-bdb --without-gui --disable-tests --disable-bench
   ```

### ZMQ Errors

**Symptom:** `ZMQ: Address already in use` or service won't start with ZMQ enabled

**Solution - Disable ZMQ:**
```bash
# Comment out ZMQ in config
sudo sed -i 's/^zmqpub/#zmqpub/' /etc/litecoin/litecoin.conf
sudo systemctl restart litecoind
```

ZMQ is optional - pool software can use RPC polling instead.

**Solution - Fix ZMQ port conflict:**
```bash
# Check what's using ZMQ ports
sudo ss -tlnp | grep -E "28332|28333"

# Kill conflicting process or change ports in config
sudo nano /etc/litecoin/litecoin.conf
# Change 28332/28333 to different ports
```

**Solution - Verify ZMQ is compiled in:**
```bash
litecoind --help | grep zmq
# Should show zmqpub options
```

### Sync is Extremely Slow

**Symptom:** Syncing at < 100 blocks/second, estimated days to complete

**Causes and solutions:**

1. **Spinning HDD (not SSD):**
   - Litecoin requires high random I/O
   - HDDs are 10-100x slower than SSDs for blockchain sync
   - If stuck with HDD, expect 1-2 weeks for initial sync
   - **Strongly recommend migrating to SSD**

2. **Low peer connections:**
   ```bash
   # Check peer count
   litecoin-cli -conf=/etc/litecoin/litecoin.conf getnetworkinfo | jq '.connections'
   
   # Should be 8+ for good sync speed
   # If < 8, check port 9333 forwarding
   ```

3. **Port 9333 not accessible:**
   ```bash
   # Check UFW
   sudo ufw status | grep 9333
   
   # Check if accepting incoming
   litecoin-cli -conf=/etc/litecoin/litecoin.conf getnetworkinfo | jq '.connections_in'
   # If 0, port forwarding isn't working
   ```

4. **Slow network:**
   ```bash
   # Test network speed
   curl -o /dev/null -w "%{speed_download}" https://speed.hetzner.de/100MB.bin
   ```

5. **Increase database cache:**
   ```bash
   # Edit config
   sudo nano /etc/litecoin/litecoin.conf
   # Add or increase:
   dbcache=1000  # Use more RAM for faster sync
   
   sudo systemctl restart litecoind
   ```

### Can't Connect to RPC

**Symptom:** `error: couldn't connect to server` or `Connection refused`

**Check service is running:**
```bash
sudo systemctl status litecoind
```

**Check RPC binding:**
```bash
grep -E "rpcbind|rpcport|rpcuser" /etc/litecoin/litecoin.conf
# Mining pool mode: rpcbind=127.0.0.1 (localhost only)
```

**Check RPC is listening:**
```bash
sudo ss -tlnp | grep 9332
```

**Check firewall:**
```bash
sudo ufw status
# For remote RPC, port 9332 must be allowed
```

**Test RPC directly:**
```bash
# Use the correct credentials from config
RPC_USER=$(grep rpcuser /etc/litecoin/litecoin.conf | cut -d= -f2)
RPC_PASS=$(grep rpcpassword /etc/litecoin/litecoin.conf | cut -d= -f2)

curl -s --user "$RPC_USER:$RPC_PASS" \
  --data-binary '{"jsonrpc":"1.0","id":"test","method":"getblockchaininfo","params":[]}' \
  -H 'content-type: text/plain;' \
  http://127.0.0.1:9332/
```

### Node Stuck / Not Syncing

**Symptom:** Block height not increasing for extended period

**Check if actually stuck:**
```bash
# Run twice, 60 seconds apart
litecoin-cli -conf=/etc/litecoin/litecoin.conf getblockchaininfo | jq '.blocks'
sleep 60
litecoin-cli -conf=/etc/litecoin/litecoin.conf getblockchaininfo | jq '.blocks'
```

**Force refresh peers:**
```bash
sudo systemctl restart litecoind
```

**Clear peer database:**
```bash
sudo systemctl stop litecoind
sudo rm /var/lib/litecoin/peers.dat
sudo systemctl start litecoind
```

**Add manual peers:**
```bash
# Add to /etc/litecoin/litecoin.conf
addnode=seed-a.litecoin.loshan.co.uk
addnode=dnsseed.thrasher.io
addnode=dnsseed.litecointools.com
addnode=dnsseed.litecoinpool.org
```

**Check for network issues:**
```bash
# Some ISPs/networks block crypto traffic
# Test from a different network or use a VPN
```

### Out of Disk Space

**Check space:**
```bash
df -h /var/lib/litecoin
du -sh /var/lib/litecoin/*
```

**Clean up old logs:**
```bash
sudo truncate -s 0 /var/log/litecoin/debug.log
```

**Switch to pruned mode:**
```bash
sudo systemctl stop litecoind

# Edit config
sudo nano /etc/litecoin/litecoin.conf
# Add: prune=550  (keep only 550MB of blocks)

# Note: Pruned mode is NOT compatible with txindex
# Remove txindex=1 if present

sudo systemctl start litecoind
```

### High Memory Usage

**Symptom:** System becomes unresponsive during sync or operation

Litecoin can use 1-4GB RAM depending on configuration. Solutions:

1. **Reduce database cache:**
   ```bash
   # Edit /etc/litecoin/litecoin.conf
   dbcache=300  # Reduce from 450 default
   ```

2. **Add swap space:**
   ```bash
   sudo fallocate -l 4G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   ```

3. **Reduce max connections:**
   ```bash
   # In litecoin.conf
   maxconnections=32  # Reduce from 256
   ```

### Wallet Issues

**Can't create wallet:**
```bash
# Check if node is synced enough
litecoin-cli -conf=/etc/litecoin/litecoin.conf getblockchaininfo | jq '.initialblockdownload'
# Should be false for wallet operations

# Create wallet manually
litecoin-cli -conf=/etc/litecoin/litecoin.conf createwallet "pool-wallet"
```

**Wallet file locked:**
```bash
# Only one process can access wallet at a time
# Stop any other litecoind or litecoin-cli processes
sudo pkill litecoind
sudo systemctl start litecoind
```

**Lost wallet password:**
- The password is stored in `/etc/litecoin/pool-wallet.conf`
- If file is deleted, wallet cannot be recovered without backup
- Always backup wallet files and credentials!

## Complete Reset

If all else fails, start fresh (keeps blockchain data):

```bash
# Stop service
sudo systemctl stop litecoind
sudo systemctl disable litecoind

# Remove config (not blockchain!)
sudo rm -rf /etc/litecoin
sudo rm -f /etc/systemd/system/litecoind.service
sudo rm -f /usr/local/bin/litecoin*
sudo rm -rf /opt/litecoin

# Remove user
sudo userdel litecoin 2>/dev/null

# Reload systemd
sudo systemctl daemon-reload

# Re-run setup (will reuse existing blockchain in /var/lib/litecoin)
sudo ./setup-litecoin.sh
```

## Full Uninstall

Remove everything including blockchain:

```bash
# Use the uninstall script
sudo ./uninstall-litecoin.sh --force

# Or manual commands:
sudo systemctl stop litecoind
sudo systemctl disable litecoind
sudo rm -rf /opt/litecoin /etc/litecoin /var/log/litecoin /var/lib/litecoin
sudo rm -rf /usr/local/src/litecoin /usr/local/BerkeleyDB.4.8
sudo rm -f /etc/systemd/system/litecoind.service /usr/local/bin/litecoin*
sudo userdel litecoin 2>/dev/null
sudo systemctl daemon-reload
echo "Done - Litecoin completely removed"
```

## Log Analysis

### Finding Errors in Logs

```bash
# Service logs
sudo journalctl -u litecoind | grep -i error | tail -20

# Debug log
sudo grep -i "error\|warning\|fatal" /var/log/litecoin/debug.log | tail -50

# Recent activity
sudo tail -100 /var/log/litecoin/debug.log
```

### Common Log Messages

| Message | Meaning | Action |
|---------|---------|--------|
| `UpdateTip` | New block received | Normal operation |
| `socket recv error` | Network hiccup | Usually recovers automatically |
| `Corrupted block database` | Database corruption | Re-verify or resync |
| `Disk space is low` | Running out of space | Free disk or enable pruning |
| `connect() to ... failed` | Can't reach peer | Check network/firewall |

## Performance Tuning

### For Mining Pool Backend

```ini
# /etc/litecoin/litecoin.conf optimizations
dbcache=1000         # More RAM = faster (if available)
maxconnections=256   # Allow more peers
maxuploadtarget=0    # Unlimited upload
par=4                # Parallel verification threads
txindex=1            # Required for pool
```

### For Low-Resource Systems

```ini
# Conservative settings
dbcache=200
maxconnections=32
maxuploadtarget=1000  # Limit to 1GB/day upload
par=1                 # Single-threaded
```

## Getting Help

If you're still stuck:

1. **Check logs carefully:**
   ```bash
   sudo journalctl -u litecoind -n 200 | grep -i error
   ```

2. **Litecoin community:**
   - [Reddit r/Litecoin](https://www.reddit.com/r/litecoin/)
   - [Litecoin Talk](https://litecointalk.io/)
   - [Litecoin Discord](https://discord.gg/litecoin)
   - [Bitcoin Stack Exchange](https://bitcoin.stackexchange.com/) (many answers apply to Litecoin)

3. **Include in your question:**
   - Output of `sudo systemctl status litecoind`
   - Last 50 lines of logs
   - Your config (remove any passwords!)
   - Debian version: `cat /etc/debian_version`
   - Litecoin version: `litecoind --version`
   - Disk space: `df -h /var/lib/litecoin`

