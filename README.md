# Evolution API - Docker Deployment

This repository contains a Docker Compose setup for deploying Evolution API, a WhatsApp API solution that supports multiple integration channels.

## 📋 Requirements

- **Docker** (version 20.10 or higher)
- **Docker Compose** (version 2.0 or higher)
- **Minimum 2GB RAM** (4GB recommended)
- **Ports available**: 8089, 3006, 5439, 6389

## 🚀 Quick Start

### 1. Clone and Navigate

```bash
git clone <repository-url>
cd evolution-api
```

### 2. Configure Environment Variables

```bash
# Copy the sample environment file
cp env.sample .env

# Edit .env with your configuration
nano .env  # or use your preferred editor
```

> Read all the configuration before running `docker-compose up -d`
> It is crutial to change the `AUTHENTICATION_API_KEY` since it is used as APIKEY and
> the password for the UI

### 3. Start Services

```bash
docker-compose up -d
```

### 4. Verify Services

```bash
docker-compose ps
```

### 5. Access Services

**Local Access:**

- **API**: http://127.0.0.1:8089 or http://localhost:8089
- **Frontend Manager**: http://localhost:3006
- **PostgreSQL**: localhost:5439
- **Redis**: localhost:6389

**Network Access (from other devices on your LAN):**

- **API**: http://YOUR_RASPBERRY_IP:8089 (e.g., http://192.168.1.58:8089)
- **Frontend Manager**: http://YOUR_RASPBERRY_IP:3006 (e.g., http://192.168.1.58:3006)
- **PostgreSQL**: YOUR_RASPBERRY_IP:5439
- **Redis**: YOUR_RASPBERRY_IP:6389

> **Note**: Ports are configured to bind to `0.0.0.0` to allow network access. For security, consider restricting API access in production environments.

## ⚙️ Configuration

### Important Environment Variables

The `.env` file must contain the following critical variables:

#### PostgreSQL Configuration

```env
POSTGRES_DATABASE=evolution
POSTGRES_USERNAME=evolution_user_test
POSTGRES_PASSWORD=evolution_password_safe
```

#### Evolution API Database Configuration

```env
DATABASE_PROVIDER=postgresql
DATABASE_CONNECTION_URI=postgresql://evolution_user_test:evolution_password_safe@evolution-postgres:5432/evolution
DATABASE_CONNECTION_CLIENT_NAME=evolution
```

**Important**: The `DATABASE_CONNECTION_URI` must use port `5432` (container port), not `5439` (host port), for inter-container communication.

#### Redis Configuration

```env
CACHE_REDIS_ENABLED=true
CACHE_REDIS_URI=redis://evolution-redis:6389
CACHE_REDIS_PREFIX_KEY=evolution-cache
CACHE_REDIS_TTL=604800
CACHE_REDIS_SAVE_INSTANCES=true
```

#### Server Configuration

```env
SERVER_NAME=evolution
SERVER_TYPE=http
SERVER_PORT=8080
SERVER_URL=http://localhost:8089
SERVER_DISABLE_DOCS=false
SERVER_DISABLE_MANAGER=false
```

#### Authentication

```env
AUTHENTICATION_API_KEY=BQYHJGJHJ
AUTHENTICATION_EXPOSE_IN_FETCH_INSTANCES=false
```

**Security Note**: Change `AUTHENTICATION_API_KEY` to a strong, unique value in production.

### Port Configuration

The following ports are configured to avoid conflicts:

| Service    | Host Port | Container Port | Purpose                  |
| ---------- | --------- | -------------- | ------------------------ |
| API        | 8089      | 8080           | Evolution API endpoint   |
| Frontend   | 3006      | 80             | Web management interface |
| PostgreSQL | 5439      | 5432           | Database access          |
| Redis      | 6389      | 6389           | Cache service            |

**Note**: When connecting from containers, always use container ports (e.g., `5432` for PostgreSQL, not `5439`).

## 🔧 Issues Fixed

### 1. Missing Environment Variables

**Problem**: PostgreSQL environment variables were not set, causing warnings and connection failures.

**Solution**: Created `.env` file with all required PostgreSQL and Evolution API configuration variables.

### 2. External Network Not Found

**Problem**: `dokploy-network` was declared as external but didn't exist.

**Solution**: Changed from `external: true` to `driver: bridge` so Docker Compose creates it automatically.

### 3. Redis Command Syntax Error

**Problem**: Comment inside YAML multiline command block was being executed, causing `exec: #: not found` error.

**Solution**: Moved comment outside the command block in `docker-compose.yml`.

### 4. Frontend Nginx Configuration Error

**Problem**: Invalid nginx configuration in container with `must-revalidate` as standalone value.

**Solution**: Created custom nginx config file (`nginx-config/nginx.conf`) and mounted it to override the broken configuration.

### 5. Database Provider Invalid

**Problem**: API was using `DATABASE_PROVIDER=postgres` which is invalid.

**Solution**: Changed to `DATABASE_PROVIDER=postgresql` as per Evolution API requirements.

### 6. Database Connection Port Mismatch

**Problem**: API was trying to connect to PostgreSQL on port `5439` (host port) instead of `5432` (container port).

**Solution**: Updated `DATABASE_CONNECTION_URI` to use port `5432` for inter-container communication.

### 7. PostgreSQL Authentication Failure

**Problem**: PostgreSQL volume was initialized with different credentials, causing authentication failures.

**Solution**: Removed and recreated the PostgreSQL volume to initialize with correct credentials from `.env`.

## 📝 Usage

### Starting Services

```bash
docker-compose up -d
```

### Stopping Services

```bash
docker-compose down
```

### Viewing Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f api
docker-compose logs -f frontend
docker-compose logs -f evolution-postgres
docker-compose logs -f redis
```

### Restarting a Service

```bash
docker-compose restart api
```

### Checking Service Status

```bash
docker-compose ps
```

### Accessing Database

```bash
# From host machine
psql -h localhost -p 5439 -U evolution_user_test -d evolution

# From within container
docker-compose exec evolution-postgres psql -U evolution_user_test -d evolution
```

## 🔐 Security Considerations

1. **Change Default Passwords**: Update all default passwords in `.env` before production use.
2. **API Key**: Generate a strong, unique `AUTHENTICATION_API_KEY`.
3. **Database Credentials**: Use strong passwords for PostgreSQL.
4. **Network Security**:
   - Ports are bound to `0.0.0.0` to allow network access (useful for Raspberry Pi deployments)
   - For production, consider using a reverse proxy (nginx/traefik) with SSL/TLS
   - If you need to restrict to localhost only, change `0.0.0.0:8089` to `127.0.0.1:8089` in `docker-compose.yml`
   - Use firewall rules to restrict access to trusted IPs only
5. **Environment File**: Never commit `.env` to version control (already in `.gitignore`).
6. **Raspberry Pi Deployment**: Ensure your Raspberry Pi firewall allows incoming connections on ports 8089 and 3006.

## 🐛 Troubleshooting

For detailed troubleshooting information, including firewall configuration for Raspberry Pi deployments, see [docs/Troubleshooting.md](docs/Troubleshooting.md).

### Services Won't Start

```bash
# Check logs for errors
docker-compose logs

# Verify environment variables
docker-compose config

# Check if ports are in use
lsof -i :8089 -i :3006 -i :5439 -i :6389
```

### Database Connection Issues

```bash
# Verify PostgreSQL is healthy
docker-compose ps evolution-postgres

# Check database credentials
docker-compose exec evolution-postgres psql -U evolution_user_test -d evolution -c "SELECT version();"

# If authentication fails, recreate the volume
docker-compose down
docker volume rm evolution-api_postgres_data
docker-compose up -d
```

### API Not Responding

```bash
# Check API logs
docker-compose logs api

# Verify API is running
curl http://127.0.0.1:8089/instance/fetchInstances -H "apikey: YOUR_API_KEY"

# Check if all dependencies are running
docker-compose ps
```

### Frontend Not Loading

```bash
# Check frontend logs
docker-compose logs frontend

# Verify nginx config is mounted
docker-compose exec frontend ls -la /etc/nginx/conf.d/
```

## 📚 Documentation Links

- **Official Documentation**: https://doc.evolution-api.com/
- **GitHub Repository**: https://github.com/EvolutionAPI/evolution-api
- **API Reference**: https://doc.evolution-api.com/docs/api-reference

## 📦 Project Structure

```
evolution-api/
├── docker-compose.yml          # Docker Compose configuration
├── .env                        # Environment variables (not in git)
├── env.sample                  # Environment variables template
├── nginx-config/
│   └── nginx.conf             # Custom nginx configuration for frontend
├── docs/
│   └── Troubleshooting.md     # Detailed troubleshooting guide
├── README.md                   # This file
└── .gitignore                 # Git ignore rules
```

## ✅ TODO

- [ ] **Script to setup and configure using Docker-compose**
  - Create automated setup script that:
    - Checks Docker and Docker Compose installation
    - Validates port availability
    - Generates secure `.env` file with random passwords
    - Sets up initial configuration
    - Runs health checks
    - Provides setup summary
- [ ] **Script to rotate the API KEY using Docker-compose**
  - Create maintenance script that:
    - Will generate a new API KEY with the selected length
    - Update `.env` file
    - Restart docker-compose to update the instances

## 🔄 Maintenance

### Updating Images

```bash
docker-compose pull
docker-compose up -d
```

### Backup Database

```bash
docker-compose exec evolution-postgres pg_dump -U evolution_user_test evolution > backup.sql
```

### Restore Database

```bash
docker-compose exec -T evolution-postgres psql -U evolution_user_test evolution < backup.sql
```

### Clean Up

```bash
# Stop and remove containers, networks
docker-compose down

# Remove volumes (WARNING: deletes all data)
docker-compose down -v
```

## 📞 Support

For issues related to:

- **Evolution API**: Check [GitHub Issues](https://github.com/EvolutionAPI/evolution-api/issues)
- **This Setup**: Check the troubleshooting section above or create an issue in this repository

## 📄 License

This setup follows the license of Evolution API. Please refer to the [Evolution API repository](https://github.com/EvolutionAPI/evolution-api) for license information.
