# Issue #005: No Support for IPv6 Traffic Limiting

## Severity: MEDIUM (Feature/Completeness)

## Description
The script only filters IPv4 traffic (`protocol ip` in tc filter commands). IPv6 traffic bypasses all limits completely.

## Affected Code
- Line 277-279: `sudo tc filter add dev "$iface" parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev "$IFB_DEV"`
- Line 305: `sudo tc filter add dev "$IFB_DEV" parent 1: protocol ip u32 match u32 0 0 flowid 1:1`
- Line 330: `sudo tc filter add dev "$iface" parent 1: protocol ip u32 match u32 0 0 flowid 1:1`

## Current Filters (IPv4 Only)
```bash
# Download - ingress redirect (line 277-279)
tc filter add dev "$iface" parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev "$IFB_DEV"

# Download - IFB egress shaping (line 305)
tc filter add dev "$IFB_DEV" parent 1: protocol ip u32 match u32 0 0 flowid 1:1

# Upload - egress shaping (line 330)
tc filter add dev "$iface" parent 1: protocol ip u32 match u32 0 0 flowid 1:1
```

## Impact
- All IPv6 traffic (websites, downloads, streaming, etc.) is UNLIMITED
- Users think they're limited but IPv6 bypasses it
- Modern networks increasingly use IPv6 (Google, Facebook, Netflix, etc. all support IPv6)
- Security/bandwidth control bypass

## Test to Verify
```bash
# Apply limit
intlimiter --down --mbit 10

# Test IPv4 (should be limited)
curl -4 http://speedtest.example.com/largefile

# Test IPv6 (will be UNLIMITED)
curl -6 http://speedtest.example.com/largefile
```

## Recommended Fix

### Add IPv6 Filters Alongside IPv4
```bash
setup_interface() {
    local iface=$1
    local direction=$2

    sudo tc qdisc del dev "$iface" root 2>/dev/null || debug "root qdisc removal warning on $iface during setup"
    sudo tc qdisc add dev "$iface" root handle 1: htb || die "Failed to add qdisc on $iface"

    if [[ "$direction" == "up" ]]; then
        return 0
    fi

    # Ingress qdisc for redirect (IPv4 + IPv6)
    sudo tc qdisc add dev "$iface" handle ffff: ingress 2>/dev/null || debug "ingress qdisc add warning on $iface during setup"

    # Redirect IPv4
    sudo tc filter add dev "$iface" parent ffff: protocol ip u32 \
        match u32 0 0 action mirred egress redirect dev "$IFB_DEV" || \
        die "Failed to redirect IPv4 ingress traffic to IFB"

    # Redirect IPv6
    sudo tc filter add dev "$iface" parent ffff: protocol ipv6 u32 \
        match u32 0 0 action mirred egress redirect dev "$IFB_DEV" || \
        die "Failed to redirect IPv6 ingress traffic to IFB"
}

set_download_limit() {
    ...
    # Shape IPv4 on IFB
    sudo tc filter add dev "$IFB_DEV" parent 1: protocol ip u32 match u32 0 0 flowid 1:1 || debug "IFB IPv4 filter add warning"

    # Shape IPv6 on IFB
    sudo tc filter add dev "$IFB_DEV" parent 1: protocol ipv6 u32 match u32 0 0 flowid 1:1 || debug "IFB IPv6 filter add warning"
    ...
}

set_upload_limit() {
    ...
    # Shape IPv4 egress
    sudo tc filter add dev "$iface" parent 1: protocol ip u32 match u32 0 0 flowid 1:1 || debug "upload IPv4 filter add warning"

    # Shape IPv6 egress
    sudo tc filter add dev "$iface" parent 1: protocol ipv6 u32 match u32 0 0 flowid 1:1 || debug "upload IPv6 filter add warning"
    ...
}
```

### Alternative: Protocol-Independent Filter (if supported)
```bash
# Use protocol all (if kernel supports it)
tc filter add dev "$iface" parent ffff: protocol all u32 match u32 0 0 action mirred egress redirect dev "$IFB_DEV"
```
Note: `protocol all` may not be supported on all kernel versions.

### Better: Use `match u16 0x0800 0xFFFF at -2` for IPv4 and `match u16 0x86DD 0xFFFF at -2` for IPv6
This is more explicit and works with older kernels:
```bash
# IPv4 ethertype 0x0800
tc filter add dev "$iface" parent ffff: protocol 802.1q u32 match u16 0x0800 0xFFFF at -2 action mirred egress redirect dev "$IFB_DEV"

# IPv6 ethertype 0x86DD
tc filter add dev "$iface" parent ffff: protocol 802.1q u32 match u16 0x86DD 0xFFFF at -2 action mirred egress redirect dev "$IFB_DEV"
```

## Additional Consideration: teardown_interface()
Must also remove IPv6 filters:
```bash
teardown_interface() {
    local iface=$1
    local err=

    # Remove IPv4 ingress filter
    sudo tc filter del dev "$iface" parent ffff: protocol ip pref 49152 2>/dev/null || true

    # Remove IPv6 ingress filter
    sudo tc filter del dev "$iface" parent ffff: protocol ipv6 pref 49152 2>/dev/null || true

    if ! sudo tc qdisc del dev "$iface" ingress 2>/dev/null; then
        err="ingress_qdisc_removal_failed"
    fi
    ...
}
```

## References
- tc filter protocol matching: `man tc-filter`
- IPv6 ethertype: 0x86DD
- IPv4 ethertype: 0x0800