<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# Boas Práticas com Prisma

## Estrutura de Projeto

### Organização de Arquivos

```
project/
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts
├── src/
│   ├── lib/
│   │   ├── prisma.ts          # Cliente singleton
│   │   └── validations.ts     # Schemas de validação
│   ├── services/
│   │   ├── user.service.ts    # Lógica de negócio
│   │   └── post.service.ts
│   ├── types/
│   │   └── prisma.ts          # Tipos customizados
│   └── utils/
│       └── database.ts        # Utilitários de DB
└── tests/
    ├── setup.ts
    └── integration/
```

### Cliente Singleton

```typescript
// src/lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma = globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' 
      ? ['query', 'error', 'warn'] 
      : ['error'],
    errorFormat: 'pretty',
  })

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma
}

// Graceful shutdown
process.on('beforeExit', async () => {
  await prisma.$disconnect()
})

process.on('SIGINT', async () => {
  await prisma.$disconnect()
  process.exit(0)
})

process.on('SIGTERM', async () => {
  await prisma.$disconnect()
  process.exit(0)
})
```

## Schema Design

### Nomenclatura Consistente

```prisma
// ✅ Boas práticas
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  firstName String   @map("first_name")
  lastName  String   @map("last_name")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  
  // Relacionamentos sempre no plural quando array
  posts     Post[]
  comments  Comment[]
  
  @@map("users")
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  slug      String   @unique
  content   String
  published Boolean  @default(false)
  
  // Chaves estrangeiras com sufixo Id
  authorId  Int      @map("author_id")
  author    User     @relation(fields: [authorId], references: [id])
  
  @@map("posts")
  @@index([authorId])
  @@index([published])
}
```

### Campos Obrigatórios

```prisma
model BaseModel {
  id        Int      @id @default(autoincrement())
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  
  // Para soft delete
  deletedAt DateTime? @map("deleted_at")
  
  @@map("base_model")
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  
  // Campos de auditoria
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  createdBy String?  @map("created_by")
  updatedBy String?  @map("updated_by")
  
  // Versionamento otimista
  version   Int      @default(1)
  
  @@map("users")
}
```

### Índices Estratégicos

```prisma
model Post {
  id          Int      @id @default(autoincrement())
  title       String
  content     String
  published   Boolean  @default(false)
  publishedAt DateTime?
  authorId    Int
  categoryId  Int?
  
  // Índices para queries comuns
  @@index([published])                    // Posts publicados
  @@index([authorId])                     // Posts por autor
  @@index([published, publishedAt])       // Posts publicados ordenados
  @@index([categoryId, published])        // Posts por categoria
  @@index([title(ops: raw("gin_trgm_ops"))]) // Busca textual (PostgreSQL)
  
  @@map("posts")
}
```

## Validação e Type Safety

### Schemas de Validação

```typescript
// src/lib/validations.ts
import { z } from 'zod'
import type { Prisma } from '@prisma/client'

// Schema para criação de usuário
export const createUserSchema = z.object({
  email: z.string().email('Email inválido'),
  firstName: z.string().min(1, 'Nome é obrigatório'),
  lastName: z.string().min(1, 'Sobrenome é obrigatório'),
  age: z.number().min(0).max(120).optional(),
}) satisfies z.ZodType<Prisma.UserCreateInput>

// Schema para atualização
export const updateUserSchema = createUserSchema.partial()

// Schema para filtros
export const userFiltersSchema = z.object({
  email: z.string().optional(),
  ageMin: z.number().min(0).optional(),
  ageMax: z.number().max(120).optional(),
  createdAfter: z.date().optional(),
})

export type CreateUserInput = z.infer<typeof createUserSchema>
export type UpdateUserInput = z.infer<typeof updateUserSchema>
export type UserFilters = z.infer<typeof userFiltersSchema>
```

### Tipos Customizados

```typescript
// src/types/prisma.ts
import type { User, Post, Comment, Prisma } from '@prisma/client'

// Tipos com relacionamentos
export type UserWithPosts = User & {
  posts: Post[]
}

export type PostWithAuthor = Post & {
  author: User
}

export type PostWithComments = Post & {
  comments: (Comment & {
    author: User
  })[]
}

// Tipos para DTOs
export type UserCreateDTO = Omit<User, 'id' | 'createdAt' | 'updatedAt'>
export type UserUpdateDTO = Partial<UserCreateDTO>

// Tipos para responses
export type UserPublicData = Pick<User, 'id' | 'firstName' | 'lastName' | 'createdAt'>

// Tipos para queries complexas
export type UserWithStats = User & {
  _count: {
    posts: number
    comments: number
  }
}
```

## Services e Repository Pattern

### Service Layer

```typescript
// src/services/user.service.ts
import { prisma } from '../lib/prisma'
import { createUserSchema, updateUserSchema } from '../lib/validations'
import type { CreateUserInput, UpdateUserInput, UserFilters } from '../lib/validations'
import type { UserWithPosts, UserPublicData } from '../types/prisma'

export class UserService {
  async create(data: CreateUserInput): Promise<UserPublicData> {
    const validatedData = createUserSchema.parse(data)
    
    const user = await prisma.user.create({
      data: validatedData,
      select: {
        id: true,
        firstName: true,
        lastName: true,
        email: true,
        createdAt: true,
      }
    })
    
    return user
  }
  
  async findById(id: number): Promise<UserWithPosts | null> {
    return prisma.user.findUnique({
      where: { id },
      include: {
        posts: {
          where: { published: true },
          orderBy: { createdAt: 'desc' }
        }
      }
    })
  }
  
  async findMany(filters: UserFilters = {}) {
    const where: any = {}
    
    if (filters.email) {
      where.email = { contains: filters.email, mode: 'insensitive' }
    }
    
    if (filters.ageMin || filters.ageMax) {
      where.age = {}
      if (filters.ageMin) where.age.gte = filters.ageMin
      if (filters.ageMax) where.age.lte = filters.ageMax
    }
    
    if (filters.createdAfter) {
      where.createdAt = { gte: filters.createdAfter }
    }
    
    return prisma.user.findMany({
      where,
      select: {
        id: true,
        firstName: true,
        lastName: true,
        email: true,
        createdAt: true,
      },
      orderBy: { createdAt: 'desc' }
    })
  }
  
  async update(id: number, data: UpdateUserInput): Promise<UserPublicData> {
    const validatedData = updateUserSchema.parse(data)
    
    return prisma.user.update({
      where: { id },
      data: validatedData,
      select: {
        id: true,
        firstName: true,
        lastName: true,
        email: true,
        createdAt: true,
      }
    })
  }
  
  async delete(id: number): Promise<void> {
    await prisma.user.delete({
      where: { id }
    })
  }
  
  async softDelete(id: number): Promise<void> {
    await prisma.user.update({
      where: { id },
      data: { deletedAt: new Date() }
    })
  }
}

export const userService = new UserService()
```

## Error Handling

### Tratamento de Erros Específicos

```typescript
// src/utils/errors.ts
import { 
  PrismaClientKnownRequestError, 
  PrismaClientUnknownRequestError,
  PrismaClientValidationError 
} from '@prisma/client/runtime/library'

export class DatabaseError extends Error {
  constructor(
    message: string,
    public code?: string,
    public meta?: any
  ) {
    super(message)
    this.name = 'DatabaseError'
  }
}

export function handlePrismaError(error: unknown): never {
  if (error instanceof PrismaClientKnownRequestError) {
    switch (error.code) {
      case 'P2002':
        throw new DatabaseError(
          'Registro duplicado. Este valor já existe.',
          'DUPLICATE_ENTRY',
          error.meta
        )
      case 'P2025':
        throw new DatabaseError(
          'Registro não encontrado.',
          'NOT_FOUND',
          error.meta
        )
      case 'P2003':
        throw new DatabaseError(
          'Violação de chave estrangeira.',
          'FOREIGN_KEY_VIOLATION',
          error.meta
        )
      case 'P2014':
        throw new DatabaseError(
          'Violação de relacionamento obrigatório.',
          'REQUIRED_RELATION_VIOLATION',
          error.meta
        )
      default:
        throw new DatabaseError(
          `Erro de banco de dados: ${error.message}`,
          error.code,
          error.meta
        )
    }
  }
  
  if (error instanceof PrismaClientValidationError) {
    throw new DatabaseError(
      'Dados inválidos fornecidos.',
      'VALIDATION_ERROR'
    )
  }
  
  if (error instanceof PrismaClientUnknownRequestError) {
    throw new DatabaseError(
      'Erro desconhecido no banco de dados.',
      'UNKNOWN_ERROR'
    )
  }
  
  throw error
}

// Wrapper para operações do Prisma
export async function safeExecute<T>(
  operation: () => Promise<T>
): Promise<T> {
  try {
    return await operation()
  } catch (error) {
    handlePrismaError(error)
  }
}
```

### Service com Error Handling

```typescript
// src/services/user.service.ts (versão com error handling)
import { safeExecute, DatabaseError } from '../utils/errors'

export class UserService {
  async create(data: CreateUserInput): Promise<UserPublicData> {
    return safeExecute(async () => {
      const validatedData = createUserSchema.parse(data)
      
      return prisma.user.create({
        data: validatedData,
        select: {
          id: true,
          firstName: true,
          lastName: true,
          email: true,
          createdAt: true,
        }
      })
    })
  }
  
  async findByIdOrThrow(id: number): Promise<UserWithPosts> {
    const user = await this.findById(id)
    
    if (!user) {
      throw new DatabaseError(
        `Usuário com ID ${id} não encontrado.`,
        'USER_NOT_FOUND'
      )
    }
    
    return user
  }
}
```

## Performance e Otimização

### Connection Pooling

```typescript
// src/lib/prisma.ts
export const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL,
    },
  },
  log: ['query', 'error', 'warn'],
  errorFormat: 'pretty',
})

// Configuração de pool no DATABASE_URL
// postgresql://user:pass@host:5432/db?connection_limit=20&pool_timeout=20
```

### Query Optimization

```typescript
// ✅ Bom: Select apenas campos necessários
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true,
  }
})

// ✅ Bom: Usar include com filtros
const userWithRecentPosts = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: {
      take: 10,
      orderBy: { createdAt: 'desc' },
      where: { published: true }
    }
  }
})

// ❌ Ruim: Buscar tudo sem necessidade
const users = await prisma.user.findMany({
  include: {
    posts: {
      include: {
        comments: {
          include: {
            author: true
          }
        }
      }
    }
  }
})
```

### Batch Operations

```typescript
// ✅ Bom: Usar createMany para múltiplos inserts
const users = await prisma.user.createMany({
  data: [
    { email: 'user1@example.com', name: 'User 1' },
    { email: 'user2@example.com', name: 'User 2' },
    { email: 'user3@example.com', name: 'User 3' },
  ],
  skipDuplicates: true
})

// ✅ Bom: Usar transações para operações relacionadas
const result = await prisma.$transaction([
  prisma.user.create({ data: userData }),
  prisma.profile.create({ data: profileData }),
  prisma.post.createMany({ data: postsData })
])
```

## Middleware e Hooks

### Middleware para Auditoria

```typescript
// src/lib/middleware.ts
import { prisma } from './prisma'

// Middleware para soft delete
prisma.$use(async (params, next) => {
  // Interceptar delete e transformar em update
  if (params.action === 'delete') {
    params.action = 'update'
    params.args['data'] = { deletedAt: new Date() }
  }
  
  if (params.action === 'deleteMany') {
    params.action = 'updateMany'
    if (params.args.data != undefined) {
      params.args.data['deletedAt'] = new Date()
    } else {
      params.args['data'] = { deletedAt: new Date() }
    }
  }
  
  return next(params)
})

// Middleware para filtrar registros deletados
prisma.$use(async (params, next) => {
  if (params.action === 'findUnique' || params.action === 'findFirst') {
    params.args.where['deletedAt'] = null
  }
  
  if (params.action === 'findMany') {
    if (params.args.where) {
      if (params.args.where.deletedAt == undefined) {
        params.args.where['deletedAt'] = null
      }
    } else {
      params.args['where'] = { deletedAt: null }
    }
  }
  
  return next(params)
})

// Middleware para logging
prisma.$use(async (params, next) => {
  const before = Date.now()
  const result = await next(params)
  const after = Date.now()
  
  console.log(
    `Query ${params.model}.${params.action} took ${after - before}ms`
  )
  
  return result
})
```

## Testing

### Setup de Testes

```typescript
// tests/setup.ts
import { PrismaClient } from '@prisma/client'
import { execSync } from 'child_process'
import { randomBytes } from 'crypto'

const generateDatabaseURL = (schema: string) => {
  if (!process.env.DATABASE_URL) {
    throw new Error('DATABASE_URL não definida')
  }
  
  const url = new URL(process.env.DATABASE_URL)
  url.searchParams.set('schema', schema)
  return url.toString()
}

const schemaId = randomBytes(16).toString('hex')
const databaseUrl = generateDatabaseURL(schemaId)
process.env.DATABASE_URL = databaseUrl

const prisma = new PrismaClient()

beforeAll(async () => {
  execSync('npx prisma migrate deploy', {
    env: {
      ...process.env,
      DATABASE_URL: databaseUrl,
    },
  })
})

afterAll(async () => {
  await prisma.$executeRawUnsafe(
    `DROP SCHEMA IF EXISTS "${schemaId}" CASCADE`
  )
  await prisma.$disconnect()
})

export { prisma }
```

### Testes de Integração

```typescript
// tests/integration/user.test.ts
import { prisma } from '../setup'
import { userService } from '../../src/services/user.service'

describe('UserService', () => {
  beforeEach(async () => {
    await prisma.user.deleteMany()
  })
  
  describe('create', () => {
    it('deve criar um usuário válido', async () => {
      const userData = {
        email: 'test@example.com',
        firstName: 'Test',
        lastName: 'User'
      }
      
      const user = await userService.create(userData)
      
      expect(user).toMatchObject({
        email: userData.email,
        firstName: userData.firstName,
        lastName: userData.lastName
      })
      expect(user.id).toBeDefined()
      expect(user.createdAt).toBeDefined()
    })
    
    it('deve falhar com email duplicado', async () => {
      const userData = {
        email: 'test@example.com',
        firstName: 'Test',
        lastName: 'User'
      }
      
      await userService.create(userData)
      
      await expect(userService.create(userData))
        .rejects
        .toThrow('Registro duplicado')
    })
  })
})
```

## Deployment e Produção

### Variáveis de Ambiente

```bash
# .env.production
DATABASE_URL="postgresql://user:pass@host:5432/prod_db?connection_limit=20&pool_timeout=20&sslmode=require"
NODE_ENV="production"
LOG_LEVEL="error"
```

### Scripts de Deploy

```json
{
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "db:deploy": "prisma migrate deploy",
    "db:seed": "prisma db seed",
    "postinstall": "prisma generate"
  }
}
```

### Health Check

```typescript
// src/utils/health.ts
import { prisma } from '../lib/prisma'

export async function checkDatabaseHealth(): Promise<boolean> {
  try {
    await prisma.$queryRaw`SELECT 1`
    return true
  } catch (error) {
    console.error('Database health check failed:', error)
    return false
  }
}
```

---

[Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>