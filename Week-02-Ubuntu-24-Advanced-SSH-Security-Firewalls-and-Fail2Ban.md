# Week 2: Advanced SSH Security, Firewalls & Fail2Ban

**Challenge:** 6-Week Linux System Administration Challenge
**Date:** 05/07/2026

## Goal

To establish secure SSH connections using ED25519 keys, configure custom SSH ports with UFW and OVH Edge firewalls, and implement intrusion prevention using Fail2Ban.

## Tasks

* Removed old IP addresses from known hosts to resolve host ID change warnings following a new installation.
* Created new user accounts, assigned supplementary groups, and applied strict directory permissions for role-based access.
* Generated ED25519 SSH keys and disabled password authentication to secure login procedures.
* Changed the default SSH port to 50222 and configured UFW to allow specific web and SSH traffic.
* Configured OVH Cloud Edge Network Firewall rules to authorise specific ports and drop unauthorised TCP traffic.
* Installed Fail2Ban, created a custom local configuration, and whitelisted a home IP address to prevent administrative lockouts..

## Commands Used

### 1. Connection & User Management Review

The following commands were used to handle SSH warnings and manage directory permissions.

```bash
# Remove an IP address from the known hosts file to overcome host ID change warnings
ssh-keygen -f '/home/<username>/.ssh/known_hosts' -R '<ip_address>'

# Update and upgrade package list
sudo apt update && sudo apt upgrade -y

# Create users, add to groups, and build directory structures
sudo adduser andy
sudo adduser betty
sudo groupadd developers
sudo groupadd hr
sudo usermod -aG developers andy
sudo usermod -aG hr betty
sudo mkdir -p /data/developers /data/hr

# Assign ownership and strict octal permissions
sudo chown root:developers /data/developers
sudo chown root:hr /data/hr
sudo chmod 750 /data/hr
sudo chmod 770 /data/developers

# System monitoring and file removal
htop
rm -r new-project
```

#### Key Findings: File Management and Monitoring

* The `-R` flag in `ssh-keygen` is necessary to remove outdated IP records when connecting to a freshly installed server.
* Recursive removal using `rm -r` deletes a directory and all of its contents, requiring extreme caution.
* The `htop` command launches a progress monitor to check system statistics such as server uptime.

### 2. SSH Key Generation & Configuration

These commands generated secure keys and disabled vulnerable password logins.

```bash
# Generate a new ED25519 key with 100 derivation rounds and a comment
ssh-keygen -t ed25519 -a 100 -C "key_name"

# Copy the public key and set up the authorized_keys file on the server
cat ~/.ssh/id_ed25519.pub
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Test the connection
ssh <username>@<ip_address>

# Audit the authentication logs to confirm public key acceptance
sudo grep "Accepted" /var/log/auth.log | tail -n 5

# Locate and disable password authentication overrides
sudo grep -Hn "PasswordAuthentication" /etc/ssh/sshd_config.d/*
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo systemctl restart ssh
```

#### Key Findings: Securing SSH

* The `-t` flag defines the algorithm type, the `-a` flag dictates the key derivation function rounds, and `-C` adds a descriptive comment to the key.
* By saving the key in the default location (`~/.ssh/id_ed25519`), the SSH client automatically offers it during login without requiring the `-i` identity flag.
* Disabling `PasswordAuthentication` may require updating external configuration files like `50-cloud-init.conf` to fully apply the restriction.

### 3. Custom SSH Port & UFW Firewall

The internal system socket and Uncomplicated Firewall (UFW) were updated to lock down standard access points.

```bash
# Edit the SSH socket to change the listening port to 50222
sudo systemctl edit ssh.socket

# Reload the system daemon and restart the SSH services
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
sudo systemctl restart ssh

# Install, configure, and enable the Uncomplicated Firewall (UFW)
sudo apt install ufw
sudo ufw allow 50222/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# Verify UFW status and test the new port connection
sudo ufw status verbose
ssh -p 50222 <username>@<ip_address>
```

#### Key Findings: Port Management and Edge Firewalls

* Adding `ListenStream=50222` to the `ssh.socket` override file directs the system to monitor the new custom port.
* The OVH Cloud Edge Network Firewall requires authorisation rules for port 50222, web traffic (ports 80 and 443), and system updates (port 18).
* A "deny" rule must be created in the Edge Firewall to block all other traffic, but it should only be enabled after the internal server configuration is completely verified.
  
  

### 4. Intrusion Prevention (Fail2Ban)

Fail2Ban was installed to monitor logs and automatically ban suspicious IP addresses.

```bash
# Install Fail2Ban automatically confirming prompts
sudo apt install fail2ban -y

# Start the service and enable it to run on boot
sudo systemctl start fail2ban
sudo systemctl enable fail2ban

# Copy the master configuration to a local file to prevent overwrite on updates
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local

# Check the Fail2Ban status and view the real-time block log
sudo fail2ban-client status sshd
sudo fail2ban-client get sshd ignoreip
sudo tail -f /var/log/fail2ban.log

# Review the last 20 failed login attempts
sudo lastb | head -n 20
```

**File: `/etc/fail2ban/jail.local`**

```ini, TOML
[DEFAULT]
# Whitelist 
ignoreip = 127.0.0.1/8 ::1 <home_ip_address>

# Ban Settings
bantime  = 24h
findtime = 10m
maxretry = 3

# Incremental Banning
bantime.increment = true
bantime.factor = 1
bantime.maxtime = 5w

[sshd]
enabled = true
port    = 50222
```

#### Key Findings: Configuring Jails

* The `jail.local` configuration file takes precedence over the master `jail.conf` file, ensuring custom settings are preserved during distribution updates.
* The Fail2Ban configuration utilises `bantime.increment = true` to increase penalties for repeat offenders.
* The `tail -f` command allows administrators to view log file updates in real time to monitor active bans.

---

## Error/Resolution Log

### 1. SSH Host Key Mismatch

* **Error:** `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`
* **Symptom:** The initial SSH connection to the freshly provisioned VPS was refused due to a strict host key checking failure.
* **Root Cause:** Wiping and reinstalling the OS on the existing IP address generated a new cryptographic host key. The local system actively blocked the connection upon detecting a mismatch between this new key and the old key stored in the local `known_hosts` file.
* **Resolution:** Purged the stale IP record from the local machine to reset the verification process using:
  `ssh-keygen -f '/home/<username>/.ssh/known_hosts' -R '<ip_address>'`

### 2. Password Authentication Override

* **Error:** Password login remained active despite being disabled in the primary configuration.
* **Symptom:** The server continued to prompt for a password during login even after setting `PasswordAuthentication no` in `/etc/ssh/sshd_config` and restarting the SSH service.
* **Root Cause:** A secondary cloud-provider drop-in file was silently overriding the primary SSH daemon configuration. 
* **Resolution:** Searched the configuration directory for conflicting directives using:
  `sudo grep -Hn "PasswordAuthentication" /etc/ssh/sshd_config.d/*`
  This isolated the active override to `/etc/ssh/sshd_config.d/50-cloud-init.conf`. After updating this specific drop-in file, a final `sudo systemctl restart ssh` successfully enforced the key-only authentication policy.
