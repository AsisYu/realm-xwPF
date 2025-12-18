# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**realm-xwPF** is a comprehensive Bash-based network forwarding management system built around [Realm](https://github.com/zhboner/realm). It provides a complete solution for managing TCP/UDP port forwarding with advanced features like load balancing, failover, traffic monitoring, and MPTCP support.

The project consists of multiple shell scripts that work together:
- **xwPF.sh** - Main management script for Realm forwarding rules
- **port-traffic-dog.sh** - Port traffic monitoring and rate limiting
- **speedtest.sh** - Network link testing and diagnostics
- **xwFailover.sh** - Automatic failover management
- **xw_realm_OCR.sh** - Import existing Realm configurations

## Core Architecture

### File Structure After Installation

```
/usr/local/bin/
├── realm                    # Realm binary (v2.9.2)
├── xwPF.sh                  # Main script
├── port-traffic-dog.sh      # Traffic monitoring
├── pf                       # Shortcut: xwPF.sh
└── dog                      # Shortcut: port-traffic-dog.sh

/etc/realm/
├── manager.conf             # State management (core persistence)
├── config.json              # Generated Realm config (DO NOT edit manually)
├── rules/                   # Individual rule files
│   ├── rule-1.conf          # Rule configuration files
│   └── rule-*.conf          # Each contains forwarding parameters
├── health/                  # Failover health checks
│   └── health_status.conf   # Target health states
├── speedtest.sh             # Network testing (downloaded on demand)
└── xwFailover.sh            # Failover script (downloaded on demand)

/etc/port-traffic-dog/
├── config.json              # Port monitoring config
├── data/                    # Traffic statistics
├── snapshots/               # Daily snapshots
├── notifications/           # Telegram/WeChat webhook modules
│   ├── telegram.sh
│   └── wecom.sh
└── logs/                    # Notification logs

/etc/systemd/system/
├── realm.service            # Main Realm service
├── realm-health-check.service
└── realm-health-check.timer # Runs every 4 seconds
```

### Critical Design Patterns

1. **Rule-Based Configuration System**
   - Each forwarding rule is stored as `/etc/realm/rules/rule-{id}.conf`
   - Rules contain shell variables: `RULE_ID`, `RULE_NAME`, `LISTEN_PORT`, `REMOTE_HOST`, `REMOTE_PORT`, etc.
   - The main script reads all rule files and generates `/etc/realm/config.json` dynamically
   - **NEVER edit config.json manually** - always modify rule files and regenerate

2. **Configuration Generation Flow**
   ```
   rule-*.conf files → read_rule_file() → generate_realm_config() → config.json → systemd restart
   ```

3. **State Persistence**
   - `manager.conf` stores service state and global settings
   - Rule files store individual forwarding configurations
   - Health status files track failover states

4. **Failover Mechanism**
   - Systemd timer triggers health checks every 4 seconds
   - Uses `nc -z` (netcat) for TCP connectivity tests
   - Tracks consecutive failures/successes (threshold: 2)
   - 120-second cooldown period prevents flapping
   - Updates trigger config regeneration via marker files

## Common Development Commands

### Running the Scripts

```bash
# Main script (must run as root)
sudo bash xwPF.sh

# Quick shortcuts after installation
pf                           # Main menu
dog                          # Port traffic dog menu

# One-line installation
wget -qO- https://raw.githubusercontent.com/AsisYu/realm-xwPF/main/xwPF.sh | sudo bash -s install
```

### Service Management

```bash
# Restart Realm service
systemctl restart realm

# View real-time logs
journalctl -u realm -f

# Check failover timer
systemctl status realm-health-check.timer

# Manually trigger failover check
systemctl start realm-health-check.service
```

### Testing and Debugging

```bash
# Generate config without restarting
bash xwPF.sh --generate-config-only

# Restart service programmatically
bash xwPF.sh --restart-service

# Test network link (requires iperf3, hping3, nexttrace)
bash speedtest.sh

# Check traffic statistics
nft list table inet port_traffic_dog
```

### Working with Rule Files

```bash
# List all rules
ls -l /etc/realm/rules/

# Read a rule file
cat /etc/realm/rules/rule-1.conf

# Rule file format example:
RULE_ID="1"
RULE_NAME="Web Server Forward"
LISTEN_PORT="8080"
LISTEN_IP="0.0.0.0"
THROUGH_IP="::"
REMOTE_HOST="192.168.1.100"
REMOTE_PORT="80"
RULE_ROLE="1"                # 1=relay, 2=exit
SECURITY_LEVEL="none"        # none/tls/ws/wss
BALANCE_MODE="off"           # off/roundrobin/iphash
WEIGHTS="1"
ENABLE_FAILOVER="false"
```

## Key Script Functions

### xwPF.sh Main Functions

- `generate_realm_config()` - Converts rule files to config.json
- `read_rule_file()` - Sources a rule file and loads variables
- `service_restart()` - Regenerates config and restarts systemd service
- `create_forward_rule()` - Interactive wizard for new rules
- `rules_management_menu()` - Main rule management interface
- `smart_install()` - Installation with dependency detection

### Load Balancing & Failover

- Multiple rules can share the same `LISTEN_PORT` for load balancing
- `BALANCE_MODE`: `roundrobin` (rotating), `iphash` (sticky sessions)
- `WEIGHTS`: Comma-separated integers (e.g., "2,1,3" for 3 targets)
- Failover automatically removes unhealthy targets from rotation

### Port Traffic Dog Features

- Uses `nftables` for packet counting
- Uses `tc` (traffic control) for rate limiting
- Supports monthly quotas with auto-reset
- Telegram/WeChat webhook notifications
- Persistent data across reboots

## Important Implementation Notes

### When Modifying Scripts

1. **Preserve Compatibility**: These scripts run on various Linux distributions (primarily Debian/Ubuntu)
2. **Lightweight Dependencies**: Prefer built-in tools over external dependencies
3. **Atomic Operations**: Use temp files + mv for config updates to prevent corruption
4. **Error Handling**: Always check command success and provide clear error messages
5. **Color Output**: Use defined color variables (`$RED`, `$GREEN`, etc.)

### Security Considerations

1. **Root Required**: Most operations require root privileges (systemd, nftables, tc)
2. **Input Validation**: Port numbers, IP addresses, file paths must be validated
3. **TLS Certificates**: When `SECURITY_LEVEL=tls`, cert/key paths must be verified
4. **Default SNI**: Uses `www.tesla.com` as default TLS server name

### Multi-Architecture Support

The scripts detect and download the correct Realm binary:
- x86_64 (most servers)
- aarch64 (ARM64)
- armv7 (ARM32, e.g., Raspberry Pi)

### Configuration File Safety

- **Never parse or modify config.json directly** - it's regenerated from rule files
- Always use `read_rule_file()` to load rule configurations
- Use `generate_realm_config()` after any rule changes
- The script uses `jq` for JSON manipulation - ensure it's installed

## Network Scenarios Explained

### Single-Realm Architecture (Common)
Relay server runs Realm, exit server runs business application. Realm transparently forwards packets.

### Dual-Realm Architecture (Tunnel)
Both relay and exit run Realm. Adds encryption layer (TLS/WS/WSS) between Realm instances. **Both sides must use matching encryption settings**.

### Load Balancing + Failover
- Same port can forward to multiple targets
- `roundrobin`: Cycles through targets
- `iphash`: Consistent hashing by source IP
- Failover removes failed targets automatically

### MPTCP Configuration
- Uses `ip mptcp endpoint` commands to configure subflows
- Requires kernel support (Linux 5.6+)
- Both relay and exit must enable MPTCP in Realm config
- Script provides visual interface for endpoint management

## Download Strategy

The scripts implement multi-source downloads with automatic fallback:
```bash
DOWNLOAD_SOURCES=(
    ""                          # Direct GitHub
    "https://ghfast.top/"       # Mirror 1
    "https://free.cn.eu.org/"   # Mirror 2
    "https://ghproxy.net/"      # Mirror 3
)
```

Uses two timeout strategies:
- Short (5s connect, 7s total) - Fast failure for checks
- Long (15s connect, 20s total) - Important operations

## Testing Workflow

When adding new features:

1. Test installation on clean Debian/Ubuntu VM
2. Verify service starts: `systemctl status realm`
3. Test rule creation and config generation
4. Check log output: `journalctl -u realm`
5. Test service restart and persistence across reboots
6. Validate cleanup: Run uninstall and verify file removal

## Common Issues and Solutions

### Port Conflicts
- Script detects conflicts using `ss -tunlp`
- Uses `netstat` as fallback if `ss` unavailable
- Supports port reuse when same service binds multiple targets

### Config Regeneration Failures
- Check for syntax errors in rule files
- Verify `jq` is installed
- Ensure `/etc/realm/rules/` directory exists
- Check file permissions (must be readable by root)

### Failover Not Working
- Verify timer is active: `systemctl status realm-health-check.timer`
- Check connectivity tests: `nc -z <target> <port>`
- Review health status: `cat /etc/realm/health/health_status.conf`
- Ensure `inotify-tools` is installed for marker file detection

### Traffic Monitoring Issues
- Verify `nftables` is installed and loaded: `nft list tables`
- Check if table exists: `nft list table inet port_traffic_dog`
- Ensure `tc` qdisc is configured: `tc qdisc show`
- Review logs: `journalctl -u port-traffic-dog`
