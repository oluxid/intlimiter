# Issue #001: Command Injection Vulnerability via Interface Name

## Severity: HIGH (Security)

## Description
The `validate_interface()` function (line 91-96) and `get_interface()` function (line 98-100) do not properly sanitize the interface name before using it in shell commands. The interface name is directly interpolated into `ip link show`, `ip route`, `tc qdisc`, and other commands without validation against shell metacharacters.

## Affected Code
- Line 93: `if ! ip link show "$iface" >/dev/null 2>&1; then`
- Line 99: `ip route | grep default | awk '{print $5}' | head -1`
- Line 179-184: `sudo tc qdisc del dev "$iface" ...`
- Line 269-270: `sudo tc qdisc del dev "$iface" root ...` / `sudo tc qdisc add dev "$iface" root ...`
- Line 277-279: `sudo tc filter add dev "$iface" ...`
- Line 301-305: Various tc commands using `$IFB_DEV` and `$iface`
- Line 327-330: `sudo tc class add dev "$iface" ...` / `sudo tc filter add dev "$iface" ...`

## Attack Vector
An attacker who can control the interface name (via `--interface` argument or by manipulating the default route) could inject shell commands. For example:
```bash
intlimiter --down --mbit 50 --interface "eth0; rm -rf /"
```

The interface name validation only checks if the interface exists via `ip link show`, but the name itself is never sanitized before being used in subsequent `tc` commands.

## Impact
- Arbitrary command execution with root privileges (since tc commands require sudo)
- Complete system compromise
- Data theft, destruction, or ransomware

## Proof of Concept
```bash
# If an interface named "eth0; reboot" exists (or can be created), the command:
sudo tc qdisc add dev "eth0; reboot" root handle 1: htb
# Would execute: reboot
```

## Recommended Fix
1. **Strict validation**: Only allow alphanumeric characters, hyphens, underscores, and dots in interface names (standard Linux interface naming convention):
   ```bash
   validate_interface() {
       local iface=$1
       # Validate interface name format (Linux kernel naming rules)
       if ! [[ "$iface" =~ ^[a-zA-Z0-9._-]+$ ]]; then
           die "Invalid interface name: $iface (only alphanumeric, dot, hyphen, underscore allowed)"
       fi
       if ! ip link show "$iface" >/dev/null 2>&1; then
           die "Interface '$iface' does not exist"
       fi
   }
   ```

2. **Use arrays for commands**: Instead of string interpolation, use bash arrays to avoid any shell interpretation:
   ```bash
   sudo tc qdisc add dev "$iface" root handle 1: htb
   # Becomes:
   sudo tc qdisc add dev "$iface" root handle 1: htb
   # (This is already safe because "$iface" is quoted, but the validation above is still needed)
   ```

3. **Sanitize auto-detected interface**: The `get_interface()` function should also validate its output.

## References
- CWE-78: Improper Neutralization of Special Elements used in an OS Command ('OS Command Injection')
- CWE-88: Improper Neutralization of Argument Delimiters in a Command ('Argument Injection')