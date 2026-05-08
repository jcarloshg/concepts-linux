# Linux Groups - Best Practices & Commands

## Research Date
2026-05-07

---

## Overview

Linux groups are a way to organize users and manage permissions. Each user can belong to multiple groups, and groups can have shared permissions for files, directories, and system resources.

---

## Key Files

| File | Description | Permissions |
|------|-------------|-------------|
| `/etc/group` | Group definitions | Readable by all |
| `/etc/gshadow` | Group passwords (encrypted) | Root only |
| `/etc/passwd` | User-to-group mapping | Readable by all |

### /etc/group Format
```
group_name:password:GID:user1,user2,user3
```

| Field | Description |
|-------|-------------|
| `group_name` | Name of the group |
| `password` | Group password (or `x` for shadow, `!` for locked) |
| `GID` | Numeric group ID |
| `userlist` | Comma-separated list of member users |

---

## Essential Commands

### Viewing Groups

```bash
# Show current user's groups
groups

# Show specific user's groups
groups username

# Display all groups (from /etc/group)
getent group

# Show group membership for all users
getent group | cut -d: -f1,4

# Display GID and members for a group
getent group groupname
```

### Creating Groups

```bash
# Create a new group
sudo groupadd developers

# Create group with specific GID
sudo groupadd -g 1500 developers

# Create system group (lower GID range)
sudo groupadd -r systemgroup
```

### Deleting Groups

```bash
# Delete a group
sudo groupdel groupname

# Cannot delete if it's primary group of a user
# Must remove user from group first
```

### Modifying Groups

```bash
# Rename a group
sudo groupmod -n oldname newname

# Change GID
sudo groupmod -g 1500 groupname

# Set group password
sudo gpasswd groupname

# Remove group password
sudo gpasswd -r groupname
```

### Managing Members

```bash
# Add user to group (primary or supplementary)
sudo usermod -aG groupname username

# Add user to multiple groups
sudo usermod -aG group1,group2,group3 username

# Remove user from group
sudo gpasswd -d username groupname

# Add multiple users to group
sudo gpasswd -M user1,user2,user3 groupname

# Clear all members from group
sudo gpasswd -M "" groupname
```

### Group Administration with gpasswd

```bash
# Add administrator to group
sudo gpasswd -A username groupname

# List group administrators
sudo gpasswd -A groupname

# Remove all administrators
sudo gpasswd -A "" groupname

# View group info
sudo gpasswd -i groupname  # Interactive
```

---

## Primary vs Supplementary Groups

### Primary Group (Login Shell)
- Set in `/etc/passwd` (4th field, GID reference)
- Used when creating files (files owned by user's primary group)
- One user = one primary group
- Changed with: `usermod -g GID_or_groupname`

```bash
# View user's primary group
id -gn username

# Change user's primary group
sudo usermod -g developers username
```

### Supplementary Groups
- Additional groups a user belongs to
- Used for permission sharing across multiple groups
- Unlimited number per user
- Set with: `usermod -aG group1,group2 username`

```bash
# View all user's groups (primary + supplementary)
id username

# View only supplementary groups
id -Gn username
```

---

## Practical Examples

### Setting Up Development Team

```bash
# 1. Create development group
sudo groupadd developers

# 2. Create shared directory
sudo mkdir -p /opt/dev-project
sudo chown root:developers /opt/dev-project
sudo chmod 2775 /opt/dev-project  # SGID for automatic group ownership

# 3. Add developers to group
sudo usermod -aG developers alice
sudo usermod -aG developers bob
sudo usermod -aG developers charlie

# 4. Verify
getent group developers
# developers:x:1001:alice,bob,charlie
```

### Project-Based Access Control

```bash
# Create project group
sudo groupadd project-alpha

# Add members
sudo usermod -aG project-alpha alice
sudo usermod -aG project-alpha bob

# Set directory permissions
sudo chgrp -R project-alpha /path/to/project
sudo chmod -R 2770 /path/to/project  # rwxrws--- (SGID)

# New files automatically owned by project-alpha group
```

### Managing Sudo Access via Groups

```bash
# Create admin group (alternative to wheel)
sudo groupadd admin

# Add user to admin group
sudo usermod -aG admin username

# Edit sudoers to allow admin group
sudo visudo
# Add: %admin ALL=(ALL) ALL

# Verify sudo access
sudo -l -U username
```

### Group for Shared Resources (e.g., /mnt/shared)

```bash
# 1. Create group
sudo groupadd shared-storage

# 2. Create mount point
sudo mkdir /mnt/shared

# 3. Configure group ownership and permissions
sudo chgrp shared-storage /mnt/shared
sudo chmod 2775 /mnt/shared  # SGID + rwxrwxr-x

# 4. Add users
sudo usermod -aG shared-storage user1
sudo usermod -aG shared-storage user2

# 5. Users must log out/in for group to take effect
# Or use: newgrp shared-storage (temporary switch)
```

---

## Special Groups

| Group | Purpose | Typical GID |
|-------|---------|-------------|
| `root` | Superuser | 0 |
| `wheel` | Sudo access (RHEL/CentOS) | 10 |
| `sudo` | Sudo access (Debian/Ubuntu) | 27 |
| `adm` | System monitoring | 4 |
| `cdrom` | CD-ROM access | 24 |
| `docker` | Docker daemon | 996 |
| `plugdev` | Plug devices access | 46 |
| `video` | Video devices | 39 |
| `audio` | Audio devices | 63 |
| `dialout` | Serial ports | 20 |
| `www-data` | Web server | 33 |
| `nobody` | Unprivileged user | 65534 |

---

## SGID (Set Group ID) for Shared Directories

SGID is powerful for shared directories — new files inherit the group, not the creator's primary group.

```bash
# Set SGID on directory
chmod 2775 /shared/dir  # 2 = SGID bit

# Verify SGID is set
ls -ld /shared/dir
# drwxrwsr-x  2 root group 4096 May  7  /shared/dir
#        ^ (s indicates SGID)

# Remove SGID
chmod 775 /shared/dir
```

### Why SGID for Shared Directories?
- User A creates file → file group = shared group (not user A's primary)
- User B in same group → can access/edit file immediately
- Without SGID: files would be owned by user's primary group, others can't access

---

## Group Passwords (gshadow)

Less commonly used, but groups can have passwords for temporary access:

```bash
# Set group password
sudo gpasswd groupname

# User joins group with password
newgrp groupname  # Prompts for password

# Exit temporary group
exit  # Returns to primary group

# Remove password
sudo gpasswd -r groupname
```

---

## Troubleshooting

### User Can't Access Shared Directory

```bash
# 1. Check group membership
groups username
id username

# 2. Check directory permissions
ls -ld /path/to/dir

# 3. Check group exists
getent group groupname

# 4. If user just added, they need to:
#    a) Log out and back in, OR
#    b) Run: newgrp groupname (temporary)
#    c) Start new shell with: sg groupname -c "command"
```

### "User is already a member" Error

```bash
# usermod -aG won't add if already member
# Use -a (append) to add without removing from other groups
sudo usermod -aG groupname username  # -a is critical

# If error persists, check exact group name (case sensitive)
getent group GroupName
```

### Files Have Wrong Group Owner

```bash
# Recursively change group ownership
sudo chgrp -R groupname /path/to/dir

# Fix SGID on directory
chmod 2775 /path/to/dir

# Find files not owned by correct group
find /path -not -group correctgroup -ls
```

---

## Best Practices

### ✅ DO's

1. **Use descriptive group names** — `project-alpha`, `dev-team`, not `g1`
2. **Use supplementary groups for access control** — not primary group
3. **Apply SGID to shared directories** — automatic group inheritance
4. **Keep primary groups for personal use** — one per user
5. **Use GID ranges consistently** — system: <1000, users: >1000
6. **Log group changes** — track who modifies group membership
7. **Audit group membership quarterly** — remove inactive users

### ❌ DON'Ts

1. **Don't use same group for everything** — separate concerns
2. **Don't put users in wheel/sudo unless needed** — principle of least privilege
3. **Don't leave groups with no members** — clean up stale groups
4. **Don't use group passwords in production** — harder to audit
5. **Don't mix user GIDs and system GIDs** — keep separate ranges

---

## Security Checklist

- [ ] All users in appropriate supplementary groups
- [ ] No unnecessary users in `wheel` or `sudo`
- [ ] Shared directories use SGID properly
- [ ] Stale groups with no members are removed
- [ ] GID ranges separated (system vs users)
- [ ] Group membership changes are logged
- [ ] Regular audit of `/etc/group` vs actual needs
- [ ] No shared accounts with broad group membership

---

## Quick Reference

```bash
# View current user's groups
groups

# View all groups
getent group

# Add user to group
sudo usermod -aG groupname username

# List group members
getent group groupname

# Change file group ownership
chgrp groupname file.txt

# Change directory group recursively
chgrp -R groupname /path/to/dir

# Set SGID on directory
chmod 2775 /path/to/dir

# View user's complete group list
id username

# Change primary group
sudo usermod -g groupname username

# Remove user from group
sudo gpasswd -d username groupname
```

---

## Related Topics

- `/etc/passwd` - User account information
- `/etc/login.defs` - UID/GID configuration
- `/etc/sudoers` - Sudo permissions
- File permissions (`chmod`, `chown`)
- SGID/SUID/Sticky bit special permissions

---

## Sources

- Linux man pages: `group(5)`, `groups(1)`, `groupadd(8)`, `groupdel(8)`, `groupmod(8)`, `gpasswd(1)`, `usermod(8)`
- `sg(1)` - execute command as different group
- `newgrp(1)` - change primary group
- Red Hat Enterprise Linux System Administration Guide

---

*Generated by Boti using general-researcher skill*