# Linux & Shell Reference

## Table of Contents
1. Essential Commands
2. File Permissions
3. Shell Scripting
4. Systemd Services
5. Cron Jobs
6. Networking Commands
7. Process Management
8. File Structure

---

## 1. Essential Commands

```bash
# ── Navigation ─────────────────────────────────────────
pwd                         # Print working directory
ls -la                      # List all files with details
cd /var/log                 # Change directory
cd ~                        # Go to home directory
cd -                        # Go to previous directory

# ── Files & Directories ───────────────────────────────
mkdir -p app/src/utils      # Create nested directories
cp -r src/ backup/          # Copy directory recursively
mv old-name.txt new-name.txt
rm -rf dist/                # ⚠️ Force delete (no undo!)
ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/app  # Symlink

# ── Viewing files ─────────────────────────────────────
cat file.txt                # Print whole file
less file.txt               # Paginated view (q to quit)
head -n 20 file.txt         # First 20 lines
tail -n 50 file.txt         # Last 50 lines
tail -f /var/log/app.log    # Follow live log output

# ── Search ────────────────────────────────────────────
grep -r "error" /var/log/   # Search recursively
grep -n "TODO" src/*.js     # Show line numbers
find /app -name "*.env"     # Find files by name
find /app -mtime -1         # Files modified in last 24h
locate nginx.conf           # Fast file search (uses index)

# ── Disk & Memory ─────────────────────────────────────
df -h                       # Disk usage per filesystem
du -sh /var/log/*           # Size of each item
free -h                     # RAM usage
lsblk                       # List block devices

# ── Text processing ───────────────────────────────────
cat access.log | grep "POST" | awk '{print $1}' | sort | uniq -c | sort -rn
#              filter method   extract IP          sort  count dups  sort by count

sed -i 's/old-value/new-value/g' config.txt   # In-place find & replace
cut -d',' -f1,3 data.csv                      # Extract columns 1 and 3
jq '.items[] | .name' data.json               # Parse JSON
```

---

## 2. File Permissions

```bash
# Permission format: [type][owner][group][others]
# Example: -rwxr-xr-- = file, owner=rwx, group=r-x, others=r--

chmod 755 script.sh         # rwxr-xr-x (owner full, others read+exec)
chmod 644 config.txt        # rw-r--r-- (owner read+write, others read)
chmod +x deploy.sh          # Add execute permission
chmod -R 755 /var/www/      # Recursive

chown deploy:deploy app/    # Change owner:group
chown -R www-data /var/www/ # Recursive

# Numeric shorthand:
# 4=read  2=write  1=execute
# 7=rwx   6=rw-   5=r-x   4=r--
```

---

## 3. Shell Scripting

```bash
#!/bin/bash
# scripts/deploy.sh — Example deploy script
set -euo pipefail   # Exit on error, undefined vars, pipe failures

# Variables
APP_NAME="my-app"
DEPLOY_DIR="/opt/${APP_NAME}"
LOG_FILE="/var/log/${APP_NAME}/deploy.log"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

# Functions
log() {
    echo "[${TIMESTAMP}] $1" | tee -a "$LOG_FILE"
}

check_dependencies() {
    for cmd in docker kubectl git; do
        if ! command -v "$cmd" &> /dev/null; then
            log "ERROR: $cmd is not installed"
            exit 1
        fi
    done
}

# Conditional
if [[ -z "${DEPLOY_ENV:-}" ]]; then
    log "ERROR: DEPLOY_ENV is not set"
    exit 1
fi

# Loop
for service in api worker scheduler; do
    log "Restarting $service..."
    kubectl rollout restart deployment/"${APP_NAME}-${service}"
done

# Array
ENVIRONMENTS=("staging" "production")
for env in "${ENVIRONMENTS[@]}"; do
    echo "Processing: $env"
done

# Read output of a command
CURRENT_IMAGE=$(kubectl get deployment my-app -o jsonpath='{.spec.template.spec.containers[0].image}')
log "Current image: $CURRENT_IMAGE"

# Check exit code
if kubectl rollout status deployment/"$APP_NAME" --timeout=120s; then
    log "Deployment successful"
else
    log "Deployment FAILED — rolling back"
    kubectl rollout undo deployment/"$APP_NAME"
    exit 1
fi
```

---

## 4. Systemd Services

```ini
# /etc/systemd/system/my-app.service
[Unit]
Description=My Application
After=network.target          # Start after network is up
Requires=postgresql.service   # Require postgres to be running

[Service]
Type=simple
User=deploy
Group=deploy
WorkingDirectory=/opt/my-app
ExecStart=/usr/bin/node /opt/my-app/dist/index.js
ExecReload=/bin/kill -HUP $MAINPID
Restart=always                # Restart if crashes
RestartSec=5s
StandardOutput=journal        # Logs go to journald
StandardError=journal
EnvironmentFile=/opt/my-app/.env

[Install]
WantedBy=multi-user.target    # Start on boot
```

```bash
# Systemd commands
sudo systemctl daemon-reload          # After editing .service file
sudo systemctl enable my-app          # Start on boot
sudo systemctl start my-app
sudo systemctl stop my-app
sudo systemctl restart my-app
sudo systemctl status my-app

# View logs
journalctl -u my-app -f               # Follow logs
journalctl -u my-app --since "1 hour ago"
```

---

## 5. Cron Jobs

```bash
# Edit crontab
crontab -e

# Format: minute hour day-of-month month day-of-week command
# ─────── ──── ──────────────── ───── ────────────── ───────
  0       2    *               *     *              /scripts/backup.sh
# At 2:00 AM every day

  */15    *    *               *     *              /scripts/healthcheck.sh
# Every 15 minutes

  0       0    1               *     *              /scripts/monthly-report.sh
# At midnight on the 1st of every month

  0       9    *               *     1-5            /scripts/weekday-job.sh
# 9 AM Monday through Friday
```

```bash
# Best practices for cron scripts
#!/bin/bash
# Always use full paths — cron has minimal $PATH
/usr/bin/docker ps
/usr/local/bin/kubectl get pods

# Redirect output to log
exec >> /var/log/my-cron.log 2>&1

# Use flock to prevent overlapping runs
flock -n /tmp/my-job.lock /scripts/long-running-job.sh
```

---

## 6. Networking Commands

```bash
# Connectivity
ping google.com
curl -I https://myapi.example.com/health    # HTTP headers only
curl -v https://myapi.example.com           # Verbose (shows TLS, headers, body)
wget -O - https://example.com/file.tar.gz | tar -xz

# DNS
nslookup myapp.example.com
dig myapp.example.com A          # A record
dig myapp.example.com MX         # Mail records

# Ports & connections
ss -tlnp                         # Listening TCP ports (modern netstat)
netstat -tlnp                    # Same (older systems)
lsof -i :3000                    # What's using port 3000?
nc -zv db.example.com 5432       # Test if port is reachable

# Firewall (ufw — Ubuntu)
sudo ufw status
sudo ufw allow 80/tcp
sudo ufw allow from 10.0.0.0/24 to any port 5432
sudo ufw deny 23

# Network interfaces
ip addr show
ip route show
```

---

## 7. Process Management

```bash
# View processes
ps aux                              # All running processes
ps aux | grep node                  # Filter by name
top                                 # Interactive process viewer
htop                                # Better interactive viewer

# Signals
kill -15 <PID>                      # Graceful shutdown (SIGTERM)
kill -9 <PID>                       # Force kill (SIGKILL)
kill -1 <PID>                       # Reload config (SIGHUP)
killall node                        # Kill all processes named "node"
pkill -f "node dist/index.js"       # Kill by full command

# Background processes
nohup ./script.sh &                 # Run in background, survives logout
./script.sh > output.log 2>&1 &    # Background with logging
jobs                                # List background jobs
fg %1                               # Bring job 1 to foreground

# Limits
ulimit -n                           # Max open files (default: 1024)
ulimit -n 65535                     # Increase for high-traffic apps
```

---

## 8. File Structure

```
scripts/
├── setup.sh            # Initial server setup (deps, users, dirs)
├── deploy.sh           # Application deployment
├── backup.sh           # Database + file backups
├── healthcheck.sh      # Check services are running
└── cleanup.sh          # Log rotation, temp file cleanup

/etc/systemd/system/
└── my-app.service      # App service definition

/etc/cron.d/
└── my-app              # App-specific cron jobs

/var/log/my-app/
├── app.log             # Application logs
└── deploy.log          # Deployment history
```