# Issue #006: Incomplete Dependency Checking - Missing Kernel Modules

## Severity: MEDIUM (Reliability/Usability)

## Description
The `check_dependencies()` function (lines 51-64) only checks for user-space commands (`ip`, `tc`, `bc`, `sudo`) but does not verify that required kernel modules are available. The script requires:
- `ifb` module (for download limiting)
- `sch_htb` module (Hierarchical Token Bucket qdisc)
- `cls_u32` module (u32 classifier for filters)
- `act_mirred` module (mirred action for ingress redirect)

If these modules are not loaded or not compiled into the kernel, the script fails with cryptic error messages later.

## Affected Code
- Lines 51-64: `check_dependencies()`
- Line 257: `sudo modprobe ifb 2>/dev/null || die "Failed to load ifb module"` (only checks ifb, late in execution)
- Lines 270, 301, 327: HTB qdisc usage (requires sch_htb)
- Lines 278, 305, 330: u32 classifier usage (requires cls_u32)
- Line 278: mirred action usage (requires act_mirred)

## Current check_dependencies()
```bash
check_dependencies() {
    local missing=()
    local cmd

    for cmd in ip tc bc sudo; do
        if ! command -v "$cmd" >/dev/null 2>&1; then
            missing+=("$cmd")
        fi
    done

    if (( ${#missing[@]} > 0 )); then
        die "Missing required commands: ${missing[*]}"
    fi
}
```

## Problems

### 1. Late Failure with Poor Error Messages
```bash
intlimiter --down --mbit 50
# ... runs for a bit ...
# sudo modprobe ifb 2>/dev/null || die "Failed to load ifb module"
# Error: "Failed to load ifb module"
```
User doesn't know WHY it failed (module not compiled, permissions, etc.)

### 2. No Check for HTB/u32/mirred
These are required but never checked. Failures manifest as:
- `tc qdisc add ... htb` → "RTNETLINK answers: Operation not supported"
- `tc filter add ... u32` → "RTNETLINK answers: Operation not supported"
- `tc filter add ... mirred` → "RTNETLINK answers: Operation not supported"

### 3. No Check for Root/Capabilities
The script uses `sudo` but doesn't verify it actually works or that the user has sufficient privileges.

## Impact
- Confusing error messages for users
- Wasted time debugging kernel configuration
- Silent failures if modules load but don't work correctly

## Recommended Fix

### Enhanced Dependency Check
```bash
check_dependencies() {
    local missing_cmds=()
    local missing_modules=()
    local cmd

    # Check user-space commands
    for cmd in ip tc bc sudo; do
        if ! command -v "$cmd" >/dev/null 2>&1; then
            missing_cmds+=("$cmd")
        fi
    done

    # Check kernel modules (try to load or verify loaded)
    local modules=("ifb" "sch_htb" "cls_u32" "act_mirred")
    local mod
    for mod in "${modules[@]}"; do
        if ! lsmod | grep -q "^${mod} "; then
            # Try to load it
            if ! sudo modprobe "$mod" 2>/dev/null; then
                missing_modules+=("$mod")
            fi
        fi
    done

    # Check sudo works
    if ! sudo -n true 2>/dev/null; then
        # Try with password prompt (timeout after 5s)
        if ! timeout 5 sudo -v 2>/dev/null; then
            missing_cmds+=("sudo (requires password or not configured)")
        fi
    fi

    if (( ${#missing_cmds[@]} > 0 )); then
        die "Missing required commands: ${missing_cmds[*]}"
    fi

    if (( ${#missing_modules[@]} > 0 )); then
        die "Missing required kernel modules: ${missing_modules[*]}. Ensure they are compiled in kernel or available as modules."
    fi

    debug "All dependencies satisfied"
}
```

### Better Error Messages on tc Failures
```bash
run_tc() {
    local output
    output=$(sudo tc "$@" 2>&1)
    local rc=$?
    if [[ $rc -ne 0 ]]; then
        case "$output" in
            *"Operation not supported"*)
                die "tc command failed: $output. Required kernel module may be missing (sch_htb, cls_u32, act_mirred)."
                ;;
            *"Permission denied"*)
                die "tc command failed: Permission denied. Run with sudo or check capabilities."
                ;;
            *"File exists"*)
                debug "tc command warning (already exists): $output"
                return 0
                ;;
            *)
                die "tc command failed: $output"
                ;;
        esac
    fi
    echo "$output"
}

# Usage:
run_tc qdisc add dev "$iface" root handle 1: htb
run_tc class add dev "$IFB_DEV" parent 1: classid 1:1 htb rate "${rate_mbit}mbit" ceil "${rate_mbit}mbit"
```

### Verify Capabilities Alternative to sudo
```bash
check_capabilities() {
    # Check if we have CAP_NET_ADMIN (required for tc)
    if command -v capsh >/dev/null 2>&1; then
        if capsh --print | grep -q "cap_net_admin"; then
            debug "Running with CAP_NET_ADMIN capability"
            return 0
        fi
    fi

    # Fall back to sudo check
    if sudo -n true 2>/dev/null; then
        debug "sudo available without password"
        return 0
    fi

    die "Requires root privileges (CAP_NET_ADMIN or sudo)."
}
```

## References
- Linux Traffic Control kernel modules
- NETLINK error codes
- Linux capabilities (CAP_NET_ADMIN)