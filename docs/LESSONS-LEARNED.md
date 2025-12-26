# Lessons Learned: Litecoin Node Installer

This document captures the issues encountered and solutions discovered while building the Litecoin node installer for Debian 13. These lessons are valuable for building similar installers for other cryptocurrencies (Dogecoin, Zcash, etc.).

---

## Table of Contents
1. [Build Issues](#build-issues)
2. [Dependency Issues](#dependency-issues)
3. [Script Development Issues](#script-development-issues)
4. [Runtime Issues](#runtime-issues)
5. [Best Practices Discovered](#best-practices-discovered)

---

## Build Issues

### 1. Berkeley DB 4.8 Mutex Error on Debian 13

**Problem:**
```
configure: WARNING: NO SHARED LATCH IMPLEMENTATION FOUND FOR THIS PLATFORM.
configure: error: Unable to find a mutex implementation
```

**Cause:**
Berkeley DB 4.8 is old software (2010) and its configure script doesn't properly detect mutex implementations on modern Linux kernels/glibc versions in Debian 13.

**Solution:**
Add `--with-mutex=POSIX/pthreads` to the Berkeley DB configure command:
```bash
../dist/configure \
    --enable-cxx \
    --disable-shared \
    --with-pic \
    --prefix="$BDB_PREFIX" \
    --with-mutex=POSIX/pthreads
```

**Why it works:**
Forces Berkeley DB to use POSIX pthread mutexes instead of trying to auto-detect, which is available on all modern Linux systems.

---

### 2. MiniUPnPc API Incompatibility

**Problem:**
```
net.cpp:1640:25: error: too few arguments to function 'int UPNP_GetValidIGD(...)'
```

**Cause:**
Debian 13 ships with a newer version of `libminiupnpc` that changed its API. The `UPNP_GetValidIGD` function now requires 2 additional parameters that Litecoin Core v0.21.4 doesn't provide.

**Solution:**
Disable miniupnpc support entirely by adding `--without-miniupnpc` to Litecoin's configure:
```bash
./configure \
    --without-miniupnpc \
    # ... other flags
```

**Why this is acceptable:**
- MiniUPnPc is for automatic router port forwarding (UPnP)
- Server deployments should use manual port forwarding anyway (more secure)
- The feature is optional and not required for node operation

**Alternative solution (not implemented):**
Patch the Litecoin source code to match the new API, but this is risky and requires maintenance.

---

### 3. Missing libfmt

**Problem:**
```
checking for main in -lfmt... no
configure: error: libfmt missing
```

**Cause:**
Litecoin Core v0.21.4 requires the `fmt` library for string formatting, which isn't installed by default.

**Solution:**
Add `libfmt-dev` to the dependencies:
```bash
apt-get install -y libfmt-dev
```

---

## Dependency Issues

### Complete Dependency List for Debian 13

Through trial and error, we discovered the complete list of dependencies needed:

```bash
apt-get install -y \
    build-essential \
    libtool \
    autotools-dev \
    autoconf \
    automake \
    pkg-config \
    bsdmainutils \
    python3 \
    libevent-dev \
    libboost-dev \
    libboost-system-dev \
    libboost-filesystem-dev \
    libboost-test-dev \
    libboost-thread-dev \
    libboost-chrono-dev \
    libboost-program-options-dev \
    libzmq3-dev \
    libssl-dev \
    libminiupnpc-dev \      # Still install even if we don't use it
    libnatpmp-dev \
    libsqlite3-dev \
    libqrencode-dev \
    libfmt-dev \            # CRITICAL - often missed
    libprotobuf-dev \
    protobuf-compiler \
    git \
    wget \
    curl \
    ufw \
    jq \
    openssl \
    ca-certificates \
    gnupg
```

### Dependency Verification

Always verify critical libraries are present before attempting to build:
```bash
# Check for critical headers
[[ ! -f /usr/include/event2/event.h ]] && echo "Missing libevent"
[[ ! -f /usr/include/boost/version.hpp ]] && echo "Missing libboost"
[[ ! -f /usr/include/zmq.h ]] && echo "Missing libzmq"
[[ ! -f /usr/include/openssl/ssl.h ]] && echo "Missing libssl"
[[ ! -f /usr/include/sqlite3.h ]] && echo "Missing libsqlite3"
[[ ! -f /usr/include/fmt/core.h ]] && echo "Missing libfmt"
```

---

## Script Development Issues

### 1. Silent Crashes with `set -e`

**Problem:**
The script would exit silently with no error message when a command failed.

**Cause:**
Using `set -e` at the top of the script causes immediate exit on any non-zero return code, but doesn't provide any useful feedback to the user.

**Solution:**
Remove `set -e` and handle errors explicitly:
```bash
# BAD - silent exit
set -e
some_command  # If this fails, script exits with no message

# GOOD - explicit handling
if ! some_command; then
    print_error "some_command failed!"
    echo "Last 20 lines of log:"
    tail -20 "$log_file"
    exit 1
fi
```

### 2. Interactive Prompts Not Displaying (Command Substitution Issue)

**Problem:**
When using functions like `prompt_choice` inside command substitution `$(...)`, the visual prompts (boxes, colors, instructions) weren't being displayed to the user.

**Example of broken code:**
```bash
choice=$(prompt_choice "Select option" "Option 1" "Option 2")
# User sees nothing - all echo output is captured by $()
```

**Cause:**
Command substitution `$(...)` captures ALL stdout from the function, including the visual elements meant for the user.

**Solution:**
Redirect all visual output to `/dev/tty` (the terminal) and read input from `/dev/tty`:
```bash
prompt_choice() {
    local prompt="$1"
    shift
    local options=("$@")
    
    # Visual elements go to /dev/tty, not stdout
    echo -e "Select an option:" >/dev/tty
    for i in "${!options[@]}"; do
        echo "  [$((i+1))] ${options[$i]}" >/dev/tty
    done
    
    # Read from /dev/tty, not stdin
    read -r choice </dev/tty
    
    # Only the result goes to stdout (for capture)
    echo "$choice"
}
```

### 3. Build Errors Hidden by Output Suppression

**Problem:**
Build commands like `./configure > /dev/null 2>&1` hide all output, making it impossible to diagnose failures.

**Solution:**
Log output to a file and display relevant portions on failure:
```bash
local log_file="/tmp/build.log"

if ! ./configure > "$log_file" 2>&1; then
    print_error "Configure failed!"
    echo ""
    echo "Last 30 lines of log:"
    tail -30 "$log_file"
    echo ""
    print_info "Full log: $log_file"
    return 1
fi
```

---

## Runtime Issues

### 1. RPC Credentials Not Accessible

**Problem:**
After installation, running `litecoin-cli getblockchaininfo` as a regular user fails:
```
error: Could not locate RPC credentials. No authentication cookie could be found, and RPC password is not set.
```

**Cause:**
The config file `/etc/litecoin/litecoin.conf` is owned by root with restricted permissions (for security - it contains the RPC password).

**Solution:**
Users must run litecoin-cli with sudo:
```bash
sudo litecoin-cli -conf=/etc/litecoin/litecoin.conf getblockchaininfo
```

Or create an alias:
```bash
alias ltc="sudo litecoin-cli -conf=/etc/litecoin/litecoin.conf"
```

### 2. Service Appears Running But Node Not Responding

**Problem:**
`systemctl status litecoind` shows "active (running)" but RPC calls fail.

**Possible Causes:**
1. Node is still starting up (can take 30+ seconds)
2. Config file has errors
3. Port conflicts

**Diagnostic Steps:**
```bash
# Check actual daemon logs
sudo journalctl -u litecoind -f

# Check if process is listening on ports
sudo ss -tlnp | grep litecoin

# Check config syntax (daemon will log errors on start)
sudo cat /etc/litecoin/litecoin.conf
```

---

## Best Practices Discovered

### 1. Always Show Progress and Context

Users get anxious when builds take a long time with no output. Show:
- What step we're on (e.g., "STEP 5/10")
- What's happening ("Compiling Litecoin - this takes 15-60 minutes")
- How many parallel jobs are being used

### 2. Save Build Logs

Always save build output to log files:
- `/tmp/bdb-build.log` - Berkeley DB build
- `/tmp/litecoin-build.log` - Litecoin build

These are invaluable for debugging failed builds.

### 3. Verify After Each Critical Step

Don't assume success - verify:
```bash
# After Berkeley DB install
if [[ ! -f "$BDB_PREFIX/lib/libdb_cxx.a" ]]; then
    print_error "Berkeley DB installation verification failed!"
    return 1
fi

# After Litecoin build
if [[ ! -f "$INSTALL_DIR/bin/litecoind" ]]; then
    print_error "litecoind binary not found after build!"
    return 1
fi
```

### 4. Provide Clear Completion Messages

When done, show:
- Big "COMPLETE" banner (users need visual confirmation)
- RPC credentials (they need to save these)
- Wallet address (if created)
- Port forwarding instructions
- Useful commands to run next

### 5. Make Uninstall Thorough

The uninstall script must remove EVERYTHING:
- Stop processes (multiple methods - systemd, cli, pkill)
- Remove binaries, symlinks
- Remove source code
- Remove Berkeley DB
- Remove config, logs, data
- Remove user/group
- Remove firewall rules
- Remove temp files and build logs
- Clean up .litecoin in home directories
- Update library cache

### 6. Version Management

- Check for script updates from GitHub at startup
- Auto-detect latest cryptocurrency version from GitHub API
- Allow fallback to a known-good version
- Bump version numbers when making changes

### 7. Test the Full Cycle

Always test:
1. Fresh install on clean VM
2. Re-running setup after install (update mode)
3. Uninstall
4. Re-install after uninstall

---

## Applying These Lessons to Other Cryptocurrency Installers

When building installers for Dogecoin, Zcash, etc.:

1. **Research first** - Check the official build docs for that cryptocurrency
2. **Expect Berkeley DB issues** - Most Bitcoin-derived coins need BDB 4.8
3. **Expect library version mismatches** - Especially on newer distros like Debian 13
4. **Test on the target distro** - Don't assume Ubuntu fixes work on Debian
5. **Use this Litecoin installer as a template** - The structure and error handling patterns are battle-tested
6. **Document as you go** - Add to this lessons learned document

---

## Quick Reference: Debian 13 Build Flags

```bash
# Berkeley DB 4.8
../dist/configure \
    --enable-cxx \
    --disable-shared \
    --with-pic \
    --prefix=/usr/local/BerkeleyDB.4.8 \
    --with-mutex=POSIX/pthreads

# Litecoin/Bitcoin-derived coins
./configure \
    LDFLAGS="-L/usr/local/BerkeleyDB.4.8/lib/" \
    CPPFLAGS="-I/usr/local/BerkeleyDB.4.8/include/" \
    --prefix=/opt/litecoin \
    --disable-tests \
    --disable-bench \
    --disable-gui-tests \
    --with-daemon \
    --with-utils \
    --without-gui \
    --without-miniupnpc \
    --enable-zmq
```

---

*Last updated: December 2024*
*Tested on: Debian 13 (Trixie)*
*Litecoin Core: v0.21.4*

