# Issue #003: Missing Cleanup on Signal/Interruption

## Severity: MEDIUM (Reliability)

## Description
The script does not trap signals (SIGINT, SIGTERM, SIGHUP) to ensure proper cleanup of traffic control configuration when interrupted. If a user presses Ctrl+C or the process is killed during setup, the system can be left in an inconsistent state with partial qdiscs/filters applied.

## Affected Code
- Lines 292-293: `state_lock` / `trap 'release_state_lock' RETURN` in `set_download_limit()`
- Lines 321-322: Same pattern in `set_upload_limit()`
- No signal traps anywhere in the script

## Problem Scenarios

### 1. Ctrl+C During Setup
```bash
intlimiter --down --mbit 50
# User presses Ctrl+C after IFB is created but before tc class is added
```
**Result**: IFB device `ifb0` remains up, ingress qdisc on interface remains, but no rate limit class exists. Traffic may be redirected to IFB but not limited.

### 2. SIGTERM During Setup
```bash
# System shutdown or kill command
kill <pid>
```
**Result**: Same as above - partial configuration left behind.

### 3. Crash in Middle of teardown_interface()
If `teardown_interface` fails partway through (e.g., ingress removal succeeds but root qdisc removal fails), the error is only logged as debug but the function returns success.

## Current Trap (Lines 292-293, 321-322)
```bash
state_lock
trap 'release_state_lock' RETURN
```
This only releases the *state lock* on function return. It does NOT:
- Clean up IFB device
- Remove qdiscs/filters
- Handle signals (SIGINT, SIGTERM, SIGHUP)

## Impact
- Zombie IFB devices (`ifb0`) left up
- Orphaned qdiscs and filters consuming kernel resources
- Network connectivity issues (traffic redirected to non-functional IFB)
- Inability to reapply limits until manual cleanup
- State file may exist but actual tc config doesn't match

## Recommended Fix

### Add Comprehensive Signal Handling
```bash
# At script top, after variable declarations
cleanup_on_exit() {
    local exit_code=$?
    debug "Cleanup on exit (code: $exit_code)"
    # Only clean up if we were in the middle of setting up
    # (This is tricky - need a flag to track setup progress)
    release_state_lock
    # Note: We can't blindly tear down because another process might be using the interface
}
trap cleanup_on_exit EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
trap 'exit 129' HUP
```

### Better: Track Setup Phase and Clean Up on Interruption
```bash
SETUP_PHASE=""

set_download_limit() {
    ...
    SETUP_PHASE="ifb_created"
    setup_ifb "$IFB_DEV"

    SETUP_PHASE="interface_setup"
    setup_interface "$iface" "down"

    SETUP_PHASE="ifb_qdisc"
    sudo tc qdisc del dev "$IFB_DEV" root 2>/dev/null || debug "IFB root qdisc removal warning"
    sudo tc qdisc add dev "$IFB_DEV" root handle 1: htb || die "Failed to add qdisc on $IFB_DEV"

    SETUP_PHASE="ifb_class"
    sudo tc class add dev "$IFB_DEV" parent 1: classid 1:1 htb rate "${rate_mbit}mbit" ceil "${rate_mbit}mbit" || \
        sudo tc class replace dev "$IFB_DEV" parent 1: classid 1:1 htb rate "${rate_mbit}mbit" ceil "${rate_mbit}mbit" || \
        die "Failed to set rate limit on IFB"

    SETUP_PHASE="ifb_filter"
    sudo tc filter add dev "$IFB_DEV" parent 1: protocol ip u32 match u32 0 0 flowid 1:1 || debug "IFB filter add warning, may already exist"

    SETUP_PHASE="state_saved"
    save_state "$iface" "down" "$speed" "$unit" "$rate_mbit"
    SETUP_PHASE=""
    ...
}

cleanup_on_signal() {
    local sig=$1
    debug "Caught signal $sig during phase: $SETUP_PHASE"

    case "$SETUP_PHASE" in
        "ifb_class"|"ifb_filter"|"ifb_qdisc"|"interface_setup")
            # Partial IFB setup - tear down IFB
            teardown_ifb "$IFB_DEV"
            teardown_interface "$INTERFACE"
            ;;
        "state_saved")
            # State saved but we're exiting - this is actually OK, state matches
            ;;
        *)
            # Before IFB creation or after completion - minimal cleanup
            ;;
    esac

    release_state_lock
    exit 128
}

trap 'cleanup_on_signal INT' INT
trap 'cleanup_on_signal TERM' TERM
trap 'cleanup_on_signal HUP' HUP
```

### Ensure teardown_functions Return Proper Errors
```bash
teardown_interface() {
    local iface=$1
    local errors=()

    sudo tc qdisc del dev "$iface" ingress 2>/dev/null || errors+=("ingress_qdisc_removal_failed")
    sudo tc qdisc del dev "$iface" root 2>/dev/null || errors+=("root_qdisc_removal_failed")

    if [[ ${#errors[@]} -gt 0 ]]; then
        debug "teardown warnings on $iface: ${errors[*]}"
        return 1
    fi
    return 0
}
```

## References
- CWE-459: Incomplete Cleanup
- Signal handling best practices for system administration scripts