# 🚀 Story Protocol Advanced Snapshot Service Setup Guide v2.0

## Complete guide to create your own professional snapshot service with ZSTD + Node.js API + Dual Interface

---

## 📋 Table of Contents

- [System Requirements](#-system-requirements)
- [Initial Server Setup](#-initial-server-setup)
- [Dependencies Installation](#-dependencies-installation)
- [Nginx Installation & Configuration](#-nginx-installation--configuration)
- [SSL Certificate Setup](#-ssl-certificate-setup)
- [Directory Structure Creation](#-directory-structure-creation)
- [Node.js API Server Setup](#-nodejs-api-server-setup)
- [Enhanced Snapshot Creation Script](#-enhanced-snapshot-creation-script)
- [Web Interface Setup (Dual Mode)](#-web-interface-setup-dual-mode)
- [Automated Scheduling & Service Management](#-automated-scheduling--service-management)
- [Download Script for Users](#-download-script-for-users)
- [Monitoring & Maintenance](#-monitoring--maintenance)
- [Troubleshooting](#-troubleshooting)

---

## 📋 System Requirements

### Minimum Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **CPU** | 4 cores | 8+ cores |
| **RAM** | 16 GB | 32 GB |
| **Storage** | 500 GB NVMe SSD | 1 TB+ NVMe SSD |
| **OS** | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |
| **Network** | 100 Mbps | 1 Gbps |
| **Domain** | Required | Required |

### Prerequisites
- Ubuntu 22.04 LTS server
- Domain name (e.g., `snaps.yourdomain.com`)
- Root or sudo access
- Story Protocol node already synced and running

---

## 🚀 Initial Server Setup

### Step 1: System Update & User Management

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install basic tools
sudo apt install -y curl git wget htop tmux build-essential jq make gcc unzip

# Create dedicated snapshot user
sudo useradd -m -s /bin/bash snapshot
sudo usermod -aG sudo snapshot

# Set up passwordless sudo for specific operations
echo "snapshot ALL=(ALL) NOPASSWD: /bin/systemctl" | sudo tee /etc/sudoers.d/snapshot
sudo chmod 440 /etc/sudoers.d/snapshot

# Verify user creation
id snapshot
```

---

## 📦 Dependencies Installation

### Step 1: Install Core Packages

```bash
# Install nginx, SSL tools, compression utilities, and Node.js
sudo apt install -y nginx certbot python3-certbot-nginx aria2 zstd jq python3 bc

# Install Node.js (v18 LTS)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify installations
nginx -v
certbot --version
aria2c --version
zstd --version
node --version
npm --version
```

---

## 🌐 Nginx Installation & Configuration

### Step 1: Enhanced Nginx Configuration

```bash
# Stop default nginx if running
sudo systemctl stop nginx

# Remove default configuration
sudo rm -f /etc/nginx/sites-enabled/default

# Create enhanced snapshot service configuration
sudo nano /etc/nginx/sites-available/YOUR_DOMAIN
```

**Add this enhanced configuration (replace YOUR_DOMAIN with your actual domain):**

```nginx
server {
    listen 80;
    server_name YOUR_DOMAIN;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name YOUR_DOMAIN;
    
    # SSL certificates (will be configured with certbot)
    ssl_certificate /etc/letsencrypt/live/YOUR_DOMAIN/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/YOUR_DOMAIN/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
    
    # Root directory
    root /var/www/snapshots;
    index index.html;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    
    # Static home page
    location / {
        try_files $uri $uri/ =404;
    }
    
    # Dynamic web interface
    location /dynamic {
        proxy_pass http://localhost:3001/dynamic.html;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # API endpoints
    location /api/ {
        proxy_pass http://localhost:3001/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Snapshot files
    location /story/aeneid/ {
        alias /var/www/snapshots/story/aeneid/;
        
        location ~* \.(tar\.zst|tar\.lz4|txt|json)$ {
            add_header Cache-Control "public, max-age=3600";
            add_header X-Snapshot-Service "YOUR_SERVICE_NAME";
        }
        
        location = /story/aeneid/ {
            return 301 /#snapshots;
        }
    }
    
    # Access and error logs
    access_log /var/log/nginx/YOUR_DOMAIN.access.log;
    error_log /var/log/nginx/YOUR_DOMAIN.error.log;
}
```

### Step 2: Enable Site Configuration

```bash
# Enable your site
sudo ln -s /etc/nginx/sites-available/YOUR_DOMAIN /etc/nginx/sites-enabled/

# Test nginx configuration
sudo nginx -t
```

---

## 🔒 SSL Certificate Setup

### Step 1: Get Let's Encrypt Certificate

```bash
# Get SSL certificate (replace YOUR_DOMAIN and YOUR_EMAIL)
sudo certbot --nginx -d YOUR_DOMAIN --email YOUR_EMAIL --agree-tos --no-eff-email

# Test auto-renewal
sudo certbot renew --dry-run

# Set up automatic renewal
echo "0 12 * * * /usr/bin/certbot renew --quiet" | sudo crontab -

# Start nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

---

## 📁 Directory Structure Creation

### Step 1: Create Enhanced Directory Structure

```bash
# Create comprehensive web directory structure
sudo mkdir -p /var/www/snapshots/story/aeneid
sudo mkdir -p /var/www/snapshots/api
sudo mkdir -p /var/www/snapshots/public
sudo mkdir -p /var/lock/YOUR_SERVICE_NAME

# Set ownership and permissions
sudo chown -R snapshot:snapshot /var/www/snapshots
sudo chown -R snapshot:snapshot /var/lock/YOUR_SERVICE_NAME
sudo chmod -R 755 /var/www/snapshots
sudo chmod -R 755 /var/lock/YOUR_SERVICE_NAME

# Verify structure
ls -la /var/www/snapshots/
```

---

## 🔧 Node.js API Server Setup

### Step 1: Initialize Node.js Project

```bash
# Change to web directory
cd /var/www/snapshots

# Initialize npm project
sudo -u snapshot npm init -y

# Install dependencies
sudo -u snapshot npm install express cors

# Create API directory structure
sudo -u snapshot mkdir -p api public
```

### Step 2: Create API Endpoint

```bash
# Create snapshot API endpoint
sudo -u snapshot tee api/snapshots.js > /dev/null << 'EOF'
const express = require('express');
const fs = require('fs');
const path = require('path');

const router = express.Router();

// Get current snapshot info
router.get('/info', (req, res) => {
    try {
        const infoPath = path.join(__dirname, '../story/aeneid/YOUR_SERVICE_NAME-info.json');
        const heightPath = path.join(__dirname, '../story/aeneid/YOUR_SERVICE_NAME-height.txt');
        
        if (fs.existsSync(infoPath)) {
            const info = JSON.parse(fs.readFileSync(infoPath, 'utf8'));
            const height = fs.readFileSync(heightPath, 'utf8').trim();
            
            res.json({
                ...info,
                current_height: height,
                status: 'active',
                last_updated: new Date().toISOString()
            });
        } else {
            res.status(404).json({ error: 'Snapshot info not found' });
        }
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Get file sizes
router.get('/files', (req, res) => {
    try {
        const snapshotDir = path.join(__dirname, '../story/aeneid');
        const files = fs.readdirSync(snapshotDir);
        
        const snapshots = files
            .filter(file => file.endsWith('.tar.zst'))
            .map(file => {
                const filePath = path.join(snapshotDir, file);
                const stats = fs.statSync(filePath);
                return {
                    name: file,
                    size: stats.size,
                    size_gb: (stats.size / (1024 * 1024 * 1024)).toFixed(1),
                    modified: stats.mtime
                };
            });
            
        res.json({ snapshots });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

module.exports = router;
EOF
```

### Step 3: Create Main Express Application

```bash
# Create main Express app
sudo -u snapshot tee app.js > /dev/null << 'EOF'
const express = require('express');
const cors = require('cors');
const path = require('path');

const app = express();
const PORT = 3001;

// Middleware
app.use(cors());
app.use(express.json());
app.use(express.static('public'));

// API routes
app.use('/api', require('./api/snapshots'));

// Serve static files from story directory
app.use('/story', express.static('story'));

app.listen(PORT, () => {
    console.log(`🌟 YOUR_SERVICE_NAME API Server running on port ${PORT}`);
});
EOF
```

### Step 4: Create Systemd Service for API

```bash
# Create systemd service for API server
sudo tee /etc/systemd/system/YOUR_SERVICE_NAME-api.service > /dev/null << 'EOF'
[Unit]
Description=YOUR_SERVICE_NAME API Server
After=network.target

[Service]
Type=simple
User=snapshot
WorkingDirectory=/var/www/snapshots
ExecStart=/usr/bin/node app.js
Restart=always
RestartSec=10
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
EOF

# Enable and start API service
sudo systemctl daemon-reload
sudo systemctl enable YOUR_SERVICE_NAME-api
sudo systemctl start YOUR_SERVICE_NAME-api

# Check status
sudo systemctl status YOUR_SERVICE_NAME-api
```

---

## 📸 Enhanced Snapshot Creation Script

### Step 1: Create Advanced Script with Atomic Lock

```bash
# Create the enhanced snapshot creation script
sudo -u snapshot tee /home/snapshot/snapshot_creator.sh > /dev/null << 'EOF'
#!/bin/bash
# Advanced Snapshot Creator with Atomic Lock and Safe Updates
set -e

# Configuration
SNAPSHOT_DIR="/var/www/snapshots/story/aeneid"
STORY_HOME="/root/.story"
LOG_FILE="/var/log/YOUR_SERVICE_NAME-snapshot.log"
LOCK_FILE="/var/lock/YOUR_SERVICE_NAME/snapshot.lock"
SERVICE_NAME="YOUR_SERVICE_NAME"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a $LOG_FILE
}

# Atomic lock using flock to prevent race conditions
check_lock() {
    # Create lock directory if it doesn't exist
    mkdir -p /var/lock/$SERVICE_NAME
    
    # Use file descriptor 200 for lock file
    exec 200>"$LOCK_FILE"
    
    # Try to acquire exclusive lock (non-blocking)
    if ! flock -n 200; then
        log "ERROR: Another snapshot process is already running"
        exit 1
    fi
    
    # Write PID to lock file for monitoring
    echo $$ >&200
    log "Lock acquired successfully (PID: $$)"
}

cleanup_lock() {
    # Lock will be automatically released when script exits
    rm -f "$LOCK_FILE"
    log "Lock released and cleaned up"
}

# Set trap for cleanup
trap cleanup_lock EXIT INT TERM

# Auto-detect Story RPC port from config
detect_rpc_port() {
    local config_file="$STORY_HOME/story/config/config.toml"
    local default_port="26657"
    
    if [[ -f "$config_file" ]]; then
        local port=$(grep -E "^laddr = \"tcp://.*:[0-9]+\"" "$config_file" | grep -oE "[0-9]+" | head -1)
        if [[ -n "$port" && "$port" =~ ^[0-9]+$ ]]; then
            echo "$port"
            return
        fi
    fi
    
    echo "$default_port"
}

get_block_height() {
    local rpc_port=$(detect_rpc_port)
    local height=$(curl -s "localhost:$rpc_port/status" | jq -r '.result.sync_info.latest_block_height' 2>/dev/null)
    if [[ "$height" =~ ^[0-9]+$ ]]; then
        echo $height
    else
        echo "0"
    fi
}

is_synced() {
    local rpc_port=$(detect_rpc_port)
    local catching_up=$(curl -s "localhost:$rpc_port/status" | jq -r '.result.sync_info.catching_up' 2>/dev/null)
    [[ "$catching_up" == "false" ]]
}

update_static_web() {
    local current_height=$1
    local consensus_file=$2
    local execution_file=$3
    
    log "Updating static web interface..."
    
    # Safe targeted updates - avoid full file replacement
    # Update block height in stats (specific pattern)
    sed -i "s/\(<div class=\"stat-value\">\)[0-9]\{7\}\(<\/div>\)/\1$current_height\2/g" /var/www/snapshots/index.html
    
    # Update consensus file name (specific pattern)
    sed -i "s/\($SERVICE_NAME-aeneid-consensus-\)[0-9]\{7\}-[0-9]\{8\}\(\.tar\.zst\)/\1${current_height}-$(date '+%Y%m%d')\2/g" /var/www/snapshots/index.html
    
    # Update execution file name (specific pattern)
    sed -i "s/\($SERVICE_NAME-aeneid-execution-\)[0-9]\{7\}-[0-9]\{8\}\(\.tar\.zst\)/\1${current_height}-$(date '+%Y%m%d')\2/g" /var/www/snapshots/index.html
    
    # Ensure proper permissions
    chown snapshot:snapshot /var/www/snapshots/index.html 2>/dev/null || true
    chmod 644 /var/www/snapshots/index.html 2>/dev/null || true
    
    log "Static web interface updated with block height: $current_height"
}

create_snapshots() {
    local current_height=$(get_block_height)
    local timestamp=$(date '+%Y%m%d')
    local consensus_name="$SERVICE_NAME-aeneid-consensus-${current_height}-${timestamp}.tar.zst"
    local execution_name="$SERVICE_NAME-aeneid-execution-${current_height}-${timestamp}.tar.zst"
    
    log "Starting $SERVICE_NAME snapshot creation for block height: $current_height"
    
    if ! is_synced; then
        log "ERROR: Node is not synced"
        return 1
    fi
    
    log "Removing old snapshots..."
    rm -f $SNAPSHOT_DIR/$SERVICE_NAME-aeneid-*.tar.zst
    
    log "Stopping services safely..."
    sudo systemctl stop story
    log "Waiting for Story service to fully stop..."
    sleep 10
    
    sudo systemctl stop story-geth
    log "Waiting for Geth service to fully stop..."
    sleep 15
    
    log "Waiting for database files to be fully released..."
    sleep 20
    
    # Verify services are truly stopped
    for i in {1..5}; do
        if ! systemctl is-active --quiet story && ! systemctl is-active --quiet story-geth; then
            log "Services confirmed stopped"
            break
        fi
        log "Services still shutting down, waiting..."
        sleep 5
    done
    
    local temp_dir=$(mktemp -d)
    if [[ -f "$STORY_HOME/story/data/priv_validator_state.json" ]]; then
        log "Backing up validator state..."
        sudo cp "$STORY_HOME/story/data/priv_validator_state.json" "$temp_dir/validator_backup.json"
        sudo chown $(whoami):$(whoami) "$temp_dir/validator_backup.json"
    fi
    
    log "Creating consensus snapshot with ZSTD..."
    if sudo bash -c "cd $STORY_HOME/story && tar -cf - data" | zstd -3 -T0 > "$SNAPSHOT_DIR/$consensus_name"; then
        log "Consensus snapshot created successfully"
    else
        log "ERROR: Consensus snapshot creation failed"
        return 1
    fi
    
    log "Creating execution snapshot with ZSTD..."
    if sudo bash -c "cd $STORY_HOME/geth/aeneid/geth && tar -cf - chaindata" | zstd -3 -T0 > "$SNAPSHOT_DIR/$execution_name"; then
        log "Execution snapshot created successfully"
    else
        log "ERROR: Execution snapshot creation failed"
        return 1
    fi
    
    # Verify snapshot sizes
    consensus_size=$(stat -c%s "$SNAPSHOT_DIR/$consensus_name")
    execution_size=$(stat -c%s "$SNAPSHOT_DIR/$execution_name")
    
    log "Snapshot sizes: Consensus: ${consensus_size} bytes, Execution: ${execution_size} bytes"
    
    if [ $consensus_size -lt 1000000 ] || [ $execution_size -lt 1000000 ]; then
        log "ERROR: Snapshot files are too small, something went wrong"
        return 1
    fi
    
    # Set permissions properly
    chown snapshot:snapshot "$SNAPSHOT_DIR/$consensus_name"
    chown snapshot:snapshot "$SNAPSHOT_DIR/$execution_name"
    chmod 644 "$SNAPSHOT_DIR/$consensus_name"
    chmod 644 "$SNAPSHOT_DIR/$execution_name"
    
    # Create metadata for API
    cat > "$SNAPSHOT_DIR/$SERVICE_NAME-info.json" << EOL
{
  "service": "$SERVICE_NAME Snapshot Service",
  "network": "story-aeneid",
  "block_height": $current_height,
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "compression": "zstd",
  "snapshots": {
    "consensus": "$consensus_name",
    "execution": "$execution_name"
  }
}
EOL
    
    echo "$current_height" > "$SNAPSHOT_DIR/$SERVICE_NAME-height.txt"
    chown snapshot:snapshot "$SNAPSHOT_DIR/$SERVICE_NAME-info.json"
    chown snapshot:snapshot "$SNAPSHOT_DIR/$SERVICE_NAME-height.txt"
    chmod 644 "$SNAPSHOT_DIR/$SERVICE_NAME-info.json"
    chmod 644 "$SNAPSHOT_DIR/$SERVICE_NAME-height.txt"
    
    # Update static web interface (safe version)
    update_static_web "$current_height" "$consensus_name" "$execution_name"
    
    # Restore validator state
    if [[ -f "$temp_dir/validator_backup.json" ]]; then
        log "Restoring validator state..."
        sudo cp "$temp_dir/validator_backup.json" "$STORY_HOME/story/data/priv_validator_state.json"
    fi
    
    log "Starting services in correct order..."
    sudo systemctl start story-geth
    log "Waiting for Geth to initialize..."
    sleep 15
    
    sudo systemctl start story
    log "Waiting for Story to initialize..."
    sleep 10
    
    # Cleanup
    rm -rf "$temp_dir"
    log "$SERVICE_NAME snapshot completed: $consensus_name, $execution_name"
}

main() {
    # Acquire atomic lock first
    check_lock
    
    log "=== $SERVICE_NAME Snapshot Creation Started ==="
    
    if ! systemctl is-active --quiet story || ! systemctl is-active --quiet story-geth; then
        log "ERROR: Services not running"
        exit 1
    fi
    
    if create_snapshots; then
        log "=== $SERVICE_NAME Snapshot Creation Completed ==="
    else
        log "=== $SERVICE_NAME Snapshot Creation Failed ==="
        exit 1
    fi
}

main "$@"
EOF

# Set permissions
sudo chmod +x /home/snapshot/snapshot_creator.sh
sudo chown snapshot:snapshot /home/snapshot/snapshot_creator.sh

# Create log file
sudo touch /var/log/YOUR_SERVICE_NAME-snapshot.log
sudo chown snapshot:snapshot /var/log/YOUR_SERVICE_NAME-snapshot.log
```

---

## 🎨 Web Interface Setup (Dual Mode)

### Step 1: Create Static Interface

```bash
# Create enhanced static index.html
sudo -u snapshot tee /var/www/snapshots/index.html > /dev/null << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>YOUR_SERVICE_NAME Story Snapshots</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white; min-height: 100vh; padding: 20px;
        }
        .container { max-width: 1000px; margin: 0 auto; }
        .header { text-align: center; margin-bottom: 40px; }
        .header h1 { font-size: 2.8em; margin-bottom: 10px; text-shadow: 2px 2px 4px rgba(0,0,0,0.3); }
        .stats-grid {
            display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
            gap: 20px; margin-bottom: 40px;
        }
        .stat-card {
            background: rgba(255, 255, 255, 0.15); border-radius: 15px; padding: 20px;
            text-align: center; backdrop-filter: blur(10px);
        }
        .stat-value { font-size: 1.4em; font-weight: bold; color: #ffd700; }
        .snapshots-section {
            background: rgba(255, 255, 255, 0.1); border-radius: 20px; padding: 30px;
            backdrop-filter: blur(15px);
        }
        .file-item {
            background: rgba(255, 255, 255, 0.1); border-radius: 12px; padding: 20px;
            margin-bottom: 15px; display: grid; grid-template-columns: 1fr auto;
        }
        .download-btn {
            background: linear-gradient(45deg, #ffd700, #ffed4e); color: #333;
            padding: 10px 20px; border-radius: 20px; text-decoration: none;
            font-weight: bold;
        }
        .instructions-btn {
            background: linear-gradient(45deg, #ffd700, #ffed4e); color: #333;
            padding: 15px 30px; border-radius: 25px; text-decoration: none;
            font-weight: bold; display: inline-block; margin: 10px;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🌟 YOUR_SERVICE_NAME Story Snapshots</h1>
            <p>Advanced ZSTD Compression • High-Performance Infrastructure</p>
        </div>
        
        <div class="stats-grid">
            <div class="stat-card">
                <div style="font-size: 2.5em;">📊</div>
                <div>Block Height</div>
                <div class="stat-value">0000000</div>
            </div>
            <div class="stat-card">
                <div style="font-size: 2.5em;">🗜️</div>
                <div>Compression</div>
                <div class="stat-value">ZSTD</div>
            </div>
            <div class="stat-card">
                <div style="font-size: 2.5em;">⚡</div>
                <div>Download Tool</div>
                <div class="stat-value">aria2c</div>
            </div>
            <div class="stat-card">
                <div style="font-size: 2.5em;">🔄</div>
                <div>Update Freq</div>
                <div class="stat-value">6 Hours</div>
            </div>
        </div>
        
        <div class="snapshots-section">
            <h2 style="text-align: center; color: #ffd700; margin-bottom: 20px;">📂 Latest Snapshots</h2>
            
            <div class="file-item">
                <div>
                    <h3 style="color: #ffd700;">🔹 YOUR_SERVICE_NAME-aeneid-consensus-0000000-00000000.tar.zst</h3>
                    <p>Story consensus layer (36G)</p>
                </div>
                <a href="/story/aeneid/YOUR_SERVICE_NAME-aeneid-consensus-0000000-00000000.tar.zst" class="download-btn">📥 Download</a>
            </div>
            
            <div class="file-item">
                <div>
                    <h3 style="color: #ffd700;">🟣 YOUR_SERVICE_NAME-aeneid-execution-0000000-00000000.tar.zst</h3>
                    <p>Geth execution layer (15G)</p>
                </div>
                <a href="/story/aeneid/YOUR_SERVICE_NAME-aeneid-execution-0000000-00000000.tar.zst" class="download-btn">📥 Download</a>
            </div>
            
            <div style="text-align: center; margin-top: 30px;">
                <a href="https://github.com/YOUR_GITHUB/Story-Aeneid#-snapshot-service" class="instructions-btn">
                    📋 Download Instructions
                </a>
                <a href="/dynamic" class="instructions-btn" style="background: linear-gradient(45deg, #00ff88, #44ff44);">
                    ⚡ Live Dynamic Interface
                </a>
            </div>
        </div>
        
        <div style="text-align: center; margin-top: 40px; opacity: 0.8;">
            <p>🚀 Powered by <strong>YOUR_SERVICE_NAME</strong> | ZSTD compression</p>
        </div>
    </div>
</body>
</html>
EOF
```

### Step 2: Create Dynamic Interface

```bash
# Create real-time dynamic interface
sudo -u snapshot tee /var/www/snapshots/public/dynamic.html > /dev/null << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>YOUR_SERVICE_NAME Story Snapshots - Live</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white; min-height: 100vh; padding: 20px;
        }
        .container { max-width: 1000px; margin: 0 auto; }
        .header { text-align: center; margin-bottom: 40px; }
        .header h1 { font-size: 2.8em; margin-bottom: 10px; text-shadow: 2px 2px 4px rgba(0,0,0,0.3); }
        .live-indicator {
            background: #00ff88; color: #000; padding: 5px 15px; border-radius: 20px;
            display: inline-block; font-size: 0.9em; font-weight: bold; margin-left: 10px;
            animation: pulse 2s infinite;
        }
        @keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.7; } }
        .stats-grid {
            display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
            gap: 20px; margin-bottom: 40px;
        }
        .stat-card {
            background: rgba(255, 255, 255, 0.15); border-radius: 15px; padding: 20px;
            text-align: center; backdrop-filter: blur(10px);
            border: 1px solid rgba(255, 255, 255, 0.2); transition: transform 0.3s ease;
        }
        .stat-card:hover { transform: translateY(-5px); }
        .stat-value { font-size: 1.4em; font-weight: bold; color: #ffd700; }
        .snapshots-section {
            background: rgba(255, 255, 255, 0.1); border-radius: 20px; padding: 30px;
            backdrop-filter: blur(15px); border: 1px solid rgba(255, 255, 255, 0.2);
        }
        .file-item {
            background: rgba(255, 255, 255, 0.1); border-radius: 12px; padding: 20px;
            margin-bottom: 15px; display: grid; grid-template-columns: 1fr auto; align-items: center;
            transition: all 0.3s ease; border-left: 4px solid #ffd700;
        }
        .download-btn {
            background: linear-gradient(45deg, #ffd700, #ffed4e); color: #333;
            padding: 10px 20px; border-radius: 20px; text-decoration: none;
            font-weight: bold; transition: transform 0.3s ease;
        }
        .download-btn:hover { transform: scale(1.05); }
        .loading { text-align: center; padding: 40px; }
        .error { background: rgba(255, 0, 0, 0.2); padding: 20px; border-radius: 10px; margin: 20px 0; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🌟 YOUR_SERVICE_NAME Story Snapshots <span class="live-indicator">● LIVE</span></h1>
            <p>Real-time ZSTD Compression • High-Performance Infrastructure</p>
        </div>
        
        <div class="stats-grid" id="stats">
            <div class="loading">📊 Loading real-time data...</div>
        </div>
        
        <div class="snapshots-section">
            <h2 style="text-align: center; color: #ffd700; margin-bottom: 20px;">📂 Live Snapshot Data</h2>
            <div id="snapshots">
                <div class="loading">🔄 Fetching latest snapshots...</div>
            </div>
        </div>
    </div>

    <script>
        // Fetch real-time data
        async function updateData() {
            try {
                const response = await fetch('/api/info');
                const data = await response.json();
                
                // Update stats
                document.getElementById('stats').innerHTML = \`
                    <div class="stat-card">
                        <div style="font-size: 2.5em; margin-bottom: 10px;">📊</div>
                        <div style="font-size: 0.9em; opacity: 0.8;">Block Height</div>
                        <div class="stat-value">\${data.block_height}</div>
                    </div>
                    <div class="stat-card">
                        <div style="font-size: 2.5em; margin-bottom: 10px;">🗜️</div>
                        <div style="font-size: 0.9em; opacity: 0.8;">Compression</div>
                        <div class="stat-value">ZSTD</div>
                    </div>
                    <div class="stat-card">
                        <div style="font-size: 2.5em; margin-bottom: 10px;">⚡</div>
                        <div style="font-size: 0.9em; opacity: 0.8;">Download Tool</div>
                        <div class="stat-value">aria2c</div>
                    </div>
                    <div class="stat-card">
                        <div style="font-size: 2.5em; margin-bottom: 10px;">🕐</div>
                        <div style="font-size: 0.9em; opacity: 0.8;">Last Update</div>
                        <div class="stat-value">\${new Date(data.timestamp).toLocaleTimeString()}</div>
                    </div>
                \`;
                
                // Update snapshots
                document.getElementById('snapshots').innerHTML = \`
                    <div class="file-item">
                        <div>
                            <h3 style="color: #ffd700;">🔹 \${data.snapshots.consensus}</h3>
                            <p style="opacity: 0.8;">Story consensus layer (ZSTD compressed)</p>
                        </div>
                        <a href="/story/aeneid/\${data.snapshots.consensus}" class="download-btn">📥 Download</a>
                    </div>
                    <div class="file-item">
                        <div>
                            <h3 style="color: #ffd700;">🟣 \${data.snapshots.execution}</h3>
                            <p style="opacity: 0.8;">Geth execution layer (ZSTD compressed)</p>
                        </div>
                        <a href="/story/aeneid/\${data.snapshots.execution}" class="download-btn">📥 Download</a>
                    </div>
                \`;
                
            } catch (error) {
                document.getElementById('snapshots').innerHTML = \`
                    <div class="error">❌ Error loading data: \${error.message}</div>
                \`;
            }
        }
        
        // Initial load and auto-refresh every 30 seconds
        updateData();
        setInterval(updateData, 30000);
    </script>
</body>
</html>
EOF
```

---

## ⏰ Automated Scheduling & Service Management

### Step 1: Setup Enhanced Cron Job

```bash
# Setup cron job for snapshot user
sudo -u snapshot crontab << 'EOF'
# Advanced Snapshot Service - ZSTD Compression with Atomic Lock
# Create snapshots every 6 hours at 00:00, 06:00, 12:00, 18:00
0 0,6,12,18 * * * /home/snapshot/snapshot_creator.sh >> /var/log/YOUR_SERVICE_NAME-snapshot.log 2>&1

# Cleanup logs monthly
0 0 1 * * find /var/log -name "YOUR_SERVICE_NAME-snapshot.log" -size +100M -exec truncate -s 50M {} \;
EOF

# Verify cron job
sudo -u snapshot crontab -l
```

### Step 2: Service Status Check

```bash
# Check all services are running
sudo systemctl status nginx
sudo systemctl status YOUR_SERVICE_NAME-api
```

---

## 📥 Download Script for Users

### Create Auto-Discovery User Script

```bash
# Create user download documentation
sudo -u snapshot tee /var/www/snapshots/download-script.md > /dev/null << 'EOF'
# YOUR_SERVICE_NAME Advanced Download Script

## Prerequisites

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install -y aria2 zstd jq curl

# Verify installation
aria2c --version && zstd --version
```

## Auto-Discovery Download Script

```bash
#!/bin/bash
# YOUR_SERVICE_NAME Advanced Snapshot Download
# Using aria2c + ZSTD with automatic file discovery

set -e

echo "🌟 YOUR_SERVICE_NAME Advanced Snapshot Download"
echo "==============================================="

# Configuration
SNAPSHOT_BASE="https://YOUR_DOMAIN/story/aeneid"
STORY_DATA="$HOME/.story"
TEMP_DIR="/tmp/snapshot_sync"

# Auto-detect Story RPC port from user's config
detect_story_port() {
    local config_file="$STORY_DATA/story/config/config.toml"
    local default_port="26657"
    
    if [[ -f "$config_file" ]]; then
        local port=$(grep -E "^laddr = \"tcp://.*:[0-9]+\"" "$config_file" | grep -oE "[0-9]+" | head -1)
        if [[ -n "$port" && "$port" =~ ^[0-9]+$ ]]; then
            echo "$port"
            return
        fi
    fi
    
    echo "$default_port"
}

STORY_PORT=$(detect_story_port)
LOCAL_RPC="localhost:$STORY_PORT"

# Check dependencies
echo "📦 Checking dependencies..."
for cmd in aria2c zstd jq; do
    if ! command -v $cmd &> /dev/null; then
        echo "❌ Missing dependency: $cmd"
        echo "💡 Install with: sudo apt install aria2 zstd jq"
        exit 1
    fi
done
echo "✅ All dependencies ready"

# Auto-discover snapshots via JSON API
echo "🔍 Discovering latest snapshots..."
METADATA=$(curl -s "$SNAPSHOT_BASE/YOUR_SERVICE_NAME-info.json")
CONSENSUS_FILE=$(echo "$METADATA" | jq -r '.snapshots.consensus')
EXECUTION_FILE=$(echo "$METADATA" | jq -r '.snapshots.execution')
BLOCK_HEIGHT=$(echo "$METADATA" | jq -r '.block_height')

if [ "$CONSENSUS_FILE" = "null" ] || [ "$EXECUTION_FILE" = "null" ]; then
    echo "❌ Could not discover snapshot files"
    exit 1
fi

echo "📊 Snapshot Information:"
echo "  Block Height: $BLOCK_HEIGHT"
echo "  Consensus: $CONSENSUS_FILE"
echo "  Execution: $EXECUTION_FILE"

# Confirmation
read -p "🚀 Continue with download? (y/N): " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    echo "Download cancelled"
    exit 0
fi

# Prepare environment
mkdir -p "$TEMP_DIR"

echo "🛑 Stopping services..."
sudo systemctl stop story story-geth

echo "💾 Backing up validator state..."
if [[ -f "$STORY_DATA/story/data/priv_validator_state.json" ]]; then
    cp "$STORY_DATA/story/data/priv_validator_state.json" "$TEMP_DIR/validator_backup.json"
    echo "✅ Validator state backed up"
fi

echo "🧹 Cleaning target directories..."
rm -rf "$STORY_DATA/story/data"
rm -rf "$STORY_DATA/geth/aeneid/geth/chaindata"
mkdir -p "$STORY_DATA/geth/aeneid/geth"

echo ""
echo "📥 Downloading with aria2c (multi-connection)..."

# Download consensus with aria2c
echo "🔹 Downloading consensus snapshot..."
aria2c \
    --max-connection-per-server=8 \
    --split=8 \
    --min-split-size=1M \
    --continue=true \
    --max-tries=3 \
    --retry-wait=3 \
    --timeout=60 \
    --user-agent="YOUR_SERVICE_NAME-Client/2.0" \
    --dir="$TEMP_DIR" \
    --out="$CONSENSUS_FILE" \
    "$SNAPSHOT_BASE/$CONSENSUS_FILE"

# Download execution with aria2c
echo "🔸 Downloading execution snapshot..."
aria2c \
    --max-connection-per-server=8 \
    --split=8 \
    --min-split-size=1M \
    --continue=true \
    --max-tries=3 \
    --retry-wait=3 \
    --timeout=60 \
    --user-agent="YOUR_SERVICE_NAME-Client/2.0" \
    --dir="$TEMP_DIR" \
    --out="$EXECUTION_FILE" \
    "$SNAPSHOT_BASE/$EXECUTION_FILE"

echo ""
echo "📂 Extracting with ZSTD compression..."

# Extract consensus
echo "🔹 Extracting consensus snapshot..."
zstd -d --stdout "$TEMP_DIR/$CONSENSUS_FILE" | tar -xf - -C "$STORY_DATA/story"

# Extract execution
echo "🔸 Extracting execution snapshot..."
zstd -d --stdout "$TEMP_DIR/$EXECUTION_FILE" | tar -xf - -C "$STORY_DATA/geth/aeneid/geth"

echo "🔄 Restoring validator state..."
if [[ -f "$TEMP_DIR/validator_backup.json" ]]; then
    cp "$TEMP_DIR/validator_backup.json" "$STORY_DATA/story/data/priv_validator_state.json"
    echo "✅ Validator state restored"
fi

echo "🚀 Starting services..."
sudo systemctl start story-geth
sleep 10
sudo systemctl start story

echo "🧹 Cleaning up temporary files..."
rm -rf "$TEMP_DIR"

echo ""
echo "✅ YOUR_SERVICE_NAME snapshot download completed successfully!"
echo "📊 Monitor sync with: sudo journalctl -u story -u story-geth -f"

# Show final status
sleep 5
LOCAL_HEIGHT=$(curl -s "$LOCAL_RPC/status" 2>/dev/null | jq -r '.result.sync_info.latest_block_height' || echo "Checking...")
echo "🎯 Node restarted at block: $LOCAL_HEIGHT (RPC: $LOCAL_RPC)"
echo "📈 Target height was: $BLOCK_HEIGHT"
```
```
EOF
```

---

## 📊 Monitoring & Maintenance

### Step 1: Create Comprehensive Monitoring Script

```bash
# Create enhanced monitoring script
sudo -u snapshot tee /home/snapshot/monitor.sh > /dev/null << 'EOF'
#!/bin/bash

echo "📊 YOUR_SERVICE_NAME Snapshot Service Monitor"
echo "============================================="

# Check web service
echo "🌐 Web Service:"
curl -I https://YOUR_DOMAIN/story/aeneid/YOUR_SERVICE_NAME-height.txt 2>/dev/null | head -1 || echo "❌ Web service not accessible"

# Check API service
echo ""
echo "📡 API Service:"
curl -s https://YOUR_DOMAIN/api/info | jq '.block_height, .status' 2>/dev/null || echo "❌ API service not responding"

# Check snapshot files
echo ""
echo "📁 Current Snapshots:"
ls -lah /var/www/snapshots/story/aeneid/ | grep -E "\.(tar\.zst|json|txt)$"

# Check services status
echo ""
echo "🔧 Services Status:"
systemctl is-active nginx && echo "✅ Nginx: Running" || echo "❌ Nginx: Not running"
systemctl is-active YOUR_SERVICE_NAME-api && echo "✅ API: Running" || echo "❌ API: Not running"

# Check disk usage
echo ""
echo "💾 Disk Usage:"
df -h /var/www/snapshots | tail -1

# Check cron status
echo ""
echo "📅 Cron Status:"
sudo -u snapshot crontab -l | grep -E "^[0-9]"

# Check recent logs
echo ""
echo "📋 Recent Activity:"
tail -5 /var/log/YOUR_SERVICE_NAME-snapshot.log 2>/dev/null || echo "No logs found"

# Check Story node status
echo ""
echo "🔗 Story Node Status:"
curl -s localhost:26657/status 2>/dev/null | jq -r '.result.sync_info | "Height: \(.latest_block_height) | Synced: \(.catching_up == false)"' || echo "Cannot connect to Story node"

echo ""
echo "============================================="
EOF

# Make executable
sudo chmod +x /home/snapshot/monitor.sh
```

### Step 2: Setup Log Rotation

```bash
# Create logrotate configuration
sudo tee /etc/logrotate.d/YOUR_SERVICE_NAME > /dev/null << 'EOF'
/var/log/YOUR_SERVICE_NAME-snapshot.log {
    daily
    missingok
    rotate 30
    compress
    notifempty
    create 644 snapshot snapshot
}
EOF
```

---

## 🔧 Final Setup Steps

### Step 1: Replace Placeholders

```bash
# Replace all placeholders with your actual values
SERVICE_NAME="YourServiceName"
DOMAIN="snaps.yourdomain.com"
EMAIL="admin@yourdomain.com"

# Replace in all configuration files
sudo sed -i "s/YOUR_SERVICE_NAME/$SERVICE_NAME/g" /home/snapshot/snapshot_creator.sh
sudo sed -i "s/YOUR_SERVICE_NAME/$SERVICE_NAME/g" /var/www/snapshots/index.html
sudo sed -i "s/YOUR_SERVICE_NAME/$SERVICE_NAME/g" /var/www/snapshots/public/dynamic.html
sudo sed -i "s/YOUR_SERVICE_NAME/$SERVICE_NAME/g" /var/www/snapshots/api/snapshots.js
sudo sed -i "s/YOUR_SERVICE_NAME/$SERVICE_NAME/g" /etc/systemd/system/$SERVICE_NAME-api.service
sudo sed -i "s/YOUR_DOMAIN/$DOMAIN/g" /etc/nginx/sites-available/$DOMAIN
```

### Step 2: Test Complete System

```bash
# Test manual snapshot creation
sudo -u snapshot /home/snapshot/snapshot_creator.sh

# Test API endpoints
curl https://$DOMAIN/api/info

# Test web interfaces
curl https://$DOMAIN
curl https://$DOMAIN/dynamic

# Check monitoring
sudo -u snapshot /home/snapshot/monitor.sh
```

---

## 🎉 Congratulations!

You have successfully set up your professional Story Protocol snapshot service with:

### ✅ **Advanced Features**
- **ZSTD compression** (superior to LZ4)
- **Atomic lock system** (prevents race conditions)  
- **Dual web interface** (Static + Dynamic with real-time updates)
- **Node.js API server** (JSON endpoints with auto-discovery)
- **Professional monitoring** (Comprehensive status checking)
- **Safe auto-updates** (HTML corruption prevention)

### ✅ **Enterprise-Grade Infrastructure**
- **SSL security** with automatic renewal
- **Systemd service management** 
- **Automated 6-hour scheduling**
- **Validator-safe operations**
- **aria2c multi-connection downloads**
- **Global accessibility** with JSON API

### ✅ **User Experience**
- **3-5 minute sync time** (vs days from genesis)
- **Auto-discovery downloads** (no manual file name updates)
- **Real-time interface** with live data updates
- **Professional documentation** for end users

Your users can now sync Story Protocol nodes in **minutes instead of days** with automatically updated snapshot files!

---

*🌟 You've built an enterprise-grade snapshot service that will significantly benefit the Story Protocol community!*
