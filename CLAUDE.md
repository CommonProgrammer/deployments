# Deployments Monorepo

Centralized deployment infrastructure for all apps on VPS `46.224.183.242`.
GitHub org: `CommonProgrammer`.

## Architecture

Push to app's branch → trigger workflow (repository_dispatch) → deploy workflow SSHs to VPS → `git pull` + `docker compose up --build -d`.

Each app has its own:
- Standalone Docker Compose stack (app + postgres + optional services)
- nginx reverse proxy config (managed on VPS host, not containerized)
- SSL via certbot (Let's Encrypt, auto-configured by `certbot --nginx`)

## Apps

| App | Domain | VPS Path | Port | Branch | Health Check |
|-----|--------|----------|------|--------|-------------|
| hookrelay | hookrelay.duckdns.org | /home/hookrelay | 3001→8080 | master | /health |
| budzet-rodzinny | budzet-rodzinny.duckdns.org | /home/budzet-rodzinny | 3000 | main | /health |
| floriboxy | floriboxy.pl | /home/floriboxy | 5000 | master | / |

## Repo Structure

```
deployments/
├── .github/workflows/
│   ├── deploy-hookrelay.yml
│   ├── deploy-budzet-rodzinny.yml
│   └── deploy-floriboxy.yml
├── <app-name>/
│   ├── docker-compose.prod.yml   # reference config (also in app repo)
│   ├── nginx.conf
│   └── .env.example
└── README.md
```

## Adding a New App

### 1. Create deployment configs

```bash
mkdir <app-name>
```

Create these files:

**`<app-name>/docker-compose.prod.yml`:**
- Use `build: .` (builds from app repo Dockerfile on VPS)
- Bind to `127.0.0.1:<port>:<container-port>` (nginx proxies from 80/443)
- Add `env_file: .env.prod` to ALL services (app AND postgres)
- For postgres: set `POSTGRES_USER` and `POSTGRES_DB` in `environment:`, password comes from `.env.prod` via `POSTGRES_PASSWORD`
- NEVER use `${VAR}` interpolation for secrets — Docker Compose reads those from shell/.env, NOT from .env.prod
- Add healthcheck, restart: unless-stopped, logging config
- Use named volumes for data persistence

**`<app-name>/nginx.conf`:**
```nginx
server {
    listen 80;
    server_name <domain>;
    location / {
        proxy_pass http://127.0.0.1:<port>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**`<app-name>/.env.example`:**
- Include ALL env vars the app needs
- Include `POSTGRES_PASSWORD=CHANGE_ME`
- Use `CHANGE_ME` as placeholder for secrets

### 2. Create deploy workflow

**`.github/workflows/deploy-<app-name>.yml`:**
```yaml
name: Deploy <App Name>
on:
  repository_dispatch:
    types: [deploy-<app-name>]
  workflow_dispatch:
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          password: ${{ secrets.VPS_PASSWORD }}
          script: |
            cd /home/<app-name>
            git pull origin <branch>
            docker compose -f docker-compose.prod.yml up --build -d
            sleep 10
            curl -sf http://localhost:<port>/<health-path> && echo 'Deploy OK' || echo 'HEALTH CHECK FAILED'
```

### 3. Add trigger workflow to app repo

**`<app-repo>/.github/workflows/deploy.yml`:**
```yaml
name: Trigger Deploy
on:
  push:
    branches: [<branch>]
jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.DEPLOY_PAT }}
          repository: CommonProgrammer/deployments
          event-type: deploy-<app-name>
```

### 4. Copy docker-compose.prod.yml to app repo

The app repo also needs `docker-compose.prod.yml` because the VPS clones the app repo (not deployments).

### 5. GitHub Secrets

- **deployments repo**: `VPS_HOST`, `VPS_USER`, `VPS_PASSWORD` (already set)
- **app repo**: `DEPLOY_PAT` — GitHub PAT with `repo` scope

### 6. VPS setup

```bash
# Clone
cd /home && git clone https://github.com/CommonProgrammer/<app>.git

# Create env
cp <app>/.env.example <app>/.env.prod   # or create manually
nano <app>/.env.prod                      # fill real values

# Nginx
cp deployments/<app>/nginx.conf /etc/nginx/sites-available/<app>
ln -sf /etc/nginx/sites-available/<app> /etc/nginx/sites-enabled/<app>
nginx -t && systemctl restart nginx

# First deploy
cd /home/<app> && docker compose -f docker-compose.prod.yml up --build -d

# SSL
certbot --nginx -d <domain>
```

## Common Patterns

### Docker Compose postgres password
ALWAYS use `env_file: .env.prod` on the postgres service. Put `POSTGRES_PASSWORD=<value>` in `.env.prod`. NEVER use `${VAR}` interpolation — it reads from shell, not from .env.prod.

### Port allocation
Each app gets a unique localhost port. Current: 3000 (budzet), 3001 (hookrelay), 5000 (floriboxy). Next available: 3002, 5001, 8081.

### Data migration
When migrating an existing app to this pattern:
1. Dump data from old postgres: `docker exec <old-container> pg_dump -U <user> <db> > /tmp/backup.sql`
2. Deploy new stack
3. Restore: `docker exec -i <new-postgres> psql -U <user> -d <db> < /tmp/backup.sql`
4. Copy uploaded files: `docker cp /old/path/uploads/. <app-container>:/app/static/uploads/`
