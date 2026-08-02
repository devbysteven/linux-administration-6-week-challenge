# Week 6: Rocky Production Deployment

**Challenge:** 6-Week Linux System Administration Challenge  
**Date:** 02/08/2026

## Goal

To complete the enterprise transition by deploying a Node.js application behind an Nginx reverse proxy, securing a custom SSH port with SELinux contexts, and hardening the perimeter using Firewalld and Fail2Ban.

## Tasks

- Created a permanent administrative user with `wheel` group privileges.
- Updated system packages, cleaned caches, and installed the EPEL repository.
- Secured SSH access via ED25519 public keys and disabled password authentication.
- Configured `firewalld` to allow HTTP, HTTPS, and a custom SSH port (50222), while removing default port 22 access.
- Configured SELinux to enforce security policies and applied a custom port context using `semanage`.
- Deployed a modern Node.js web application and configured it to run persistently as a Systemd service.
- Configured Nginx as a reverse proxy and secured the domain with Let's Encrypt SSL via Certbot.
- Updated SELinux booleans to authorise Nginx to connect to the internal Node network port.
- Installed and configured Fail2Ban with Firewalld integration to prevent brute-force attacks.

## Commands Used

### 1. Connection, User Setup & System Updates

The following commands were used to establish a secure connection, create an administrative user, and apply system updates.

```bash
# Connect to the Rocky Linux installation
ssh <default_user>@<ip_address>

# Create a new administrative user and set the password  
sudo useradd <username>  
sudo passwd <username>

# Grant sudo permissions by adding the user to the wheel group  
sudo usermod -aG wheel <username>

# Switch to the new user and verify group membership  
su - <username>  
groups <username>

# Return to the default user, exit, and reconnect as the administrator  
exit  
ssh <username>@<ip_address>

# Check for updates and install them  
sudo dnf check-update && sudo dnf upgrade -y

# Clean cached packages  
sudo dnf autoremove -y  
sudo dnf clean all

# Install EPEL repository and basic utilities  
sudo dnf install epel-release -y  
sudo dnf install nano -y
```

### 2. Custom SSH Port, Firewalld & SELinux Port Contexts

The SSH service was moved to a non-standard port. This required explicitly telling SELinux and Firewalld to allow traffic on the new port before restarting the service.  

```bash
# Set up the SSH directory and apply strict permissions for the identity key
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys 
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Check secure logs to confirm successful key-based login
sudo grep "Accepted" /var/log/secure | tail -n 5

# Edit the SSH daemon configuration file to disable passwords and change the port
sudo nano /etc/ssh/sshd_config

# Tell SELinux to allow SSH to listen on the new custom port (50222)
sudo semanage port -a -t ssh_port_t -p tcp 50222

# Configure Firewalld to start automatically and open standard web ports
sudo dnf install firewalld -y
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

# Add the new custom SSH port to the firewall
sudo firewall-cmd --permanent --add-port=50222/tcp
sudo firewall-cmd --reload

# Test SSH configuration syntax and restart the daemon
sudo sshd -t
sudo systemctl restart sshd

# Connect via the new port (requires purging the old known_hosts record)
ssh-keygen -f "/home/<username>/.ssh/known_hosts" -R "[<ip_address>]:50222"
ssh -p 50222 <username>@<ip_address>

# Remove the default SSH port (22) from the firewall to lock down the perimeter
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --reload
sudo firewall-cmd --list-all

# Ensure SELinux remains strictly in enforcing mode
getenforce
sudo nano /etc/selinux/config
```

#### Key Findings: Custom Ports on Enterprise Linux

- Changing the SSH port on Rocky Linux requires using the `semanage port` command so SELinux recognises the new port as a valid `ssh_port_t` target.  
- Old host keys associated with the custom IP and port combination must be purged from the local `known_hosts` file using `ssh-keygen -R` to prevent security warnings.  

### 3. Application Deployment & Systemd Service

A modern Express web server was deployed using the NodeSource repository and managed via Systemd.  

```bash
# Fetch the NodeSource v24 repository and install Node.js
curl -fsSL https://rpm.nodesource.com/setup_24.x | sudo -E bash -
sudo dnf install nodejs -y
node -v

# Initialise the project and install Express
mkdir -p ~/node-challenge
cd ~/node-challenge
npm init -y
npm install express

# Update package.json to use modern ES modules ("type": "module")
nano package.json

# Create and configure the systemd unit file
sudo nano /etc/systemd/system/node-challenge.service

# Reload the daemon, start the service, and enable it on boot
sudo systemctl daemon-reload
sudo systemctl start node-challenge
sudo systemctl enable node-challenge
```

**File:** `/etc/systemd/system/node-challenge.service`

```TOML
[Unit]
Description=Node Express Challenge
After=network.target

[Service]
Environment=NODE_ENV=production
Environment=PORT=3000
Type=simple
User=<username>
WorkingDirectory=/home/<username>/node-challenge
ExecStart=/usr/bin/node index.js
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
LimitNOFILE=50000

[Install]
WantedBy=multi-user.target
```

### 4. Nginx Reverse Proxy, SSL & SELinux Booleans

Nginx was configured to serve the application securely, requiring a specific SELinux boolean toggle to allow proxying.  

```bash
# Install Nginx and Curl
sudo dnf install nginx curl -y

# Create the Nginx server block configuration
sudo nano /etc/nginx/conf.d/node-challenge.conf

# Check Nginx syntax and restart the service
sudo nginx -t
sudo systemctl restart nginx

# Install Certbot and the Nginx plugin
sudo dnf install -y certbot python3-certbot-nginx
certbot --version

# Generate the Let's Encrypt SSL certificate
sudo certbot --nginx -d <your_domain_name>

# Tell SELinux to allow the Nginx web server to initiate network connections (proxying)
sudo setsebool -P httpd_can_network_connect 1

# Verify the connection headers locally and over HTTPS
curl -I http://localhost:3000
curl -I https://<your_domain_name>
```

#### Key Findings: SELinux Web Proxying

- By default, SELinux prevents the web server (`httpd`/`nginx`) from initiating outbound network connections.
- The command `setsebool -P httpd_can_network_connect 1` is strictly required to allow Nginx to act as a reverse proxy and forward traffic to the internal Node.js port (3000).

### 5. Intrusion Prevention (Fail2Ban & Firewalld)

Fail2Ban was installed and configured to integrate directly with Firewalld rather than standard iptables.

```bash
# Install Fail2Ban and the specific Firewalld integration package
sudo dnf install fail2ban fail2ban-firewalld -y

# Start and enable the service
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
fail2ban-client --version

# Copy the default configuration to create a safe local override
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local

# Test the configuration and restart the service
sudo fail2ban-client -t
sudo systemctl restart fail2ban

# Monitor real-time Fail2Ban logs and check the status of the SSH jail
sudo journalctl -u fail2ban -f
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

**File:** `/etc/fail2ban/jail.local` **(Excerpt)**

```TOML
[sshd]
enabled = true
port = 50222
ignoreip = 127.0.0.1/8 ::1 <home_ip_address>
bantime  = 24h
findtime = 10m
maxretry = 3
bantime.increment = true
bantime.factor = 1
bantime.maxtime = 5w
```

#### Key Findings: Fail2Ban Integration

- On Enterprise Linux systems running Firewalld, the `fail2ban-firewalld` package must be installed to ensure Fail2Ban routes its ban rules through the correct firewall manager