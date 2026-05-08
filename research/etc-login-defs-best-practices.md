# /etc/login.defs - Best Practices & Configuration Guide

## Research Date
2026-05-07

---

## Overview

The `/etc/login.defs` file is a configuration file on Unix/Linux systems that defines certain parameters and behaviors for the `login` program and user account management. It controls settings like password aging, UID/GID ranges, home directory creation, and login constraints.

This file is read by many programs including `useradd`, `userdel`, `usermod`, `login`, and `passwd`.

---

## Key Configuration Sections

### 1. Password Aging (PASS_*)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `PASS_MAX_DAYS` | 99999 | Maximum days before password expires |
| `PASS_MIN_DAYS` | 0 | Minimum days between password changes |
| `PASS_WARN_AGE` | 7 | Days before expiration to warn user |
| `PASS_MIN_LEN` | 5 | Minimum password length |

**Best Practices:**
- Set `PASS_MAX_DAYS` to 90-180 days (not unlimited)
- Set `PASS_MIN_DAYS` to 1-7 (prevents rapid cycling)
- Set `PASS_WARN_AGE` to 7-14 days
- `PASS_MIN_LEN` should be at least 12 in modern security standards

```bash
# Recommended for security-conscious environments
PASS_MAX_DAYS   90
PASS_MIN_DAYS   7
PASS_WARN_AGE   14
PASS_MIN_LEN    12
```

### 2. UID/GID Ranges (UID_* and GID_*)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `UID_MIN` | 1000 | Minimum UID for new users |
| `UID_MAX` | 60000 | Maximum UID for new users |
| `GID_MIN` | 1000 | Minimum GID for new groups |
| `GID_MAX` | 60000 | Maximum GID for new groups |
| `SYS_UID_MIN` | 201 | Minimum for system users |
| `SYS_UID_MAX` | 999 | Maximum for system users |

**Best Practices:**
- Keep system UIDs (< 1000) strictly reserved
- Consider `UID_MAX` of 65534 (RHEL/CentOS standard)
- Use separate ranges for system vs regular users

```bash
# Standard Linux configuration
UID_MIN          1000
UID_MAX          60000
SYS_UID_MIN     201
SYS_UID_MAX     999

GID_MIN          1000
GID_MAX          60000
SYS_GID_MIN     201
SYS_GID_MAX     999
```

### 3. Home Directory Creation (HOME_MODE)

| Parameter | Description |
|-----------|-------------|
| `HOME_MODE` | Mode (permissions) for new home directories |

**Best Practices:**
- Set to `0750` or `0700` for privacy
- Group-readable only if team sharing is needed

```bash
HOME_MODE        0750
```

### 4. Login Retries (LOGIN_RETRIES)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `LOGIN_RETRIES` | 5 | Maximum authentication attempts |
| `LOGIN_TIMEOUT` | 60 | Login process timeout in seconds |

**Best Practices:**
- Keep `LOGIN_RETRIES` at 3-5 (lower = more secure)
- `LOGIN_TIMEOUT` of 60-120 seconds is reasonable

```bash
LOGIN_RETRIES    3
LOGIN_TIMEOUT    60
```

### 5. Umask Settings (UMASK and HOME_MAIL)

| Parameter | Description |
|-----------|-------------|
| `UMASK` | Default umask for new users |
| `HOME_MAIL` | Location of user mailboxes |

**Best Practices:**
- Default umask `022` is standard
- More restrictive `077` for high-security environments
- Mail spool location typically `/var/spool/mail/`

```bash
UMASK            077
HOME_MAIL       /var/spool/mail
```

### 6. User Environment (ENV_*)

| Parameter | Description |
|-----------|-------------|
| `ENV_PATH` | Default PATH for regular users |
| `ENV_SUPATH` | Default PATH for root |
| `ENV_HZ` | TerminalHZ setting |

**Best Practices:**
- Never include `.` in PATH for security
- Root's PATH should be minimal and explicitly defined
- Regular user PATH should exclude current directory

```bash
ENV_PATH         /usr/local/bin:/usr/bin:/bin
ENV_SUPATH       /usr/local/sbin:/usr/sbin:/sbin
```

### 7. Password Hashing (ENCRYPT_METHOD)

| Parameter | Recommended | Description |
|-----------|-------------|-------------|
| `ENCRYPT_METHOD` | SHA512 | Hash algorithm for passwords |

**Best Practices:**
- Use `SHA512` (or `YESCRYPT` on newer systems)
- Never use `DES` or `MD5` (cryptographically broken)
- Ensure system supports the chosen algorithm

```bash
ENCRYPT_METHOD   SHA512
```

### 8. Skeleton Directory (SKEL)

| Parameter | Description |
|-----------|-------------|
| `SKEL` | Directory containing default files for new homes |

**Best Practices:**
- Include `.bashrc`, `.profile`, `.bash_history` templates
- Add `.bash_logout` for security (clears history on exit)
- Ensure skeleton files don't contain sensitive data

```bash
SKEL            /etc/skel
```

### 9. Default Shell (DEFAULT_HOME and SHELL)

| Parameter | Description |
|-----------|-------------|
| `DEFAULT_HOME` | Allow login if home directory exists |
| `SHELL` | Default login shell |

**Best Practices:**
- `DEFAULT_HOME yes` — allow login without home (some disable)
- Default shell `/bin/bash` is standard
- Consider `/usr/bin/bash` on newer distros

```bash
DEFAULT_HOME     yes
SHELL            /bin/bash
```

### 10. Account Creation (CREATE_HOME and USERGROUPS_ENAB)

| Parameter | Description |
|-----------|-------------|
| `CREATE_HOME` | Create home directory by default |
| `USERGROUPS_ENAB` | Create group with same name as user |

**Best Practices:**
- `CREATE_HOME yes` — always create home dirs
- `USERGROUPS_ENAB yes` — user's own group by default

```bash
CREATE_HOME      yes
USERGROUPS_ENAB  yes
```

---

## Complete Example Configuration

```bash
# /etc/login.defs - Configuration file for login(1)

# Password aging
PASS_MAX_DAYS   90
PASS_MIN_DAYS   7
PASS_WARN_AGE   14
PASS_MIN_LEN    12

# UID/GID ranges
UID_MIN          1000
UID_MAX          60000
GID_MIN          1000
GID_MAX          60000
SYS_UID_MIN     201
SYS_UID_MAX     999
SYS_GID_MIN     201
SYS_GID_MAX     999

# Home directory
HOME_MODE        0750

# Login settings
LOGIN_RETRIES    3
LOGIN_TIMEOUT    60

# Umask and mail
UMASK            077
HOME_MAIL        /var/spool/mail

# Environment
ENV_PATH         /usr/local/bin:/usr/bin:/bin
ENV_SUPATH       /usr/local/sbin:/usr/sbin:/sbin

# Password encryption
ENCRYPT_METHOD   SHA512

# Skeleton
SKEL             /etc/skel

# Defaults
DEFAULT_HOME     yes
SHELL            /bin/bash
CREATE_HOME      yes
USERGROUPS_ENAB  yes
```

---

## Security Hardening Checklist

- [ ] `PASS_MAX_DAYS` ≤ 180 (90 is recommended)
- [ ] `PASS_MIN_LEN` ≥ 12
- [ ] `ENCRYPT_METHOD` is `SHA512` or `YESCRYPT`
- [ ] `UMASK` is `077` or `027` (not `022`)
- [ ] `LOGIN_RETRIES` ≤ 5 (3 is better)
- [ ] System UIDs (1-200) are reserved and unused
- [ ] `SKEL` directory is secure and audited
- [ ] `ENV_PATH` excludes current directory (`.`)
- [ ] `HOME_MODE` is `0750` or stricter

---

## Interaction with Other Systems

### PAM (Pluggable Authentication Modules)
`/etc/login.defs` works alongside PAM configuration. For password policies, many systems use `pam_pwquality.so` or `pam_unix.so` instead, which provide more granular control.

### chage Command
Use `chage` to modify password aging for existing users:
```bash
# View password aging info
chage -l username

# Set maximum days
chage -M 90 username

# Set minimum days
chage -m 7 username

# Set warning period
chage -W 14 username
```

### useradd Defaults
```bash
# View current defaults
useradd -D

# Modify defaults
useradd -D -K UID_MIN=500 -K UID_MAX=30000
```

---

## Common Mistakes

| Mistake | Correct Practice |
|---------|------------------|
| `PASS_MAX_DAYS 99999` | Set to 90-180 |
| `UMASK 022` | Use `077` for privacy |
| `ENCRYPT_METHOD MD5` | Use `SHA512` |
| `LOGIN_RETRIES 10` | Keep at 3-5 |
| Unchecked `SKEL` | Audit skeleton files regularly |

---

## Related Files

| File | Purpose |
|------|---------|
| `/etc/passwd` | User account information |
| `/etc/shadow` | Encrypted password data |
| `/etc/group` | Group membership |
| `/etc/skel/` | Default home directory template |
| `/etc/pam.d/login` | PAM configuration for login |

---

## Sources

- Linux man pages: `login.defs(5)`
- `useradd(8)`, `userdel(8)`, `passwd(5)`
- Red Hat Enterprise Linux Security Guide
- CIS Linux Benchmarks

---

*Generated by Boti using general-researcher skill*