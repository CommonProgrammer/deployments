# Deployments

Centralized deployment configs for all apps on VPS `46.224.183.242`.

## Apps

| App | Domain | Port | Branch | Stack |
|-----|--------|------|--------|-------|
| hookrelay | hookrelay.duckdns.org | 3001 | master | Go + React + Postgres + Redis |
| budzet-rodzinny | budzet-rodzinny.duckdns.org | 3000 | main | Node.js + Express + Postgres |
| floriboxy | floriboxy.pl | 5000 | master | Python Flask + Gunicorn + Postgres |

## How it works

1. Push to app's branch (`main` or `master`)
2. App repo's trigger workflow sends `repository_dispatch` to this repo
3. This repo's deploy workflow SSHs into VPS, pulls code, rebuilds containers
4. Health check verifies the deploy

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
# Clone app repo
cd /home
git clone https://github.com/CommonProgrammer/<app>.git
cd <app>

# Create env file from example
nano .env.prod  # fill in real secrets (see deployments/<app>/.env.example)

# Nginx config
cat > /etc/nginx/sites-available/<app> << 'EOF'
# paste from deployments/<app>/nginx.conf
EOF
ln -sf /etc/nginx/sites-available/<app> /etc/nginx/sites-enabled/<app>
nginx -t && systemctl restart nginx

# First deploy
docker compose -f docker-compose.prod.yml up --build -d

# SSL
certbot --nginx -d <domain>
```

## Adding a new app

1. Create `<app-name>/` directory with `docker-compose.prod.yml`, `nginx.conf`, `.env.example`
2. Create `.github/workflows/deploy-<app-name>.yml`
3. Copy `docker-compose.prod.yml` to the app repo as well
4. Add trigger workflow to the app repo (`.github/workflows/deploy.yml`)
5. Add `DEPLOY_PAT` secret to app repo
6. Run VPS one-time setup

See `CLAUDE.md` for detailed templates and conventions.
