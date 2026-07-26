# Week 5: Rocky Linux Security, Firewalld & SELinux

**Challenge:** 6-Week Linux System Administration Challenge
**Date:** 26/07/2026

## Goal

To establish secure user access, configure SSH key-based authentication, manage network security using Firewalld, and monitor and configure SELinux enforcing modes on a Rocky Linux enterprise environment.

## Tasks

* Created a new administrative user with `wheel` privileges to securely manage the server.
* Updated system packages, cleaned cached dependencies, and installed required repositories such as EPEL.
* Secured SSH access by configuring public keys, disabling password authentication, and verifying connections via system logs.
* Installed, enabled, and configured `firewalld` to manage permanent services and ports.
* Interrogated SELinux policies, audited denial logs in real-time, and managed enforcing and permissive states.

## Commands Used

### 1. Connection & Administrative User Setup

The following commands were used to log in, create a permanent administrative user, and update the system packages.

```bash
# Connect to the Rocky Linux installation
ssh rocky@<ip_address>

# Create a new administrative user and set the password
sudo useradd <username>
sudo passwd <username>

# Grant sudo permissions by adding the user to the wheel group
sudo usermod -aG wheel <username>

# Switch to the new user and verify group membership
su - <username>
groups <username>

# Return to the default user and reconnect as the new administrator
exit
ssh <username>@<ip_address>

# Check for available system updates and apply them
sudo dnf check-update && sudo dnf upgrade -y
sudo dnf upgrade -y

# Clean cached packages
sudo dnf autoremove -y
sudo dnf clean all

# Add the Extra Packages for Enterprise Linux (EPEL) repository and install nano
sudo dnf install epel-release -y
sudo dnf install nano
```

#### Key Findings: User Setup & Package Management

- Rocky Linux utilises the `wheel` group for sudo privileges.  
- The `&&` operator effectively chains the update and upgrade commands, while `dnf autoremove` and `dnf clean all` keep the system free of redundant packages.  

### 2. Secure Server Configuration (SSH)

These commands secured the SSH daemon using ED25519 keys and restricted root access.  

```bash
# Display the local public key (Run on local machine)
cat ~/.ssh/id_ed25519.pub

# Create directories, add public key, and set strict permissions
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys 
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Check secure logs to confirm successful key-based login
sudo grep "Accepted" /var/log/secure | tail -n 5

# Edit the SSH daemon configuration file
sudo nano /etc/ssh/sshd_config

# Test the SSH configuration for syntax errors before restarting
sudo sshd -t

# Restart the SSH daemon to apply changes
sudo systemctl restart sshd

# Filter SSH logs to see real-time login attempts
sudo journalctl -u sshd -f 

# View all accepted SSH logins via journalctl
sudo journalctl -u sshd | grep "Accepted"
```

**File:** `/etc/ssh/sshd_config` **(Excerpt)** Updated the configuration to enforce secure access:  

```TOML
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

#### Key Findings: Securing SSH on Rocky Linux

- Unlike Ubuntu, which logs authentications to `/var/log/auth.log`, Rocky Linux logs authentication events to `/var/log/secure`.  
- The SSH service on Enterprise Linux is managed as `sshd` rather than `ssh`.  
- Running `sshd -t` is a crucial safety check to catch syntax errors before restarting the service and potentially locking yourself out.  

### 3. Firewall Setup & Configuration (Firewalld)

The default enterprise firewall manager, `firewalld`, was installed and configured to control network traffic.  

```bash
# Install Firewalld
sudo dnf install firewalld -y

# Start and enable the service on boot (or disable/stop as needed)
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo systemctl stop firewalld
sudo systemctl disable firewalld

# Reload the system daemon (required on some minimal images) and enable immediately
sudo systemctl daemon-reload
sudo systemctl enable --now firewalld

# Check Firewalld state and service status
sudo firewall-cmd --state
sudo systemctl status firewalld

# Display all active rules, ports, and services in the public zone
sudo firewall-cmd --list-all --zone=public
sudo firewall-cmd --list-ports
sudo firewall-cmd --list-services
sudo firewall-cmd --list-all

# Open HTTP/HTTPS services permanently and reload to apply
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# Inspect what specific ports a service contains (eg. http)
sudo firewall-cmd --info-service=http

# Open a specific port, check it, and remove it
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --remove-port=8080/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

#### Key Findings: Managing Firewalld

- `firewalld` utilises the `--permanent` flag to ensure rules survive a reboot.  
- Any changes made with `--permanent` require a `firewall-cmd --reload` command to become actively enforced.  

### 4. Security-Enhanced Linux (SELinux)

SELinux policies were interrogated and logs were audited to understand enterprise access controls.  

```bash
# Check the current SELinux enforcement mode and status
getenforce
sestatus

# View the raw audit log where SELinux records all events
sudo tail -f /var/log/audit/audit.log

# Filter the audit log to see only SELinux denials (last 10 or live)
sudo grep denied /var/log/audit/audit.log
sudo grep denied /var/log/audit/audit.log | tail -n 10
sudo tail -n 10 /var/log/audit/audit.log
sudo tail -n 10 -f /var/log/audit/audit.log

# Temporarily switch SELinux modes (0 = Permissive, 1 = Enforcing)
sudo setenforce 0
getenforce
sudo setenforce 1
getenforce

# Edit the SELinux configuration file to make changes permanent across reboots
sudo nano /etc/selinux/config

# Search the Access Vector Cache (AVC) for specific denials and activity
sudo ausearch -m avc
sudo ausearch -m avc -ts recent
sudo ausearch -m avc -c httpd
sudo ausearch -m avc -c sshd

# Search for specific authentication and login events
sudo ausearch -m USER_LOGIN
sudo ausearch -m USER_AUTH

# View the last 10 audit messages for logins or specific commands
sudo ausearch -m USER_LOGIN | tail -n 10
sudo ausearch -c sshd | tail -n 10
```

**File:** `/etc/selinux/config` **(Excerpt)**

```TOML
SELINUX=enforcing
```

#### Key Findings: SELinux Auditing

- SELinux logs its events, including permission denials, in `/var/log/audit/audit.log`, which should not be manually edited.  
- The `setenforce` command toggles modes temporarily, whereas `/etc/selinux/config` dictates the persistent state upon reboot.
- The `ausearch` utility provides a powerful, filtered way to query the audit logs for specific services (`-c`) or message types (`-m`).