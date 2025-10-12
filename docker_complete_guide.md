# Complete Docker Guide: Beginner to Advanced

## Table of Contents

1. [Introduction to Docker](#introduction-to-docker)
2. [Beginner Level](#beginner-level)
   - [Installation and Setup](#installation-and-setup)
   - [Basic Docker Concepts](#basic-docker-concepts)
   - [Your First Container](#your-first-container)
   - [Working with Images](#working-with-images)
   - [Basic Docker Commands](#basic-docker-commands)
3. [Intermediate Level](#intermediate-level)
   - [Creating Dockerfiles](#creating-dockerfiles)
   - [Docker Compose](#docker-compose)
   - [Volumes and Data Management](#volumes-and-data-management)
   - [Networking](#networking)
   - [Environment Variables](#environment-variables)
4. [Advanced Level](#advanced-level)
   - [Multi-stage Builds](#multi-stage-builds)
   - [Docker Security](#docker-security)
   - [Performance Optimization](#performance-optimization)
   - [Production Deployment](#production-deployment)
   - [Docker Swarm](#docker-swarm)
5. [Real-world Examples](#real-world-examples)
6. [Best Practices](#best-practices)
7. [Troubleshooting](#troubleshooting)

---

## Introduction to Docker

Docker is a containerization platform that packages applications and their dependencies into lightweight, portable containers. It solves the "it works on my machine" problem by ensuring consistent environments across development, testing, and production.

### Key Benefits:
- **Consistency**: Same environment everywhere
- **Portability**: Run anywhere Docker is installed
- **Efficiency**: Lightweight compared to VMs
- **Scalability**: Easy to scale applications
- **Isolation**: Applications run in isolated environments

---

## Beginner Level

### Installation and Setup

#### Windows Installation
```powershell
# Using Chocolatey
choco install docker-desktop

# Or download from Docker Hub
# https://hub.docker.com/editions/community/docker-ce-desktop-windows
```

#### Linux Installation (Ubuntu)
```bash
# Update package index
sudo apt-get update

# Install prerequisites
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Set up stable repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

# Add user to docker group
sudo usermod -aG docker $USER
```

#### macOS Installation
```bash
# Using Homebrew
brew install --cask docker

# Or download from Docker Hub
# https://hub.docker.com/editions/community/docker-ce-desktop-mac
```

#### Verify Installation
```bash
docker --version
docker run hello-world
```

### Basic Docker Concepts

#### Images vs Containers
- **Image**: Read-only template used to create containers
- **Container**: Running instance of an image

#### Registry
- **Docker Hub**: Public registry for Docker images
- **Private Registry**: Your own registry for proprietary images

#### Dockerfile
- Text file with instructions to build Docker images

### Your First Container

#### Running a Simple Container
```bash
# Run a simple web server
docker run -d -p 8080:80 --name my-nginx nginx

# Check running containers
docker ps

# Stop the container
docker stop my-nginx

# Remove the container
docker rm my-nginx
```

#### Interactive Container
```bash
# Run Ubuntu container interactively
docker run -it ubuntu:20.04 /bin/bash

# Inside the container
apt update
apt install curl
curl --version
exit
```

### Working with Images

#### Pulling Images
```bash
# Pull specific version
docker pull ubuntu:20.04

# Pull latest version
docker pull nginx

# List local images
docker images

# Remove an image
docker rmi ubuntu:20.04
```

#### Searching Images
```bash
# Search Docker Hub
docker search nodejs

# Get image information
docker inspect nginx
```

### Basic Docker Commands

#### Container Management
```bash
# Run container
docker run [OPTIONS] IMAGE [COMMAND]

# Run in background
docker run -d nginx

# Run with port mapping
docker run -p 8080:80 nginx

# Run with name
docker run --name my-app nginx

# Run with environment variables
docker run -e ENV_VAR=value nginx

# Execute command in running container
docker exec -it container_name /bin/bash

# View container logs
docker logs container_name

# Follow logs in real-time
docker logs -f container_name

# Copy files to/from container
docker cp file.txt container_name:/path/
docker cp container_name:/path/file.txt ./
```

#### Container Lifecycle
```bash
# Start stopped container
docker start container_name

# Stop running container
docker stop container_name

# Restart container
docker restart container_name

# Pause/unpause container
docker pause container_name
docker unpause container_name

# Remove container
docker rm container_name

# Remove all stopped containers
docker container prune
```

---

## Intermediate Level

### Creating Dockerfiles

#### Basic Dockerfile Structure
```dockerfile
# Use official base image
FROM node:16-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source code
COPY . .

# Expose port
EXPOSE 3000

# Define entry point
CMD ["npm", "start"]
```

#### Dockerfile Instructions
```dockerfile
# FROM - Base image
FROM ubuntu:20.04

# LABEL - Metadata
LABEL maintainer="your-email@example.com"
LABEL version="1.0"

# ENV - Environment variables
ENV NODE_ENV=production
ENV PORT=3000

# ARG - Build-time variables
ARG BUILD_DATE
ARG VERSION

# RUN - Execute commands during build
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    && rm -rf /var/lib/apt/lists/*

# COPY - Copy files from host to image
COPY src/ /app/src/

# ADD - Copy and extract files (use COPY instead)
ADD archive.tar.gz /app/

# WORKDIR - Set working directory
WORKDIR /app

# USER - Set user for subsequent instructions
USER node

# VOLUME - Create mount points
VOLUME ["/data"]

# EXPOSE - Document ports
EXPOSE 8080 8443

# CMD - Default command (can be overridden)
CMD ["node", "server.js"]

# ENTRYPOINT - Always executed command
ENTRYPOINT ["./entrypoint.sh"]
```

#### Building Images
```bash
# Build image from Dockerfile
docker build -t my-app:1.0 .

# Build with build args
docker build --build-arg VERSION=1.0 -t my-app .

# Build from specific Dockerfile
docker build -f Dockerfile.prod -t my-app:prod .

# Build without cache
docker build --no-cache -t my-app .
```

#### Real Example: Node.js Application
```dockerfile
FROM node:16-alpine

# Create app directory
WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Bundle app source
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Change ownership of the app directory
RUN chown -R nodejs:nodejs /usr/src/app
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

### Docker Compose

#### Basic docker-compose.yml
```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    volumes:
      - .:/app
      - /app/node_modules
    depends_on:
      - db

  db:
    image: postgres:13
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  postgres_data:
```

#### Advanced Compose Configuration
```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - web
    networks:
      - frontend
      - backend

  web:
    build:
      context: .
      dockerfile: Dockerfile.prod
      args:
        BUILD_DATE: ${BUILD_DATE}
        VERSION: ${VERSION}
    image: myapp:${VERSION:-latest}
    container_name: web-app
    restart: unless-stopped
    environment:
      - NODE_ENV=${NODE_ENV:-production}
      - DB_HOST=db
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./logs:/app/logs
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - backend
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M

  db:
    image: postgres:13-alpine
    container_name: postgres-db
    restart: always
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:6-alpine
    container_name: redis-cache
    restart: always
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - backend

  monitoring:
    image: prom/prometheus
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - monitoring

volumes:
  postgres_data:
  redis_data:
  prometheus_data:

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
  monitoring:
    driver: bridge
```

#### Docker Compose Commands
```bash
# Start services
docker-compose up

# Start in background
docker-compose up -d

# Build and start
docker-compose up --build

# Start specific service
docker-compose up web

# Scale services
docker-compose up --scale web=3

# View logs
docker-compose logs
docker-compose logs -f web

# Execute commands
docker-compose exec web bash

# Stop services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# List running services
docker-compose ps

# Validate compose file
docker-compose config
```

### Volumes and Data Management

#### Volume Types
```bash
# Named volumes
docker volume create my-volume
docker run -v my-volume:/data nginx

# Bind mounts
docker run -v /host/path:/container/path nginx

# Tmpfs mounts (Linux only)
docker run --tmpfs /tmp nginx
```

#### Volume Management
```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect my-volume

# Remove volume
docker volume rm my-volume

# Remove unused volumes
docker volume prune

# Backup volume
docker run --rm -v my-volume:/data -v $(pwd):/backup ubuntu tar czf /backup/backup.tar.gz -C /data .

# Restore volume
docker run --rm -v my-volume:/data -v $(pwd):/backup ubuntu tar xzf /backup/backup.tar.gz -C /data
```

#### Data Container Pattern
```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    image: myapp
    volumes_from:
      - data

  data:
    image: busybox
    volumes:
      - /data
    command: /bin/true
```

### Networking

#### Network Types
```bash
# Bridge network (default)
docker network create my-bridge

# Host network
docker run --network host nginx

# None network
docker run --network none nginx

# Custom bridge network
docker network create --driver bridge my-network
docker run --network my-network nginx
```

#### Network Management
```bash
# List networks
docker network ls

# Inspect network
docker network inspect bridge

# Create custom network
docker network create --driver bridge \
  --subnet=172.20.0.0/16 \
  --ip-range=172.20.240.0/20 \
  my-custom-network

# Connect container to network
docker network connect my-network container_name

# Disconnect container from network
docker network disconnect my-network container_name

# Remove network
docker network rm my-network
```

#### Service Discovery
```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx
    networks:
      - frontend

  api:
    image: myapi
    networks:
      - frontend
      - backend

  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
  backend:
```

### Environment Variables

#### Setting Environment Variables
```bash
# Single variable
docker run -e NODE_ENV=production myapp

# Multiple variables
docker run -e NODE_ENV=production -e PORT=3000 myapp

# From file
docker run --env-file .env myapp

# In Dockerfile
ENV NODE_ENV=production
```

#### Environment File (.env)
```env
# .env file
NODE_ENV=production
PORT=3000
DB_HOST=localhost
DB_USER=admin
DB_PASSWORD=secret
```

#### Using in Docker Compose
```yaml
version: '3.8'

services:
  web:
    image: myapp
    environment:
      - NODE_ENV=${NODE_ENV}
      - PORT=${PORT}
    env_file:
      - .env
```

---

## Advanced Level

### Multi-stage Builds

#### Basic Multi-stage Build
```dockerfile
# Build stage
FROM node:16-alpine as builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Production stage
FROM node:16-alpine as production

WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .

EXPOSE 3000
CMD ["node", "server.js"]
```

#### Advanced Multi-stage Example
```dockerfile
# Base stage with common dependencies
FROM node:16-alpine as base
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Development stage
FROM base as development
ENV NODE_ENV=development
RUN npm ci
COPY . .
CMD ["npm", "run", "dev"]

# Build stage
FROM base as build
COPY . .
RUN npm run build

# Test stage
FROM build as test
ENV NODE_ENV=test
RUN npm run test

# Production stage
FROM node:16-alpine as production
WORKDIR /app
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
COPY --from=base --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=build --chown=nodejs:nodejs /app/dist ./dist
COPY --chown=nodejs:nodejs package*.json ./

USER nodejs
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

#### Building Specific Stages
```bash
# Build development stage
docker build --target development -t myapp:dev .

# Build production stage
docker build --target production -t myapp:prod .

# Build and run tests
docker build --target test -t myapp:test .
```

### Docker Security

#### Security Best Practices
```dockerfile
# Use official base images
FROM node:16-alpine

# Don't run as root
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Set working directory
WORKDIR /app

# Copy and set ownership
COPY --chown=nodejs:nodejs . .

# Install dependencies as root, then switch user
RUN npm ci --only=production

# Switch to non-root user
USER nodejs

# Use COPY instead of ADD
COPY package*.json ./

# Remove unnecessary packages
RUN apk del .build-deps

# Set read-only filesystem
# docker run --read-only myapp
```

#### Security Scanning
```bash
# Scan image for vulnerabilities
docker scout cves myapp:latest

# Use Trivy scanner
trivy image myapp:latest

# Use Clair scanner
docker run -d --name clair-db arminc/clair-db:latest
docker run -p 6060:6060 --link clair-db:postgres -d --name clair arminc/clair-local-scan:latest
```

#### Runtime Security
```bash
# Run with limited capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp

# Set resource limits
docker run -m 512m --cpus="1.0" myapp

# Read-only root filesystem
docker run --read-only myapp

# No new privileges
docker run --security-opt=no-new-privileges myapp

# User namespace
docker run --userns-remap=default myapp
```

#### Secrets Management
```bash
# Docker secrets (Swarm mode)
echo "mysecret" | docker secret create my_secret -

# Using external secrets manager
docker run -e SECRET_KEY="$(vault kv get -field=key secret/myapp)" myapp

# Mount secrets from file
docker run -v /etc/ssl/certs:/etc/ssl/certs:ro myapp
```

### Performance Optimization

#### Image Size Optimization
```dockerfile
# Use Alpine Linux
FROM node:16-alpine

# Multi-stage builds
FROM node:16 as builder
# ... build steps
FROM node:16-alpine as production
COPY --from=builder /app/dist ./dist

# Minimize layers
RUN apt-get update && apt-get install -y \
    curl \
    vim \
    && rm -rf /var/lib/apt/lists/*

# Use .dockerignore
# Create .dockerignore file
node_modules
*.log
.git
.env
```

#### Build Cache Optimization
```dockerfile
# Copy dependency files first
COPY package*.json ./
RUN npm ci

# Copy source code last
COPY . .
```

#### Resource Optimization
```bash
# Set memory limits
docker run -m 512m myapp

# Set CPU limits
docker run --cpus="1.5" myapp

# Set swap limits
docker run --memory=1g --memory-swap=2g myapp
```

#### Monitoring and Profiling
```bash
# Monitor container resources
docker stats

# Get container resource usage
docker stats container_name

# Memory usage details
docker exec container_name cat /sys/fs/cgroup/memory/memory.usage_in_bytes
```

### Production Deployment

#### Health Checks
```dockerfile
# Health check in Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Or using wget
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
```

```bash
# Health check at runtime
docker run --health-cmd="curl -f http://localhost:3000/health" \
  --health-interval=30s \
  --health-timeout=3s \
  --health-retries=3 \
  myapp
```

#### Logging
```yaml
# docker-compose.yml with logging
version: '3.8'

services:
  web:
    image: myapp
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  app-with-syslog:
    image: myapp
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://192.168.1.42:123"
```

#### Restart Policies
```bash
# Always restart
docker run --restart=always myapp

# Restart on failure
docker run --restart=on-failure:3 myapp

# Restart unless stopped
docker run --restart=unless-stopped myapp
```

### Docker Swarm

#### Initialize Swarm
```bash
# Initialize swarm mode
docker swarm init

# Join as worker
docker swarm join --token SWMTKN-1-token manager-ip:2377

# Join as manager
docker swarm join-token manager
```

#### Deploy Stack
```yaml
# docker-stack.yml
version: '3.8'

services:
  web:
    image: myapp:latest
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == worker
    ports:
      - "80:3000"
    networks:
      - webnet

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints:
          - node.role == manager
    networks:
      - webnet

networks:
  webnet:
```

```bash
# Deploy stack
docker stack deploy -c docker-stack.yml mystack

# List stacks
docker stack ls

# List services in stack
docker stack services mystack

# Remove stack
docker stack rm mystack
```

#### Service Management
```bash
# Create service
docker service create --replicas 3 --name web nginx

# Scale service
docker service scale web=5

# Update service
docker service update --image nginx:alpine web

# List services
docker service ls

# Inspect service
docker service inspect web

# View service logs
docker service logs web
```

---

## Real-world Examples

### Full-Stack Application
```yaml
# docker-compose.production.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
      - static_volume:/var/www/html/static
    depends_on:
      - web
    networks:
      - frontend

  web:
    build:
      context: ./backend
      dockerfile: Dockerfile.prod
    expose:
      - "8000"
    volumes:
      - static_volume:/app/staticfiles
    env_file:
      - .env.prod
    depends_on:
      - db
      - redis
    networks:
      - frontend
      - backend

  db:
    image: postgres:13-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    env_file:
      - .env.prod.db
    networks:
      - backend

  redis:
    image: redis:6-alpine
    networks:
      - backend

  celery:
    build:
      context: ./backend
      dockerfile: Dockerfile.prod
    command: celery -A myproject worker -l info
    volumes:
      - static_volume:/app/staticfiles
    env_file:
      - .env.prod
    depends_on:
      - db
      - redis
    networks:
      - backend

  celery-beat:
    build:
      context: ./backend
      dockerfile: Dockerfile.prod
    command: celery -A myproject beat -l info
    volumes:
      - static_volume:/app/staticfiles
    env_file:
      - .env.prod
    depends_on:
      - db
      - redis
    networks:
      - backend

volumes:
  postgres_data:
  static_volume:

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
```

### Microservices Architecture
```yaml
# docker-compose.microservices.yml
version: '3.8'

services:
  api-gateway:
    image: kong:latest
    environment:
      KONG_DATABASE: "off"
      KONG_DECLARATIVE_CONFIG: /kong/declarative/kong.yml
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    volumes:
      - ./kong.yml:/kong/declarative/kong.yml
    ports:
      - "8000:8000"
      - "8001:8001"
    networks:
      - microservices

  user-service:
    build: ./user-service
    environment:
      - DB_HOST=user-db
    depends_on:
      - user-db
    networks:
      - microservices

  user-db:
    image: postgres:13-alpine
    environment:
      POSTGRES_DB: users
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - user_db_data:/var/lib/postgresql/data
    networks:
      - microservices

  product-service:
    build: ./product-service
    environment:
      - DB_HOST=product-db
    depends_on:
      - product-db
    networks:
      - microservices

  product-db:
    image: mongo:4.4
    volumes:
      - product_db_data:/data/db
    networks:
      - microservices

  order-service:
    build: ./order-service
    environment:
      - DB_HOST=order-db
      - USER_SERVICE_URL=http://user-service:3000
      - PRODUCT_SERVICE_URL=http://product-service:3000
    depends_on:
      - order-db
      - user-service
      - product-service
    networks:
      - microservices

  order-db:
    image: postgres:13-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - order_db_data:/var/lib/postgresql/data
    networks:
      - microservices

  message-queue:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: password
    ports:
      - "15672:15672"
    networks:
      - microservices

volumes:
  user_db_data:
  product_db_data:
  order_db_data:

networks:
  microservices:
    driver: bridge
```

---

## Best Practices

### Development Best Practices

1. **Use Multi-stage Builds**
   ```dockerfile
   FROM node:16 as build
   # Build steps
   
   FROM node:16-alpine as production
   COPY --from=build /app/dist ./dist
   ```

2. **Optimize Layer Caching**
   ```dockerfile
   # Copy package files first
   COPY package*.json ./
   RUN npm install
   
   # Copy source code last
   COPY . .
   ```

3. **Use .dockerignore**
   ```
   node_modules
   .git
   .env
   *.log
   coverage/
   .nyc_output
   ```

4. **Run as Non-root User**
   ```dockerfile
   RUN adduser -D myuser
   USER myuser
   ```

### Production Best Practices

1. **Health Checks**
   ```dockerfile
   HEALTHCHECK --interval=30s --timeout=3s \
     CMD curl -f http://localhost:3000/health || exit 1
   ```

2. **Resource Limits**
   ```yaml
   deploy:
     resources:
       limits:
         memory: 512M
         cpus: '0.5'
   ```

3. **Secrets Management**
   ```bash
   docker secret create my_secret secret.txt
   ```

4. **Monitoring and Logging**
   ```yaml
   logging:
     driver: json-file
     options:
       max-size: "10m"
       max-file: "3"
   ```

### Security Best Practices

1. **Scan Images for Vulnerabilities**
   ```bash
   docker scout cves myapp:latest
   ```

2. **Use Official Base Images**
   ```dockerfile
   FROM node:16-alpine
   ```

3. **Keep Images Updated**
   ```bash
   docker pull node:16-alpine
   ```

4. **Minimal Privileges**
   ```bash
   docker run --cap-drop=ALL myapp
   ```

---

## Troubleshooting

### Common Issues and Solutions

#### Container Won't Start
```bash
# Check logs
docker logs container_name

# Check exit code
docker ps -a

# Run interactively
docker run -it image_name /bin/bash
```

#### Port Already in Use
```bash
# Find process using port
netstat -tulpn | grep :8080
lsof -i :8080

# Kill process
kill -9 PID

# Use different port
docker run -p 8081:80 nginx
```

#### Out of Disk Space
```bash
# Clean up unused resources
docker system prune

# Remove unused images
docker image prune

# Remove unused volumes
docker volume prune

# Check disk usage
docker system df
```

#### Performance Issues
```bash
# Monitor resource usage
docker stats

# Limit resources
docker run -m 512m --cpus="1.0" myapp

# Check container processes
docker exec container_name ps aux
```

#### Network Issues
```bash
# Test connectivity
docker exec container_name ping google.com

# Check network configuration
docker network inspect bridge

# Test DNS resolution
docker exec container_name nslookup google.com
```

### Debugging Commands

```bash
# Enter running container
docker exec -it container_name /bin/bash

# Check container configuration
docker inspect container_name

# View real-time events
docker events

# Check Docker daemon logs
journalctl -u docker.service

# Test image locally
docker run --rm -it image_name /bin/bash
```

### Useful Scripts

#### Cleanup Script
```bash
#!/bin/bash
# cleanup-docker.sh

echo "Cleaning up Docker resources..."

# Remove stopped containers
docker container prune -f

# Remove unused images
docker image prune -a -f

# Remove unused volumes
docker volume prune -f

# Remove unused networks
docker network prune -f

# Show remaining usage
docker system df
```

#### Backup Script
```bash
#!/bin/bash
# backup-volumes.sh

VOLUME_NAME=$1
BACKUP_NAME=$2

if [ -z "$VOLUME_NAME" ] || [ -z "$BACKUP_NAME" ]; then
    echo "Usage: $0 <volume_name> <backup_name>"
    exit 1
fi

docker run --rm \
    -v $VOLUME_NAME:/data \
    -v $(pwd):/backup \
    alpine \
    tar czf /backup/$BACKUP_NAME.tar.gz -C /data .

echo "Backup created: $BACKUP_NAME.tar.gz"
```

This comprehensive Docker guide covers everything from basic concepts to advanced production deployments. Each section builds upon the previous one, providing practical examples and real-world scenarios you'll encounter when working with Docker.