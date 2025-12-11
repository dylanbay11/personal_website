# VPS Website Setup Reference
Set up I believe a mix of this Claude-original advice, perhaps others
This document represents Claude's understanding of my setup only.

## Infrastructure
- **VPS:** 30GB disk, Ubuntu 22.04 64-bit, 2GB RAM, IPv4 address
- **Domain:** dylanbay.dev (via Cloudflare)
- **Web Server:** Nginx
- **SSL:** Let's Encrypt via Certbot
- **Deployment:** GitHub Actions auto-deploy on push
- **PostgreSQL:** Instance also runs on VPS

# 2. Connect via Client (DBeaver/Python)
- Host: 127.0.0.1
- Port: 5432

## Key Directories & Files
```
/var/www/dylanbay.dev/          # Website files (git repo)
/etc/nginx/sites-available/     # Nginx config
/etc/letsencrypt/               # SSL certificates
~/.ssh/github_actions           # SSH key for GitHub Actions
```

## Nginx Configuration
```nginx
# /etc/nginx/sites-available/dylanbay.dev
server {
    listen 80;
    server_name dylanbay.dev www.dylanbay.dev;
    root /var/www/dylanbay.dev;
    index index.html index.htm;
    location / {
        try_files $uri $uri/ =404;
    }
}
```
*(Certbot modifies this to add HTTPS/443 and redirect)*

## GitHub Actions Workflow
**File:** `.github/workflows/deploy.yml`
```yaml
on: push to main branch
→ SSH to VPS
→ git pull in /var/www/dylanbay.dev
→ fix permissions
→ reload nginx
```

**Required GitHub Secrets:**
- `HOST`: VPS IP address
- `USERNAME`: SSH username (root)
- `KEY`: Private SSH key (~/.ssh/github_actions)

## Cloudflare DNS
- **A Record:** `@` → VPS IP (proxied/orange cloud)
- **A Record:** `www` → VPS IP (proxied/orange cloud)

## Maintenance Commands
```bash
# System updates
apt update && apt upgrade -y

# Check services
systemctl status nginx
certbot certificates

# Check disk space
df -h
# or
du -h --max-depth=1 / 2>/dev/null | sort -hr

# View Nginx logs
tail -f /var/log/nginx/error.log

# Test SSL renewal
certbot renew --dry-run

# Deep Clean (Kernels & Dependencies)
apt autoremove --purge -y

# Clean Snap Revisions (If /snap is huge)
set -eu; snap list --all | awk '/disabled/{print $1, $3}' | while read snapname revision; do snap remove "$snapname" --revision="$revision"; done
```

## Deployment Workflow
1. Edit files locally or on GitHub
2. `git push origin main`
3. GitHub Actions auto-deploys to VPS
4. Live at https://dylanbay.dev

## Security Notes
- SSH keys only (github_actions key for automation)
- Certbot auto-renews SSL (cron job)
- Cloudflare provides DDoS protection & CDN
- Firewall allows: OpenSSH, Nginx Full

## System Maintenance & Automation
* **Timezone:** `America/Denver` (set via `timedatectl`).
* **Update Method:** `unattended-upgrades` (Security patches only).
* **Schedule:** Weekly on Tuesdays @ 04:00 Local.
    * *Config location:* `systemctl edit apt-daily-upgrade.timer`
* **Auto-Reboot:** Enabled @ 04:30 Local (if kernel update requires it).
    * *Config location:* `/etc/apt/apt.conf.d/50unattended-upgrades`
* **Space-Saving** Also removed linux-firmware extras and unused kernels are at latest +1 backup for disk space
    * Capped logs at 200MB in `/etc/systemd/journald.conf`

## Backup Strategy
- Website content: GitHub repository
- VPS config: `/etc/nginx` and SSH keys (backup manually)

# Python Flask Applet
Located without linking at tools.dylanbay.dev
.md file below probably needs compression for conciseness but erring on side of caution for now

## Infrastructure Additions
- Python Runtime: uv (Fast Python package manager)
- App Server: Gunicorn (Python WSGI HTTP Server)
- Framework: Flask
- Cloned in Desktop/Learning, like personal_website

```
New Key Directories
/var/www/website-tools/         # Python App (git repo)
/root/.local/bin/uv             # uv binary location
/etc/systemd/system/website-tools.service  # App Service config
Nginx Configuration (Subdomain Proxy)
File: /etc/nginx/sites-available/website-tools

Nginx

server {
    server_name tools.dylanbay.dev;
    location / {
        proxy_pass http://127.0.0.1:8000; # Forward to Python
        # + Standard Proxy Headers
    }
}
Systemd Service (Python App)
File: /etc/systemd/system/website-tools.service

User: root

Working Directory: /var/www/website-tools

Command: /var/www/website-tools/.venv/bin/gunicorn --workers 3 --bind 127.0.0.1:8000 app:app

Environment: PATH includes .venv/bin
```

## GitHub Actions for App
Repo: website-tools Workflow: Same logic as main site, but runs:
git pull
/root/.local/bin/uv sync (Update dependencies)
systemctl restart website-tools (Restart app)

## New Maintenance Commands
**Restart the Python App (after manual changes)**
systemctl restart website-tools

**Check App Status/Logs**
systemctl status website-tools
journalctl -u website-tools -f

**Update Python Dependencies manually**
cd /var/www/website-tools && /root/.local/bin/uv sync

**Check Database Status**
systemctl status postgresql

**Enter SQL Shell (on server)**
sudo -i -u postgres psql

**Backup Database**
pg_dump analysis > analysis_backup.sql
