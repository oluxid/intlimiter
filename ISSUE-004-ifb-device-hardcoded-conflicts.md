# Issue #004: IFB Device Name Hardcoded, Causing Conflicts

## Severity: MEDIUM (Usability/Correctness)

## Description
The IFB device name is hardcoded as `ifb0` (line 5: `IFB_DEV="ifb0"`). This causes conflicts when:
1. Multiple users run intlimiter simultaneously
2. Other software uses `ifb0`
3. Multiple interfaces need download limiting simultaneously

## Affected Code
- Line 5: `IFB_DEV="ifb0"`
- Line 126: `IFB_DEV=$IFB_DEV` in state file
- Line 150-172: `teardown_ifb()` uses hardcoded name
- Line 254-263: `setup_ifb()` uses hardcoded name
- Line 297: `setup_ifb "$IFB_DEV"` in `set_download_limit()`
- Line 300-305: tc commands on `$IFB_DEV`

## Problems

### 1. Single IFB Device Per System
Linux supports multiple IFB devices (ifb0, ifb1, ifb2, ...), but intlimiter only uses ifb0. This means:
- Only ONE interface can have download limiting at a time
- If user tries to limit download on eth0, then on wlan0, the second will fail or overwrite the first

### 2. Conflict with Other Tools
Other traffic shaping tools (wondershaper, fireqos, etc.) may also use ifb0.

### 3. State File Doesn't Track Per-Interface IFB
The state file stores `IFB_DEV=ifb0` but doesn't associate it with the interface. If you run:
```bash
intlimiter --down --mbit 50 --interface eth0
intlimiter --down --mbit 100 --interface wlan0
```
The second command will tear down ifb0 (used by eth0) and recreate it for wlan0, breaking eth0's limit.

### 4. No Cleanup of Unused IFB Devices
`teardown_ifb()` only tears down the hardcoded `ifb0`. If somehow ifb1 was created, it would leak.

## Impact
- Cannot limit download on multiple interfaces simultaneously
- Conflicts with other network management tools
- Unpredictable behavior when switching interfaces

## Recommended Fix

### Option A: Dynamic IFB Allocation Per Interface
```bash
# Map interface to IFB device
get_ifb_for_interface() {
    local iface=$1
    # Hash interface name to get consistent IFB number
    local hash=$(echo -n "$iface" | cksum | awk '{print $1}')
    local ifb_num=$((hash % 256))  # ifb0-ifb255
    echo "ifb${ifb_num}"
}

# Or simpler: use interface name suffix
get_ifb_for_interface() {
    local iface=$1
    # Sanitize interface name for IFB device
    local suffix=$(echo "$iface" | sed 's/[^a-zA-Z0-9]//g')
    echo "ifb${suffix}"
}
```

### Option B: Auto-allocate First Available IFB
```bash
allocate_ifb() {
    local ifb_dev=""
    for i in {0..255}; do
        ifb_dev="ifb${i}"
        if ! ip link show "$ifb_dev" >/dev/null 2>&1; then
            echo "$ifb_dev"
            return 0
        fi
    done
    die "No free IFB devices available"
}

setup_ifb() {
    local ifb_dev=$1
    sudo modprobe ifb numifbs=256 2>/dev/null || die "Failed to load ifb module"
    ...
}
```

### Update State File to Track Per-Interface IFB
```bash
# State file format change:
INTERFACE=eth0
DIRECTION=down
SPEED=50
UNIT=mbit
RATE_MBIT=50
IFB_DEV=ifb0  # Specific to this interface
```

### Update teardown to Use State's IFB_DEV
```bash
teardown_ifb() {
    local ifb_dev=$1
    ...
}

ensure_clean_state() {
    if ! load_and_check_direction; then
        return 0
    fi
    debug "Found existing state: interface=$INTERFACE, direction=$DIRECTION, ifb=$IFB_DEV"
    teardown_ifb "$IFB_DEV"  # Use IFB from state, not global
    teardown_interface "$INTERFACE"
}
```

## Additional Consideration: IFB Module Parameter
The `ifb` kernel module supports `numifbs` parameter (default 1). The script should ensure enough IFB devices are available:
```bash
# In setup_ifb or check_dependencies
sudo modprobe ifb numifbs=256 2>/dev/null || true
```

## References
- Linux IFB documentation: https://www.kernel.org/doc/html/latest/networking/ifb.html
- tc mirred action for ingress shaping