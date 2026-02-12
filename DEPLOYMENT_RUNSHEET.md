# TEMCO SRS Website — Deployment Runsheet
**Document Version:** 1.1  
**Created:** February 13, 2026  
**Last Updated:** February 13, 2026  
**Author:** DevOps Team  
**Status:** Deployed & Live

---

## 1. Project Summary

| Item | Detail |
|------|--------|
| **Project Name** | TEMCO Banking App — SRS Presentation Website |
| **Domain** | `temcosrs.temcobank.com` |
| **GitHub Repo** | https://github.com/ishanthasiribaddana/temco-srs |
| **Branch** | `main` |
| **Type** | Static HTML website (no backend, no database) |
| **Container Name** | `temco-srs` |
| **Docker Network** | `srs-network` (isolated) |
| **Internal Port** | `8093` (localhost only) |
| **Production Server** | `109.123.227.166` (Contabo VPS) |
| **SSH Alias** | `temco-prod` |
| **Server Path** | `/apps/temco-srs` |
| **SSL** | Let's Encrypt (Certbot auto-renewal) |
| **CDN / Proxy** | Cloudflare (Proxied, orange cloud) |
| **Nginx Config Path** | `/etc/nginx/conf.d/temcosrs.temcobank.com.conf` |
| **SSL Cert Path** | `/etc/letsencrypt/live/temcosrs.temcobank.com/` |
| **SSL Expiry** | May 13, 2026 (auto-renews) |

---

## 2. Technology Stack

| Layer | Technology |
|-------|-----------|
| Web Server | Nginx Alpine (Docker) |
| Container | Docker + Docker Compose |
| SSL Termination | Nginx (host-level) + Let's Encrypt |
| CDN / Proxy | Cloudflare (Proxied mode, Full SSL) |
| Content | Static HTML, CSS, JavaScript, SVG |
| Version Control | Git / GitHub |

---

## 3. File Structure

```
/apps/temco-srs/
├── index.html              # Overview page
├── architecture.html       # Architecture page
├── features.html           # Services & Features page
├── database.html           # Database Design page
├── api.html                # API Specification page
├── timeline.html           # Timeline & Risks page
├── runsheet.html           # Runsheet page
├── styles.css              # Stylesheet (brand colors)
├── script.js               # Navigation & animations
├── logo.svg                # TEMCO Bank logo
├── Dockerfile              # Docker build config
├── docker-compose.yml      # Docker Compose config
├── nginx.conf              # Container-level Nginx config
├── temcosrs-nginx-host.conf# Host-level Nginx config
├── .dockerignore           # Docker ignore rules
└── DEPLOYMENT_RUNSHEET.md  # This document
```

---

## 4. Port Map (All TEMCO Services)

| Application | Container Name | Port | Domain |
|-------------|---------------|------|--------|
| AdminApp | admin-frontend | 8089 | admin.temcobank.com |
| FinanceApp | finance-frontend | 8091 | finance.temcobank.com |
| Customer Portal | customer-portal | 8092 | portal.temcobank.com |
| Finance API (WildFly) | finance-api | 8087 | — (internal) |
| **SRS Website** | **temco-srs** | **8093** | **temcosrs.temcobank.com** |
| WildFly (SSO/Core) | — (host) | 8080 | — (internal) |
| MariaDB | temco-admin-mariadb | 3306 | — (internal) |

---

## 5. Pre-Deployment Checklist

- [x] DNS A record created: `temcosrs.temcobank.com → 109.123.227.166` (Cloudflare, Proxied)
- [x] SSH access verified: `ssh temco-prod`
- [x] Docker & Docker Compose installed on server
- [x] Port 8093 is free (not used by other services)
- [x] Nginx installed on host
- [x] Certbot installed for SSL
- [x] SSL certificate obtained via Certbot
- [x] Cloudflare SSL mode set to Full (origin must have SSL)

---

## 6. Deployment Steps (First Time)

### Step 1: Add DNS Record

In your DNS provider (domain registrar), add:

```
Type:  A
Name:  temcosrs
Value: 109.123.227.166
TTL:   300
```

Wait for DNS propagation (usually 5–15 minutes). Verify:

```bash
dig temcosrs.temcobank.com +short
# Should return: 109.123.227.166
```

### Step 2: SSH into Production Server

```bash
ssh temco-prod
# or: ssh root@109.123.227.166 -i ~/.ssh/id_ed25519_temco
```

### Step 3: Clone the Repository

```bash
mkdir -p /apps/temco-srs
cd /apps/temco-srs
git clone https://github.com/ishanthasiribaddana/temco-srs.git .
```

### Step 4: Build & Start Docker Container

```bash
cd /apps/temco-srs
docker-compose up -d --build
```

Verify container is running:

```bash
docker ps | grep temco-srs
```

Expected output:
```
CONTAINER ID  IMAGE           STATUS                   PORTS                    NAMES
xxxxxxxxxxxx  temco-srs-...   Up X seconds (healthy)   127.0.0.1:8093->80/tcp   temco-srs
```

Test locally on server:

```bash
curl -I http://127.0.0.1:8093
# Should return: HTTP/1.1 200 OK
```

### Step 5: Obtain SSL Certificate

> **IMPORTANT:** Cloudflare connects to origin on port 443 (Full SSL mode).
> The Nginx config MUST have a `listen 443 ssl` block or the site will return blank/404.

```bash
certbot certonly --nginx -d temcosrs.temcobank.com
```

### Step 6: Configure Host-Level Nginx

> **NOTE:** On this server, active configs go in `/etc/nginx/conf.d/` (not `sites-available`).
> All other TEMCO sites follow this pattern.

Copy an existing config and adapt it (avoids PowerShell/shell variable escaping issues):

```bash
cp /etc/nginx/conf.d/finance.temcobank.com.conf /etc/nginx/conf.d/temcosrs.temcobank.com.conf

# Replace domain and port
sed -i 's/finance.temcobank.com/temcosrs.temcobank.com/g; s|http://127.0.0.1:8091|http://127.0.0.1:8093|g' /etc/nginx/conf.d/temcosrs.temcobank.com.conf

# Remove API proxy blocks (not needed for static site)
sed -i '/location \/temco-bank-system-project/,/}/d; /location \/api/,/}/d' /etc/nginx/conf.d/temcosrs.temcobank.com.conf
```

The final config should look like:

```nginx
# temcosrs.temcobank.com - SRS Website (Static, No SSO)
server {
    listen 80;
    server_name temcosrs.temcobank.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name temcosrs.temcobank.com;

    ssl_certificate /etc/letsencrypt/live/temcosrs.temcobank.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/temcosrs.temcobank.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:8093;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

Test and reload:

```bash
nginx -t
nginx -s reload
```

### Step 7: Final Verification

```bash
# Test HTTPS
curl -I https://temcosrs.temcobank.com

# Expected response:
# HTTP/2 200
# server: nginx
# x-frame-options: SAMEORIGIN
# x-content-type-options: nosniff
# strict-transport-security: max-age=31536000; includeSubDomains
```

Open in browser: **https://temcosrs.temcobank.com**

---

## 7. Update / Redeploy Steps

When SRS website content is updated:

### Option A: From Local Machine (Push → Pull → Rebuild)

**On local machine:**
```bash
cd f:\TemcoERP\docs\srs-website
git add -A
git commit -m "Update SRS website content"
git push origin main
```

**On production server:**
```bash
ssh temco-prod
cd /apps/temco-srs
git pull origin main
docker-compose up -d --build
```

### Option B: Quick One-Liner (from local)

```bash
ssh temco-prod "cd /apps/temco-srs && git pull origin main && docker-compose down && docker-compose up -d --build"
```

---

## 8. Maintenance Commands

### View Container Status
```bash
docker ps | grep temco-srs
```

### View Container Logs
```bash
docker logs temco-srs
docker logs temco-srs --tail 50 -f    # Follow live
```

### Restart Container
```bash
cd /apps/temco-srs
docker-compose restart
```

### Stop Container
```bash
cd /apps/temco-srs
docker-compose down
```

### Rebuild from Scratch
```bash
cd /apps/temco-srs
docker-compose down
docker-compose up -d --build --force-recreate
```

### Check Container Health
```bash
docker inspect --format='{{.State.Health.Status}}' temco-srs
# Should return: healthy
```

### Check Disk Usage
```bash
docker system df
docker images | grep temco-srs
```

---

## 9. Nginx Management

### Test Config
```bash
nginx -t
```

### Reload (no downtime)
```bash
nginx -s reload
```

### View Nginx Logs for SRS Site
```bash
# Container-level logs
docker logs temco-srs

# Host-level Nginx logs
tail -f /var/log/nginx/access.log | grep temcosrs
tail -f /var/log/nginx/error.log | grep temcosrs
```

### Renew SSL Certificate
```bash
certbot renew --dry-run    # Test renewal
certbot renew              # Actual renewal
```

Certbot auto-renewal is typically set up via cron/systemd timer. Verify:
```bash
systemctl status certbot.timer
```

---

## 10. Troubleshooting

### Container Won't Start
```bash
docker-compose logs temco-srs
docker-compose down
docker-compose up -d --build
```

### Port 8093 Already in Use
```bash
lsof -i :8093
# Kill the conflicting process or change port in docker-compose.yml
```

### 502 Bad Gateway
```bash
# Check if container is running
docker ps | grep temco-srs

# Check if port is listening
curl http://127.0.0.1:8093

# Restart if needed
cd /apps/temco-srs
docker-compose restart
```

### SSL Certificate Issues
```bash
certbot certificates                          # List all certs
certbot renew --force-renewal -d temcosrs.temcobank.com  # Force renew
nginx -s reload
```

### DNS Not Resolving
```bash
dig temcosrs.temcobank.com +short
nslookup temcosrs.temcobank.com
# If empty, DNS record hasn't propagated yet — wait or check DNS provider
```

---

## 11. Cloudflare Notes

### SSL Mode
Cloudflare is set to **Full SSL** for `temcobank.com`. This means:
- Cloudflare connects to origin on **port 443**
- Origin **must** have a valid SSL certificate (Let's Encrypt)
- Nginx **must** have a `listen 443 ssl` server block
- Without this, Cloudflare returns 404 or blank page

### Cache Purging
If you update the site and see stale content:
1. Go to **Cloudflare Dashboard** → `temcobank.com` → **Caching** → **Configuration**
2. Click **Purge Everything**
3. Or use Custom Purge: `https://temcosrs.temcobank.com/*`

Alternatively, users can hard-refresh with **Ctrl + Shift + R**.

### DNS Record
```
Type: A
Name: temcosrs
Content: 109.123.227.166
Proxy status: Proxied (orange cloud)
TTL: Auto
```

---

## 12. Isolation & Security Notes

| Concern | Mitigation |
|---------|-----------|
| **Cross-contamination** | Separate Docker network (`srs-network`), separate container, no shared volumes |
| **Port conflict** | Port 8093 bound to `127.0.0.1` only (not exposed to internet) |
| **No backend** | Pure static site — zero attack surface from APIs/databases |
| **Independent lifecycle** | Can be stopped/rebuilt without affecting any other TEMCO service |
| **Separate repo** | `ishanthasiribaddana/temco-srs` — not part of main TemcoERP repo |
| **SSL** | Let's Encrypt with auto-renewal via Certbot |

---

## 13. Key Contacts & Access

| Resource | Detail |
|----------|--------|
| **Production Server** | `109.123.227.166` |
| **SSH** | `ssh temco-prod` or `ssh root@109.123.227.166 -i ~/.ssh/id_ed25519_temco` |
| **GitHub Repo** | https://github.com/ishanthasiribaddana/temco-srs |
| **Live URL** | https://temcosrs.temcobank.com |
| **Local Dev** | `python -m http.server 5500` in `f:\TemcoERP\docs\srs-website\` |
| **Local Preview** | http://localhost:5500 |

---

## 14. Lessons Learned (Feb 13, 2026 Deployment)

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Blank page after deployment | Nginx config only had port 80 listener. Cloudflare connects on 443 (Full SSL). | Added `listen 443 ssl` block + Let's Encrypt cert. |
| CSS/JS returning 404 | Cloudflare cached the 404 from before SSL was configured. | Purged Cloudflare cache. |
| PowerShell mangling Nginx variables | `$host`, `$remote_addr` etc. interpreted as PS variables when using heredoc over SSH. | Copy existing config + `sed` to replace values instead. |
| Config in wrong directory | Initially placed in `/etc/nginx/sites-available/`. Server loads from `/etc/nginx/conf.d/`. | Moved config to `/etc/nginx/conf.d/temcosrs.temcobank.com.conf`. |
| `ContainerConfig` KeyError on redeploy | `docker-compose` v1.29.2 bug when recreating containers with newer Docker engine. | Use `docker-compose down` then `docker-compose up -d --build` instead of just `up --build`. |

---

## 15. Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│              TEMCO SRS WEBSITE — QUICK REFERENCE            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  DEPLOY (first time):                                       │
│    ssh temco-prod                                           │
│    mkdir -p /apps/temco-srs && cd /apps/temco-srs           │
│    git clone https://github.com/ishanthasiribaddana/        │
│      temco-srs.git .                                        │
│    docker-compose up -d --build                             │
│    certbot certonly --nginx -d temcosrs.temcobank.com        │
│    cp finance config → temcosrs + sed replace (see Step 6)  │
│    nginx -t && nginx -s reload                              │
│                                                             │
│  UPDATE:                                                    │
│    ssh temco-prod "cd /apps/temco-srs &&                    │
│      git pull origin main &&                                │
│      docker-compose down &&                                 │
│      docker-compose up -d --build"                          │
│                                                             │
│  STATUS:                                                    │
│    docker ps | grep temco-srs                               │
│    curl -I https://temcosrs.temcobank.com                   │
│                                                             │
│  LOGS:                                                      │
│    docker logs temco-srs --tail 50 -f                       │
│                                                             │
│  RESTART:                                                   │
│    cd /apps/temco-srs && docker-compose restart             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

*End of Runsheet — TEMCO SRS Website Deployment*
