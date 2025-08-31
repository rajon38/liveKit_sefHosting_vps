# LiveKit Self-Hosting with PM2 Guide

## Prerequisites
- VPS with Ubuntu 20.04+ (recommended 2+ CPU cores, 4GB+ RAM)
- Domain name pointed to your VPS IP
- PM2 already installed
- Node.js installed
- Root or sudo access

## Step 1: Install Redis (Required for LiveKit)

```bash
# Install Redis
sudo apt update
sudo apt install redis-server -y

# Configure Redis
sudo systemctl enable redis-server
sudo systemctl start redis-server

# Test Redis
redis-cli ping
# Should return: PONG
```

## Step 2: Download and Install LiveKit Server Binary

```bash
# Create LiveKit directory
mkdir ~/livekit-server
cd ~/livekit-server

# Download latest LiveKit server binary
wget https://github.com/livekit/livekit/releases/latest/download/livekit_linux_amd64.tar.gz

# Extract
tar -xzf livekit_linux_amd64.tar.gz

# Make executable
chmod +x livekit-server

# Move to system path (optional)
sudo mv livekit-server /usr/local/bin/
```

## Step 3: Set Up SSL Certificate (Let's Encrypt)

```bash
# Install Certbot
sudo apt install certbot -y

# Get SSL certificate
sudo certbot certonly --standalone -d yourdomain.com

# Create certs directory and copy certificates
mkdir -p ~/livekit-server/certs
sudo cp /etc/letsencrypt/live/yourdomain.com/fullchain.pem ~/livekit-server/certs/
sudo cp /etc/letsencrypt/live/yourdomain.com/privkey.pem ~/livekit-server/certs/

# Change ownership
sudo chown $USER:$USER ~/livekit-server/certs/*
```

## Step 4: Generate API Keys

```bash
# Generate API keys using the binary
cd ~/livekit-server
livekit-server generate-keys

# Save the output - you'll need these keys
# Example output:
# API Key: APIxxxxxxxxxxxxxxx
# API Secret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Step 5: Create LiveKit Configuration

Create `~/livekit-server/livekit.yaml`:

```yaml
port: 7880
bind_addresses:
  - ""

# RTC configuration
rtc:
  tcp_port: 7881
  port_range_start: 50000
  port_range_end: 60000
  use_external_ip: true
  # Uncomment and set your VPS IP if needed
  # external_ip: "YOUR_VPS_IP"

# Redis configuration
redis:
  address: localhost:6379
  # Add authentication if your Redis is secured
  # username: ""
  # password: ""

# TURN server for NAT traversal
turn:
  enabled: true
  domain: yourdomain.com
  cert_file: certs/fullchain.pem
  key_file: certs/privkey.pem
  tls_port: 5349
  udp_port: 3478

# API keys - REPLACE WITH YOUR GENERATED KEYS
keys:
  # Generate these with: docker run --rm livekit/livekit-server:latest generate-keys
  # Format: API_KEY: SECRET_KEY
  APIxxxxxxxxxxxxxxx: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Logging configuration
logging:
  level: info
  pion_level: warn
  sample: false

# Room settings
room:
  auto_create: true
  enable_recording: false
  max_participants: 100
  empty_timeout: 300s
  departure_timeout: 20s

# Node configuration for clustering (single node for now)
node:
  ip: ""
  
# Optional webhook configuration
# webhook:
#   url: https://yourapp.com/livekit-webhook
#   api_key: your_webhook_secret

# Development settings (remove in production)
# dev_mode: false
```
## Step 6: Create PM2 Ecosystem File
Create `~/livekit-server/ecosystem.config.js`:

```javascript
module.exports = {
  apps: [
    {
      name: 'livekit-server',
      script: '/usr/local/bin/livekit-server',
      args: '--config /root/livekit-server/livekit.yaml',
      cwd: '/root/livekit-server',
      instances: 1,
      exec_mode: 'fork',
      autorestart: true,
      watch: false,
      max_memory_restart: '1G',
      env: {
        NODE_ENV: 'production',
        LIVEKIT_CONFIG: '/root/livekit-server/livekit.yaml'
      },
      env_production: {
        NODE_ENV: 'production'
      },
      error_file: '/root/livekit-server/logs/livekit-error.log',
      out_file: '/root/livekit-server/logs/livekit-out.log',
      log_file: '/root/livekit-server/logs/livekit-combined.log',
      time: true,
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      merge_logs: true,
      kill_timeout: 5000,
      restart_delay: 1000,
      max_restarts: 10,
      min_uptime: 10
    }
  ]
};
```

## Step 7: Configure Firewall

```bash
# Allow necessary ports
sudo ufw allow 22          # SSH
sudo ufw allow 80          # HTTP
sudo ufw allow 443         # HTTPS
sudo ufw allow 7880        # LiveKit HTTP/WebSocket
sudo ufw allow 7881        # LiveKit TCP
sudo ufw allow 3478/udp    # TURN UDP
sudo ufw allow 5349        # TURN TLS
sudo ufw allow 50000:60000/udp  # RTC port range

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status
```

## Step 8: Start LiveKit with PM2

```bash
cd ~/livekit-server

# Create logs directory
mkdir -p logs

# Start with PM2
pm2 start ecosystem.config.js

# Save PM2 configuration
pm2 save

# Setup PM2 to start on boot
pm2 startup
# Follow the instructions shown by the command above
```

## Step 9: Set Up Nginx Reverse Proxy (Recommended)

Install and configure Nginx:

```bash
# Install Nginx
sudo apt install nginx -y

# Create Nginx configuration
sudo nano /etc/nginx/sites-available/livekit
```

Add this configuration:

```nginx
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    location / {
        proxy_pass http://localhost:7880;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support
        proxy_buffering off;
        proxy_cache off;
    }
}
```

Enable the site:

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/livekit /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
sudo systemctl enable nginx
```

## Step 10: Test Your Installation

```bash
# Check PM2 status
pm2 status

# Check LiveKit logs
pm2 logs livekit-server

# Test server health
curl https://yourdomain.com/

# Test WebSocket connection
wscat -c wss://yourdomain.com
```

## Step 11: Create Environment Variables

Create `~/livekit-server/.env`:

```bash
LIVEKIT_URL=wss://yourdomain.com
LIVEKIT_API_KEY=APIxxxxxxxxxxxxxxx
LIVEKIT_API_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Step 12: Update Your Application

Update your client application to use your self-hosted server:

### Frontend (JavaScript/React):
```javascript
const serverUrl = 'wss://yourdomain.com';
const token = await generateToken(); // Generate server-side

const room = new Room();
await room.connect(serverUrl, token);
```

### Backend Token Generation:
```javascript
const { AccessToken } = require('livekit-server-sdk');

function createToken(roomName, participantName) {
  const at = new AccessToken(
    'APIxxxxxxxxxxxxxxx',
    'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
    {
      identity: participantName,
      ttl: '10m',
    }
  );
  
  at.addGrant({
    roomJoin: true,
    room: roomName,
    canPublish: true,
    canSubscribe: true,
  });
  
  return at.toJwt();
}
```

## Step 13: PM2 Management Commands

```bash
# View all PM2 processes
pm2 list

# View LiveKit logs
pm2 logs livekit-server

# Restart LiveKit
pm2 restart livekit-server

# Stop LiveKit
pm2 stop livekit-server

# Monitor resources
pm2 monit

# Reload PM2 after config changes
pm2 reload ecosystem.config.js

# Delete process
pm2 delete livekit-server
```

## Step 14: Set Up Log Rotation

Create `~/livekit-server/logrotate.conf`:

```bash
/home/yourusername/livekit-server/logs/*.log {
    daily
    missingok
    rotate 14
    compress
    notifempty
    create 644 yourusername yourusername
    postrotate
        pm2 reloadLogs
    endscript
}
```

Add to system logrotate:
```bash
sudo cp logrotate.conf /etc/logrotate.d/livekit
```

## Step 15: SSL Certificate Auto-Renewal

Create renewal script `~/livekit-server/renew-certs.sh`:

```bash
#!/bin/bash
sudo certbot renew --quiet
sudo cp /etc/letsencrypt/live/yourdomain.com/fullchain.pem ~/livekit-server/certs/
sudo cp /etc/letsencrypt/live/yourdomain.com/privkey.pem ~/livekit-server/certs/
sudo chown $USER:$USER ~/livekit-server/certs/*
pm2 restart livekit-server
sudo systemctl reload nginx
```

Make executable and add to crontab:
```bash
chmod +x renew-certs.sh

# Add to crontab
crontab -e
# Add: 0 2 * * 1 /home/yourusername/livekit-server/renew-certs.sh
```

## Monitoring and Maintenance

- **Status Check**: `pm2 status`
- **Logs**: `pm2 logs livekit-server --lines 100`
- **Restart**: `pm2 restart livekit-server`
- **Update**: Download new binary and restart with PM2
- **Health Check**: `curl https://yourdomain.com/`

## Common Issues

1. **Port conflicts**: Ensure ports 7880, 7881 aren't used by other services
2. **Certificate permissions**: Make sure cert files are readable by your user
3. **Redis connection**: Verify Redis is running with `redis-cli ping`
4. **Firewall**: Double-check all required ports are open

Your LiveKit server will be accessible at `wss://yourdomain.com` (or `wss://yourdomain.com:7880` if not using Nginx proxy).

## Thank you