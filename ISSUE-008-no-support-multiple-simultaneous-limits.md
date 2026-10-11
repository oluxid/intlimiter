# Issue #008: No Support for Multiple Simultaneous Limits

## Severity: MEDIUM (Feature/Usability)

## Description
The script only supports ONE active limit at a time (either download OR upload on ONE interface). The state file stores a single limit, and applying a new limit tears down the previous one completely.

## Affected Code
- Lines 192-200: `ensure_clean_state()` tears down ALL existing config before applying new
- Lines 113-128: `save_state()` overwrites single state file
- Lines 130-144: `load_and_check_direction()` only loads single state
- Lines 422-446: Main logic only handles one direction/speed at a time

## Current Behavior
```bash
# Set download limit
intlimiter --down --mbit 50

# Try to also set upload limit
intlimiter --up --mbit 10
# Result: Download limit is REMOVED, only upload limit applied
```

## Problems

### 1. Cannot Limit Both Directions Simultaneously
Users commonly want to limit both upload and download at the same time (e.g., for a server or shared connection).

### 2. Cannot Limit Multiple Interfaces
```bash
intlimiter --down --mbit 50 --interface eth0
intlimiter --down --mbit 100 --interface wlan0
# Second command tears down eth0's limit
```

### 3. State File Design Prevents Multi-Limit
```bash
# State file format (single entry)
INTERFACE=eth0
DIRECTION=down
SPEED=50
UNIT=mbit
RATE_MBIT=50
IFB_DEV=ifb0
```

## Impact
- Major usability limitation
- Forces users to run multiple instances or use other tools
- Inconsistent with typical traffic shaping needs

## Recommended Fix

### Option A: Multi-State File (Per Interface + Direction)
```bash
# State directory instead of file
STATE_DIR="/var/run/intlimiter"

save_state() {
    local iface=$1
    local direction=$2
    local speed=$3
    local unit=$4
    local rate_mbit=$5

    mkdir -p "$STATE_DIR"
    local state_file="$STATE_DIR/${iface}_${direction}.state"

    cat > "$state_file" <<STATE_EOF
INTERFACE=$iface
DIRECTION=$direction
SPEED=$speed
UNIT=$unit
RATE_MBIT=$rate_mbit
IFB_DEV=$IFB_DEV
STATE_EOF
}

load_state_for() {
    local iface=$1
    local direction=$2
    local state_file="$STATE_DIR/${iface}_${direction}.state"

    if [[ ! -f "$state_file" ]]; then
        return 1
    fi

    set -a
    source "$state_file"
    set +a

    [[ -n "${DIRECTION:-}" && -n "${INTERFACE:-}" ]]
}

clear_state_for() {
    local iface=$1
    local direction=$2
    rm -f "$STATE_DIR/${iface}_${direction}.state" 2>/dev/null || true
}

# List all active limits
list_active_limits() {
    for f in "$STATE_DIR"/*.state; do
        [[ -f "$f" ]] || continue
        set -a
        source "$f"
        set +a
        echo "$INTERFACE $DIRECTION $SPEED $UNIT"
    done
}
```

### Option B: Extended State File (Array-Based)
```bash
# State file with multiple entries
LIMITS=(
    "eth0|down|50|mbit|50|ifb0"
    "eth0|up|10|mbit|10|"
    "wlan0|down|100|mbit|100|ifb1"
)

save_state() {
    local iface=$1
    local direction=$2
    local speed=$3
    local unit=$4
    local rate_mbit=$5

    local entry="${iface}|${direction}|${speed}|${unit}|${rate_mbit}|${IFB_DEV}"
    # Update or add entry in state file
    ...
}
```

### Update ensure_clean_state() to Only Clean Specific Interface+Direction
```bash
ensure_clean_state() {
    local iface=$1
    local direction=$2

    if ! load_state_for "$iface" "$direction"; then
        return 0
    fi

    debug "Found existing state for $iface $direction: ifb=$IFB_DEV"
    teardown_ifb "$IFB_DEV"
    teardown_interface "$iface"
}
```

### Update Main Logic to Support Multiple Operations
```bash
# Allow multiple --down/--up in single invocation
# Or support applying additional limits without removing existing

# New usage:
# intlimiter --down --mbit 50 --up --mbit 10
# intlimiter --down --mbit 50 --interface eth0 --down --mbit 100 --interface wlan0
```

### Backward Compatibility
Keep `--clear` working for all limits, add `--clear --interface eth0 --down` for specific.

## References
- Traffic control multi-qdisc configurations
- Linux tc filter priorities for multiple rules