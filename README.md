# Element Server Suite - Docker Deployment

A complete, production-ready Matrix homeserver deployment using Docker Compose, featuring Synapse, Element Web, Coturn TURN server, and Synapse Admin.

[Русская версия](README.ru.md) | [English](#)

## Features

- **Synapse** - Matrix homeserver (latest version)
- **Element Web** - Modern Matrix web client
- **PostgreSQL 16** - High-performance database backend
- **Coturn** - TURN/STUN server for VoIP calls
- **Synapse Admin** - Web-based admin interface
- **Nginx-ready** - Pre-configured reverse proxy configs
- **SSL/TLS** - Full HTTPS support with Let's Encrypt
- **Docker Compose** - Easy deployment and management

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Nginx Proxy                         │
│  (Handles SSL termination and routing)                      │
└────────┬──────────────┬─────────────┬──────────────────────┘
         │              │             │
    Port 443        Port 443      Port 443
         │              │             │
   ┌─────▼─────┐  ┌────▼─────┐ ┌────▼─────┐
   │  Synapse  │  │ Element  │ │  Admin   │
   │   :8008   │  │  Web     │ │  :8091   │
   └─────┬─────┘  │  :8090   │ └──────────┘
         │        └──────────┘
    ┌────▼─────┐
    │PostgreSQL│       ┌──────────┐
    │  :5432   │       │ Coturn   │
    └──────────┘       │ :3479    │
                       └──────────┘
```

## Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- Domain name with DNS configured
- SSL/TLS certificates (Let's Encrypt recommended)
- Minimum 2GB RAM, 2 CPU cores
- 20GB disk space

## Quick Start

### 1. Clone and Configure

```bash
git clone <your-repo-url>
cd element

# Copy environment template
cp .env.example .env

# Generate secure passwords
openssl rand -base64 32  # For POSTGRES_PASSWORD
openssl rand -base64 64  # For TURN_SHARED_SECRET

# Edit .env with your values
nano .env
```

### 2. Configure Coturn

```bash
# Copy Coturn config template
cp config/coturn/turnserver.conf.example config/coturn/turnserver.conf

# Edit and set your TURN secret (same as in .env)
nano config/coturn/turnserver.conf
```

Update these values:
- `realm` - Your domain (e.g., turn.yourdomain.com)
- `server-name` - Same as realm
- `static-auth-secret` - Same as TURN_SHARED_SECRET from .env
- `external-ip` - Your server's public IP (optional)

### 3. Initial Synapse Setup

```bash
# Generate Synapse configuration
docker compose up synapse

# Wait for the message "Generating config file..."
# Then stop with Ctrl+C
docker compose down
```

### 4. Configure Synapse

Edit `data/synapse/homeserver.yaml`:

```yaml
# Server
server_name: "matrix.yourdomain.com"
public_baseurl: "https://matrix.yourdomain.com"

# Database
database:
  name: psycopg2
  args:
    user: synapse
    password: "YOUR_POSTGRES_PASSWORD"  # From .env
    database: synapse
    host: postgres
    port: 5432

# TURN Server
turn_uris:
  - "turn:matrix.yourdomain.com:3479?transport=tcp"
  - "turn:matrix.yourdomain.com:3479?transport=udp"
turn_shared_secret: "YOUR_TURN_SHARED_SECRET"  # From .env
turn_user_lifetime: 86400000
turn_allow_guests: true

# Registration
enable_registration: true
enable_registration_without_verification: true

# Media
max_upload_size: 50M
```

### 5. Configure Element Web

Edit `config/element-web/config.json`:

```json
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://matrix.yourdomain.com",
            "server_name": "matrix.yourdomain.com"
        }
    }
}
```

### 6. Setup Nginx

Copy nginx configurations to your nginx server:

```bash
# On your nginx server
sudo cp config/nginx/*.conf /etc/nginx/sites-available/
sudo ln -s /etc/nginx/sites-available/matrix.yourdomain.com.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/element.yourdomain.com.conf /etc/nginx/sites-enabled/

# Test and reload
sudo nginx -t
sudo systemctl reload nginx
```

**Important:** Update the following in nginx configs:
- Replace `deliev.net` with your domain
- Update SSL certificate paths
- Update `proxy_pass` IPs to your Docker host IP

### 7. Start All Services

```bash
docker compose up -d
```

### 8. Create Admin User

```bash
docker exec -it element-synapse register_new_matrix_user \
  http://localhost:8008 -c /data/homeserver.yaml -a
```

Follow the prompts to create your admin account.

## Service URLs

After deployment, services will be available at:

- **Element Web:** https://element.yourdomain.com
- **Matrix API:** https://matrix.yourdomain.com
- **Synapse Admin:** http://your-server-ip:8091 (local only)
- **Federation:** https://matrix.yourdomain.com:8448

## Ports

### Docker Containers
- `8008` - Synapse HTTP API
- `8448` - Synapse Federation
- `8090` - Element Web
- `8091` - Synapse Admin (local only)
- `3479` - Coturn TURN/STUN
- `5349` - Coturn TLS
- `49152-49252` - Coturn UDP relay range

### Firewall Requirements
Open these ports on your firewall:
- `80/tcp` - HTTP (redirects to HTTPS)
- `443/tcp` - HTTPS
- `8448/tcp` - Matrix Federation
- `3479/tcp,udp` - TURN/STUN
- `5349/tcp` - TURN/STUN TLS (optional)

## Management

### Docker Compose Commands

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs
docker compose logs -f

# View specific service logs
docker compose logs -f synapse

# Restart a service
docker compose restart synapse

# Update images
docker compose pull
docker compose up -d
```

### User Management

**Via Synapse Admin (Recommended):**
1. Open http://your-server-ip:8091
2. Login with your admin credentials
3. Manage users, rooms, and settings through the web UI

**Via Command Line:**
```bash
# Create user
docker exec -it element-synapse register_new_matrix_user \
  http://localhost:8008 -c /data/homeserver.yaml

# Make user admin
docker exec -it element-synapse \
  sqlite3 /data/homeserver.db \
  "UPDATE users SET admin = 1 WHERE name = '@username:yourdomain.com'"
```

## Backup

### Important Data

Always backup these directories:
- `data/postgres/` - Database
- `data/synapse/` - Homeserver config and media
- `config/` - All configurations

### Backup Script

```bash
#!/bin/bash
BACKUP_DIR="/backups/element"
DATE=$(date +%Y%m%d)

# Stop services
docker compose down

# Create backup
tar -czf "$BACKUP_DIR/element-backup-$DATE.tar.gz" \
  data/ config/ docker-compose.yml .env

# Restart services
docker compose up -d
```

### Restore

```bash
# Stop services
docker compose down

# Extract backup
tar -xzf element-backup-YYYYMMDD.tar.gz

# Start services
docker compose up -d
```

## Troubleshooting

### Synapse Won't Start

Check logs:
```bash
docker compose logs synapse
```

Common issues:
- Database connection failed → Check POSTGRES_PASSWORD in .env and homeserver.yaml
- Port already in use → Check if another service uses port 8008
- Config file errors → Validate YAML syntax in homeserver.yaml

### Voice/Video Calls Don't Work

1. Check Coturn is running:
```bash
docker compose ps coturn
```

2. Verify TURN secret matches in:
   - `.env` (TURN_SHARED_SECRET)
   - `config/coturn/turnserver.conf` (static-auth-secret)
   - `data/synapse/homeserver.yaml` (turn_shared_secret)

3. Test TURN server:
```bash
# Check if port is open
nc -zv your-server-ip 3479
```

### Federation Not Working

1. Test federation:
   - Visit https://federationtester.matrix.org/
   - Enter your homeserver domain

2. Check port 8448 is accessible:
```bash
curl https://matrix.yourdomain.com:8448/_matrix/federation/v1/version
```

3. Verify `.well-known` endpoints:
```bash
curl https://matrix.yourdomain.com/.well-known/matrix/server
curl https://matrix.yourdomain.com/.well-known/matrix/client
```

### Database Issues

Connect to PostgreSQL:
```bash
docker exec -it element-postgres psql -U synapse -d synapse
```

Check database size:
```bash
docker exec -it element-postgres \
  psql -U synapse -c "SELECT pg_size_pretty(pg_database_size('synapse'));"
```

## Security Best Practices

1. **Use Strong Passwords**
   - Generate with `openssl rand -base64 32`
   - Never commit `.env` file to git

2. **Enable Registration Restrictions**
   - Set `enable_registration: false` after creating users
   - Or use `registrations_require_3pid` for email verification

3. **Regular Updates**
   ```bash
   docker compose pull
   docker compose up -d
   ```

4. **Firewall Configuration**
   - Only expose necessary ports
   - Use fail2ban for brute force protection

5. **SSL/TLS**
   - Always use HTTPS
   - Keep certificates updated
   - Use strong ciphers (already configured in nginx)

6. **Backups**
   - Automate regular backups
   - Test restore procedures
   - Store backups securely off-site

## Monitoring

### Health Checks

```bash
# Synapse health
curl http://localhost:8008/health

# Element Web
curl http://localhost:8090

# PostgreSQL
docker exec element-postgres pg_isready -U synapse
```

### Resource Usage

```bash
# View container stats
docker stats

# Disk usage
du -sh data/
```

## Upgrading

```bash
# Pull latest images
docker compose pull

# Stop services
docker compose down

# Backup before upgrade (recommended)
tar -czf backup-pre-upgrade.tar.gz data/ config/

# Start with new images
docker compose up -d

# Check logs for errors
docker compose logs -f
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is provided as-is for educational and production use.

## Support

- [Matrix Specification](https://spec.matrix.org/)
- [Synapse Documentation](https://element-hq.github.io/synapse/)
- [Element Documentation](https://element.io/help)
- [Coturn Documentation](https://github.com/coturn/coturn/wiki)

## Acknowledgments

Based on the official [Element Server Suite Helm Charts](https://github.com/element-hq/ess-helm) adapted for Docker Compose deployment.
# ess-helm-docker
