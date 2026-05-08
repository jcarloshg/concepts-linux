# /etc/shadow - Best Practices & Security Guide

## Research Date
2026-05-07

---

## Overview

The `/etc/shadow` file is a critical system file in Unix/Linux that stores encrypted passwords and account aging information for user accounts. Unlike `/etc/passwd` which is readable by everyone, `/etc/shadow` is only readable by root and system processes.

**File permissions:** `000` or `0400` (root-only read/write)

---

## File Format

Each line in `/etc/shadow` contains nine fields separated by colons (`:`):

```
username:password:last_change:min_days:max_days:warn_days:inactive:expire:reserved
```

### Field Breakdown

| Field | Description | Example |
|-------|-------------|---------|
| `username` | User login name | `jose` |
| `password` | Encrypted password (or `*` or `!`) | `$6$salt$hash...` |
| `last_change` | Days since Jan 1, 1970 password was changed | `19650` |
| `min_days` | Minimum days before password can be changed | `7` |
| `max_days` | Maximum days before password expires | `90` |
| `warn_days` | Days before expiration to warn user | `14` |
| `inactive` | Days after expiration before account is disabled | `30` |
| `expire` | Days since Jan 1, 1970 when account expires | `19756` |
| `reserved` | Reserved for future use | empty |

---

## Password Field Examples

### Valid Encrypted Passwords

| Format | Algorithm | Example Hash Prefix |
|--------|-----------|---------------------|
| `$1$` | MD5 | `$1$5x9G3k$PwzJ1tZ5yH8Z7a4qQ6vK0` |
| `$2a$` | Blowfish | `$2a$10$...` |
| `$5$` | SHA-256 | `$5$rounds=535000$salt$hash` |
| `$6$` | SHA-512 | `$6$salt$hash` (most common) |
| `$y$` | yescrypt | `$y$j9T$salt$hash` |
| `$gy$` | gost-yescrypt | `$gy$j9T$salt$hash` |

### Special Characters

| Value | Meaning |
|-------|---------|
| `*` | Account locked, no password |
| `!` | Account locked, no password |
| `!!` | Password never set |
| `NP` | No password (NIS+) |
| Empty | No password required |

---

## Password Aging Commands

### chage - Change User Aging Information

```bash
# View current aging settings
chage -l username

# Set password to never expire
chage -M -1 username

# Set password to expire in 90 days
chage -M 90 username

# Set minimum days between changes
chage -m 7 username

# Set warning days before expiration
chage -W 14 username

# Set account inactive period after expiration
chage -I 30 username

# Set account expiration date
chage -E 2026-12-31 username

# Set all parameters interactively
chage -d 0 username    # Force password change on next login
```

### passwd with aging

```bash
# Lock account (add ! to password)
passwd -l username

# Unlock account
passwd -u username

# Delete password
passwd -d username

# Show status
passwd -S username
```

---

## Security Best Practices

### File Permissions

```bash
# Correct permissions
chmod 000 /etc/shadow          # Most secure
chmod 0400 /etc/shadow         # Alternative

# Verify ownership
chown root:shadow /etc/shadow
```

### Password Policy Configuration

```bash
# Enforce strong passwords (via PAM)
# Edit /etc/pam.d/common-password or /etc/security/pwquality.conf

minlen = 12                    # Minimum length
dcredit = -1                  # At least 1 digit
ucredit = -1                  # At least 1 uppercase
lcredit = -1                  # At least 1 lowercase
ocredit = -1                  # At least 1 special char
```

### Regular Auditing

```bash
# Find users with no password
awk -F: '($2 == "") {print $1}' /etc/shadow

# Find users with older hash algorithms
awk -F: '/^\$1\$/ {print $1}' /etc/shadow    # MD5
awk -F: '/^\$2a\$/ {print $1}' /etc/shadow   # Blowfish
awk -F: '/^\$5\$/ {print $1}' /etc/shadow    # SHA-256

# Find accounts with no expiration
awk -F: '($8 == "" || $8 == "19701") {print $1}' /etc/shadow

# Find passwordless accounts
grep -E '^[^:]+::' /etc/shadow
```

---

## Common Scenarios

### Disable Password Expiration for Service Accounts

```bash
# For service accounts that can't have password changes
chage -M -1 -m 0 -W 7 service_account
```

### Force Password Change on Next Login

```bash
# When user forgets password or security incident
chage -d 0 username
# This sets last_change to 0, forcing change on next login
```

### Implement 90-Day Password Policy

```bash
# For all users (run periodically or add to new user template)
for user in $(awk -F: '($3 >= 1000) {print $1}' /etc/passwd); do
    chage -M 90 -m 7 -W 14 $user
done
```

### Set Account Expiration

```bash
# For temporary employees/contractors
chage -E 2026-12-31 contractor_user
```

### Lock Account After Failed Attempts

This requires PAM configuration, but in `/etc/login.defs` you can set:
```bash
FAIL_DELAY  5                # Seconds between attempts
LOGIN_RETRIES 3             # Max attempts
```

---

## Understanding the Fields

### last_change (Days Since Epoch)

| Value | Meaning |
|-------|---------|
| `0` | User must change password on next login |
| `19650` | Specific date (example: ~2024) |
| Calculating: | `date +%s` divided by 86400 |

```bash
# Convert last_change to date
last_change=19650
date -d "1970-01-01 + $last_change days" "+%Y-%m-%d"
```

### min_days

- Prevents users from changing passwords too frequently
- Common: `7` (one week minimum between changes)
- Prevents cycling through passwords to reuse old ones

### max_days

| Value | Policy |
|-------|--------|
| `-1` or empty | Password never expires |
| `90` | Change every 90 days (recommended) |
| `180` | Change every 6 months |
| `30` | Change monthly |

### warn_days

- Typically `14` days (2 weeks warning)
- Gives users time to prepare for password change

### inactive

| Value | Meaning |
|-------|---------|
| Empty or `-1` | No grace period |
| `30` | Account disabled 30 days after expiration |

### expire

| Value | Meaning |
|-------|---------|
| Empty | Account never expires |
| `19701` | Never expires (Jan 1, 2024) |
| `20300` | Example: ~2025 |

---

## Reading /etc/shadow Safely

### Using getent

```bash
# Proper way to query shadow file
getent shadow username
```

### Parsing Examples (Python)

```python
with open('/etc/shadow', 'r') as f:
    for line in f:
        fields = line.strip().split(':')
        username = fields[0]
        password = fields[1]
        last_change = int(fields[2])
        # ... etc
```

### Using awk

```bash
# Extract specific user
awk -F: '/^username:/ {print $0}' /etc/shadow

# Count users with SHA-512 passwords
awk -F: '/^\$6\$/ {count++} END {print count}' /etc/shadow
```

---

## Related Files and Commands

| File/Command | Purpose |
|--------------|---------|
| `/etc/shadow` | Encrypted password storage |
| `/etc/passwd` | User account info (no passwords) |
| `/etc/login.defs` | Default aging parameters |
| `/etc/pam.d/` | PAM authentication modules |
| `chage` | Modify user aging info |
| `passwd` | Change user password |
| `pwck` | Verify integrity of password files |
| `vipw` | Safely edit password files |

---

## PAM Configuration for Password Quality

File: `/etc/security/pwquality.conf` or via PAM modules

```bash
# /etc/security/pwquality.conf example
minlen = 12
dcredit = -1           # At least 1 digit
ucredit = -1           # At least 1 uppercase
lcredit = -1           # At least 1 lowercase
ocredit = -1           # At least 1 special
maxrepeat = 2          # Max consecutive same chars
maxclassrepeat = 4     # Max consecutive chars from same class
difok = 3              # Min different chars from old password
```

---

## Security Checklist

- [ ] `/etc/shadow` permissions are `0400` or `000`
- [ ] Ownership is `root:shadow`
- [ ] All service accounts have strong passwords
- [ ] No accounts have empty password fields
- [ ] Password hash algorithm is `$6$` (SHA-512) or better
- [ ] No legacy hash formats (`$1$`, `$2a$`, `$5$`)
- [ ] `max_days` is set to 90-180 (not unlimited)
- [ ] `min_days` prevents rapid password cycling
- [ ] `warn_days` gives at least 7-14 days warning
- [ ] Inactive period is set (30 days recommended)
- [ ] Temporary accounts have explicit `expire` dates
- [ ] Regular auditing is scheduled
- [ ] PAM enforces password quality requirements

---

## Quick Reference Commands

```bash
# View user aging info
chage -l username

# Set all aging parameters
chage -m 7 -M 90 -W 14 -I 30 -E 2026-12-31 username

# List all users with password expiration info
for u in $(cut -d: -f1 /etc/shadow); do echo -n "$u: "; chage -l $u | grep "Password expires"; done

# Check for weak hash algorithms
grep -E '^\w+:\$1\$|\:\$2a\$|\:\$5\$' /etc/shadow

# Find accounts that never expire
awk -F: '($8 == "") {print $1}' /etc/shadow

# Find users with password not set
grep -E ':\!\!:' /etc/shadow
grep -E ':\*:' /etc/shadow
```

---

## Sources

- Linux man pages: `shadow(5)`, `chage(1)`, `passwd(1)`
- `pwck(8)` - verify password file integrity
- `/etc/login.defs` - default configuration
- Red Hat Enterprise Linux Security Guide
- CIS Linux Distribution Security Benchmarks

---

*Generated by Boti using general-researcher skill*