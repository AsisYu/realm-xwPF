# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Core Instruction for CodeX MCP

在任何时刻，你必须思考当前过程可以如何与codex进行协作，如何调用Codex 为你提供的MCP工具作为你客观全面分析的保障。
其中你**务必执行**以下几个步骤：
**1** 在你对用户需求形成初步分析后，将用户需求、初始思路告知codex，并要求其完善需求分析和实施计划。
**2** 在实施具体编码任务前，**必须向codex索要代码实现原型（要求codex仅给出unified diff patch，严禁对代码做任何真实修改）**。在获取代码原型后，你**只能以此为逻辑参考，再次对代码修改进行重写**，形成企业生产级别、可读性极高、可维护性极高的代码后，才能实施具体编程修改任务。
**3** 无论何时，只要完成切实编码行为后，**必须立即使用codex review代码改动和对应需求完成程度**。
**4** codex只能给出参考，你**必须有自己的思考，甚至需要对codex的回答提出置疑**。尽信书则不如无书，你与codex的最终使命都是达成统一、全面、精准的意见，所以你们必须不断争辩已找到通向真理的唯一途径。


## Codex Tool Invocation Specification

 1. 工具概述

  codex MCP 提供了一个工具 `codex`，用于执行 AI 辅助的编码任务。该工具**通过 MCP 协议调用**，无需使用命令行。

  2. 工具参数

  **必选**参数：
  - PROMPT (string): 发送给 codex 的任务指令
  - cd (Path): codex 执行任务的工作目录根路径

  可选参数：
  - sandbox (string): 沙箱策略，可选值：
    - "read-only" (默认): 只读模式，最安全
    - "workspace-write": 允许在工作区写入
    - "danger-full-access": 完全访问权限
  - SESSION_ID (UUID | null): 用于继续之前的会话以与codex进行多轮交互，默认为 None（开启新会话）
  - skip_git_repo_check (boolean): 是否允许在非 Git 仓库中运行，默认 False
  - return_all_messages (boolean): 是否返回所有消息（包括推理、工具调用等），默认 False
  - image (List[Path] | null): 附加一个或多个图片文件到初始提示词，默认为 None
  - model (string | null): 指定使用的模型，默认为 None（使用用户默认配置）
  - yolo (boolean | null): 无需审批运行所有命令（跳过沙箱），默认 False
  - profile (string | null): 从 `~/.codex/config.toml` 加载的配置文件名称，默认为 None（使用用户默认配置）

  返回值：
  {
    "success": true,
    "SESSION_ID": "uuid-string",
    "agent_messages": "agent回复的文本内容",
    "all_messages": []  // 仅当 return_all_messages=True 时包含
  }
  或失败时：
  {
    "success": false,
    "error": "错误信息"
  }

  3. 使用方式

  开启新对话：
  - 不传 SESSION_ID 参数（或传 None）
  - 工具会返回新的 SESSION_ID 用于后续对话

  继续之前的对话：
  - 将之前返回的 SESSION_ID 作为参数传入
  - 同一会话的上下文会被保留

  4. 调用规范

  **必须遵守**：
  - 每次调用 codex 工具时，必须保存返回的 SESSION_ID，以便后续继续对话
  - cd 参数必须指向存在的目录，否则工具会静默失败
  - 严禁codex对代码进行实际修改，使用 sandbox="read-only" 以避免意外，并要求codex仅给出unified diff patch即可

  推荐用法：
  - 如需详细追踪 codex 的推理过程和工具调用，设置 return_all_messages=True
  - 对于精准定位、debug、代码原型快速编写等任务，优先使用 codex 工具

  5. 注意事项

  - 会话管理：始终追踪 SESSION_ID，避免会话混乱
  - 工作目录：确保 cd 参数指向正确且存在的目录
  - 错误处理：检查返回值的 success 字段，处理可能的错误

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
