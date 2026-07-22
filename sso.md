# Linux Users, Groups, and Permissions – Quick Reference

## 1. Users in Linux

A **user** is an account that can log in and use the system.
### Types of Users

* **Root User (UID 0):** Has complete control over the system.
* **Regular User:** Used for daily tasks with limited privileges.
* **System User:** Created for running services (e.g., nginx, mysql).

### Useful Commands

```bash
whoami          # Show current user
id              # Display UID, GID, and groups
who             # Show logged-in users
passwd          # Change password
sudo command    # Run a command as root
```

---

## 2. Groups in Linux

A **group** is a collection of users used to manage permissions efficiently.

### Why Use Groups?

* Simplifies permission management.
* Allows multiple users to access the same resources.

### Useful Commands

```bash
groups                  # Show groups of current user
groupadd developers     # Create a group
groupdel developers     # Delete a group
usermod -aG developers alice   # Add user to a group
```

---

## 3. File Ownership

Every file and directory has:

* **Owner (User)**
* **Group**
* **Others**

View ownership:

```bash
ls -l
```

Example:

```text
-rw-r--r-- 1 alice developers 1200 Jul 23 notes.txt
```

Change ownership:

```bash
chown bob notes.txt
chown bob:developers notes.txt
chgrp developers notes.txt
```

---

## 4. Linux File Permissions

Permission Types:

* **r (Read)** = 4
* **w (Write)** = 2
* **x (Execute)** = 1

Example:

```text
-rwxr-xr--
```

| User   | Permission | Numeric |
| ------ | ---------- | ------- |
| Owner  | rwx        | 7       |
| Group  | r-x        | 5       |
| Others | r--        | 4       |

Numeric permission:

```text
754
```

Change permissions:

```bash
chmod 755 script.sh
chmod u+x script.sh
chmod g-w file.txt
chmod o+r file.txt
```

---

## 5. Permission Examples

```bash
chmod 644 file.txt
```

* Owner: Read + Write
* Group: Read
* Others: Read

```bash
chmod 755 script.sh
```

* Owner: Read + Write + Execute
* Group: Read + Execute
* Others: Read + Execute

---

## 6. Best Practices

* Use **sudo** instead of logging in as root.
* Follow the **Principle of Least Privilege**.
* Grant only the permissions users actually need.
* Manage access using **groups** rather than assigning permissions user by user.

### Quick Cheat Sheet

```bash
whoami                    # Current user
id                        # User and group IDs
groups                    # User's groups
ls -l                     # View permissions
chmod 755 file            # Change permissions
chown user file           # Change owner
chown user:group file     # Change owner and group
groupadd groupname        # Create group
usermod -aG group user    # Add user to group
```
