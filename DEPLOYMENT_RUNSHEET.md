# TEMCO SRS Website — Deployment Runsheet
**Document Version:** 1.0  
**Created:** February 13, 2026  
**Author:** DevOps Team  
**Status:** Ready for Deployment

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

---

## 2. Technology Stack

| Layer | Technology |
|-------|-----------|
| Web Server | Nginx Alpine (Docker) |
| Container | Docker + Docker Compose |
| SSL Termination | Nginx (host-level) + Let's Encrypt |
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

- [ ] DNS A record created: `temcosrs.temcobank.com → 109.123.227.166`
- [ ] SSH access verified: `ssh temco-prod`
- [ ] Docker & Docker Compose installed on server
- [ ] Port 8093 is free (not used by other services)
- [ ] Nginx installed on host
- [ ] Certbot installed for SSL

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

### Step 5: Configure Host-Level Nginx

```bash
# Copy the provided config
cp /apps/temco-srs/temcosrs-nginx-host.conf /etc/nginx/sites-available/temcosrs.temcobank.com

# Enable the site
ln -s /etc/nginx/sites-available/temcosrs.temcobank.com /etc/nginx/sites-enabled/

# Test Nginx config
nginx -t
```

If `nginx -t` shows errors about SSL certificates (expected on first run), temporarily comment out the HTTPS server block and the HTTP→HTTPS redirect, then:

```bash
nginx -s reload
```

### Step 6: Obtain SSL Certificate

```bash
certbot --nginx -d temcosrs.temcobank.com
```

Certbot will:
1. Verify domain ownership
2. Generate SSL certificate
3. Auto-configure the Nginx server block
4. Set up auto-renewal

After Certbot completes:

```bash
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
ssh temco-prod "cd /apps/temco-srs && git pull origin main && docker-compose up -d --build"
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

## 11. Isolation & Security Notes

| Concern | Mitigation |
|---------|-----------|
| **Cross-contamination** | Separate Docker network (`srs-network`), separate container, no shared volumes |
| **Port conflict** | Port 8093 bound to `127.0.0.1` only (not exposed to internet) |
| **No backend** | Pure static site — zero attack surface from APIs/databases |
| **Independent lifecycle** | Can be stopped/rebuilt without affecting any other TEMCO service |
| **Separate repo** | `ishanthasiribaddana/temco-srs` — not part of main TemcoERP repo |
| **SSL** | Let's Encrypt with auto-renewal via Certbot |

---

## 12. Key Contacts & Access

| Resource | Detail |
|----------|--------|
| **Production Server** | `109.123.227.166` |
| **SSH** | `ssh temco-prod` or `ssh root@109.123.227.166 -i ~/.ssh/id_ed25519_temco` |
| **GitHub Repo** | https://github.com/ishanthasiribaddana/temco-srs |
| **Live URL** | https://temcosrs.temcobank.com |
| **Local Dev** | `python -m http.server 5500` in `f:\TemcoERP\docs\srs-website\` |
| **Local Preview** | http://localhost:5500 |

---

## 13. Quick Reference Card

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
│    cp temcosrs-nginx-host.conf                              │
│      /etc/nginx/sites-available/temcosrs.temcobank.com      │
│    ln -s /etc/nginx/sites-available/                        │
│      temcosrs.temcobank.com /etc/nginx/sites-enabled/       │
│    certbot --nginx -d temcosrs.temcobank.com                │
│    nginx -s reload                                          │
│                                                             │
│  UPDATE:                                                    │
│    ssh temco-prod "cd /apps/temco-srs &&                    │
│      git pull origin main &&                                │
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
