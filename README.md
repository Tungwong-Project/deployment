# Tungwong Project

A microservices-based application with authentication, user management, and upload services.

## Services

- **Auth Management API** (Port 3000): Handles authentication, authorization, and permission roles
- **User Management API** (Port 3001): Manages user profiles and addresses
- **Upload Management API** (Port 3002): Handles file uploads
- **Video Upload API** (Coming soon): Handles video uploads and publishes to message queue
- **Video Worker** (Coming soon): Processes videos with FFmpeg for HLS streaming
- **PostgreSQL Database** (Port 5433): Shared database for all services
- **NATS JetStream** (Port 4222): Message queue for video processing
- **Nginx Reverse Proxy** (Port 80/443): API gateway and load balancer

## Architecture

```
                         ┌──────────────┐
                         │    Nginx     │
                         │   Port 80    │
                         └──────┬───────┘
                                │
               ┌────────────────┼────────────────┬──────────────┐
               │                │                │              │
         ┌─────▼─────┐    ┌─────▼─────┐    ┌────▼──────┐  ┌───▼──────┐
         │  Auth API │    │ User API  │    │Upload API │  │Video API │
         │ Port 3000 │    │Port 3001  │    │ Port 3002 │  │Port 3003 │
         └─────┬─────┘    └─────┬─────┘    └─────┬─────┘  └────┬─────┘
               │                │                 │             │
               └────────────────┼─────────────────┼─────────────┘
                                │                 │
                         ┌──────▼────────┐        │ Publish
                         │   PostgreSQL  │        │
                         │   Port 5433   │        │
                         └───────────────┘        │
                                                  │
                                           ┌──────▼──────────┐
                                           │ NATS JetStream  │
                                           │   Port 4222     │
                                           └──────┬──────────┘
                                                  │ Subscribe
                                           ┌──────▼──────────┐
                                           │  Video Worker   │
                                           │ FFmpeg/HLS      │
                                           └─────────────────┘
```

## Prerequisites

- Docker and Docker Compose
- Git

## Getting Started

### 1. Clone the repository

```bash
git clone <repository-url>
cd tungwong-project
```

### 2. Set up environment variables

Each service needs its own `.env` file. Copy the example files:

```bash
# Auth API
cp tungwong-auth-management-api/.env.example tungwong-auth-management-api/.env

# User API
cp tungwong-user-management-api/.env.example tungwong-user-management-api/.env

# Upload API
cp tungwong_upload_management_api/.env.example tungwong_upload_management_api/.env

# Root environment (optional)
cp .env.example .env
```

Edit each `.env` file with your configuration.

### 3. Build and run all services

```bash
# Build all services
docker-compose build

# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# ChNATS Monitoring**: http://localhost:8222
- **eck service status
docker-compose ps
```

### 4. Access the services

- **Nginx Gateway**: http://localhost
- **Auth API**: http://localhost/api/auth or http://localhost:3000
- **User API**: http://localhost/api/users or http://localhost:3001
- **Upload API**: http://localhost/api/uploads or http://localhost:3002
- **NATS Monitoring**: http://localhost:8222

**Swagger Documentation (Subdomain Access):**
- **Auth Swagger**: http://auth-swagger.localhost/swagger/
- **User Swagger**: http://user-swagger.localhost/swagger/
- **Upload Swagger**: http://upload-swagger.localhost/swagger/

> **Note**: `.localhost` subdomains work automatically on modern browsers. Alternatively, use direct port access: http://localhost:3000/swagger/, http://localhost:3001/swagger/, http://localhost:3002/swagger/

> Note: Swagger UI is accessed via direct ports due to JavaScript path limitations when proxying through nginx subpaths.

## API Routes

### Through Nginx (Recommended)

**Auth Management API:**
- `http://localhost/api/auth/login` - User login
- `http://localhost/api/auth/register` - User registration
- `http://localhost/api/auth/refresh` - Refresh token
- `http://localhost/api/auth/verify` - Verify token (internal service calls)
- `http://localhost/api/auth/permission-roles` - Permission roles management
- `http://localhost/api/users/:id` - Get/Update user by ID (admin)

**User Management API:**
- `http://localhost/api/users/profile` - User profiles (CRUD)
- `http://localhost/api/users/profile/upload-image` - Upload profile image
- `http://localhost/api/users/addresses` - User addresses (CRUD)

**Upload Management API:**
- `http://localhost/api/uploads` - File uploads (CRUD)

### Direct Access (Development)

- `http://localhost:3000/*` → Auth Service
- `http://localhost:3001/*` → User Service
- `http://localhost:3002/*` → Upload Service

## Docker Commands

```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# Rebuild a specific service
docker-compose build auth-api

# View logs for a specific service
docker-compose logs -f auth-api

# Restart a specific service
docker-compose restart user-api

# Remove all containers and volumes
docker-compose down -v

# Scale a service (if needed)
docker-compose up -d --scale user-api=3
```

## Development

### Rebuilding after code changes

```bash
# Rebuild and restart all services
docker-compose up -d --build

# Rebuild specific service
docker-compose up -d --build auth-api
```

### Database migrations

Migrations run automatically when the services start. To run manually:

```bash
# Access a service container
docker-compose exec auth-api sh

# Run migrations from inside the container
./app migrate
```

### Accessing the database

```bash
# Connect to PostgreSQL
docker-compose exec postgres psql -U postgres -d tungwong_db
```

## Nginx Configuration

The nginx configuration provides:
- Reverse proxy for all APIs
- Rate limiting
- Request buffering for uploads
- Health check endpoint at `/health`
- Gzip compression
- SSL/TLS support (commented out by default)

### Enabling HTTPS

1. Generate or obtain SSL certificates
2. Create `nginx/ssl` directory and place certificates there
3. Uncomment the HTTPS server block in `nginx.conf`
4. Restart nginx: `docker-compose restart nginx`

## Monitoring

### Health Check

```bash
# Check nginx health
curl http://localhost/health

# Check specific service
curl http://localhost:3000/health
```

### Container Status

```bash
# View container status
docker-compose ps

# View resource usage
docker stats
```

## Troubleshooting

### Service won't start

```bash
# Check logs
docker-compose logs [service-name]

# Check if port is already in use
netstat -ano | findstr :3000
```

### Database connection issues

```bash
# Verify postgres is running
docker-compose ps postgres

# Check database logs
docker-compose logs postgres

# Test connection
docker-compose exec postgres pg_isready -U postgres
```

### Network issues

```bash
# Recreate network
docker-compose down
docker network prune
docker-compose up -d
```

## Production Deployment

For production:

1. Use environment-specific `.env` files
2. Enable HTTPS in nginx configuration
3. Set up proper logging and monitoring
4. Use Docker secrets for sensitive data
5. Configure backup strategy for PostgreSQL
6. Set up health checks and restart policies
7. Use reverse proxy with proper DNS

## License

[Your License]

## Contributors

[Your Team]
