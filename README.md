# Deployments

Centralized deployment configs for all apps on VPS `46.224.183.242`.

## Apps

| App | Domain | Port | Stack |
|-----|--------|------|-------|
| hookrelay | hookrelay.duckdns.org | 3001 | Go + React + Postgres + Redis |
| budzet-rodzinny | budzet-rodzinny.duckdns.org | 3000 | Node.js + Express + Postgres |
| floriboxy | floriboxy.pl | 5000 | Python Flask + Gunicorn + Postgres |

## How it works

1. Push to app's `main` branch
2. App repo's trigger workflow sends `repository_dispatch` to this repo
3. This repo's deploy workflow SSHs into VPS, pulls code, rebuilds containers

## Setup

### GitHub Secrets

**This repo (deployments):**
- `VPS_HOST` - VPS IP address
- `VPS_USER` - SSH username
- `VPS_PASSWORD` - SSH password

**Each app repo:**
- `DEPLOY_PAT` - GitHub Personal Access Token with `repo` scope

### VPS one-time setup per app

```bash
cd /home
git clone https://github.com/CommonProgrammer/<app>.git
cd <app>
cp /path/to/deployments/<app>/.env.example .env.prod
nano .env.prod  # fill in real secrets

# Copy nginx config
cp /path/to/deployments/<app>/nginx.conf /etc/nginx/sites-available/<app>
ln -sf /etc/nginx/sites-available/<app> /etc/nginx/sites-enabled/<app>
nginx -t && systemctl restart nginx

# First deploy
docker compose -f docker-compose.prod.yml up --build -d

# SSL
certbot --nginx -d <domain> --non-interactive --agree-tos --email admin@example.com
```

## Adding a new app

1. Create `<app-name>/` directory with `docker-compose.prod.yml`, `nginx.conf`, `.env.example`
2. Create `.github/workflows/deploy-<app-name>.yml`
3. Add trigger workflow to the app repo
4. Run VPS one-time setup
