# Issue #002: Race Condition in State File and Lock Mechanism

## Severity: HIGH (Reliability/Correctness)

## Description
The state file (`/var/run/intlimiter.state`) and lock directory (`/var/run/intlimiter.locks/state`) mechanism has multiple race conditions that can lead to:
- Corrupted state
- Stale locks preventing legitimate operations
- Inconsistent traffic control configuration

## Affected Code
- Lines 11-26: `acquire_state_lock()` - uses `mkdir` for locking with retry
- Lines 28-30: `release_state_lock()` - uses `rmdir`
- Lines 113-128: `save_state()` - writes state file with heredoc
- Lines 130-144: `load_and_check_direction()` - reads state file with `source`
- Lines 146-148: `clear_state()` - removes state file
- Lines 192-200: `ensure_clean_state()` - loads state and tears down

## Race Conditions Identified

### 1. TOCTOU in Lock Acquisition (Lines 11-26)
```bash
while (( count < attempts )); do
    if mkdir "$STATE_FILE_LOCK" >/dev/null 2>&1; then
        return 0
    fi
    sleep "$delay"
    count=$(( count + 1 ))
done
```
**Issue**: The check-then-act pattern between `mkdir` attempts is vulnerable. If the lock directory is created by another process between the check and the next attempt, the lock may be acquired by the wrong process. More critically, there's no verification that the lock was acquired by *this* process (no PID file inside the lock dir).

### 2. State File Write is Not Atomic (Lines 113-128)
```bash
cat > "$STATE_FILE" <<STATE_EOF
INTERFACE=$iface
DIRECTION=$direction
SPEED=$speed
UNIT=$unit
RATE_MBIT=$rate_mbit
IFB_DEV=$IFB_DEV
STATE_EOF
```
**Issue**: The heredoc write is not atomic. If the process crashes mid-write, the state file will be corrupted/partial. A concurrent reader (e.g., `--status`) could read partial state.

### 3. State File Read Without Lock (Lines 130-144)
```bash
load_and_check_direction() {
    if [[ ! -f "$STATE_FILE" ]]; then
        return 1
    fi
    set -a
    source "$STATE_FILE"
    set +a
    ...
}
```
**Issue**: `load_and_check_direction()` reads the state file *without* holding the lock. This means it can read state while another process is writing it (partial read) or after the lock is released but before the state is cleaned up.

### 4. Stale Lock on Crash (Lines 28-30)
```bash
release_state_lock() {
    rmdir "$STATE_FILE_LOCK" >/dev/null 2>&1 || true
}
```
**Issue**: If the process crashes after acquiring the lock but before releasing it (e.g., killed by SIGKILL, power loss, OOM killer), the lock directory remains forever. Subsequent runs will fail with "Another intlimiter operation is in progress" until manually cleaned up.

### 5. Lock Not Held During Critical Section in set_download_limit/set_upload_limit
```bash
state_lock
trap 'release_state_lock' RETURN
ensure_clean_state  # This calls load_and_check_direction which reads state WITHOUT lock!
...
save_state  # Writes state
```
**Issue**: `ensure_clean_state()` calls `load_and_check_direction()` which reads state *without* the lock, creating a window where state can be modified by another process.

## Impact
- State corruption leading to incorrect traffic control configuration
- Permanent lock requiring manual intervention (`rm -rf /var/run/intlimiter.locks/state`)
- Inconsistent limits applied (e.g., download limit state says 50mbit but actual tc config is different)
- `--status` showing wrong information

## Recommended Fix

### Option A: Use flock (Recommended)
```bash
# Use a lock file with flock for proper advisory locking
LOCK_FILE="/var/run/intlimiter.lock"
exec 9>"$LOCK_FILE"
flock -n 9 || die "Another intlimiter operation is in progress"

# All state operations happen while lock is held
# Lock automatically released on exit (including crash)
```

### Option B: Improve mkdir-based lock with PID verification
```bash
acquire_state_lock() {
    local lock_dir="$STATE_FILE_LOCK"
    local pid_file="$lock_dir/pid"
    local delay=0.25
    local attempts=20
    local count=0

    while (( count < attempts )); do
        if mkdir "$lock_dir" >/dev/null 2>&1; then
            # We got the lock, write our PID
            echo $$ > "$pid_file"
            # Verify we still own it (defense against rare race)
            if [[ "$(cat "$pid_file" 2>/dev/null)" == "$$" ]]; then
                return 0
            fi
            # Lost race, clean up and retry
            rmdir "$lock_dir" 2>/dev/null || true
        fi
        sleep "$delay"
        count=$(( count + 1 ))
    done
    die "Another intlimiter operation is in progress"
}

release_state_lock() {
    local pid_file="$STATE_FILE_LOCK/pid"
    # Only release if we own it
    if [[ -f "$pid_file" ]] && [[ "$(cat "$pid_file")" == "$$" ]]; then
        rmdir "$STATE_FILE_LOCK" >/dev/null 2>&1 || true
    fi
}
```

### Atomic State Write
```bash
save_state() {
    local tmp_file="${STATE_FILE}.tmp.$$"
    cat > "$tmp_file" <<STATE_EOF
INTERFACE=$iface
DIRECTION=$direction
SPEED=$speed
UNIT=$unit
RATE_MBIT=$rate_mbit
IFB_DEV=$IFB_DEV
STATE_EOF
    mv "$tmp_file" "$STATE_FILE"  # Atomic rename
}
```

### Stale Lock Detection
```bash
acquire_state_lock() {
    ...
    # Check for stale lock (process dead)
    if [[ -d "$lock_dir" ]] && [[ -f "$pid_file" ]]; then
        local other_pid=$(cat "$pid_file" 2>/dev/null)
        if [[ -n "$other_pid" ]] && ! kill -0 "$other_pid" 2>/dev/null; then
            # Process dead, remove stale lock
            rmdir "$lock_dir" 2>/dev/null || true
        fi
    fi
    ...
}
```