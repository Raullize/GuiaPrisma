<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# Troubleshooting: Problemas Comuns e Soluções

## Problemas de Instalação e Configuração

### 1. Erro: "prisma command not found"

**Problema:**
```bash
prisma: command not found
```

**Soluções:**

```bash
# Opção 1: Instalar globalmente
npm install -g prisma

# Opção 2: Usar npx (recomendado)
npx prisma --version

# Opção 3: Adicionar ao package.json scripts
{
  "scripts": {
    "prisma": "prisma",
    "db:migrate": "prisma migrate dev",
    "db:generate": "prisma generate"
  }
}
```

### 2. Erro de Conexão com Banco de Dados

**Problema:**
```
Error: P1001: Can't reach database server
```

**Verificações:**

```bash
# 1. Verificar se o banco está rodando
# PostgreSQL
sudo systemctl status postgresql
# ou
ps aux | grep postgres

# MySQL
sudo systemctl status mysql
# ou
ps aux | grep mysql

# 2. Testar conexão direta
# PostgreSQL
psql -h localhost -U username -d database_name

# MySQL
mysql -h localhost -u username -p database_name

# 3. Verificar variáveis de ambiente
echo $DATABASE_URL
```

**Soluções Comuns:**

```bash
# .env - Verificar formato da URL
# PostgreSQL
DATABASE_URL="postgresql://username:password@localhost:5432/database_name"

# MySQL
DATABASE_URL="mysql://username:password@localhost:3306/database_name"

# SQLite
DATABASE_URL="file:./dev.db"

# MongoDB
DATABASE_URL="mongodb://username:password@localhost:27017/database_name"
```

### 3. Erro de Permissões

**Problema:**
```
Error: P1010: User does not have permission to access the database
```

**Soluções:**

```sql
-- PostgreSQL
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO myuser;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO myuser;

-- MySQL
GRANT ALL PRIVILEGES ON mydb.* TO 'myuser'@'localhost';
FLUSH PRIVILEGES;
```

## Problemas com Migrations

### 1. Migration Falhou

**Problema:**
```
Migration failed to apply cleanly to the shadow database
```

**Soluções:**

```bash
# 1. Reset do banco de desenvolvimento
npx prisma migrate reset

# 2. Resolver conflitos manualmente
npx prisma migrate resolve --applied "20231201000000_migration_name"

# 3. Forçar aplicação (cuidado em produção)
npx prisma db push --force-reset
```

### 2. Schema Drift Detectado

**Problema:**
```
Your database schema is not in sync with your migration history
```

**Diagnóstico:**

```bash
# Verificar status das migrations
npx prisma migrate status

# Ver diferenças
npx prisma db diff
```

**Soluções:**

```bash
# 1. Aplicar migrations pendentes
npx prisma migrate deploy

# 2. Criar migration para sincronizar
npx prisma migrate diff --from-empty --to-schema-datamodel prisma/schema.prisma --script > fix.sql

# 3. Marcar migration como aplicada (se já foi aplicada manualmente)
npx prisma migrate resolve --applied "migration_name"
```

### 3. Rollback de Migration

**Problema:** Necessidade de reverter uma migration

**Soluções:**

```bash
# 1. Criar migration reversa
npx prisma migrate diff \
  --from-schema-datamodel prisma/schema.prisma \
  --to-migrations ./prisma/migrations/20231201000000_previous_state \
  --script > rollback.sql

# 2. Aplicar rollback manual
# Execute o SQL gerado no banco

# 3. Marcar migration como não aplicada
npx prisma migrate resolve --rolled-back "20231201000000_migration_to_rollback"
```

## Problemas com Prisma Client

### 1. Prisma Client Não Gerado

**Problema:**
```
Cannot find module '@prisma/client'
```

**Soluções:**

```bash
# 1. Gerar o client
npx prisma generate

# 2. Verificar se está no package.json
npm install @prisma/client

# 3. Adicionar ao postinstall
{
  "scripts": {
    "postinstall": "prisma generate"
  }
}
```

### 2. Tipos TypeScript Desatualizados

**Problema:** IntelliSense não funciona ou tipos incorretos

**Soluções:**

```bash
# 1. Regenerar client após mudanças no schema
npx prisma generate

# 2. Reiniciar TypeScript server (VS Code)
# Ctrl+Shift+P -> "TypeScript: Restart TS Server"

# 3. Limpar cache do TypeScript
rm -rf node_modules/.prisma
npm install
npx prisma generate
```

### 3. Erro de Instância Múltipla

**Problema:**
```
Warn: Multiple Prisma Client instances detected
```

**Solução - Singleton Pattern:**

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma = globalForPrisma.prisma ??
  new PrismaClient({
    log: ['query', 'error', 'warn'],
  })

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma
}
```

## Problemas de Performance

### 1. Queries Lentas

**Diagnóstico:**

```typescript
// Habilitar logs de query
const prisma = new PrismaClient({
  log: [
    {
      emit: 'event',
      level: 'query',
    },
  ],
})

prisma.$on('query', (e) => {
  console.log('Query: ' + e.query)
  console.log('Duration: ' + e.duration + 'ms')
})
```

**Soluções:**

```typescript
// 1. Adicionar índices
model User {
  id    Int    @id @default(autoincrement())
  email String @unique
  name  String
  
  @@index([name]) // Índice simples
  @@index([email, name]) // Índice composto
}

// 2. Otimizar queries com select
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    // Não buscar campos desnecessários
  },
})

// 3. Usar paginação
const users = await prisma.user.findMany({
  skip: (page - 1) * limit,
  take: limit,
})

// 4. Batch operations
const users = await prisma.user.createMany({
  data: userData,
  skipDuplicates: true,
})
```

### 2. Problema de N+1 Queries

**Problema:**
```typescript
// ❌ Gera N+1 queries
const users = await prisma.user.findMany()
for (const user of users) {
  const posts = await prisma.post.findMany({
    where: { authorId: user.id }
  })
}
```

**Solução:**
```typescript
// ✅ Uma única query com include
const users = await prisma.user.findMany({
  include: {
    posts: true,
  },
})

// ✅ Ou com select específico
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    posts: {
      select: {
        id: true,
        title: true,
      },
    },
  },
})
```

### 3. Connection Pool Esgotado

**Problema:**
```
Error: P2024: Timed out fetching a new connection from the connection pool
```

**Soluções:**

```typescript
// 1. Configurar connection pool
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL + '?connection_limit=10&pool_timeout=20',
    },
  },
})

// 2. Sempre fechar conexões
try {
  const result = await prisma.user.findMany()
  return result
} finally {
  await prisma.$disconnect()
}

// 3. Usar singleton pattern
// (ver exemplo anterior)
```

## Problemas em Produção

### 1. Erro de Deploy

**Problema:** Migrations não aplicadas em produção

**Soluções:**

```bash
# 1. Script de deploy
#!/bin/bash
set -e

echo "Aplicando migrations..."
npx prisma migrate deploy

echo "Gerando Prisma Client..."
npx prisma generate

echo "Iniciando aplicação..."
npm start

# 2. Docker multi-stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY prisma ./prisma/
RUN npx prisma generate
COPY . .
RUN npm run build

FROM node:18-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/prisma ./prisma
CMD ["sh", "-c", "npx prisma migrate deploy && npm start"]
```

### 2. Problemas de SSL

**Problema:**
```
Error: P1002: The database server was reached but timed out
```

**Soluções:**

```bash
# 1. Configurar SSL na URL
DATABASE_URL="postgresql://user:pass@host:5432/db?sslmode=require"

# 2. Para desenvolvimento local
DATABASE_URL="postgresql://user:pass@host:5432/db?sslmode=disable"

# 3. Certificado customizado
DATABASE_URL="postgresql://user:pass@host:5432/db?sslmode=require&sslcert=./client-cert.pem&sslkey=./client-key.pem&sslrootcert=./ca-cert.pem"
```

### 3. Monitoramento e Logs

**Setup de Monitoramento:**

```typescript
// middleware/prisma-logger.ts
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
      level: 'warn',
    },
  ],
})

prisma.$on('query', (e) => {
  if (e.duration > 1000) { // Queries > 1s
    console.warn(`Slow query detected: ${e.duration}ms`, {
      query: e.query,
      params: e.params,
    })
  }
})

prisma.$on('error', (e) => {
  console.error('Prisma error:', e)
})

// Health check endpoint
app.get('/health', async (req, res) => {
  try {
    await prisma.$queryRaw`SELECT 1`
    res.json({ status: 'healthy', database: 'connected' })
  } catch (error) {
    res.status(503).json({ 
      status: 'unhealthy', 
      database: 'disconnected',
      error: error.message 
    })
  }
})
```

## Ferramentas de Debug

### 1. Prisma Studio

```bash
# Abrir interface visual
npx prisma studio

# Em produção (cuidado!)
npx prisma studio --browser none --port 5555
```

### 2. Debug de Queries

```typescript
// Habilitar debug detalhado
const prisma = new PrismaClient({
  log: [
    {
      emit: 'event',
      level: 'query',
    },
  ],
})

prisma.$on('query', (e) => {
  console.log('Query: ' + e.query)
  console.log('Params: ' + e.params)
  console.log('Duration: ' + e.duration + 'ms')
  console.log('---')
})

// Ou usar DEBUG environment variable
// DEBUG=prisma:query npm start
```

### 3. Validação de Schema

```bash
# Validar schema
npx prisma validate

# Formatar schema
npx prisma format

# Verificar diferenças
npx prisma db diff
```

## Checklist de Troubleshooting

### Antes de Reportar um Bug:

1. **Versões:**
   ```bash
   npx prisma --version
   node --version
   npm --version
   ```

2. **Logs:**
   ```bash
   DEBUG=prisma:* npm start
   ```

3. **Schema:**
   ```bash
   npx prisma validate
   npx prisma format
   ```

4. **Migrations:**
   ```bash
   npx prisma migrate status
   ```

5. **Conexão:**
   ```bash
   npx prisma db pull
   ```

### Comandos Úteis para Debug:

```bash
# Informações do sistema
npx prisma --version
npx envinfo --binaries --databases --npmPackages prisma,@prisma/client

# Reset completo (desenvolvimento)
npx prisma migrate reset --force
npx prisma db push --force-reset

# Regenerar tudo
rm -rf node_modules/.prisma
npm install
npx prisma generate

# Verificar schema
npx prisma validate
npx prisma format
```

---

[Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>