# VPS Website Setup Reference
Set up I believe a mix of this Claude-original advice, perhaps others
This document represents Claude's understanding of my setup only.

## Infrastructure
- **VPS:** 30GB disk, Ubuntu 22.04 64-bit, 2GB RAM, IPv4 address
- **Domain:** dylanbay.dev (via Cloudflare)
- **Web Server:** Nginx
- **SSL:** Let's Encrypt via Certbot
- **Deployment:** GitHub Actions auto-deploy on push

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

# View Nginx logs
tail -f /var/log/nginx/error.log

# Test SSL renewal
certbot renew --dry-run
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

## Backup Strategy
- Website content: GitHub repository
- VPS config: `/etc/nginx` and SSH keys (backup manually)
