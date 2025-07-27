# 🚀 Story Protocol Advanced Snapshot Service Setup Guide

## Complete guide to create your own high-performance snapshot service with aria2c + ZSTD compression

---

## 📋 Table of Contents

- [System Requirements](#system-requirements)
- [Initial Server Setup](#initial-server-setup)
- [Dependencies Installation](#dependencies-installation)
- [Nginx Installation & Configuration](#nginx-installation--configuration)
- [SSL Certificate Setup](#ssl-certificate-setup)
- [Directory Structure Creation](#directory-structure-creation)
- [Snapshot Creation Script](#snapshot-creation-script)
- [Automated Scheduling](#automated-scheduling)
- [Web Interface Setup](#web-interface-setup)
- [Download Script for Users](#download-script-for-users)
- [Monitoring & Maintenance](#monitoring--maintenance)
- [Troubleshooting](#troubleshooting)

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

### Step 1: System Update

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install basic tools
sudo apt install -y curl git wget htop tmux build-essential jq make gcc unzip
```

### Step 2: User Management

```bash
# Create dedicated snapshot user
sudo useradd -m -s /bin/bash snapshots
sudo usermod -aG sudo snapshots

# Set up passwordless sudo for snapshot operations
echo "snapshots ALL=(ALL) NOPASSWD: /bin/systemctl" | sudo tee /etc/sudoers.d/snapshots
sudo chmod 440 /etc/sudoers.d/snapshots

# Verify user creation
id snapshots
```

---

## 📦 Dependencies Installation

### Step 1: Install Required Packages

```bash
# Install nginx, SSL tools, and compression utilities
sudo apt install -y nginx certbot python3-certbot-nginx aria2 zstd jq python3 bc

# Verify installations
nginx -v
certbot --version
aria2c --version
zstd --version
jq --version
```

---

## 🌐 Nginx Installation & Configuration

### Step 1: Basic Nginx Setup

```bash
# Stop default nginx if running
sudo systemctl stop nginx

# Remove default configuration
sudo rm -f /etc/nginx/sites-enabled/default

# Create your snapshot service configuration
sudo nano /etc/nginx/sites-available/YOUR_DOMAIN
```

**Add this configuration (replace YOUR_DOMAIN with your actual domain):**

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
    
    # Main page
    location / {
        try_files $uri $uri/ =404;
    }
    
    # Story snapshots directory
    location /story/aeneid/ {
        alias /var/www/snapshots/story/aeneid/;
        
        # Direct file downloads
        location ~* \.(tar\.zst|txt|json)$ {
            add_header Cache-Control "public, max-age=3600";
            add_header X-Snapshot-Service "YOUR_SERVICE_NAME";
        }
        
        # Prevent directory browsing, redirect to main page
        location = /story/aeneid/ {
            return 301 /#snapshots;
        }
    }
    
    # Access and error logs
    access_log /var/log/nginx/YOUR_DOMAIN.access.log;
    error_log /var/log/nginx/YOUR_DOMAIN.error.log;
}
```

### Step 2: Enable Site

```bash
# Enable your site
sudo ln -s /etc/nginx/sites-available/YOUR_DOMAIN /etc/nginx/sites-enabled/

# Test nginx configuration
sudo nginx -t

# Don't start nginx yet (we need SSL first)
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
```

### Step 2: Start Nginx

```bash
# Start and enable nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx
```

---

## 📁 Directory Structure Creation

### Step 1: Create Web Directories

```bash
# Create main web directory structure
sudo mkdir -p /var/www/snapshots/story/aeneid
sudo mkdir -p /var/www/snapshots/api
sudo mkdir -p /var/www/snapshots/scripts

# Set ownership and permissions
sudo chown -R snapshots:snapshots /var/www/snapshots
sudo chmod -R 755 /var/www/snapshots

# Verify structure
ls -la /var/www/snapshots/
```

---

## 🔧 Snapshot Creation Script

### Step 1: Create Advanced Snapshot Script

```bash
# Create the snapshot creation script
sudo nano /home/snapshots/snapshot_creator.sh
```

**Add this content:**

```bash
#!/bin/bash

# Advanced Snapshot Creator with ZSTD compression
# Replace SERVICE_NAME with your service name

set -e

# Configuration
SNAPSHOT_DIR="/var/www/snapshots/story/aeneid"
STORY_HOME="/root/.story"
LOG_FILE="/var/log/snapshots.log"
SERVICE_NAME="YOUR_SERVICE_NAME"

# Logging function
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a $LOG_FILE
}

# Auto-detect Story RPC port from config
detect_rpc_port() {
    local config_file="$STORY_HOME/story/config/config.toml"
    local default_port="26657"
    
    if [[ -f "$config_file" ]]; then
        # Extract port from laddr line in config.toml
        local port=$(grep -E "^laddr = \"tcp://.*:[0-9]+\"" "$config_file" | grep -oE "[0-9]+" | head -1)
        if [[ -n "$port" && "$port" =~ ^[0-9]+$ ]]; then
            echo "$port"
            return
        fi
    fi
    
    # Fallback to default port
    echo "$default_port"
}

# Get current block height
get_block_height() {
    local rpc_port=$(detect_rpc_port)
    local height=$(curl -s "localhost:$rpc_port/status" | jq -r '.result.sync_info.latest_block_height' 2>/dev/null)
    if [[ "$height" =~ ^[0-9]+$ ]]; then
        echo $height
    else
        echo "0"
    fi
}

# Check if node is synced
is_synced() {
    local rpc_port=$(detect_rpc_port)
    local catching_up=$(curl -s "localhost:$rpc_port/status" | jq -r '.result.sync_info.catching_up' 2>/dev/null)
    [[ "$catching_up" == "false" ]]
}

# Create snapshots
create_snapshots() {
    local current_height=$(get_block_height)
    local timestamp=$(date '+%Y%m%d')
    
    # Unique naming convention for your service
    local consensus_name="${SERVICE_NAME,,}-aeneid-consensus-${current_height}-${timestamp}.tar.zst"
    local execution_name="${SERVICE_NAME,,}-aeneid-execution-${current_height}-${timestamp}.tar.zst"
    
    log "Starting snapshot creation for block height: $current_height"
    
    # Check sync status
    if ! is_synced; then
        log "ERROR: Node is not synced"
        return 1
    fi
    
    # Remove old snapshots
    log "Removing old snapshots..."
    rm -f $SNAPSHOT_DIR/${SERVICE_NAME,,}-aeneid-*.tar.zst
    
    # Stop services safely
    log "Stopping services..."
    sudo systemctl stop story
    sleep 3
    sudo systemctl stop story-geth
    sleep 5
    
    # Backup validator state
    local temp_dir=$(mktemp -d)
    if [[ -f "$STORY_HOME/story/data/priv_validator_state.json" ]]; then
        cp "$STORY_HOME/story/data/priv_validator_state.json" "$temp_dir/validator_backup.json"
    fi
    
    # Create consensus snapshot with ZSTD
    log "Creating consensus snapshot with ZSTD compression..."
    cd "$STORY_HOME/story"
    tar -cf - data | zstd -3 -T0 > "$SNAPSHOT_DIR/$consensus_name"
    
    # Create execution snapshot with ZSTD
    log "Creating execution snapshot with ZSTD compression..."
    cd "$STORY_HOME/geth/aeneid/geth"
    tar -cf - chaindata | zstd -3 -T0 > "$SNAPSHOT_DIR/$execution_name"
    
    # Set permissions
    chown snapshots:snapshots "$SNAPSHOT_DIR/$consensus_name"
    chown snapshots:snapshots "$SNAPSHOT_DIR/$execution_name"
    chmod 644 "$SNAPSHOT_DIR/$consensus_name"
    chmod 644 "$SNAPSHOT_DIR/$execution_name"
    
    # Create metadata JSON
    cat > "$SNAPSHOT_DIR/snapshot-info.json" << EOL
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
    
    # Create height file
    echo "$current_height" > "$SNAPSHOT_DIR/snapshot-height.txt"
    
    # Set metadata permissions
    chown snapshots:snapshots "$SNAPSHOT_DIR/snapshot-info.json"
    chown snapshots:snapshots "$SNAPSHOT_DIR/snapshot-height.txt"
    chmod 644 "$SNAPSHOT_DIR/snapshot-info.json"
    chmod 644 "$SNAPSHOT_DIR/snapshot-height.txt"
    
    # Restore validator state
    if [[ -f "$temp_dir/validator_backup.json" ]]; then
        cp "$temp_dir/validator_backup.json" "$STORY_HOME/story/data/priv_validator_state.json"
    fi
    
    # Start services
    log "Starting services..."
    sudo systemctl start story-geth
    sleep 10
    sudo systemctl start story
    
    # Cleanup
    rm -rf "$temp_dir"
    
    log "Snapshot creation completed: $consensus_name, $execution_name"
}

# Main execution
main() {
    log "=== $SERVICE_NAME Snapshot Creation Started ==="
    
    # Check if services are running
    if ! systemctl is-active --quiet story || ! systemctl is-active --quiet story-geth; then
        log "ERROR: Story services not running"
        exit 1
    fi
    
    # Create snapshots
    if create_snapshots; then
        log "=== $SERVICE_NAME Snapshot Creation Completed ==="
    else
        log "=== $SERVICE_NAME Snapshot Creation Failed ==="
        exit 1
    fi
}

main "$@"
```

### Step 2: Configure Script

```bash
# Replace SERVICE_NAME in the script with your service name
sed -i 's/YOUR_SERVICE_NAME/MySnaps/g' /home/snapshots/snapshot_creator.sh

# Set permissions
chmod +x /home/snapshots/snapshot_creator.sh
chown snapshots:snapshots /home/snapshots/snapshot_creator.sh

# Create log file
sudo touch /var/log/snapshots.log
sudo chown snapshots:snapshots /var/log/snapshots.log
```

---

## ⏰ Automated Scheduling

### Step 1: Setup Cron Job

```bash
# Setup cron job for snapshots user
sudo -u snapshots crontab << 'EOF'
# Advanced Snapshot Service - ZSTD Compression
# Create snapshots every 6 hours at 00:00, 06:00, 12:00, 18:00
0 0,6,12,18 * * * /home/snapshots/snapshot_creator.sh >> /var/log/snapshots.log 2>&1

# Cleanup logs monthly
0 0 1 * * find /var/log -name "snapshots.log" -size +100M -exec truncate -s 50M {} \;
EOF

# Verify cron job
sudo -u snapshots crontab -l
```

---

## 🎨 Web Interface Setup

### Step 1: Create Main Index Page

```bash
# Create the main index.html
sudo nano /var/www/snapshots/index.html
```

**Add this content (customize SERVICE_NAME, DOMAIN, etc.):**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Story Protocol Snapshots</title>
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
        .header p { font-size: 1.2em; opacity: 0.9; }
        .info-banner {
            background: rgba(255, 255, 255, 0.15); border-radius: 15px; padding: 20px;
            margin-bottom: 30px; text-align: center; border: 1px solid rgba(255, 255, 255, 0.2);
        }
        .update-time {
            background: rgba(255, 255, 255, 0.2); padding: 10px 20px; border-radius: 25px;
            display: inline-block; margin-top: 15px; font-size: 0.9em;
        }
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
        .stat-icon { font-size: 2.5em; margin-bottom: 10px; }
        .stat-label { font-size: 0.9em; opacity: 0.8; margin-bottom: 5px; }
        .stat-value { font-size: 1.4em; font-weight: bold; color: #ffd700; }
        .snapshots-section {
            background: rgba(255, 255, 255, 0.1); border-radius: 20px; padding: 30px;
            backdrop-filter: blur(15px); border: 1px solid rgba(255, 255, 255, 0.2); margin-bottom: 30px;
        }
        .section-title { font-size: 1.8em; margin-bottom: 20px; color: #ffd700; text-align: center; }
        .file-grid { display: grid; gap: 15px; }
        .file-item {
            background: rgba(255, 255, 255, 0.1); border-radius: 12px; padding: 20px;
            display: grid; grid-template-columns: 1fr auto; align-items: center; gap: 20px;
            transition: all 0.3s ease; border-left: 4px solid #ffd700;
        }
        .file-item:hover { background: rgba(255, 255, 255, 0.2); transform: translateX(5px); }
        .file-info { display: flex; align-items: center; flex: 1; }
        .file-icon { font-size: 2em; margin-right: 15px; }
        .file-details h3 { margin-bottom: 5px; color: #ffd700; font-size: 1.1em; }
        .file-details p { opacity: 0.8; font-size: 0.9em; }
        .file-meta {
            display: flex; flex-direction: column; align-items: flex-end;
            gap: 10px; position: relative; z-index: 10;
        }
        .file-size {
            font-weight: bold; color: #ffd700; font-size: 1.2em;
            background: rgba(0, 0, 0, 0.2); padding: 5px 10px; border-radius: 10px;
        }
        .download-btn {
            background: linear-gradient(45deg, #ffd700, #ffed4e); color: #333;
            padding: 10px 20px; border-radius: 20px; text-decoration: none;
            font-weight: bold; font-size: 0.9em; transition: transform 0.3s ease;
            box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
        }
        .download-btn:hover { transform: scale(1.05); box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3); }
        .instructions-btn {
            background: linear-gradient(45deg, #ffd700, #ffed4e); color: #333;
            padding: 15px 30px; border-radius: 25px; text-decoration: none;
            font-weight: bold; font-size: 1.1em; transition: transform 0.3s ease;
            display: inline-block; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
        }
        .instructions-btn:hover { transform: scale(1.05); }
        .footer { text-align: center; margin-top: 40px; opacity: 0.8; }
        .footer a { color: #ffd700; text-decoration: none; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🌟 Story Protocol Snapshots</h1>
            <p>Advanced ZSTD Compression • High-Performance Infrastructure</p>
        </div>
        
        <div class="info-banner">
            <h3>⚡ ZSTD Compressed Snapshots</h3>
            <p>Download time: 2-4 hours • Updated every 6 hours • Validator safe • Superior compression</p>
            <div class="update-time">🕐 Next update: Every 00:00, 06:00, 12:00, 18:00 UTC</div>
        </div>
        
        <div class="stats-grid">
            <div class="stat-card">
                <div class="stat-icon">📊</div>
                <div class="stat-label">Block Height</div>
                <div class="stat-value">Loading...</div>
            </div>
            <div class="stat-card">
                <div class="stat-icon">🗜️</div>
                <div class="stat-label">Compression</div>
                <div class="stat-value">ZSTD</div>
            </div>
            <div class="stat-card">
                <div class="stat-icon">⚡</div>
                <div class="stat-label">Download Tool</div>
                <div class="stat-value">aria2c</div>
            </div>
            <div class="stat-card">
                <div class="stat-icon">🔄</div>
                <div class="stat-label">Update Freq</div>
                <div class="stat-value">6 Hours</div>
            </div>
        </div>
        
        <div class="snapshots-section" id="snapshots">
            <h2 class="section-title">📂 Latest Snapshots</h2>
            <div class="file-grid">
                <div class="file-item">
                    <div class="file-info">
                        <div class="file-icon">📄</div>
                        <div class="file-details">
                            <h3>snapshot-height.txt</h3>
                            <p>Current snapshot block height</p>
                        </div>
                    </div>
                    <div class="file-meta">
                        <div class="file-size">8 B</div>
                        <a href="/story/aeneid/snapshot-height.txt" class="download-btn">📥 View</a>
                    </div>
                </div>
                
                <div class="file-item">
                    <div class="file-info">
                        <div class="file-icon">🔹</div>
                        <div class="file-details">
                            <h3>SERVICE_NAME-aeneid-consensus-HEIGHT-DATE.tar.zst</h3>
                            <p>Story consensus layer (ZSTD compressed)</p>
                        </div>
                    </div>
                    <div class="file-meta">
                        <div class="file-size">~36G</div>
                        <div style="font-size: 0.8em;">Latest available</div>
                    </div>
                </div>
                
                <div class="file-item">
                    <div class="file-info">
                        <div class="file-icon">🟣</div>
                        <div class="file-details">
                            <h3>SERVICE_NAME-aeneid-execution-HEIGHT-DATE.tar.zst</h3>
                            <p>Geth execution layer (ZSTD compressed)</p>
                        </div>
                    </div>
                    <div class="file-meta">
                        <div class="file-size">~15G</div>
                        <div style="font-size: 0.8em;">Latest available</div>
                    </div>
                </div>

                <div class="file-item">
                    <div class="file-info">
                        <div class="file-icon">📋</div>
                        <div class="file-details">
                            <h3>snapshot-info.json</h3>
                            <p>Snapshot metadata and service info</p>
                        </div>
                    </div>
                    <div class="file-meta">
                        <div class="file-size">~300 B</div>
                        <a href="/story/aeneid/snapshot-info.json" class="download-btn">📥 View</a>
                    </div>
                </div>
            </div>
            
            <div style="text-align: center; margin-top: 30px;">
                <a href="#download-instructions" class="instructions-btn">
                    📋 Download Instructions (aria2c + zstd)
                </a>
            </div>
        </div>
        
        <div class="footer">
            <p>🚀 Powered by <strong>YOUR_SERVICE_NAME</strong> | Advanced ZSTD compression technology</p>
            <p>📞 Support: <a href="mailto:YOUR_EMAIL">Email</a> | 🌐 Website: <a href="https://YOUR_WEBSITE">YOUR_WEBSITE</a></p>
        </div>
    </div>

    <script>
        // Fetch and display current block height
        fetch('/story/aeneid/snapshot-height.txt')
            .then(response => response.text())
            .then(height => {
                document.querySelector('.stat-value').textContent = height.trim();
            })
            .catch(() => {
                document.querySelector('.stat-value').textContent = 'N/A';
            });
    </script>
</body>
</html>
```

### Step 2: Customize Your Index Page

```bash
# Replace placeholders with your information
sudo sed -i 's/YOUR_SERVICE_NAME/MySnaps/g' /var/www/snapshots/index.html
sudo sed -i 's/YOUR_EMAIL/admin@mydomain.com/g' /var/www/snapshots/index.html
sudo sed -i 's/YOUR_WEBSITE/mydomain.com/g' /var/www/snapshots/index.html

# Set permissions
sudo chown snapshots:snapshots /var/www/snapshots/index.html
```

---

## 📥 Download Script for Users

### Step 1: Create User Download Documentation

Create a documentation file for your users:

```bash
# Create user guide
sudo nano /var/www/snapshots/download-guide.md
```

**Add this content:**

```markdown
# Download Guide - Story Protocol Snapshots

## Prerequisites

Install required packages:

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install -y aria2 zstd jq curl

# Verify installation
aria2c --version && zstd --version
```

## Download Method

```bash
#!/bin/bash

# Your Service Snapshot Download
# Using aria2c + ZSTD for maximum performance

set -e

echo "🌟 Advanced Snapshot Download"
echo "============================="

# Configuration
SNAPSHOT_BASE="https://YOUR_DOMAIN/story/aeneid"
STORY_DATA="$HOME/.story"
TEMP_DIR="/tmp/snapshot_sync"

# Auto-detect Story RPC port from user's config
detect_story_port() {
    local config_file="$STORY_DATA/story/config/config.toml"
    local default_port="26657"
    
    if [[ -f "$config_file" ]]; then
        # Extract port from laddr line: laddr = "tcp://127.0.0.1:16657"
        local port=$(grep -E "^laddr = \"tcp://.*:[0-9]+\"" "$config_file" | grep -oE "[0-9]+" | head -1)
        if [[ -n "$port" && "$port" =~ ^[0-9]+$ ]]; then
            echo "$port"
            return
        fi
    fi
    
    # Fallback to default if config not found or port not detected
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

# Discover snapshots via JSON API
echo "🔍 Discovering latest snapshots..."
METADATA=$(curl -s "$SNAPSHOT_BASE/snapshot-info.json")
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
    --user-agent="SnapshotClient/2.0" \
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
    --user-agent="SnapshotClient/2.0" \
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
echo "✅ Snapshot download completed successfully!"
echo "📊 Monitor sync with: sudo journalctl -u story -u story-geth -f"

# Show final status
sleep 5
LOCAL_HEIGHT=$(curl -s "$LOCAL_RPC/status" 2>/dev/null | jq -r '.result.sync_info.latest_block_height' || echo "Checking...")
echo "🎯 Node restarted at block: $LOCAL_HEIGHT (RPC: $LOCAL_RPC)"
echo "📈 Target height was: $BLOCK_HEIGHT"
```
```

---

## 📊 Monitoring & Maintenance

### Step 1: Create Monitoring Script

```bash
# Create monitoring script
sudo nano /home/snapshots/monitor.sh
```

**Add monitoring function to the script:**

```bash
# Add this function after the other detection functions
detect_rpc_port() {
    local config_file="$STORY_HOME/story/config/config.toml"
    local default_port="26657"
    
    if [[ -f "$config_file" ]]; then
        # Extract port from config.toml: laddr = "tcp://127.0.0.1:16657"
        local port=$(grep -E "^laddr = \"tcp://.*:[0-9]+\"" "$config_file" | grep -oE "[0-9]+" | head -1)
        if [[ -n "$port" && "$port" =~ ^[0-9]+$ ]]; then
            echo "$port"
            return
        fi
    fi
    
    echo "$default_port"
}
```

**Add this content:**

```bash
#!/bin/bash

echo "📊 Snapshot Service Status Monitor"
echo "=================================="

# Check web service
echo "🌐 Web Service:"
curl -I https://YOUR_DOMAIN/story/aeneid/snapshot-height.txt 2>/dev/null | head -1 || echo "❌ Web service not accessible"

# Check snapshot files
echo ""
echo "📁 Current Snapshots:"
ls -lah /var/www/snapshots/story/aeneid/ | grep -E "\.(tar\.zst|json|txt)$"

# Check disk usage
echo ""
echo "💾 Disk Usage:"
df -h /var/www/snapshots

# Check cron status
echo ""
echo "📅 Cron Status:"
sudo -u snapshots crontab -l | grep -E "^[0-9]"

# Check recent logs
echo ""
echo "📋 Recent Activity:"
tail -5 /var/log/snapshots.log 2>/dev/null || echo "No logs found"

# Check Story node status
echo ""
echo "🔗 Story Node Status:"

# Auto-detect RPC port
STORY_PORT=$(detect_rpc_port)
LOCAL_RPC="localhost:$STORY_PORT"

curl -s "$LOCAL_RPC/status" 2>/dev/null | jq -r '.result.sync_info | "Height: \(.latest_block_height) | Synced: \(.catching_up == false) | Port: " + env.STORY_PORT' || echo "Cannot connect to Story node (tried port $STORY_PORT)"

echo ""
echo "=================================="
```

### Step 2: Set Up Log Rotation

```bash
# Create logrotate configuration
sudo nano /etc/logrotate.d/snapshots
```

**Add this content:**

```
/var/log/snapshots.log {
    daily
    missingok
    rotate 30
    compress
    notifempty
    create 644 snapshots snapshots
}
```

---

## 🔧 Troubleshooting

### Common Issues and Solutions

#### 1. **Nginx 502 Bad Gateway**
```bash
# Check nginx error logs
sudo tail -f /var/log/nginx/error.log

# Restart nginx
sudo systemctl restart nginx
```

#### 2. **SSL Certificate Issues**
```bash
# Renew certificate manually
sudo certbot renew

# Check certificate status
sudo certbot certificates
```

#### 3. **Snapshot Creation Fails**
```bash
# Check Story node status
sudo journalctl -u story -n 50

# Check disk space
df -h /var/www/snapshots

# Check permissions
ls -la /var/www/snapshots/story/aeneid/
```

#### 4. **Download Speed Issues**
```bash
# Test server bandwidth
wget --progress=dot:mega http://speedtest.wdc01.softlayer.com/downloads/test100.zip

# Check aria2c configuration
aria2c --help | grep -E "(max-connection|split)"
```

#### 5. **ZSTD Compression Issues**
```bash
# Test ZSTD installation
zstd --version

# Test compression
echo "test" | zstd | zstd -d
```

---

## 🎯 Final Steps

### Step 1: Test Complete System

```bash
# Run manual snapshot creation
sudo -u snapshots /home/snapshots/snapshot_creator.sh

# Check web interface
curl https://YOUR_DOMAIN

# Test download script
# (Use the script provided in Download Guide section)
```

### Step 2: Monitor First Automated Run

```bash
# Check cron logs
sudo grep CRON /var/log/syslog | tail -5

# Monitor next scheduled run
sudo -u snapshots crontab -l
```

---

## 🎉 Congratulations!

You have successfully set up your own advanced Story Protocol snapshot service with:

- ✅ **ZSTD compression** (superior to LZ4)
- ✅ **aria2c multi-connection downloads**
- ✅ **Automated 6-hour scheduling**
- ✅ **Professional web interface**
- ✅ **SSL security**
- ✅ **Validator-safe operations**
- ✅ **Monitoring and maintenance tools**

Your users can now sync in **3-5 minutes** instead of **days**!

---

*🌟 You've built a professional-grade snapshot service that will help the Story Protocol community!*
