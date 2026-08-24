# Deployment Guide - Coolify

This application is ready to be deployed on a VPS using [Coolify](https://coolify.io/).

## Prerequisites

- VPS with Docker and Docker Compose installed
- Coolify installed on the VPS
- Domain name (optional)

## Deployment with Coolify

### 1. Connect your Git Repository

In Coolify:
1. Add a new application
2. Select "Docker" as the deployment method
3. Connect your GitHub/GitLab repository

### 2. Configure Environment Variables

In Coolify's environment variables section, set:

```
NODE_ENV=production
NEXT_PUBLIC_APP_URL=https://your-domain.com
```

Copy values from `.env.example` as needed.

### 3. Docker Configuration

Coolify will automatically detect and use:
- `Dockerfile` - for building the application
- `docker-compose.yml` - for local testing (optional)

### 4. Build and Deploy

1. Coolify will build the Docker image from `Dockerfile`
2. The application will start on port 3000
3. Configure a reverse proxy (nginx, Traefik, etc.) to point to port 3000

## Local Testing with Docker

Before deploying to production, test locally:

```bash
# Build the Docker image
docker build -t yoli-nails:latest .

# Run with docker-compose
docker-compose up -d

# Or run manually
docker run -p 3000:3000 yoli-nails:latest
```

Then open [http://localhost:3000](http://localhost:3000)

## Production Considerations

- Set `NODE_ENV=production` on the VPS
- Use a reverse proxy for SSL/TLS termination
- Configure health checks (Dockerfile includes HEALTHCHECK)
- Use managed backups for any persistent data
- Monitor logs: `docker logs <container-id>`

## Troubleshooting

### Build fails
- Check that `pnpm-lock.yaml` is in version control
- Ensure all dependencies are available in `package.json`

### Port already in use
- Change the port mapping in `docker-compose.yml`
- Or ensure no other service is using port 3000

### Application not responding
- Check Docker logs: `docker logs yoli-nails-web`
- Verify environment variables are set correctly
- Ensure the health check is passing
