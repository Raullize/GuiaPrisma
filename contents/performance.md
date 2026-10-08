<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# Performance e Otimização

## Otimização de Queries

### 1. Select Específico

```typescript
// ❌ Ruim: buscar todos os campos
const users = await prisma.user.findMany()

// ✅ Bom: buscar apenas campos necessários
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true,
  },
})

// ✅ Ainda melhor: usar include para relacionamentos
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true,
    posts: {
      select: {
        id: true,
        title: true,
        published: true,
      },
    },
  },
})
```

### 2. Paginação Eficiente

```typescript
// ✅ Paginação com cursor (mais eficiente para grandes datasets)
const posts = await prisma.post.findMany({
  take: 10,
  cursor: lastPostId ? { id: lastPostId } : undefined,
  skip: lastPostId ? 1 : 0,
  orderBy: {
    createdAt: 'desc',
  },
})

// ✅ Paginação offset (boa para pequenos datasets)
const posts = await prisma.post.findMany({
  skip: (page - 1) * limit,
  take: limit,
  orderBy: {
    createdAt: 'desc',
  },
})

// ✅ Paginação com contagem otimizada
const [posts, total] = await Promise.all([
  prisma.post.findMany({
    skip: (page - 1) * limit,
    take: limit,
    orderBy: { createdAt: 'desc' },
  }),
  prisma.post.count(),
])
```

### 3. Evitar Problema N+1

```typescript
// ❌ Problema N+1: múltiplas queries
const users = await prisma.user.findMany()
for (const user of users) {
  const posts = await prisma.post.findMany({
    where: { authorId: user.id },
  })
  console.log(`${user.name} tem ${posts.length} posts`)
}

// ✅ Solução: usar include ou select
const users = await prisma.user.findMany({
  include: {
    posts: true,
  },
})
users.forEach(user => {
  console.log(`${user.name} tem ${user.posts.length} posts`)
})

// ✅ Ainda melhor: usar _count
const users = await prisma.user.findMany({
  include: {
    _count: {
      select: {
        posts: true,
      },
    },
  },
})
users.forEach(user => {
  console.log(`${user.name} tem ${user._count.posts} posts`)
})
```

## Índices e Otimização de Schema

### 1. Índices Simples

```prisma
model User {
  id       Int     @id @default(autoincrement())
  email    String  @unique
  name     String
  status   String
  
  // Índice para queries frequentes
  @@index([status])
  @@index([name])
}
```

### 2. Índices Compostos

```prisma
model Post {
  id          Int      @id @default(autoincrement())
  title       String
  published   Boolean  @default(false)
  authorId    Int
  categoryId  Int
  createdAt   DateTime @default(now())
  
  author   User     @relation(fields: [authorId], references: [id])
  category Category @relation(fields: [categoryId], references: [id])
  
  // Índices compostos para queries complexas
  @@index([published, createdAt])
  @@index([authorId, published])
  @@index([categoryId, published, createdAt])
}
```

### 3. Índices Únicos Compostos

```prisma
model UserRole {
  id     Int @id @default(autoincrement())
  userId Int
  roleId Int
  
  user User @relation(fields: [userId], references: [id])
  role Role @relation(fields: [roleId], references: [id])
  
  // Evitar duplicatas
  @@unique([userId, roleId])
}
```

## Connection Pooling

### 1. Configuração de Pool

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient({
  datasources: {
    db: {
      url: `${process.env.DATABASE_URL}?connection_limit=10&pool_timeout=20`,
    },
  },
})

export { prisma }
```

### 2. Pool com PgBouncer

```bash
# DATABASE_URL para connection pooling
DATABASE_URL="postgresql://user:password@localhost:6543/mydb?pgbouncer=true"

# DATABASE_URL para migrations (conexão direta)
DIRECT_URL="postgresql://user:password@localhost:5432/mydb"
```

```prisma
// schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}
```

### 3. Configuração Docker com PgBouncer

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  pgbouncer:
    image: pgbouncer/pgbouncer:latest
    environment:
      DATABASES_HOST: postgres
      DATABASES_PORT: 5432
      DATABASES_USER: postgres
      DATABASES_PASSWORD: password
      DATABASES_DBNAME: myapp
      POOL_MODE: transaction
      SERVER_RESET_QUERY: DISCARD ALL
      MAX_CLIENT_CONN: 25
      DEFAULT_POOL_SIZE: 5
      MIN_POOL_SIZE: 0
      RESERVE_POOL_SIZE: 0
      RESERVE_POOL_TIMEOUT: 5
    ports:
      - "6543:5432"
    depends_on:
      - postgres

volumes:
  postgres_data:
```

## Batch Operations

### 1. CreateMany

```typescript
// ❌ Ineficiente: múltiplas inserções
for (const userData of users) {
  await prisma.user.create({ data: userData })
}

// ✅ Eficiente: inserção em lote
const users = await prisma.user.createMany({
  data: [
    { name: 'Alice', email: 'alice@example.com' },
    { name: 'Bob', email: 'bob@example.com' },
    { name: 'Charlie', email: 'charlie@example.com' },
  ],
  skipDuplicates: true, // Ignorar duplicatas
})
```

### 2. UpdateMany

```typescript
// Atualizar múltiplos registros
const result = await prisma.post.updateMany({
  where: {
    published: false,
    createdAt: {
      lt: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000), // 30 dias atrás
    },
  },
  data: {
    status: 'archived',
  },
})

console.log(`${result.count} posts arquivados`)
```

### 3. DeleteMany

```typescript
// Deletar múltiplos registros
const result = await prisma.user.deleteMany({
  where: {
    lastLoginAt: {
      lt: new Date(Date.now() - 365 * 24 * 60 * 60 * 1000), // 1 ano atrás
    },
    status: 'inactive',
  },
})

console.log(`${result.count} usuários inativos removidos`)
```

## Transações Otimizadas

### 1. Transações Interativas

```typescript
// ✅ Transação otimizada
const result = await prisma.$transaction(async (tx) => {
  // Operações relacionadas em uma única transação
  const user = await tx.user.create({
    data: {
      name: 'John Doe',
      email: 'john@example.com',
    },
  })

  const profile = await tx.profile.create({
    data: {
      userId: user.id,
      bio: 'Software Developer',
    },
  })

  // Atualizar estatísticas
  await tx.stats.update({
    where: { id: 1 },
    data: {
      totalUsers: { increment: 1 },
    },
  })

  return { user, profile }
})
```

### 2. Transações com Timeout

```typescript
const result = await prisma.$transaction(
  async (tx) => {
    // Operações da transação
    const orders = await tx.order.findMany({
      where: { status: 'pending' },
    })

    for (const order of orders) {
      await tx.order.update({
        where: { id: order.id },
        data: { status: 'processing' },
      })
      
      // Simular processamento
      await new Promise(resolve => setTimeout(resolve, 100))
    }

    return orders
  },
  {
    maxWait: 5000, // Máximo 5s esperando para iniciar
    timeout: 10000, // Máximo 10s para executar
  }
)
```

## Caching Strategies

### 1. Cache com Redis

```typescript
// lib/cache.ts
import Redis from 'ioredis'

const redis = new Redis(process.env.REDIS_URL)

export class CacheService {
  static async get<T>(key: string): Promise<T | null> {
    const cached = await redis.get(key)
    return cached ? JSON.parse(cached) : null
  }

  static async set(key: string, value: any, ttl: number = 3600): Promise<void> {
    await redis.setex(key, ttl, JSON.stringify(value))
  }

  static async del(key: string): Promise<void> {
    await redis.del(key)
  }

  static async invalidatePattern(pattern: string): Promise<void> {
    const keys = await redis.keys(pattern)
    if (keys.length > 0) {
      await redis.del(...keys)
    }
  }
}
```

### 2. Cache em Queries

```typescript
// services/user.service.ts
import { CacheService } from '../lib/cache'
import { prisma } from '../lib/prisma'

export class UserService {
  static async getUser(id: number) {
    const cacheKey = `user:${id}`
    
    // Tentar buscar no cache
    let user = await CacheService.get(cacheKey)
    
    if (!user) {
      // Buscar no banco se não estiver no cache
      user = await prisma.user.findUnique({
        where: { id },
        include: {
          profile: true,
          _count: {
            select: {
              posts: true,
              followers: true,
            },
          },
        },
      })
      
      if (user) {
        // Cachear por 1 hora
        await CacheService.set(cacheKey, user, 3600)
      }
    }
    
    return user
  }

  static async updateUser(id: number, data: any) {
    const user = await prisma.user.update({
      where: { id },
      data,
    })
    
    // Invalidar cache
    await CacheService.del(`user:${id}`)
    
    return user
  }
}
```

### 3. Cache de Queries Complexas

```typescript
export class PostService {
  static async getPopularPosts(limit: number = 10) {
    const cacheKey = `popular-posts:${limit}`
    
    let posts = await CacheService.get(cacheKey)
    
    if (!posts) {
      posts = await prisma.post.findMany({
        where: { published: true },
        include: {
          author: {
            select: {
              id: true,
              name: true,
              avatar: true,
            },
          },
          _count: {
            select: {
              likes: true,
              comments: true,
            },
          },
        },
        orderBy: [
          { likes: { _count: 'desc' } },
          { comments: { _count: 'desc' } },
          { views: 'desc' },
        ],
        take: limit,
      })
      
      // Cachear por 30 minutos
      await CacheService.set(cacheKey, posts, 1800)
    }
    
    return posts
  }
}
```

## Monitoramento de Performance

### 1. Logging de Queries

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient({
  log: [
    {
      emit: 'event',
      level: 'query',
    },
    {
      emit: 'event',
      level: 'error',
    },
    {
      emit: 'event',
      level: 'info',
    },
    {
      emit: 'event',
      level: 'warn',
    },
  ],
})

// Log de queries lentas
prisma.$on('query', (e) => {
  const duration = e.duration
  if (duration > 1000) { // Queries > 1s
    console.warn('Slow query detected:', {
      query: e.query,
      params: e.params,
      duration: `${duration}ms`,
      timestamp: e.timestamp,
    })
  }
})

// Log de erros
prisma.$on('error', (e) => {
  console.error('Database error:', e)
})

export { prisma }
```

### 2. Middleware de Performance

```typescript
// middleware/performance.ts
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

prisma.$use(async (params, next) => {
  const before = Date.now()
  
  const result = await next(params)
  
  const after = Date.now()
  const duration = after - before
  
  // Log queries lentas
  if (duration > 500) {
    console.log('Query Performance:', {
      model: params.model,
      action: params.action,
      duration: `${duration}ms`,
      args: params.args,
    })
  }
  
  return result
})

export { prisma }
```

### 3. Métricas com Prometheus

```typescript
// lib/metrics.ts
import client from 'prom-client'

// Métricas de database
export const dbQueryDuration = new client.Histogram({
  name: 'prisma_query_duration_seconds',
  help: 'Duration of Prisma queries in seconds',
  labelNames: ['model', 'action'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
})

export const dbQueryTotal = new client.Counter({
  name: 'prisma_queries_total',
  help: 'Total number of Prisma queries',
  labelNames: ['model', 'action', 'status'],
})

export const dbConnectionsActive = new client.Gauge({
  name: 'prisma_connections_active',
  help: 'Number of active database connections',
})

// Middleware para coletar métricas
prisma.$use(async (params, next) => {
  const start = Date.now()
  
  try {
    const result = await next(params)
    
    const duration = (Date.now() - start) / 1000
    
    dbQueryDuration
      .labels(params.model || 'unknown', params.action)
      .observe(duration)
    
    dbQueryTotal
      .labels(params.model || 'unknown', params.action, 'success')
      .inc()
    
    return result
  } catch (error) {
    dbQueryTotal
      .labels(params.model || 'unknown', params.action, 'error')
      .inc()
    
    throw error
  }
})
```

## Raw Queries para Performance Crítica

### 1. Queries SQL Otimizadas

```typescript
// Para casos específicos onde Prisma pode não ser otimal
export class AnalyticsService {
  static async getUserStats(userId: number) {
    const result = await prisma.$queryRaw`
      SELECT 
        u.id,
        u.name,
        COUNT(DISTINCT p.id) as post_count,
        COUNT(DISTINCT c.id) as comment_count,
        COUNT(DISTINCT l.id) as like_count,
        AVG(p.views) as avg_views
      FROM "User" u
      LEFT JOIN "Post" p ON u.id = p."authorId"
      LEFT JOIN "Comment" c ON u.id = c."authorId"
      LEFT JOIN "Like" l ON u.id = l."userId"
      WHERE u.id = ${userId}
      GROUP BY u.id, u.name
    `
    
    return result[0]
  }

  static async getTopAuthors(limit: number = 10) {
    return prisma.$queryRaw`
      SELECT 
        u.id,
        u.name,
        u.avatar,
        COUNT(p.id) as post_count,
        SUM(p.views) as total_views,
        AVG(p.views) as avg_views
      FROM "User" u
      INNER JOIN "Post" p ON u.id = p."authorId"
      WHERE p.published = true
      GROUP BY u.id, u.name, u.avatar
      ORDER BY total_views DESC
      LIMIT ${limit}
    `
  }
}
```

### 2. Queries com Parâmetros Tipados

```typescript
import { Prisma } from '@prisma/client'

export class ReportService {
  static async getMonthlyStats(year: number, month: number) {
    const startDate = new Date(year, month - 1, 1)
    const endDate = new Date(year, month, 0)
    
    const result = await prisma.$queryRaw<
      Array<{
        date: Date
        user_registrations: bigint
        posts_created: bigint
        comments_created: bigint
      }>
    >`
      SELECT 
        DATE(created_at) as date,
        COUNT(CASE WHEN table_name = 'User' THEN 1 END) as user_registrations,
        COUNT(CASE WHEN table_name = 'Post' THEN 1 END) as posts_created,
        COUNT(CASE WHEN table_name = 'Comment' THEN 1 END) as comments_created
      FROM (
        SELECT created_at, 'User' as table_name FROM "User" 
        WHERE created_at BETWEEN ${startDate} AND ${endDate}
        UNION ALL
        SELECT created_at, 'Post' as table_name FROM "Post" 
        WHERE created_at BETWEEN ${startDate} AND ${endDate}
        UNION ALL
        SELECT created_at, 'Comment' as table_name FROM "Comment" 
        WHERE created_at BETWEEN ${startDate} AND ${endDate}
      ) combined
      GROUP BY DATE(created_at)
      ORDER BY date
    `
    
    return result.map(row => ({
      ...row,
      user_registrations: Number(row.user_registrations),
      posts_created: Number(row.posts_created),
      comments_created: Number(row.comments_created),
    }))
  }
}
```

## Otimizações Específicas por Banco

### PostgreSQL

```sql
-- Análise de performance
EXPLAIN ANALYZE SELECT * FROM "Post" WHERE published = true;

-- Índices parciais
CREATE INDEX idx_published_posts ON "Post" (created_at) WHERE published = true;

-- Índices de texto completo
CREATE INDEX idx_post_search ON "Post" USING gin(to_tsvector('english', title || ' ' || content));
```

### MySQL

```sql
-- Análise de performance
EXPLAIN FORMAT=JSON SELECT * FROM Post WHERE published = true;

-- Índices compostos otimizados
CREATE INDEX idx_post_lookup ON Post (published, category_id, created_at);
```

## Checklist de Performance

### ✅ Schema Design
- [ ] Índices em colunas frequentemente consultadas
- [ ] Índices compostos para queries complexas
- [ ] Tipos de dados apropriados
- [ ] Normalização adequada

### ✅ Queries
- [ ] Select apenas campos necessários
- [ ] Usar include/select em vez de queries separadas
- [ ] Implementar paginação adequada
- [ ] Evitar problema N+1

### ✅ Connection Management
- [ ] Connection pooling configurado
- [ ] Limites de conexão apropriados
- [ ] Timeout configurado

### ✅ Caching
- [ ] Cache de queries frequentes
- [ ] Invalidação de cache adequada
- [ ] TTL apropriado

### ✅ Monitoramento
- [ ] Log de queries lentas
- [ ] Métricas de performance
- [ ] Alertas configurados
- [ ] Análise regular de performance

---

[Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>