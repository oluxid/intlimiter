# Issue #009: Inconsistent Error Handling and Debug Output

## Severity: LOW (Maintainability/Usability)

## Description
The script has inconsistent error handling patterns and debug output that makes troubleshooting difficult.

## Problems

### 1. Mixed Error Output Streams
- `die()` writes to stderr (line 47): `echo "Error: $*" >&2`
- `log_error()` writes to stderr (line 43): `echo "Error: $*" >&2`
- `debug()` writes to stderr (line 39): `echo "[DEBUG] $*" >&2`
- But some `sudo` commands let stderr through directly
- `tc` command failures show raw kernel errors to stderr

### 2. Inconsistent Error Messages
```bash
# Line 48: die() - "Error: ..."
# Line 62: die() - "Missing required commands: ..."
# Line 88: die() - "Speed must be a positive number"
# Line 94: die() - "Interface '$iface' does not exist"
# Line 169: debug() - "IFB teardown warnings: $err"
# Line 188: debug() - "teardown warnings on $iface: $err"
# Line 269: debug() - "root qdisc removal warning on $iface during setup"
# Line 276: debug() - "ingress qdisc add warning on $iface during setup"
# Line 304: die() - "Failed to set rate limit on IFB"
# Line 329: die() - "Failed to set rate limit on $iface"
```

Some errors say "Error:", some don't. Some include context, some don't.

### 3. Silent Failures in teardown
```bash
teardown_ifb() {
    ...
    if ! sudo ip link set "$ifb_dev" down 2>/dev/null; then
        err="ip_link_down_failed"
    fi
    if ! sudo ip link del "$ifb_dev" type ifb 2>/dev/null; then
        err="${err:+${err};}ip_link_del_failed"
    fi
    if [[ -n "$err" ]]; then
        debug "IFB teardown warnings: $err"  # Only shown in DEBUG mode!
    fi
}
```
Teardown failures are only shown with `--debug`. Users won't know cleanup failed.

### 4. No Structured Error Codes
All failures exit with code 1. No distinction between:
- Invalid arguments (2)
- Missing dependencies (3)
- Permission denied (4)
- Kernel/tc errors (5)
- State/lock errors (6)

### 5. Debug Output Inconsistent
```bash
debug() {
    [[ "$DEBUG" -eq 1 ]] && echo "[DEBUG] $*" >&2
}
```
Some debug messages are helpful, others are cryptic:
- "IFB filter add warning, may already exist" - what does this mean?
- "root qdisc removal warning on $iface during setup" - normal or error?

## Recommended Fix

### Standardize Error Reporting
```bash
# Error codes
readonly E_ARGS=2
readonly E_DEPS=3
readonly E_PERM=4
readonly E_TC=5
readonly E_STATE=6
readonly E_GENERAL=1

die() {
    local code=${2:-$E_GENERAL}
    echo "Error: $*" >&2
    exit "$code"
}

error() {
    echo "Error: $*" >&2
}

warn() {
    echo "Warning: $*" >&2
}

info() {
    echo "$*"
}
```

### Always Show Teardown Failures (Not Just Debug)
```bash
teardown_ifb() {
    local ifb_dev=$1
    local errors=()

    debug "Tearing down IFB: $ifb_dev"
    if [[ -n "$ifb_dev" ]]; then
        if ! ip link show "$ifb_dev" >/dev/null 2>&1; then
            debug "IFB $ifb_dev not present, skip teardown"
            return 0
        fi

        if ! sudo ip link set "$ifb_dev" down 2>/dev/null; then
            errors+=("Failed to bring down $ifb_dev")
        fi

        if ! sudo ip link del "$ifb_dev" type ifb 2>/dev/null; then
            errors+=("Failed to delete $ifb_dev")
        fi

        if [[ ${#errors[@]} -gt 0 ]]; then
            warn "IFB teardown failed: ${errors[*]}"
            return 1
        fi
    fi
    return 0
}
```

### Improve Debug Messages
```bash
debug() {
    [[ "$DEBUG" -eq 1 ]] && echo "[DEBUG $(date '+%H:%M:%S')] $*" >&2
}

# More descriptive messages:
debug "Adding HTB qdisc on $iface (handle 1:)"
debug "Creating IFB device $IFB_DEV for ingress redirect"
debug "Adding u32 filter to redirect all IPv4 traffic to $IFB_DEV"
debug "Configuring HTB class 1:1 on $IFB_DEV with rate ${rate_mbit}mbit"
```

### Capture and Contextualize tc Errors
```bash
run_tc() {
    local output
    output=$(sudo tc "$@" 2>&1)
    local rc=$?
    if [[ $rc -ne 0 ]]; then
        debug "tc command failed: tc $*"
        debug "tc output: $output"
        case "$output" in
            *"Operation not supported"*)
                die "Kernel module missing for: tc $*" $E_TC
                ;;
            *"Permission denied"*)
                die "Permission denied for: tc $*. Run as root or with CAP_NET_ADMIN." $E_PERM
                ;;
            *"File exists"*)
                debug "tc object already exists (tc $*)"
                return 0
                ;;
            *)
                die "tc command failed: $output" $E_TC
                ;;
        esac
    fi
    debug "tc command succeeded: tc $*"
}
```

## References
- Linux kernel tc error messages
- Shell script error handling best practices