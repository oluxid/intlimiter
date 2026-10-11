# Issue #007: Speed Validation Allows Zero and Negative Values

## Severity: LOW (Input Validation)

## Description
The `validate_speed()` function (lines 84-89) uses `bc` to check if speed is positive, but the regex and logic have edge cases that allow invalid values.

## Affected Code
```bash
validate_speed() {
    local speed=$1
    if ! [[ "$speed" =~ ^[0-9]+(\.[0-9]+)?$ ]] || (( $(echo "$speed <= 0" | bc -l) )); then
        die "Speed must be a positive number"
    fi
}
```

## Problems

### 1. Leading Zeros Accepted (Octal Interpretation Risk)
```bash
intlimiter --down --mbit 050  # Interpreted as octal 50 = decimal 40 in some contexts
```
The regex `^[0-9]+(\.[0-9]+)?$` allows leading zeros. While `bc` handles them as decimal, it's confusing.

### 2. No Upper Bound
```bash
intlimiter --down --mbit 9999999999999999999999999999999999999999999999999
```
No maximum speed validation. Could cause integer overflow in tc or kernel.

### 3. Scientific Notation Not Handled
```bash
intlimiter --down --mbit 1e6  # Fails regex (good)
intlimiter --down --mbit 1.5e3  # Fails regex (good)
```
Actually the regex correctly rejects these, but it's worth noting.

### 4. bc Dependency for Simple Comparison
Using `bc` for a simple numeric comparison is overkill and adds a dependency. Pure bash can do this.

### 5. Floating Point Precision
```bash
intlimiter --down --mbit 0.0000000001
```
Very small values may convert to 0 mbit after `convert_to_mbit()`:
```bash
convert_to_mbit() {
    ...
    echo "$speed * 8" | bc | awk '{printf "%.0f", $1}'  # Rounds to 0
}
```

## Impact
- Confusing behavior with leading zeros
- Potential kernel issues with extremely large values
- Very small values silently become 0 (no limit applied)

## Recommended Fix

### Improved Validation
```bash
validate_speed() {
    local speed=$1
    local max_speed=1000000  # 1 Tbps max (adjust as needed)

    # Strict regex: no leading zeros (except "0" itself), optional decimal
    if ! [[ "$speed" =~ ^(0|[1-9][0-9]*)(\.[0-9]+)?$ ]]; then
        die "Speed must be a positive number (no leading zeros, no scientific notation)"
    fi

    # Pure bash comparison for integer part
    local int_part=${speed%.*}
    if (( int_part == 0 )); then
        # Check decimal part
        if [[ "$speed" == *.* ]]; then
            local dec_part=${speed#*.}
            # Remove trailing zeros
            dec_part=${dec_part%0}
            if [[ -z "$dec_part" ]] || (( dec_part == 0 )); then
                die "Speed must be greater than zero"
            fi
        else
            die "Speed must be greater than zero"
        fi
    fi

    # Check upper bound
    if (( int_part > max_speed )); then
        die "Speed exceeds maximum allowed value (${max_speed} mbit/mbyte)"
    fi

    # Warn on very small values that may round to 0
    if [[ "$speed" == *.* ]]; then
        local converted
        converted=$(convert_to_mbit "$speed" "mbit")
        if (( converted == 0 )); then
            echo "Warning: Speed $speed mbit converts to 0 mbit after unit conversion" >&2
        fi
    fi
}
```

### Simpler Alternative (Keep bc but Fix Issues)
```bash
validate_speed() {
    local speed=$1
    local max_speed=1000000

    # Regex: no leading zeros (except single 0), positive number
    if ! [[ "$speed" =~ ^(0|[1-9][0-9]*)(\.[0-9]+)?$ ]]; then
        die "Speed must be a positive number without leading zeros"
    fi

    # Use bc for comparison (handles decimals)
    if (( $(echo "$speed <= 0" | bc -l) )); then
        die "Speed must be greater than zero"
    fi

    if (( $(echo "$speed > $max_speed" | bc -l) )); then
        die "Speed exceeds maximum allowed value (${max_speed})"
    fi
}
```

### Fix convert_to_mbit() to Handle Small Values
```bash
convert_to_mbit() {
    local speed=$1
    local unit=$2

    if [ "$unit" = "mbyte" ]; then
        # Use bc with scale to preserve precision, then round up to at least 1
        local result
        result=$(echo "scale=6; $speed * 8" | bc)
        # Round up to nearest integer, minimum 1
        awk -v r="$result" 'BEGIN { i=int(r); if (r > i) i++; if (i < 1) i=1; print i }'
    else
        # Round up to nearest integer, minimum 1
        awk -v s="$speed" 'BEGIN { i=int(s); if (s > i) i++; if (i < 1) i=1; print i }'
    fi
}
```

## References
- Input validation best practices
- Bash arithmetic vs bc