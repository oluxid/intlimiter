# Issue #010: Auto-Detection of Interface Unreliable

## Severity: LOW (Usability/Reliability)

## Description
The `get_interface()` function (lines 98-100) uses a fragile method to auto-detect the default network interface.

## Current Implementation
```bash
get_interface() {
    ip route | grep default | awk '{print $5}' | head -1
}
```

## Problems

### 1. Multiple Default Routes
Systems with multiple default routes (e.g., wired + wireless, VPN, containers) will return the first one, which may not be the desired interface.

### 2. No Default Route
```bash
# No default route (e.g., static config, container without gateway)
ip route | grep default
# Returns empty
get_interface returns empty string
# Main script: [[ -n "$INTERFACE" ]] || die "No network interface found"
```
Script fails completely instead of listing available interfaces.

### 3. IPv6 Default Route Preferred
```bash
ip route
default via 192.168.1.1 dev eth0 proto dhcp metric 100
default via fe80::1 dev eth0 proto ra metric 1024 pref medium
```
`grep default` matches both, `awk '{print $5}'` on IPv6 line returns `dev` not interface name.

### 4. Metric Not Considered
The interface with lowest metric (highest priority) should be preferred, but `head -1` just takes first line.

### 5. No Fallback to List Interfaces
When auto-detection fails, the script dies instead of showing available interfaces for manual selection.

## Impact
- Fails on multi-homed systems
- Fails on IPv6-only networks
- Fails in containers without default route
- No guidance for user to specify interface manually

## Recommended Fix

### Robust Interface Detection
```bash
get_interface() {
    # Get default IPv4 route with lowest metric
    local iface
    iface=$(ip -4 route show default | sort -k9,9n | head -1 | awk '{print $5}')

    if [[ -n "$iface" ]]; then
        echo "$iface"
        return 0
    fi

    # Fallback: IPv6 default route
    iface=$(ip -6 route show default | sort -k9,9n | head -1 | awk '{print $5}')
    if [[ -n "$iface" ]]; then
        echo "$iface"
        return 0
    fi

    # Fallback: first non-loopback UP interface
    iface=$(ip -brief link show up | grep -v lo | awk '{print $1}' | head -1)
    if [[ -n "$iface" ]]; then
        echo "$iface"
        return 0
    fi

    return 1
}
```

### Better Error Message with Interface List
```bash
if [[ -z "$INTERFACE" ]]; then
    INTERFACE=$(get_interface) || {
        echo "Error: Could not auto-detect network interface." >&2
        echo "" >&2
        echo "Available interfaces:" >&2
        ip -brief link show up | grep -v lo | awk '{print "  " $1 " (" $2 ")"}' >&2
        echo "" >&2
        echo "Specify interface manually with --interface <name>" >&2
        exit 1
    }
fi
```

### Validate Auto-Detected Interface
```bash
get_interface() {
    ...
    if [[ -n "$iface" ]] && ip link show "$iface" >/dev/null 2>&1; then
        echo "$iface"
        return 0
    fi
    return 1
}
```

## References
- `ip route` man page
- Linux routing metrics