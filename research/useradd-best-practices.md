# useradd - Best Practices, Tips & Tricks

## Research Date
2026-05-07

---

## Overview

`useradd` is a Linux command used to create new user accounts. It's a low-level utility that directly modifies system files (`/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow`).

> **Note:** For interactive or more user-friendly creation, consider `adduser` (Debian/Ubuntu) or `newusers` (batch). However, `useradd` is the standard across all distributions.

---

## Basic Syntax

```bash
useradd [options] username
```

---

## Essential Options

| Option | Description | Example |
|--------|-------------|---------|
| `-c` | Add comment/full name | `-c "John Doe"` |
| `-d` | Set home directory | `-d /home/johnd` |
| `-e` | Set account expiration date | `-e 2026-12-31` |
| `-f` | Set password inactive period | `-f 30` |
| `-g` | Set primary group | `-g developers` |
| `-G` | Set supplementary groups | `-G sudo,www-data,docker` |
| `-m` | Create home directory | `-m` (default on most distros) |
| `-r` | Create system account | `-r` (no expiry, no home) |
| `-s` | Set login shell | `-s /bin/bash` |
| `-u` | Set specific UID | `-u 1500` |

---

## Common Usage Examples

### Create Standard User

```bash
# Basic user creation (home dir, shell, etc. from defaults)
sudo useradd -m -s /bin/bash -c "John Doe" johnd

# With all common options
sudo useradd -m -s /bin/bash -c "John Doe" -G sudo,developers -e 2026-12-31 johnd
```

### Create System User

```bash
# System users have UID < 1000 and no expiration
sudo useradd -r -s /usr/sbin/nologin -c "Apache Service" apache

# System user with home dir (for running services with home)
sudo useradd -r -m -s /usr/sbin/nologin -c "MongoDB Service" mongod
```

### Create User with Specific UID

```bash
sudo useradd -u 1500 -m -s /bin/bash johnd
```

### Create User with Custom Home Directory

```bash
sudo useradd -d /custom/path/johnd -m -s /bin/bash johnd
```

### Create User with Specific Shell

```bash
# Use zsh instead of bash
sudo useradd -m -s /bin/zsh johnd

# Account with no shell (disabled)
sudo useradd -m -s /usr/sbin/nologin johnd
```

### Create User with Expiration Date

```bash
# For temporary contractors
sudo useradd -m -e 2026-12-31 -c "Contractor" contractor1
```

### Create User with Multiple Groups

```bash
# Add to multiple supplementary groups
sudo useradd -m -G sudo,www-data,developers,docker -s /bin/bash johnd
```

---

## Configuration Files

`useradd` reads defaults from two places (in order of priority):

### 1. Command Line Options
Highest priority - explicitly passed options override defaults.

### 2. `/etc/default/useradd`
Contains default values for home directory mode, skeleton, shell, and expiration.

```bash
# View current defaults
useradd -D

# Example output:
# GROUP=1000
# HOME=/home
# INACTIVE=-1
# EXPIRE=
# SHELL=/bin/bash
# SKEL=/etc/skel
# CREATE_MAIL_SPOOL=yes
```

### 3. `/etc/login.defs`
Defines UID/GID ranges, password aging defaults, and other system-wide settings.

---

## Skeleton Directory (`/etc/skel`)

When creating a home directory, `useradd` copies files from `/etc/skel` to the new user's home.

```bash
# Common skeleton contents
ls -la /etc/skel/
# .bashrc
# .bash_history
# .profile
# .bash_logout (important for security - clears history on exit)
# .vimrc (optional)
```

**Best Practice:** Ensure your `/etc/skel` contains:
- Proper `.bashrc` with security settings
- `.profile` for environment variables
- `.bash_logout` that clears history
- No sensitive data

---

## Home Directory Management

### Create Home Directory Manually

```bash
# If you forgot -m
sudo mkhomedir_helper username
# or
sudo mkdir /home/username && sudo chown username:username /home/username
```

### Default Permissions

The `HOME_MODE` in `/etc/login.defs` controls permissions:

```bash
# Default is usually 0755 (world-readable)
# For more privacy:
HOME_MODE 0700
```

### Set Quotas for Home Directory

```bash
# Set disk quota for user (requires quota package)
setquota -u username 1000000 1500000 0 0 /home
```

---

## Best Practices

### Security

1. **Use strong default umask** in `/etc/login.defs`:
   ```
   UMASK 077  # Most restrictive
   UMASK 027  # Group-readable, others none
   ```

2. **Set account expiration** for temporary accounts:
   ```bash
   sudo useradd -e 2026-12-31 -c "Temp Employee" tempuser
   ```

3. **Lock accounts when not in use**:
   ```bash
   sudo usermod -L username    # Lock
   sudo usermod -U username    # Unlock
   ```

4. **Use `/usr/sbin/nologin` for service accounts**:
   ```bash
   sudo useradd -r -s /usr/sbin/nologin serviceaccount
   ```

5. **Never create users with UID 0** (root):
   ```bash
   # NEVER do this
   sudo useradd -u 0 username  # Creates pseudo-root!
   ```

### Organization

1. **Use consistent UID ranges**:
   ```
   System accounts:     UID 1-999
   Regular users:      UID 1000-60000
   Application accounts: UID 60001-65534
   ```

2. **Name groups after the role**:
   ```bash
   sudo groupadd developers
   sudo groupadd analysts
   sudo useradd -G developers alice
   ```

3. **Create a new group for each project**:
   ```bash
   sudo groupadd project-alpha
   sudo useradd -G project-alpha alice bob
   ```

4. **Use descriptive comments**:
   ```bash
   sudo useradd -c "Data Analytics Team Lead" alice
   ```

### Performance

1. **Batch create users** with `newusers`:
   ```bash
   # Create from file
   newusers users.txt

   # users.txt format:
   # username:password:UID:GID:comment:home:shell
   ```

2. **Use system accounts for services**:
   ```bash
   # Avoids UIDs changing across systems
   sudo useradd -r -s /usr/sbin/nologin nginx
   ```

---

## Tips & Tricks

### One-Liners

```bash
# Create user and set password in one line
echo "username:password" | sudo chpasswd && sudo useradd -m username

# Create user with random password
password=$(openssl rand -base64 32)
sudo useradd -m -s /bin/bash username
echo "username:$password" | sudo chpasswd
echo "Password: $password"

# Create user and force password change on first login
sudo useradd -m -s /bin/bash username
sudo passwd username
sudo chage -d 0 username

# Copy all groups from existing user to new user
sudo useradd -G $(groups existinguser | cut -d: -f2) -m newuser
```

### Interactive User Creation

For easier user creation, use `adduser` (Debian-based):

```bash
# Interactive with prompts
sudo adduser username

# Non-interactive
sudo adduser --disabled-password --gecos "Full Name" username
```

### Verify User Creation

```bash
# Check user exists
getent passwd username

# Check groups
groups username

# Check home directory
ls -la /home/username

# Check password aging
chage -l username

# Check account status
passwd -S username
```

### Migrating Users

```bash
# Export user info (to transfer to another system)
grep username /etc/passwd >> users.txt
grep username /etc/shadow >> shadows.txt (careful with security!)
grep username /etc/group >> groups.txt

# Import on new system
newusers users.txt
```

---

## Common Pitfalls

| Issue | Cause | Solution |
|-------|-------|----------|
| Home directory not created | Forgot `-m` on older systems | `mkhomedir_helper username` or `mkdir && chown` |
| Wrong shell | `useradd` defaults to `/bin/sh` on some distros | Always specify `-s /bin/bash` |
| UID conflicts | Manually chosen UID already exists | Use `getent passwd uid` to check first |
| User can't login | Shell set to `/usr/sbin/nologin` | Change with `usermod -s /bin/bash username` |
| Permission denied on home dir | Wrong ownership | `chown -R username:username /home/username` |
| User added to wrong groups | Used `-g` instead of `-G` | `-g` sets primary, `-G` adds supplementary |

---

## Related Commands

| Command | Purpose |
|---------|---------|
| `userdel` | Delete user account |
| `usermod` | Modify existing user |
| `passwd` | Change password |
| `chage` | Change password aging |
| `id` | Show user identity info |
| `groups` | Show group memberships |
| `getent` | Query user/group databases |
| `adduser` | Interactive user creation (Debian) |
| `newusers` | Batch user creation |

---

## Quick Reference

```bash
# Standard user
sudo useradd -m -s /bin/bash -c "Full Name" username

# System account
sudo useradd -r -s /usr/sbin/nologin servicename

# With expiration
sudo useradd -m -e 2026-12-31 -c "Contractor" username

# With groups
sudo useradd -m -G sudo,www-data,docker -s /bin/bash username

# With specific UID
sudo useradd -u 1500 -m username

# View defaults
useradd -D

# Modify defaults
useradd -D -s /bin/zsh -e 90
```

---

*Generated by Boti using general-researcher skill*