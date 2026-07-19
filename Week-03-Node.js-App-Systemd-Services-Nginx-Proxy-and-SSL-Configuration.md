# Week 3: Node.js App, Systemd Services, Nginx Proxy & SSL Configuration

**Challenge:** 6-Week Linux System Administration Challenge

**Date:** 12/07/2026

## Goal

To deploy a modern Node.js web application, configure it to run persistently as a background service using Systemd, securely route web traffic to it via an Nginx reverse proxy, and encrypt the connection using a Let's Encrypt SSL certificate.

## Tasks

* Verified secure environment access using the custom SSH port and public key authentication established in previous weeks.
* Installed via NodeSource v24 and initialised a new Node.js project using Express.
* Configured the application to utilise modern ES Module syntax (`import`) rather than CommonJS.
* Authored a custom Systemd unit file to ensure the Node.js application runs continuously and restarts automatically upon failure or system reboot.
* Installed Nginx, removed default conflicting configurations, and created a server block to reverse proxy incoming HTTP traffic to the internal Node.js port (3000).
* Secured the domain with an SSL certificate using Certbot, enabling encrypted HTTPS traffic.

## Commands Used

### 1. Secure Access & Environment Verification (Week 2 Recap)

The following commands were used to access the server via the secured port and verify the application of previous firewall and SSH rules.

```bash
# Connect using the configured custom port and identity key
ssh -p <custom_port> <username>@<ip_address>

# Verify previous setup configurations (SSH keys and UFW status)
sudo grep "Accepted" /var/log/auth.log | tail -n 5
sudo ufw status numbered
```

### 2. Node.js Installation & Application Setup

These commands fetched the official NodeSource repository, installed Node.js, and initialised a basic Express application.

```bash
# Install curl to fetch external data
sudo apt install -y curl

# Fetch NodeSource v24 and pipe it to bash (preserving environment variables with -E)
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -

# Install Node.js and verify the version
sudo apt install -y nodejs
node -v

# Create the project directory and initialise the npm project
mkdir ~/node-challenge
cd ~/node-challenge
npm init -y
npm install express
```

**File:** `~/node-challenge/package.json` **(Excerpt)** Updated the initialised file to include modern module syntax:

```json
{
  "name": "vps challenge",
  "version": "1.0.0",
  "main": "index.js",
  "type": "module",
  "dependencies": {
    "express": "^5.2.1"
  }
}
```

**File:** `~/node-challenge/index.js` Created the server logic to listen on port 3000:

```javascript
import express from 'express';

const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.send('Node Challenge');
});

app.listen(port, () => {
  console.log(`Server running on http://localhost:${port}`);
});
```

#### Key Findings: Application Setup

- The `-E` flag in `sudo -E bash -` ensures the user's environment variables are preserved when executing the installation script.
- Adding `"type": "module"` to `package.json` is strictly required to use the modern `import express from 'express'` syntax instead of the legacy `require()` function.

### 3. Systemd Daemon Configuration

Systemd was configured to manage the Node application as a background service.

```bash
# Create the service configuration file
sudo nano /etc/systemd/system/node-challenge.service

# Reload the systemd daemon to register the new file
sudo systemctl daemon-reload

# Start the service and enable it to run automatically on server boot
sudo systemctl start node-challenge
sudo systemctl enable node-challenge
```

**File:** `/etc/systemd/system/node-challenge.service`

```ini
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

#### Key Findings: Service Management

- Systemd provides enterprise-grade process management, ensuring the application stays online even after a crash (`Restart=on-failure`) or system reboot (`WantedBy=multi-user.target`).

### 4. Nginx Reverse Proxy Setup

Nginx was installed to act as the front-facing web server, securely passing traffic to the Node.js application behind the scenes.

```bash
# Install Nginx
sudo apt install nginx -y

# Create the server block configuration
sudo nano /etc/nginx/sites-available/node-challenge

# Create a symbolic link to enable the site configuration
sudo ln -s /etc/nginx/sites-available/node-challenge /etc/nginx/sites-enabled/

# Remove the default configuration to prevent routing conflicts
sudo rm -f /etc/nginx/sites-enabled/default

# Test the Nginx configuration for syntax errors and restart
sudo nginx -t
sudo systemctl restart nginx
```

**File:** `/etc/nginx/sites-available/node-challenge`

```nginx
server {
    listen 80;
    server_name <domain_name>;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

#### Key Findings: Reverse Proxying

- By setting `proxy_pass `[`http://localhost:3000`](http://localhost:3000)`;`, Nginx intercepts incoming port 80 traffic and funnels it to the internal Node app, preventing direct external access to port 3000.
- Using `ln -s` creates a symbolic link, a core Linux concept that allows you to keep configuration files stored in `sites-available` while selectively toggling them active in `sites-enabled`.

### 5. Certbot SSL Configuration

Traffic was encrypted by installing an automated SSL certificate.

```bash
# Install Certbot and the Nginx plugin
sudo apt install -y certbot python3-certbot-nginx

# Execute Certbot to automatically configure SSL for the domain
sudo certbot --nginx -d <domain_name>
```

#### Key Findings: Encryption

- The `python3-certbot-nginx` plugin allows Certbot to automatically read the Nginx configuration, issue the certificate from Let's Encrypt, and safely rewrite the Nginx server block to enforce HTTPS connections.