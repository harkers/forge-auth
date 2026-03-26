# Forge Auth

Self-hosted SSO infrastructure for the Forge ecosystem using Authentik.

## Architecture

```
Cloudflare/Tailscale → Nginx → Authentik → Protected Apps
                              ↓
                         PostgreSQL
```

## Components

| Component | Version | Purpose |
|-----------|---------|---------|
| Authentik | 2026.2.1 | Identity provider (SSO) |
| PostgreSQL | 16-alpine | User/config storage |
| Nginx | - | TLS termination, reverse proxy |

## Deployment

### Phase 1: Core Deploy

```bash
# Copy environment file
cp .env.example .env

# Generate secrets
openssl rand -base64 32  # PG_PASS
openssl rand -base64 60  # SECRET_KEY

# Edit .env with generated values
vim .env

# Start services
docker compose up -d
```

### Phase 2: Nginx Proxy

Create `/etc/nginx/sites-available/authentik.conf`:

```nginx
server {
    listen 80;
    server_name auth.titan.lan;
    return 301 https://$host$request_uri;
}

server {
    listen 192.168.10.80:443 ssl http2;
    server_name auth.titan.lan;

    ssl_certificate /etc/nginx/ssl/openclaw.crt;
    ssl_certificate_key /etc/nginx/ssl/openclaw.key;

    ssl_protocols TLSv1.2 TLSv1.3;

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Real-IP $remote_addr;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    location / {
        proxy_pass https://127.0.0.1:9443;
    }
}
```

Enable and reload:

```bash
sudo ln -sf /etc/nginx/sites-available/authentik.conf /etc/nginx/sites-enabled/authentik
sudo nginx -t && sudo systemctl reload nginx
```

## DNS Configuration

Add to pfSense DNS Resolver (unbound):

| Host | Domain | IP |
|------|--------|-----|
| auth | titan.lan | 192.168.10.80 |

## First-Time Setup

1. Navigate to `https://auth.titan.lan/if/flow/initial-setup/`
2. Create admin account
3. Create groups: `admins`, `users`
4. Add TOTP to admin account
5. Create break-glass admin account

## Protected Apps

### OIDC/OAuth2 (Preferred)

For apps that support OAuth2/OIDC natively:

1. Create OAuth2/OpenID Provider
2. Create Application bound to provider
3. Configure app with Authentik credentials

### Proxy Provider (Legacy Apps)

For apps without native OAuth2 support:

1. Create Proxy Provider
2. Create Application
3. Use embedded outpost (simpler) or managed outpost
4. Configure Nginx forward auth

## Runbooks

### Backup

```bash
# Database backup
docker exec authentik-postgres pg_dump -U authentik authentik > backup_$(date +%Y%m%d).sql

# Media/templates backup
docker run --rm -v authentik-data:/data -v $(pwd):/backup alpine tar czf /backup/data.tar.gz /data
```

### Restore

```bash
# Restore database
cat backup.sql | docker exec -i authentik-postgres psql -U authentik authentik

# Restart services
docker compose restart
```

### Upgrade

```bash
# 1. Backup database
docker exec authentik-postgres pg_dump -U authentik authentik > pre_upgrade.sql

# 2. Pull new image
docker compose pull

# 3. Recreate containers
docker compose up -d

# Note: Authentik does NOT support downgrades
```

### Break-glass Admin Access

1. Use secondary admin account stored in password manager
2. Or restore from backup if locked out

## File Structure

```
forge-auth/
├── docker-compose.yml    # Main deployment
├── .env.example          # Environment template
├── SSO-DESIGN.qmd        # Full design document
├── DEPLOYMENT-DECISION.qmd # Phase 0 decisions
└── README.md             # This file
```

## Resources

- [Authentik Docs](https://goauthentik.io/docs/)
- [Authentik GitHub](https://github.com/goauthentik/authentik)

## License

MIT