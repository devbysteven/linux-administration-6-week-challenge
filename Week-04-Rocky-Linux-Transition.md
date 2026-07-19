# Week 4: Rocky Linux Enterprise Transition, DNF Package Management & User Access

**Challenge:** 6-Week Linux System Administration Challenge
**Date:** 19/07/2026

## Goal

To deploy a fresh Rocky Linux installation, master the DNF package manager for enterprise environments, configure automated security updates, and implement strict role-based access controls using advanced user and group management.

## Tasks

* Established an SSH connection to the new Rocky Linux server.
* Created a new administrative user with sudo privileges by adding them to the `wheel` group.
* Updated the system and cleaned up cached package dependencies.
* Explored the `dnf` package manager to search, install, upgrade, and remove software.
* Added the EPEL repository to access enterprise packages like `htop`.
* Configured `dnf-automatic` to download and apply security updates automatically without rebooting.
* Navigated the directory structure and observed the differences in Nginx configuration directories (`conf.d`) on Rocky Linux.
* Managed multiple user accounts and groups, including locking passwords, viewing hashed credentials in the shadow file, and enforcing read-only directory permissions.

## Commands Used

### 1. Connection & Administrative User Setup

The following commands were used to log in as the default user, escalate to root, and create a permanent administrative user.

```bash
# Connect to the fresh Rocky Linux installation
ssh <default_user>@<ip_address>

# Verify current user context
whoami
id <default_user>

# Switch to the root user
sudo su -

# Create a new administrative user and set the password
useradd <admin_user>
passwd <admin_user>

# Test the new user and return to root
su - <admin_user>
exit

# Grant sudo permissions by adding the user to the wheel group
usermod -aG wheel <admin_user>

# Verify the user was successfully added to the wheel group
groups <admin_user>
```

#### Key Findings: User Setup

- In Enterprise Linux distributions like Rocky, the administrative group granting sudo access is called `wheel`, rather than `sudo`.  
- The `su -` command provides a full login shell for the target user.  

### 2. DNF Package Management & Automation

The `dnf` package manager replaced `apt` for all software installations, updates, and removals.  

```bash
# Check for available system updates and apply them
sudo dnf check-update
sudo dnf upgrade -y

# Clean up cached packages and remove unused dependencies
sudo dnf autoremove -y
sudo dnf clean all

# View help documentation and list all installed packages
dnf --help
dnf list installed

# Search for and install the 'tree' package
sudo dnf search tree
sudo dnf install tree -y

# Enable the Extra Packages for Enterprise Linux (EPEL) repository to find htop
sudo dnf search htop
sudo dnf install epel-release -y
sudo dnf search htop
sudo dnf install htop -y

# View recently updated packages
dnf list recent

# Install multiple packages simultaneously, then remove one
sudo dnf install nano fail2ban -y
sudo dnf remove fail2ban -y

# Reinstall or upgrade a specific package
sudo dnf reinstall htop
sudo dnf upgrade htop

# Install the DNF automatic updates service
sudo dnf install dnf-automatic
sudo nano /etc/dnf/automatic.conf
```

**File:** `/etc/dnf/automatic.conf` **(Excerpt)** Configured the file with the following parameters to ensure security patches are applied silently:  

```toml
upgrade_type = security
download_updates = yes
apply_updates = yes
reboot = never
```

#### Key Findings: Package Management

- The EPEL (`epel-release`) repository must be installed on Enterprise Linux to access common utility packages like `htop` that are not included in the standard repositories.
- The `dnf-automatic` utility replaces `unattended-upgrades` (from Ubuntu) to manage automated security patching.

### 3. File System Navigation & Nginx Observation

Basic directory commands were used to create, locate, and remove test files, alongside installing Nginx to observe file structure differences.

```bash
# Create nested directories and a text file
mkdir -p documents/work
nano todo.txt

# List directory contents and search for the specific file
ls
find ~/ -name "todo.txt"

# Change directories and delete the file
cd documents/work
rm documents/work/todo.txt

# Recursively delete the entire directory
rm -r documents

# Install Nginx and navigate to its configuration folder
sudo dnf install nginx -y
cd /etc/nginx/
ls
```

#### Key Findings: Directory Structures

- In Rocky Linux, Nginx utilises a `conf.d` directory for server blocks by default, differing from the `sites-available` and `sites-enabled` structure commonly used in Ubuntu.

### 4. Advanced User & Group Management

Multiple users and groups were created to test role-based access controls and account locking mechanisms.

```bash
# View options for creating users
useradd --help

# Create standard users and assign passwords
sudo useradd <user_1>
sudo useradd <user_2>
sudo useradd <user_3>
sudo passwd <user_1>
sudo passwd <user_2>
sudo passwd <user_3>

# Create departmental groups
sudo groupadd developers
sudo groupadd finance
sudo groupadd designers

# Add and remove users from secondary groups
sudo gpasswd -a <user_2> developers
sudo gpasswd -a <user_1> developers
sudo gpasswd -d <user_1> developers

# Change a user's primary group and verify
sudo usermod -gid developers <user_2>
groups <user_2>

# Rename and delete groups
sudo groupmod -n programmers developers
sudo groupdel designers

# Verify group deletion by checking the bottom of the group file
tail -n 5 /etc/group

# Create shared directories and assign group ownership to root
sudo mkdir -p /data/programmers /data/finance
sudo chown root:programmers /data/programmers
sudo chown root:finance /data/finance
ls -l /home

# View the hashed password, lock the account, and observe the hash change
sudo cat /etc/shadow | grep <user_2>
sudo passwd -l <user_2>
sudo cat /etc/shadow | grep <user_2>

# Unlock the user account
sudo passwd -u <user_2>

# View global user and group information
cat /etc/passwd
cat /etc/group

# Set strict read-only permissions for the finance group
sudo chmod 750 /data/finance

# Switch context and attempt to create a file (Requires relogin to apply groups)
cd /data/finance
touch hello.txt
```

#### Key Findings: Access Control

- Deleting a group (`groupdel`) will fail if it is currently set as a user's primary group, but will succeed if it is only a secondary group.
- Locking an account with `passwd -l` alters the password hash inside `/etc/shadow` (typically by prepending an `!`), which prevents the system from verifying logins.
- When group permissions or memberships are changed, the affected user must log out and back in for the new access levels to apply (demonstrated by the `touch hello.txt` permission denial).