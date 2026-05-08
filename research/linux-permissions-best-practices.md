# Linux Permissions Best Practices & Commands

## Research Date
2026-05-07

---

## Overview

Linux permissions control who can read, write, and execute files and directories. Understanding this system is fundamental for system administration, security, and effective Linux usage.

---

## The Permission Model

### The Three Types

| Type | Symbol | Numeric | Description |
|------|--------|---------|-------------|
| **Read** | `r` | 4 | View file contents or list directory |
| **Write** | `w` | 2 | Modify file contents or create/delete files in directory |
| **Execute** | `x` | 1 | Run file as program or access directory |

### Permission Groups

| Group | Symbol | Description |
|-------|--------|-------------|
| **Owner** | `u` | The user who owns the file |
| **Group** | `g` | Users who belong to the file's group |
| **Others** | `o` | Everyone else |
| **All** | `a` | Combination of u, g, and o |

### Example Output
```
-rwxr-xr-- 1 owner group 4096 May  7 22:18 script.sh
```

Breakdown:
- `-` → Regular file (d for directory)
- `rwx` → Owner permissions (read, write, execute)
- `r-x` → Group permissions (read, execute)
- `r--` → Others permissions (read only)

---

## Essential Commands

### Viewing Permissions

```bash
# Long listing format
ls -l file.txt

# List all files including hidden
ls -la

# Show numeric permissions
stat -c '%a' file.txt
```

### Changing Permissions (chmod)

```bash
# Using symbols
chmod u+x script.sh           # Add execute for owner
chmod g-w file.txt             # Remove write for group
chmod o+r file.txt             # Add read for others
chmod a+x script.sh            # Add execute for all
chmod 755 script.sh            # rwxr-xr-x (owner: rwx, group: r-x, others: r-x)
chmod 644 file.txt             # rw-r--r-- (owner: rw-, group: r--, others: r--)
chmod 600 file.txt             # rw------- (owner: rw-, group: ---, others: ---)
chmod 400 secret.key           # r-------- (owner: r--, group: ---, others: ---)

# Recursive (directories only)
chmod -R 755 /path/to/dir
```

### Changing Owner & Group (chown, chgrp)

```bash
# Change owner
sudo chown newowner file.txt

# Change group
sudo chgrp newgroup file.txt

# Change both owner and group
sudo chown newowner:newgroup file.txt

# Change owner recursively
sudo chown -R newowner /path/to/dir

# Change owner and group recursively
sudo chown -R newowner:newgroup /path/to/dir
```

---

## Special Permissions

### SUID (Set User ID)

Allows a user to run a program with the permissions of the file's owner.

```bash
# Set SUID
chmod u+s /usr/bin/program

# Numeric: 4755 (4 = SUID)
chmod 4755 /usr/bin/program

# Shows as 's' in owner execute position
# -rwsr-xr-x
```

**⚠️ Security Warning:** SUID binaries are a common attack vector. Use sparingly and audit regularly.

### SGID (Set Group ID)

Similar to SUID but for groups. Useful for shared directories.

```bash
# Set SGID
chmod g+s /path/to/shared-dir

# Numeric: 2755
chmod 2755 /path/to/shared-dir

# Shows as 's' in group execute position
# drwxr-sr-x
```

### Sticky Bit

On directories, only the owner of a file can delete it (even if others have write permission).

```bash
# Set sticky bit on /tmp-like directory
chmod +t /path/to/shared-dir

# Numeric: 1777
chmod 1777 /path/to/shared-dir

# Shows as 't' in others execute position
# drwxrwxrwt
```

---

## Common Permission Patterns

| Scenario | Command | Numeric |
|----------|---------|---------|
| Executable script | `chmod +x script.sh` | 755 |
| Regular file | `chmod 644 file.txt` | 644 |
| Private file | `chmod 600 file.txt` | 600 |
| Private key | `chmod 600 id_rsa` | 600 |
| Public key | `chmod 644 id_rsa.pub` | 644 |
| Web directory | `chmod 755 /var/www` | 755 |
| Web file | `chmod 644 /var/www/html/*.html` | 644 |
| Shared group dir | `chmod 2775 /shared` | 2775 |
| Temp directory | `chmod 1777 /tmp` | 1777 |

---

## Umask

Umask determines the default permissions for newly created files.

```bash
# View current umask
umask

# Set umask for session
umask 022

# Common umasks:
# 022 → files: 644, dirs: 755 (standard)
# 002 → files: 664, dirs: 775 (groups)
# 077 → files: 600, dirs: 700 (most restrictive)
```

---

## ACL (Access Control Lists)

For more granular permissions beyond traditional Unix permissions.

### Viewing ACLs

```bash
# Show ACL for a file
getfacl file.txt
```

### Setting ACLs

```bash
# Add permission for specific user
setfacl -m u:username:rx file.txt

# Add permission for specific group
setfacl -m g:groupname:rw file.txt

# Remove ACL entry
setfacl -x u:username file.txt

# Set default ACL for directory (inherited by new files)
setfacl -m d:u:username:rx /path/to/dir

# Remove all ACLs
setfacl -b file.txt
```

---

## Troubleshooting

### "Permission denied" errors

```bash
# Check ownership and permissions
ls -la /path/to/problematic-file

# Check directory permissions (need 'x' to access)
ls -ld /path/to/problematic-dir

# Am I in the right group?
groups

# Check ACLs
getfacl /path/to/problematic-file
```

### Fixing common issues

```bash
# Fix ownership of your files
sudo chown -R $USER:$USER ~/your-project

# Fix web server access
sudo chown -R www-data:www-data /var/www
sudo chmod -R 755 /var/www
sudo chmod -R 644 /var/www/*.html

# Fix SSH key permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

---

## Best Practices

### ✅ DO's

1. **Principle of least privilege** — Give only the permissions needed
2. **Use groups for shared access** — Instead of 'other' permissions
3. **Audit SUID/SGID binaries regularly** — `find / -perm -4000 -o -perm -2000`
4. **Protect SSH keys** — Always `chmod 600` for private keys
5. **Use ACLs for complex scenarios** — More flexible than traditional permissions
6. **Set proper umask** — 022 or 002 for shared environments
7. **Log permission changes** — Track who changes what

### ❌ DON'Ts

1. **Never chmod 777** — Massive security risk
2. **Don't set SUID on scripts** — Use sudo instead
3. **Don't leave files owned by root** — In user directories
4. **Don't ignore 'other' permissions** — They affect everyone
5. **Don't use world-writable directories** — /tmp should have sticky bit

---

## Security Checklist

- [ ] SSH private keys: `600`
- [ ] SSH public keys: `644`
- [ ] `.ssh` directory: `700`
- [ ] Sensitive files: `600`
- [ ] `/tmp` has sticky bit: `1777`
- [ ] SUID binaries audited
- [ ] No files with `777`
- [ ] Group memberships reviewed

---

## Quick Reference

```bash
# Permission to numeric
rwx = 4+2+1 = 7
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4

# Common numerics
777 = rwxrwxrwx  ⚠️ AVOID
755 = rwxr-xr-x
754 = rwxr-xr--
744 = rwxr--r--
644 = rw-r--r--
600 = rw-------
400 = r--------
2755 = rwxr-sr-x (SGID)
4755 = rwsr-xr-x (SUID)
1777 = rwxrwxrwt (sticky bit)
```

---

## Sources

- Linux man pages: `man chmod`, `man chown`, `man acl`
- https://www.redhat.com/sysadmin/linux-permissions
- https://www.kernel.org/doc/html/latest/admin-guide/devices.html

---

*Generated by Boti using general-researcher skill*