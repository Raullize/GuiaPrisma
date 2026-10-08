<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# Deploy e Produção

## Preparação para Produção

### Variáveis de Ambiente

```bash
# .env.production
DATABASE_URL="postgresql://user:password@prod-host:5432/proddb?sslmode=require"
NODE_ENV="production"
PORT=8080
JWT_SECRET="your-super-secret-jwt-key"
REDIS_URL="redis://redis-host:6379"

# Configurações de conexão
DATABASE_CONNECTION_LIMIT=10
DATABASE_POOL_TIMEOUT=20
DATABASE_QUERY_TIMEOUT=30

# Logs
LOG_LEVEL="info"
LOG_FORMAT="json"

# Monitoramento
SENTRY_DSN="https://your-sentry-dsn"
NEW_RELIC_LICENSE_KEY="your-newrelic-key"
```

### Configuração do Prisma Client

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma = globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' 
      ? ['query', 'error', 'warn'] 
      : ['error'],
    datasources: {
      db: {
        url: process.env.DATABASE_URL,
      },
    },
    // Configurações de produção
    ...(process.env.NODE_ENV === 'production' && {
      errorFormat: 'minimal',
      // Connection pooling
      datasources: {
        db: {
          url: `${process.env.DATABASE_URL}?connection_limit=${process.env.DATABASE_CONNECTION_LIMIT || 10}&pool_timeout=${process.env.DATABASE_POOL_TIMEOUT || 20}`,
        },
      },
    }),
  })

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma
}

// Graceful shutdown
process.on('SIGINT', async () => {
  console.log('Received SIGINT, shutting down gracefully...')
  await prisma.$disconnect()
  process.exit(0)
})

process.on('SIGTERM', async () => {
  console.log('Received SIGTERM, shutting down gracefully...')
  await prisma.$disconnect()
  process.exit(0)
})
```

### Health Checks

```typescript
// routes/health.ts
import { Router } from 'express'
import { prisma } from '../lib/prisma'

const router = Router()

router.get('/health', async (req, res) => {
  const startTime = Date.now()
  
  try {
    // Verificar conexão com banco
    await prisma.$queryRaw`SELECT 1`
    
    const dbLatency = Date.now() - startTime
    
    res.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      database: {
        status: 'connected',
        latency: `${dbLatency}ms`,
      },
      memory: {
        used: Math.round(process.memoryUsage().heapUsed / 1024 / 1024),
        total: Math.round(process.memoryUsage().heapTotal / 1024 / 1024),
      },
      version: process.env.npm_package_version || '1.0.0',
    })
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      timestamp: new Date().toISOString(),
      database: {
        status: 'disconnected',
        error: error.message,
      },
    })
  }
})

router.get('/ready', async (req, res) => {
  try {
    // Verificações mais rigorosas para readiness
    await prisma.$queryRaw`SELECT 1`
    
    // Verificar se migrations estão aplicadas
    const migrations = await prisma.$queryRaw`
      SELECT * FROM "_prisma_migrations" 
      WHERE "finished_at" IS NULL
    `
    
    if (Array.isArray(migrations) && migrations.length > 0) {
      throw new Error('Pending migrations detected')
    }
    
    res.json({ status: 'ready' })
  } catch (error) {
    res.status(503).json({
      status: 'not ready',
      error: error.message,
    })
  }
})

export { router as healthRouter }
```

## Docker e Containerização

### Dockerfile Otimizado

```dockerfile
# Dockerfile
# Usar imagem Alpine para menor tamanho
FROM node:18-alpine AS base

# Instalar dependências do sistema
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Copiar arquivos de dependências
COPY package*.json ./
COPY prisma ./prisma/

# Stage para dependências
FROM base AS deps
RUN npm ci --only=production && npm cache clean --force

# Stage para build
FROM base AS builder
RUN npm ci
RUN npx prisma generate
COPY . .
RUN npm run build

# Stage final
FROM node:18-alpine AS runner
WORKDIR /app

# Criar usuário não-root
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copiar arquivos necessários
COPY --from=deps --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nextjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nextjs:nodejs /app/prisma ./prisma
COPY --from=builder --chown=nextjs:nodejs /app/package.json ./package.json

# Gerar Prisma Client
RUN npx prisma generate

USER nextjs

EXPOSE 3000

ENV PORT 3000
ENV NODE_ENV production

# Script de inicialização
CMD ["sh", "-c", "npx prisma migrate deploy && npm start"]
```

### Docker Compose para Produção

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}?sslmode=disable
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 1G
        reservations:
          memory: 512M

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 256M
        reservations:
          memory: 128M

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app
    restart: unless-stopped

volumes:
  postgres_data:

networks:
  default:
    driver: bridge
```

### Configuração do Nginx

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream app {
        server app:3000;
    }

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;

    server {
        listen 80;
        server_name your-domain.com;
        
        # Redirect HTTP to HTTPS
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name your-domain.com;

        # SSL configuration
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;

        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";

        # API routes
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://app;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
        }

        # Auth routes with stricter rate limiting
        location /api/auth/ {
            limit_req zone=login burst=5 nodelay;
            proxy_pass http://app;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Health check
        location /health {
            proxy_pass http://app;
            access_log off;
        }

        # Static files (if serving from same container)
        location /static/ {
            expires 1y;
            add_header Cache-Control "public, immutable";
            proxy_pass http://app;
        }
    }
}
```

## Deploy em Cloud Providers

### Vercel

```json
// vercel.json
{
  "version": 2,
  "builds": [
    {
      "src": "package.json",
      "use": "@vercel/node"
    }
  ],
  "routes": [
    {
      "src": "/api/(.*)",
      "dest": "/api/$1"
    }
  ],
  "env": {
    "DATABASE_URL": "@database-url",
    "JWT_SECRET": "@jwt-secret"
  },
  "functions": {
    "pages/api/**/*.js": {
      "maxDuration": 30
    }
  }
}
```

```bash
# Script de deploy para Vercel
#!/bin/bash
set -e

echo "🔄 Gerando Prisma Client..."
npx prisma generate

echo "🚀 Aplicando migrations..."
npx prisma migrate deploy

echo "✅ Deploy concluído!"
```

### Railway

```dockerfile
# railway.dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY prisma ./prisma/
RUN npx prisma generate

COPY . .
RUN npm run build

# Railway executa migrations automaticamente
CMD ["sh", "-c", "npx prisma migrate deploy && npm start"]
```

```json
// railway.json
{
  "deploy": {
    "startCommand": "npm start",
    "healthcheckPath": "/health",
    "healthcheckTimeout": 100
  }
}
```

### AWS ECS

```json
// task-definition.json
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::account:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::account:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "your-account.dkr.ecr.region.amazonaws.com/my-app:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:database-url"
        }
      ],
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -f http://localhost:3000/health || exit 1"
        ],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### Google Cloud Run

```yaml
# cloudbuild.yaml
steps:
  # Build da imagem
  - name: 'gcr.io/cloud-builders/docker'
    args: [
      'build',
      '-t', 'gcr.io/$PROJECT_ID/my-app:$COMMIT_SHA',
      '-t', 'gcr.io/$PROJECT_ID/my-app:latest',
      '.'
    ]

  # Push da imagem
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/my-app:$COMMIT_SHA']

  # Deploy no Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args: [
      'run', 'deploy', 'my-app',
      '--image', 'gcr.io/$PROJECT_ID/my-app:$COMMIT_SHA',
      '--region', 'us-central1',
      '--platform', 'managed',
      '--allow-unauthenticated',
      '--set-env-vars', 'NODE_ENV=production',
      '--set-secrets', 'DATABASE_URL=database-url:latest',
      '--memory', '1Gi',
      '--cpu', '1',
      '--concurrency', '100',
      '--max-instances', '10'
    ]

images:
  - 'gcr.io/$PROJECT_ID/my-app:$COMMIT_SHA'
```

## Monitoramento e Observabilidade

### Logging Estruturado

```typescript
// lib/logger.ts
import winston from 'winston'

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    process.env.NODE_ENV === 'production'
      ? winston.format.json()
      : winston.format.combine(
          winston.format.colorize(),
          winston.format.simple()
        )
  ),
  defaultMeta: {
    service: 'my-app',
    version: process.env.npm_package_version,
  },
  transports: [
    new winston.transports.Console(),
    ...(process.env.NODE_ENV === 'production'
      ? [
          new winston.transports.File({
            filename: 'logs/error.log',
            level: 'error',
          }),
          new winston.transports.File({
            filename: 'logs/combined.log',
          }),
        ]
      : []),
  ],
})

export { logger }
```

### Middleware de Logging

```typescript
// middleware/logging.ts
import { Request, Response, NextFunction } from 'express'
import { logger } from '../lib/logger'

export function requestLogger(req: Request, res: Response, next: NextFunction) {
  const start = Date.now()
  
  res.on('finish', () => {
    const duration = Date.now() - start
    
    logger.info('HTTP Request', {
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      userAgent: req.get('User-Agent'),
      ip: req.ip,
      ...(req.user && { userId: req.user.id }),
    })
  })
  
  next()
}
```

### Métricas com Prometheus

```typescript
// lib/metrics.ts
import client from 'prom-client'

// Registrar métricas padrão
client.collectDefaultMetrics()

// Métricas customizadas
export const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10],
})

export const httpRequestTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
})

export const databaseQueryDuration = new client.Histogram({
  name: 'database_query_duration_seconds',
  help: 'Duration of database queries in seconds',
  labelNames: ['operation', 'model'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 3, 5],
})

export const activeConnections = new client.Gauge({
  name: 'database_connections_active',
  help: 'Number of active database connections',
})

// Endpoint de métricas
export function metricsHandler(req: Request, res: Response) {
  res.set('Content-Type', client.register.contentType)
  res.end(client.register.metrics())
}
```

### Middleware de Métricas

```typescript
// middleware/metrics.ts
import { Request, Response, NextFunction } from 'express'
import { httpRequestDuration, httpRequestTotal } from '../lib/metrics'

export function metricsMiddleware(req: Request, res: Response, next: NextFunction) {
  const start = Date.now()
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000
    const route = req.route?.path || req.path
    
    httpRequestDuration
      .labels(req.method, route, res.statusCode.toString())
      .observe(duration)
    
    httpRequestTotal
      .labels(req.method, route, res.statusCode.toString())
      .inc()
  })
  
  next()
}
```

## Estratégias de Deploy

### Blue-Green Deployment

```bash
#!/bin/bash
# deploy-blue-green.sh

set -e

CURRENT_ENV=$(curl -s http://load-balancer/current-env)
NEW_ENV=$([ "$CURRENT_ENV" = "blue" ] && echo "green" || echo "blue")

echo "🔄 Deploying to $NEW_ENV environment..."

# Build e deploy da nova versão
docker build -t my-app:$NEW_ENV .
docker tag my-app:$NEW_ENV registry.com/my-app:$NEW_ENV
docker push registry.com/my-app:$NEW_ENV

# Atualizar ambiente
kubectl set image deployment/my-app-$NEW_ENV app=registry.com/my-app:$NEW_ENV
kubectl rollout status deployment/my-app-$NEW_ENV

# Executar migrations
kubectl exec deployment/my-app-$NEW_ENV -- npx prisma migrate deploy

# Health check
echo "🔍 Running health checks..."
for i in {1..30}; do
  if curl -f http://my-app-$NEW_ENV/health; then
    echo "✅ Health check passed"
    break
  fi
  echo "⏳ Waiting for health check... ($i/30)"
  sleep 10
done

# Smoke tests
echo "🧪 Running smoke tests..."
npm run test:smoke -- --env=$NEW_ENV

# Switch traffic
echo "🔄 Switching traffic to $NEW_ENV..."
kubectl patch service my-app -p '{"spec":{"selector":{"version":"'$NEW_ENV'"}}}'

echo "✅ Deployment completed successfully!"
echo "🗑️  Old environment ($CURRENT_ENV) is ready for cleanup"
```

### Rolling Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      initContainers:
      - name: migrate
        image: my-app:latest
        command: ['npx', 'prisma', 'migrate', 'deploy']
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
      containers:
      - name: app
        image: my-app:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### Canary Deployment

```yaml
# k8s/canary-deployment.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {}
      - setWeight: 40
      - pause: {duration: 10}
      - setWeight: 60
      - pause: {duration: 10}
      - setWeight: 80
      - pause: {duration: 10}
      analysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: my-app
      trafficRouting:
        nginx:
          stableService: my-app-stable
          canaryService: my-app-canary
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:latest
        ports:
        - containerPort: 3000
```

## Backup e Disaster Recovery

### Backup Automatizado

```bash
#!/bin/bash
# backup-database.sh

set -e

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${DATE}.sql"
S3_BUCKET="my-app-backups"

echo "📦 Creating database backup..."
pg_dump $DATABASE_URL > $BACKUP_NAME

echo "📤 Uploading to S3..."
aws s3 cp $BACKUP_NAME s3://$S3_BUCKET/daily/$BACKUP_NAME

echo "🗜️ Compressing backup..."
gzip $BACKUP_NAME

echo "📤 Uploading compressed backup..."
aws s3 cp $BACKUP_NAME.gz s3://$S3_BUCKET/compressed/$BACKUP_NAME.gz

echo "🧹 Cleaning up local files..."
rm $BACKUP_NAME.gz

echo "✅ Backup completed: $BACKUP_NAME"

# Cleanup old backups (keep last 30 days)
aws s3 ls s3://$S3_BUCKET/daily/ | while read -r line; do
  createDate=$(echo $line | awk '{print $1" "$2}')
  createDate=$(date -d "$createDate" +%s)
  olderThan=$(date -d "30 days ago" +%s)
  if [[ $createDate -lt $olderThan ]]; then
    fileName=$(echo $line | awk '{print $4}')
    if [[ $fileName != "" ]]; then
      aws s3 rm s3://$S3_BUCKET/daily/$fileName
      echo "🗑️ Deleted old backup: $fileName"
    fi
  fi
done
```

### Restore Process

```bash
#!/bin/bash
# restore-database.sh

set -e

BACKUP_FILE=$1
TARGET_DB=$2

if [ -z "$BACKUP_FILE" ] || [ -z "$TARGET_DB" ]; then
  echo "Usage: $0 <backup_file> <target_database_url>"
  exit 1
fi

echo "⚠️  WARNING: This will replace the target database!"
read -p "Are you sure? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
  echo "❌ Restore cancelled"
  exit 1
fi

echo "📥 Downloading backup from S3..."
aws s3 cp s3://my-app-backups/daily/$BACKUP_FILE ./

echo "🗄️ Restoring database..."
psql $TARGET_DB < $BACKUP_FILE

echo "🔄 Running migrations..."
DATABASE_URL=$TARGET_DB npx prisma migrate deploy

echo "✅ Restore completed successfully!"

# Cleanup
rm $BACKUP_FILE
```

---

[Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>