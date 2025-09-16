# PeerTube Installation Guide

PeerTube is a free, decentralized, and federated video platform powered by ActivityPub and WebTorrent. It allows you to host your own video sharing platform while remaining connected to a larger network of servers.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

- Operating system: Linux (Debian/Ubuntu recommended)
- Node.js 16.x or 18.x
- PostgreSQL 10+ (13+ recommended)
- Redis 6.0+
- FFmpeg 4.1+ with x264 codec
- Python 3.x
- nginx or Apache as reverse proxy
- Minimum 2GB RAM (4GB+ recommended)
- Storage space for videos
- Domain name with proper DNS configuration
- SSL certificate (Let's Encrypt recommended)


## 2. Supported Operating Systems

This guide supports installation on:
- RHEL 8/9 and derivatives (CentOS Stream, Rocky Linux, AlmaLinux)
- Debian 11/12
- Ubuntu 20.04/22.04/24.04 LTS
- Arch Linux (rolling release)
- Alpine Linux 3.18+
- openSUSE Leap 15.5+ / Tumbleweed
- SUSE Linux Enterprise Server (SLES) 15+
- macOS 12+ (Monterey and later) 
- FreeBSD 13+
- Windows 10/11/Server 2019+ (where applicable)

## 3. Installation

### RHEL/CentOS/Rocky Linux/AlmaLinux

```bash
# Enable required repositories
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled powertools  # CentOS/Rocky 8
sudo dnf config-manager --set-enabled crb         # AlmaLinux 9

# Install Node.js 18
curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
sudo dnf install -y nodejs

# Install dependencies
sudo dnf install -y \
    postgresql postgresql-server postgresql-contrib \
    redis \
    python3 python3-pip \
    ffmpeg \
    gcc-c++ make \
    openssl openssl-devel \
    git wget curl \
    nginx \
    certbot python3-certbot-nginx

# Install Yarn
npm install -g yarn

# Initialize PostgreSQL
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql redis nginx
```

### Debian/Ubuntu

```bash
# Update system
sudo apt update
sudo apt upgrade -y

# Install Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install dependencies
sudo apt install -y \
    postgresql postgresql-contrib \
    redis-server \
    ffmpeg \
    python3 python3-dev python3-pip \
    git curl wget \
    build-essential \
    openssl \
    nginx \
    certbot python3-certbot-nginx

# Install Yarn
sudo npm install -g yarn

# Start services
sudo systemctl enable --now postgresql redis nginx
```

### Alpine Linux

```bash
# Install dependencies
apk add --no-cache \
    nodejs npm yarn \
    postgresql postgresql-client \
    redis \
    ffmpeg \
    python3 py3-pip \
    git curl wget \
    build-base \
    openssl openssl-dev \
    nginx

# Start services
rc-update add postgresql default
rc-update add redis default
rc-update add nginx default
rc-service postgresql start
rc-service redis start
rc-service nginx start
```

### FreeBSD

```bash
# Install from packages
pkg install -y \
    node18 npm-node18 \
    postgresql13-server postgresql13-client \
    redis \
    ffmpeg \
    python39 py39-pip \
    git curl wget \
    nginx

# Enable services
sysrc postgresql_enable="YES"
sysrc redis_enable="YES"
sysrc nginx_enable="YES"

# Initialize PostgreSQL
service postgresql initdb
service postgresql start
service redis start
service nginx start

# Install Yarn
npm install -g yarn
```

## PeerTube Installation

### Create System User

```bash
# Create peertube user
sudo useradd -m -d /var/www/peertube -s /bin/bash peertube
sudo passwd peertube

# Create directories
sudo mkdir -p /var/www/peertube
sudo chown -R peertube:peertube /var/www/peertube
```

### Database Setup

```bash
# Create PostgreSQL user and database
sudo -u postgres createuser -P peertube
sudo -u postgres createdb -O peertube -E UTF8 -T template0 peertube_prod

# Enable required extensions
sudo -u postgres psql -d peertube_prod <<EOF
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS unaccent;
EOF
```

### Download and Install PeerTube

```bash
# Switch to peertube user
sudo -u peertube -i

# Set version
VERSION=$(curl -s https://api.github.com/repos/chocobozzz/peertube/releases/latest | grep tag_name | cut -d '"' -f 4)

# Create directory structure
cd /var/www/peertube
mkdir -p config storage versions

# Download and extract
cd versions
wget -q "https://github.com/Chocobozzz/PeerTube/releases/download/${VERSION}/peertube-${VERSION}.zip"
unzip -q peertube-${VERSION}.zip
rm peertube-${VERSION}.zip

# Create symlink
cd /var/www/peertube
ln -s versions/peertube-${VERSION} ./peertube-latest

# Install dependencies
cd ./peertube-latest
yarn install --production --pure-lockfile
```

### 4. Configuration

Create `/var/www/peertube/config/production.yaml`:

```yaml
listen:
  hostname: '127.0.0.1'
  port: 9000

webserver:
  https: true
  hostname: 'peertube.example.com'
  port: 443

rates_limit:
  api:
    window: 10 minutes
    max: 500
  signup:
    window: 5 minutes
    max: 10
  login:
    window: 5 minutes
    max: 15

trust_proxy:
  - 'loopback'
  - '127.0.0.1'

database:
  hostname: 'localhost'
  port: 5432
  ssl: false
  suffix: '_prod'
  username: 'peertube'
  password: 'your_database_password'
  pool:
    max: 5

redis:
  hostname: 'localhost'
  port: 6379
  auth: null
  db: 0

smtp:
  transport: smtp
  hostname: smtp.gmail.com
  port: 465
  username: your-email@gmail.com
  password: your-app-password
  tls: true
  disable_starttls: false
  ca_file: null
  from_address: 'noreply@peertube.example.com'

email:
  body:
    signature: "PeerTube"
  subject:
    prefix: "[PeerTube]"

storage:
  tmp: '/var/www/peertube/storage/tmp/'
  bin: '/var/www/peertube/storage/bin/'
  avatars: '/var/www/peertube/storage/avatars/'
  videos: '/var/www/peertube/storage/videos/'
  streaming_playlists: '/var/www/peertube/storage/streaming-playlists/'
  redundancy: '/var/www/peertube/storage/redundancy/'
  logs: '/var/www/peertube/storage/logs/'
  previews: '/var/www/peertube/storage/previews/'
  thumbnails: '/var/www/peertube/storage/thumbnails/'
  torrents: '/var/www/peertube/storage/torrents/'
  captions: '/var/www/peertube/storage/captions/'
  cache: '/var/www/peertube/storage/cache/'
  plugins: '/var/www/peertube/storage/plugins/'
  client_overrides: '/var/www/peertube/storage/client-overrides/'
  well_known: '/var/www/peertube/storage/well-known/'
  tmp_persistent: '/var/www/peertube/storage/tmp-persistent/'

object_storage:
  enabled: false

log:
  level: 'info'
  rotation:
    enabled: true
    max_file_size: 12MB
    max_files: 20
  anonymize_ip: false
  log_ping_requests: true
  log_tracker_unknown_infohash: true
  prettify_sql: false

trending:
  videos:
    interval_days: 7
    algorithms:
      enabled:
        - 'hot'
        - 'most-viewed'
        - 'most-liked'
      default: 'most-viewed'

redundancy:
  videos:
    check_interval: '1 hour'
    strategies: []

cache:
  previews:
    size: 500
  captions:
    size: 500
  torrents:
    size: 500

admin:
  email: 'admin@example.com'

contact_form:
  enabled: true

signup:
  enabled: false
  limit: 10
  requires_email_verification: false
  filters:
    cidr:
      whitelist: []
      blacklist: []

user:
  video_quota: -1
  video_quota_daily: -1

video_channels:
  max_per_user: 20

transcoding:
  enabled: true
  allow_additional_extensions: true
  allow_audio_files: true
  threads: 1
  concurrency: 1
  profile: 'default'
  resolutions:
    0p: false
    144p: false
    240p: false
    360p: false
    480p: true
    720p: true
    1080p: true
    1440p: false
    2160p: false
  webtorrent:
    enabled: true
  hls:
    enabled: true

live:
  enabled: false
  allow_replay: true
  latency_setting:
    enabled: true
  max_duration: -1
  max_instance_lives: 20
  max_user_lives: 3
  transcoding:
    enabled: true
    threads: 2
    profile: 'default'
    resolutions:
      144p: false
      240p: false
      360p: false
      480p: false
      720p: true
      1080p: true
      1440p: false
      2160p: false

video_studio:
  enabled: false

import:
  videos:
    concurrency: 1
    http:
      enabled: true
      youtube_dl_release:
        url: 'https://api.github.com/repos/yt-dlp/yt-dlp/releases/latest'
        name: 'yt-dlp'
        python_path: '/usr/bin/python3'
    torrent:
      enabled: false

auto_blacklist:
  videos:
    of_users:
      enabled: false

followers:
  instance:
    enabled: true
    manual_approval: false

followings:
  instance:
    auto_follow_back:
      enabled: false
    auto_follow_index:
      enabled: false

theme:
  default: 'default'

broadcast_message:
  enabled: false

search:
  remote_uri:
    users: true
    anonymous: false
  search_index:
    enabled: false

instance:
  name: 'PeerTube'
  short_description: 'PeerTube, an ActivityPub-federated video streaming platform'
  description: ''
  terms: ''
  code_of_conduct: ''
  moderation_information: ''
  creation_reason: ''
  administrator: ''
  maintenance_lifetime: ''
  business_model: ''
  hardware_information: ''
  languages: []
  categories: []
  is_nsfw: false
  default_nsfw_policy: 'do_not_list'
  customizations:
    javascript: ''
    css: ''
  robots: |
    User-agent: *
    Disallow:

services:
  twitter:
    username: ''
    whitelisted: false
```

### Initialize Database

```bash
# As peertube user
cd /var/www/peertube/peertube-latest
NODE_CONFIG_DIR=/var/www/peertube/config NODE_ENV=production npm run reset-password -- -u root

# Note the generated password!
```

## Service Configuration

### Systemd Service

Create `/etc/systemd/system/peertube.service`:

```ini
[Unit]
Description=PeerTube
After=network.target postgresql.service redis.service

[Service]
Type=simple
Environment=NODE_ENV=production
Environment=NODE_CONFIG_DIR=/var/www/peertube/config
User=peertube
Group=peertube
ExecStart=/usr/bin/node /var/www/peertube/peertube-latest/dist/server
WorkingDirectory=/var/www/peertube/peertube-latest
StandardOutput=journal
StandardError=journal
SyslogIdentifier=peertube
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable peertube
sudo systemctl start peertube
sudo systemctl status peertube
```

### OpenRC Service (Alpine)

Create `/etc/init.d/peertube`:

```bash
#!/sbin/openrc-run

name="PeerTube"
description="Decentralized video platform"

command="/usr/bin/node"
command_args="/var/www/peertube/peertube-latest/dist/server"
command_user="peertube:peertube"
command_background=true
pidfile="/run/${RC_SVCNAME}.pid"

export NODE_ENV="production"
export NODE_CONFIG_DIR="/var/www/peertube/config"

depend() {
    need net
    use postgresql redis
}

start_pre() {
    checkpath -d -o peertube:peertube /var/www/peertube/storage
}
```

## Reverse Proxy Configuration

### nginx

```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name peertube.example.com;
    
    location /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/certbot;
    }
    
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# Main server block
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name peertube.example.com;

    ssl_certificate /etc/letsencrypt/live/peertube.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/peertube.example.com/privkey.pem;

    # Security headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains";
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;

    # Configure logs
    access_log /var/log/nginx/peertube.access.log;
    error_log /var/log/nginx/peertube.error.log;

    # Configure maximum upload size
    client_max_body_size 8G;
    proxy_connect_timeout 600s;
    proxy_send_timeout 600s;
    proxy_read_timeout 600s;
    send_timeout 600s;

    # Main application
    location / {
        proxy_pass http://127.0.0.1:9000;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Bypass PeerTube for performance (static files)
    location ~ ^/static/(thumbnails|avatars)/ {
        if ($request_method = 'OPTIONS') {
            add_header Access-Control-Allow-Origin '*';
            add_header Access-Control-Allow-Methods 'GET, OPTIONS';
            add_header Access-Control-Allow-Headers 'Range,DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
            add_header Access-Control-Max-Age 1728000;
            add_header Content-Type 'text/plain; charset=utf-8';
            add_header Content-Length 0;
            return 204;
        }

        add_header Access-Control-Allow-Origin '*';
        add_header Access-Control-Allow-Methods 'GET, OPTIONS';
        add_header Access-Control-Allow-Headers 'Range,DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
        add_header Cache-Control "public, max-age=7200";

        alias /var/www/peertube/storage/;
    }

    # Websocket
    location /socket.io {
        proxy_pass http://127.0.0.1:9000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Apache

```apache
<VirtualHost *:80>
    ServerName peertube.example.com
    
    RewriteEngine On
    RewriteCond %{REQUEST_URI} !^/\.well-known/acme-challenge/
    RewriteCond %{HTTPS} off
    RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
</VirtualHost>

<VirtualHost *:443>
    ServerName peertube.example.com
    
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/peertube.example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/peertube.example.com/privkey.pem
    
    # Security headers
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains"
    Header always set X-Content-Type-Options nosniff
    Header always set X-XSS-Protection "1; mode=block"
    Header always set X-Robots-Tag none
    
    # Logs
    CustomLog /var/log/apache2/peertube.access.log combined
    ErrorLog /var/log/apache2/peertube.error.log
    
    # Enable proxy modules
    ProxyPreserveHost On
    ProxyRequests Off
    ProxyTimeout 600
    
    # WebSocket
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} websocket [NC]
    RewriteCond %{HTTP:Connection} upgrade [NC]
    RewriteRule ^/?(.*) "ws://127.0.0.1:9000/$1" [P,L]
    
    # Main application
    ProxyPass / http://127.0.0.1:9000/
    ProxyPassReverse / http://127.0.0.1:9000/
    
    # Configure max upload size (8GB)
    LimitRequestBody 8589934592
    
    # Static files bypass
    Alias /static/thumbnails /var/www/peertube/storage/thumbnails
    Alias /static/avatars /var/www/peertube/storage/avatars
    
    <Directory /var/www/peertube/storage>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
```

## Post-Installation

### SSL Certificate

```bash
# Using Let's Encrypt
sudo certbot --nginx -d peertube.example.com

# Or for Apache
sudo certbot --apache -d peertube.example.com
```

### Configure Admin User

```bash
# Login to web interface with root user and generated password
# Navigate to https://peertube.example.com
# Go to Administration > Users
# Create a new admin user
# Disable or change root password
```

### Storage Configuration

```bash
# Create storage directories
sudo -u peertube mkdir -p /var/www/peertube/storage/{tmp,bin,avatars,videos,streaming-playlists,redundancy,logs,previews,thumbnails,torrents,captions,cache,plugins,client-overrides,well-known,tmp-persistent}

# For large storage on separate disk
sudo mkdir -p /mnt/peertube-storage
sudo mount /dev/sdb1 /mnt/peertube-storage
sudo chown -R peertube:peertube /mnt/peertube-storage

# Update config to use new path
# In production.yaml:
# storage:
#   videos: '/mnt/peertube-storage/videos/'
```

## Performance Optimization

### PostgreSQL Tuning

```sql
-- Optimize for PeerTube workload
ALTER SYSTEM SET shared_buffers = '2GB';
ALTER SYSTEM SET effective_cache_size = '6GB';
ALTER SYSTEM SET maintenance_work_mem = '512MB';
ALTER SYSTEM SET random_page_cost = 1.1;

-- Reload configuration
SELECT pg_reload_conf();
```

### Redis Configuration

```bash
# Edit /etc/redis/redis.conf or /etc/redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

### Transcoding Optimization

```yaml
# In production.yaml
transcoding:
  enabled: true
  threads: 4  # Adjust based on CPU cores
  concurrency: 2  # Number of parallel jobs
  
  # Hardware acceleration (if available)
  # profile: 'default'  # or 'vaapi' for Intel, 'nvenc' for NVIDIA
```

## Monitoring

### Built-in Monitoring

```bash
# Check logs
sudo journalctl -u peertube -f

# Monitor disk usage
df -h /var/www/peertube/storage

# Check database connections
sudo -u postgres psql -d peertube_prod -c "SELECT count(*) FROM pg_stat_activity;"
```

### Prometheus Metrics

PeerTube exposes metrics at `/api/v1/metrics/playback`

### Health Checks

```bash
# Check service status
curl http://localhost:9000/api/v1/config

# Check federation
curl https://peertube.example.com/api/v1/server/stats
```

## 9. Backup and Restore

### Backup Script

```bash
#!/bin/bash
# backup-peertube.sh

BACKUP_DIR="/backup/peertube/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Stop PeerTube
sudo systemctl stop peertube

# Backup database
sudo -u postgres pg_dump peertube_prod | gzip > "$BACKUP_DIR/database.sql.gz"

# Backup configuration
tar czf "$BACKUP_DIR/config.tar.gz" -C /var/www/peertube config/

# Backup avatars, thumbnails, torrents
tar czf "$BACKUP_DIR/assets.tar.gz" -C /var/www/peertube/storage avatars/ thumbnails/ torrents/

# Optional: Backup videos (can be very large)
# tar czf "$BACKUP_DIR/videos.tar.gz" -C /var/www/peertube/storage videos/

# Start PeerTube
sudo systemctl start peertube

echo "Backup completed: $BACKUP_DIR"
```

### Restore Script

```bash
#!/bin/bash
# restore-peertube.sh

BACKUP_DIR="$1"
if [ -z "$BACKUP_DIR" ]; then
    echo "Usage: $0 <backup-directory>"
    exit 1
fi

# Stop PeerTube
sudo systemctl stop peertube

# Restore database
gunzip < "$BACKUP_DIR/database.sql.gz" | sudo -u postgres psql peertube_prod

# Restore configuration
tar xzf "$BACKUP_DIR/config.tar.gz" -C /var/www/peertube/

# Restore assets
tar xzf "$BACKUP_DIR/assets.tar.gz" -C /var/www/peertube/storage/

# Restore videos if backed up
# tar xzf "$BACKUP_DIR/videos.tar.gz" -C /var/www/peertube/storage/

# Fix permissions
sudo chown -R peertube:peertube /var/www/peertube

# Start PeerTube
sudo systemctl start peertube

echo "Restore completed from: $BACKUP_DIR"
```

## Security Hardening

### Firewall Configuration

```bash
# UFW
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# firewalld
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### File Permissions

```bash
# Secure configuration
sudo chmod 600 /var/www/peertube/config/production.yaml
sudo chown peertube:peertube /var/www/peertube/config/production.yaml

# Secure storage
sudo chmod 750 /var/www/peertube/storage
```

### Rate Limiting

Configure in `production.yaml`:
```yaml
rates_limit:
  api:
    window: 10 minutes
    max: 500
  signup:
    window: 5 minutes
    max: 2
  login:
    window: 5 minutes
    max: 10
```

## 6. Troubleshooting

### Common Issues

1. **Transcoding failures**:
```bash
# Check FFmpeg
ffmpeg -version

# Test transcoding manually
sudo -u peertube ffmpeg -i input.mp4 -c:v libx264 -c:a aac output.mp4

# Check logs
sudo journalctl -u peertube | grep -i transcode
```

2. **Federation issues**:
```bash
# Check WebFinger
curl https://peertube.example.com/.well-known/webfinger?resource=acct:root@peertube.example.com

# Check NodeInfo
curl https://peertube.example.com/.well-known/nodeinfo
```

3. **Upload issues**:
```bash
# Check disk space
df -h /var/www/peertube/storage

# Check permissions
ls -la /var/www/peertube/storage/tmp

# Increase nginx timeout and upload size if needed
```

## Maintenance

### Updates

```bash
# Check current version
cd /var/www/peertube/peertube-latest
cat package.json | grep version

# Update process
sudo systemctl stop peertube
sudo -u peertube -i
cd /var/www/peertube/versions
VERSION=$(curl -s https://api.github.com/repos/chocobozzz/peertube/releases/latest | grep tag_name | cut -d '"' -f 4)
wget -q "https://github.com/Chocobozzz/PeerTube/releases/download/${VERSION}/peertube-${VERSION}.zip"
unzip -q peertube-${VERSION}.zip
cd /var/www/peertube
rm ./peertube-latest
ln -s versions/peertube-${VERSION} ./peertube-latest
cd ./peertube-latest
yarn install --production --pure-lockfile
exit
sudo systemctl start peertube
```

### Database Maintenance

```bash
# Vacuum database
sudo -u postgres vacuumdb -z peertube_prod

# Reindex
sudo -u postgres reindexdb peertube_prod
```

## Additional Resources

- [Official Documentation](https://docs.joinpeertube.org/)
- [GitHub Repository](https://github.com/Chocobozzz/PeerTube)
- [Admin Documentation](https://docs.joinpeertube.org/admin)
- [API Documentation](https://docs.joinpeertube.org/api-rest-reference.html)
- [Community Forum](https://framacolibri.org/c/peertube)
- [Federation Guide](https://docs.joinpeertube.org/admin/following-instances)

---

**Note:** This guide is part of the [HowToMgr](https://howtomgr.github.io) collection. Always refer to official documentation for the most up-to-date information.