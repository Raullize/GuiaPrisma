<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# 🧪 Testing: Testando Aplicações com Prisma

## Configuração do Ambiente de Testes

### Configuração Básica

```typescript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  testMatch: ['**/__tests__/**/*.ts', '**/?(*.)+(spec|test).ts'],
  transform: {
    '^.+\.ts$': 'ts-jest',
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/types/**',
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  setupFilesAfterEnv: ['<rootDir>/tests/setup.ts'],
  testTimeout: 30000,
}
```

```typescript
// tests/setup.ts
import { PrismaClient } from '@prisma/client'
import { execSync } from 'child_process'
import { join } from 'path'
import { URL } from 'url'

const prisma = new PrismaClient()

// Função para gerar URL de teste única
function generateTestDatabaseUrl(): string {
  const testId = Math.random().toString(36).substring(7)
  const baseUrl = process.env.DATABASE_URL || 'postgresql://postgres:password@localhost:5432/test'
  const url = new URL(baseUrl)
  url.pathname = `/test_${testId}`
  return url.toString()
}

// Setup global antes de todos os testes
beforeAll(async () => {
  // Configurar URL de teste
  process.env.DATABASE_URL = generateTestDatabaseUrl()
  
  // Executar migrations
  execSync('npx prisma migrate deploy', {
    env: {
      ...process.env,
      DATABASE_URL: process.env.DATABASE_URL,
    },
  })
})

// Cleanup após todos os testes
afterAll(async () => {
  await prisma.$disconnect()
})

// Limpar dados entre testes
afterEach(async () => {
  // Ordem importante: deletar em ordem reversa das dependências
  const tablenames = await prisma.$queryRaw<Array<{ tablename: string }>>(
    `SELECT tablename FROM pg_tables WHERE schemaname='public'`
  )

  const tables = tablenames
    .map(({ tablename }) => tablename)
    .filter((name) => name !== '_prisma_migrations')
    .map((name) => `"public"."${name}"`)
    .join(', ')

  try {
    await prisma.$executeRawUnsafe(`TRUNCATE TABLE ${tables} CASCADE;`)
  } catch (error) {
    console.log({ error })
  }
})

export { prisma }
```

### Configuração com Docker

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  test-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: test
    ports:
      - "5433:5432"
    tmpfs:
      - /var/lib/postgresql/data
    command: [
      "postgres",
      "-c", "fsync=off",
      "-c", "synchronous_commit=off",
      "-c", "full_page_writes=off",
      "-c", "checkpoint_segments=32",
      "-c", "checkpoint_completion_target=0.9",
      "-c", "wal_buffers=16MB",
      "-c", "shared_buffers=256MB"
    ]

  test-redis:
    image: redis:7-alpine
    ports:
      - "6380:6379"
    tmpfs:
      - /data
```

```bash
#!/bin/bash
# scripts/test-setup.sh

set -e

echo "🐳 Starting test containers..."
docker-compose -f docker-compose.test.yml up -d

echo "⏳ Waiting for database..."
until docker-compose -f docker-compose.test.yml exec -T test-db pg_isready -U postgres; do
  sleep 1
done

echo "🔄 Running migrations..."
DATABASE_URL="postgresql://postgres:password@localhost:5433/test" npx prisma migrate deploy

echo "✅ Test environment ready!"
```

## Testes Unitários

### Testando Modelos e Validações

```typescript
// tests/unit/user.test.ts
import { PrismaClient } from '@prisma/client'
import { prisma } from '../setup'

describe('User Model', () => {
  describe('Creation', () => {
    it('should create a user with valid data', async () => {
      const userData = {
        email: 'test@example.com',
        name: 'Test User',
        password: 'hashedpassword123',
      }

      const user = await prisma.user.create({
        data: userData,
      })

      expect(user).toMatchObject({
        id: expect.any(String),
        email: userData.email,
        name: userData.name,
        createdAt: expect.any(Date),
        updatedAt: expect.any(Date),
      })
      expect(user.password).toBe(userData.password)
    })

    it('should fail to create user with duplicate email', async () => {
      const userData = {
        email: 'duplicate@example.com',
        name: 'Test User',
        password: 'hashedpassword123',
      }

      await prisma.user.create({ data: userData })

      await expect(
        prisma.user.create({ data: userData })
      ).rejects.toThrow(/Unique constraint failed/)
    })

    it('should fail to create user without required fields', async () => {
      await expect(
        prisma.user.create({
          data: {
            name: 'Test User',
            // email missing
          } as any,
        })
      ).rejects.toThrow()
    })
  })

  describe('Relationships', () => {
    it('should create user with posts', async () => {
      const user = await prisma.user.create({
        data: {
          email: 'author@example.com',
          name: 'Author',
          password: 'password123',
          posts: {
            create: [
              {
                title: 'First Post',
                content: 'Content of first post',
                published: true,
              },
              {
                title: 'Second Post',
                content: 'Content of second post',
                published: false,
              },
            ],
          },
        },
        include: {
          posts: true,
        },
      })

      expect(user.posts).toHaveLength(2)
      expect(user.posts[0]).toMatchObject({
        title: 'First Post',
        published: true,
        authorId: user.id,
      })
    })

    it('should fetch user with posts count', async () => {
      const user = await prisma.user.create({
        data: {
          email: 'counter@example.com',
          name: 'Counter User',
          password: 'password123',
          posts: {
            create: Array.from({ length: 5 }, (_, i) => ({
              title: `Post ${i + 1}`,
              content: `Content ${i + 1}`,
              published: i % 2 === 0,
            })),
          },
        },
      })

      const userWithCount = await prisma.user.findUnique({
        where: { id: user.id },
        include: {
          _count: {
            select: {
              posts: true,
            },
          },
        },
      })

      expect(userWithCount?._count.posts).toBe(5)
    })
  })
})
```

### Testando Serviços

```typescript
// tests/unit/user.service.test.ts
import { UserService } from '../../src/services/user.service'
import { prisma } from '../setup'
import bcrypt from 'bcrypt'

// Mock do bcrypt
jest.mock('bcrypt')
const mockedBcrypt = bcrypt as jest.Mocked<typeof bcrypt>

describe('UserService', () => {
  let userService: UserService

  beforeEach(() => {
    userService = new UserService(prisma)
    jest.clearAllMocks()
  })

  describe('createUser', () => {
    it('should create user with hashed password', async () => {
      const userData = {
        email: 'test@example.com',
        name: 'Test User',
        password: 'plainpassword',
      }

      const hashedPassword = 'hashedpassword123'
      mockedBcrypt.hash.mockResolvedValue(hashedPassword as never)

      const user = await userService.createUser(userData)

      expect(bcrypt.hash).toHaveBeenCalledWith(userData.password, 10)
      expect(user).toMatchObject({
        email: userData.email,
        name: userData.name,
      })
      expect(user.password).toBe(hashedPassword)
    })

    it('should throw error for duplicate email', async () => {
      const userData = {
        email: 'duplicate@example.com',
        name: 'Test User',
        password: 'plainpassword',
      }

      mockedBcrypt.hash.mockResolvedValue('hashedpassword' as never)
      
      // Criar primeiro usuário
      await userService.createUser(userData)

      // Tentar criar segundo usuário com mesmo email
      await expect(
        userService.createUser(userData)
      ).rejects.toThrow('Email already exists')
    })
  })

  describe('authenticateUser', () => {
    it('should authenticate user with correct credentials', async () => {
      const userData = {
        email: 'auth@example.com',
        name: 'Auth User',
        password: 'plainpassword',
      }

      mockedBcrypt.hash.mockResolvedValue('hashedpassword' as never)
      mockedBcrypt.compare.mockResolvedValue(true as never)

      const user = await userService.createUser(userData)
      const authenticatedUser = await userService.authenticateUser(
        userData.email,
        userData.password
      )

      expect(bcrypt.compare).toHaveBeenCalledWith(
        userData.password,
        'hashedpassword'
      )
      expect(authenticatedUser).toMatchObject({
        id: user.id,
        email: user.email,
        name: user.name,
      })
    })

    it('should return null for invalid credentials', async () => {
      mockedBcrypt.compare.mockResolvedValue(false as never)

      const result = await userService.authenticateUser(
        'nonexistent@example.com',
        'wrongpassword'
      )

      expect(result).toBeNull()
    })
  })

  describe('getUserPosts', () => {
    it('should return user posts with pagination', async () => {
      const user = await prisma.user.create({
        data: {
          email: 'posts@example.com',
          name: 'Posts User',
          password: 'password123',
          posts: {
            create: Array.from({ length: 15 }, (_, i) => ({
              title: `Post ${i + 1}`,
              content: `Content ${i + 1}`,
              published: true,
            })),
          },
        },
      })

      const result = await userService.getUserPosts(user.id, {
        page: 1,
        limit: 10,
      })

      expect(result.posts).toHaveLength(10)
      expect(result.total).toBe(15)
      expect(result.page).toBe(1)
      expect(result.totalPages).toBe(2)
    })
  })
})
```

## Testes de Integração

### Testando APIs

```typescript
// tests/integration/auth.test.ts
import request from 'supertest'
import { app } from '../../src/app'
import { prisma } from '../setup'
import jwt from 'jsonwebtoken'

describe('Auth API', () => {
  describe('POST /api/auth/register', () => {
    it('should register a new user', async () => {
      const userData = {
        email: 'register@example.com',
        name: 'Register User',
        password: 'password123',
      }

      const response = await request(app)
        .post('/api/auth/register')
        .send(userData)
        .expect(201)

      expect(response.body).toMatchObject({
        user: {
          id: expect.any(String),
          email: userData.email,
          name: userData.name,
        },
        token: expect.any(String),
      })

      // Verificar se usuário foi criado no banco
      const user = await prisma.user.findUnique({
        where: { email: userData.email },
      })
      expect(user).toBeTruthy()
    })

    it('should return 400 for invalid email', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({
          email: 'invalid-email',
          name: 'Test User',
          password: 'password123',
        })
        .expect(400)

      expect(response.body.error).toContain('Invalid email')
    })

    it('should return 409 for duplicate email', async () => {
      const userData = {
        email: 'duplicate@example.com',
        name: 'Duplicate User',
        password: 'password123',
      }

      // Primeiro registro
      await request(app)
        .post('/api/auth/register')
        .send(userData)
        .expect(201)

      // Segundo registro (deve falhar)
      const response = await request(app)
        .post('/api/auth/register')
        .send(userData)
        .expect(409)

      expect(response.body.error).toContain('Email already exists')
    })
  })

  describe('POST /api/auth/login', () => {
    beforeEach(async () => {
      // Criar usuário para testes de login
      await request(app)
        .post('/api/auth/register')
        .send({
          email: 'login@example.com',
          name: 'Login User',
          password: 'password123',
        })
    })

    it('should login with valid credentials', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'login@example.com',
          password: 'password123',
        })
        .expect(200)

      expect(response.body).toMatchObject({
        user: {
          email: 'login@example.com',
          name: 'Login User',
        },
        token: expect.any(String),
      })

      // Verificar se token é válido
      const decoded = jwt.verify(
        response.body.token,
        process.env.JWT_SECRET || 'test-secret'
      )
      expect(decoded).toMatchObject({
        userId: expect.any(String),
      })
    })

    it('should return 401 for invalid credentials', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'login@example.com',
          password: 'wrongpassword',
        })
        .expect(401)

      expect(response.body.error).toContain('Invalid credentials')
    })
  })
})
```

### Testando Middleware de Autenticação

```typescript
// tests/integration/auth.middleware.test.ts
import request from 'supertest'
import { app } from '../../src/app'
import jwt from 'jsonwebtoken'
import { prisma } from '../setup'

describe('Auth Middleware', () => {
  let authToken: string
  let userId: string

  beforeEach(async () => {
    // Criar usuário e gerar token
    const user = await prisma.user.create({
      data: {
        email: 'middleware@example.com',
        name: 'Middleware User',
        password: 'hashedpassword',
      },
    })

    userId = user.id
    authToken = jwt.sign(
      { userId: user.id },
      process.env.JWT_SECRET || 'test-secret',
      { expiresIn: '1h' }
    )
  })

  describe('Protected Routes', () => {
    it('should allow access with valid token', async () => {
      const response = await request(app)
        .get('/api/profile')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200)

      expect(response.body.user.id).toBe(userId)
    })

    it('should deny access without token', async () => {
      const response = await request(app)
        .get('/api/profile')
        .expect(401)

      expect(response.body.error).toContain('No token provided')
    })

    it('should deny access with invalid token', async () => {
      const response = await request(app)
        .get('/api/profile')
        .set('Authorization', 'Bearer invalid-token')
        .expect(401)

      expect(response.body.error).toContain('Invalid token')
    })

    it('should deny access with expired token', async () => {
      const expiredToken = jwt.sign(
        { userId },
        process.env.JWT_SECRET || 'test-secret',
        { expiresIn: '-1h' } // Token expirado
      )

      const response = await request(app)
        .get('/api/profile')
        .set('Authorization', `Bearer ${expiredToken}`)
        .expect(401)

      expect(response.body.error).toContain('Token expired')
    })
  })
})
```

## Testes de Performance

### Testando Queries Complexas

```typescript
// tests/performance/queries.test.ts
import { prisma } from '../setup'
import { performance } from 'perf_hooks'

describe('Query Performance', () => {
  beforeEach(async () => {
    // Criar dados de teste em massa
    const users = Array.from({ length: 100 }, (_, i) => ({
      email: `user${i}@example.com`,
      name: `User ${i}`,
      password: 'hashedpassword',
    }))

    await prisma.user.createMany({ data: users })

    // Criar posts para cada usuário
    const createdUsers = await prisma.user.findMany()
    
    for (const user of createdUsers) {
      const posts = Array.from({ length: 10 }, (_, i) => ({
        title: `Post ${i} by ${user.name}`,
        content: `Content of post ${i}`,
        published: i % 2 === 0,
        authorId: user.id,
      }))
      
      await prisma.post.createMany({ data: posts })
    }
  })

  it('should fetch users with posts efficiently', async () => {
    const start = performance.now()

    const users = await prisma.user.findMany({
      include: {
        posts: {
          where: { published: true },
          take: 5,
          orderBy: { createdAt: 'desc' },
        },
        _count: {
          select: { posts: true },
        },
      },
      take: 20,
    })

    const end = performance.now()
    const duration = end - start

    expect(users).toHaveLength(20)
    expect(duration).toBeLessThan(1000) // Menos de 1 segundo
    
    console.log(`Query took ${duration.toFixed(2)}ms`)
  })

  it('should handle pagination efficiently', async () => {
    const pageSize = 10
    const totalPages = 5
    const durations: number[] = []

    for (let page = 1; page <= totalPages; page++) {
      const start = performance.now()

      const users = await prisma.user.findMany({
        skip: (page - 1) * pageSize,
        take: pageSize,
        include: {
          _count: {
            select: { posts: true },
          },
        },
        orderBy: { createdAt: 'desc' },
      })

      const end = performance.now()
      const duration = end - start
      durations.push(duration)

      expect(users).toHaveLength(pageSize)
    }

    const avgDuration = durations.reduce((a, b) => a + b, 0) / durations.length
    expect(avgDuration).toBeLessThan(500) // Média menor que 500ms
    
    console.log(`Average pagination query: ${avgDuration.toFixed(2)}ms`)
  })

  it('should detect N+1 query problems', async () => {
    // Ativar logging de queries
    const queries: string[] = []
    
    const originalQuery = prisma.$use
    prisma.$use(async (params, next) => {
      queries.push(`${params.model}.${params.action}`)
      return next(params)
    })

    // Buscar usuários e seus posts (potencial N+1)
    const users = await prisma.user.findMany({ take: 5 })
    
    for (const user of users) {
      await prisma.post.findMany({
        where: { authorId: user.id },
      })
    }

    // Deve ter 1 query para users + 5 queries para posts = 6 total
    expect(queries.length).toBe(6)
    
    // Solução otimizada
    queries.length = 0
    
    const optimizedUsers = await prisma.user.findMany({
      take: 5,
      include: {
        posts: true,
      },
    })

    // Deve ter apenas 1 query
    expect(queries.length).toBe(1)
    expect(optimizedUsers).toHaveLength(5)
  })
})
```

## Testes com Transações

```typescript
// tests/integration/transactions.test.ts
import { prisma } from '../setup'

describe('Transaction Tests', () => {
  it('should rollback on error', async () => {
    const initialUserCount = await prisma.user.count()

    await expect(
      prisma.$transaction(async (tx) => {
        // Criar usuário
        await tx.user.create({
          data: {
            email: 'transaction@example.com',
            name: 'Transaction User',
            password: 'password123',
          },
        })

        // Criar post com dados inválidos (vai falhar)
        await tx.post.create({
          data: {
            title: 'Test Post',
            content: 'Test Content',
            authorId: 'invalid-id', // ID inválido
          },
        })
      })
    ).rejects.toThrow()

    // Verificar que nenhum usuário foi criado
    const finalUserCount = await prisma.user.count()
    expect(finalUserCount).toBe(initialUserCount)
  })

  it('should commit successful transaction', async () => {
    const result = await prisma.$transaction(async (tx) => {
      const user = await tx.user.create({
        data: {
          email: 'success@example.com',
          name: 'Success User',
          password: 'password123',
        },
      })

      const post = await tx.post.create({
        data: {
          title: 'Success Post',
          content: 'Success Content',
          authorId: user.id,
        },
      })

      return { user, post }
    })

    expect(result.user.email).toBe('success@example.com')
    expect(result.post.authorId).toBe(result.user.id)

    // Verificar que dados foram persistidos
    const user = await prisma.user.findUnique({
      where: { id: result.user.id },
      include: { posts: true },
    })

    expect(user?.posts).toHaveLength(1)
  })

  it('should handle concurrent transactions', async () => {
    const user = await prisma.user.create({
      data: {
        email: 'concurrent@example.com',
        name: 'Concurrent User',
        password: 'password123',
      },
    })

    // Executar múltiplas transações concorrentes
    const promises = Array.from({ length: 5 }, (_, i) =>
      prisma.$transaction(async (tx) => {
        return tx.post.create({
          data: {
            title: `Concurrent Post ${i}`,
            content: `Content ${i}`,
            authorId: user.id,
          },
        })
      })
    )

    const posts = await Promise.all(promises)
    expect(posts).toHaveLength(5)

    // Verificar que todos os posts foram criados
    const userWithPosts = await prisma.user.findUnique({
      where: { id: user.id },
      include: { posts: true },
    })

    expect(userWithPosts?.posts).toHaveLength(5)
  })
})
```

## Mocking e Test Doubles

### Mock do Prisma Client

```typescript
// tests/mocks/prisma.mock.ts
import { PrismaClient } from '@prisma/client'
import { mockDeep, mockReset, DeepMockProxy } from 'jest-mock-extended'

export const prismaMock = mockDeep<PrismaClient>()

beforeEach(() => {
  mockReset(prismaMock)
})

export type MockPrisma = DeepMockProxy<PrismaClient>
```

```typescript
// tests/unit/user.service.mock.test.ts
import { UserService } from '../../src/services/user.service'
import { prismaMock } from '../mocks/prisma.mock'

// Mock do módulo Prisma
jest.mock('../../src/lib/prisma', () => ({
  prisma: prismaMock,
}))

describe('UserService with Mocks', () => {
  let userService: UserService

  beforeEach(() => {
    userService = new UserService(prismaMock)
  })

  it('should find user by email', async () => {
    const mockUser = {
      id: '1',
      email: 'test@example.com',
      name: 'Test User',
      password: 'hashedpassword',
      createdAt: new Date(),
      updatedAt: new Date(),
    }

    prismaMock.user.findUnique.mockResolvedValue(mockUser)

    const user = await userService.findByEmail('test@example.com')

    expect(user).toEqual(mockUser)
    expect(prismaMock.user.findUnique).toHaveBeenCalledWith({
      where: { email: 'test@example.com' },
    })
  })

  it('should return null for non-existent user', async () => {
    prismaMock.user.findUnique.mockResolvedValue(null)

    const user = await userService.findByEmail('nonexistent@example.com')

    expect(user).toBeNull()
  })

  it('should create user successfully', async () => {
    const userData = {
      email: 'new@example.com',
      name: 'New User',
      password: 'hashedpassword',
    }

    const mockCreatedUser = {
      id: '2',
      ...userData,
      createdAt: new Date(),
      updatedAt: new Date(),
    }

    prismaMock.user.create.mockResolvedValue(mockCreatedUser)

    const user = await userService.createUser(userData)

    expect(user).toEqual(mockCreatedUser)
    expect(prismaMock.user.create).toHaveBeenCalledWith({
      data: userData,
    })
  })
})
```

## Testes E2E (End-to-End)

### Configuração com Playwright

```typescript
// tests/e2e/setup.ts
import { test as base, expect } from '@playwright/test'
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

export const test = base.extend({
  // Fixture para limpar dados antes de cada teste
  cleanDatabase: async ({}, use) => {
    await use(async () => {
      // Limpar dados de teste
      await prisma.post.deleteMany()
      await prisma.user.deleteMany()
    })
  },

  // Fixture para criar usuário de teste
  testUser: async ({ cleanDatabase }, use) => {
    await cleanDatabase()
    
    const user = await prisma.user.create({
      data: {
        email: 'e2e@example.com',
        name: 'E2E User',
        password: 'hashedpassword123',
      },
    })

    await use(user)
  },
})

export { expect }
```

```typescript
// tests/e2e/auth.spec.ts
import { test, expect } from './setup'

test.describe('Authentication Flow', () => {
  test('should register and login user', async ({ page }) => {
    // Ir para página de registro
    await page.goto('/register')

    // Preencher formulário
    await page.fill('[data-testid="email"]', 'e2e-new@example.com')
    await page.fill('[data-testid="name"]', 'E2E New User')
    await page.fill('[data-testid="password"]', 'password123')
    await page.fill('[data-testid="confirmPassword"]', 'password123')

    // Submeter formulário
    await page.click('[data-testid="submit"]')

    // Verificar redirecionamento
    await expect(page).toHaveURL('/dashboard')
    await expect(page.locator('[data-testid="welcome"]')).toContainText('Welcome, E2E New User')

    // Logout
    await page.click('[data-testid="logout"]')
    await expect(page).toHaveURL('/login')

    // Login novamente
    await page.fill('[data-testid="email"]', 'e2e-new@example.com')
    await page.fill('[data-testid="password"]', 'password123')
    await page.click('[data-testid="submit"]')

    // Verificar login bem-sucedido
    await expect(page).toHaveURL('/dashboard')
  })

  test('should show error for invalid credentials', async ({ page }) => {
    await page.goto('/login')

    await page.fill('[data-testid="email"]', 'invalid@example.com')
    await page.fill('[data-testid="password"]', 'wrongpassword')
    await page.click('[data-testid="submit"]')

    await expect(page.locator('[data-testid="error"]')).toContainText('Invalid credentials')
    await expect(page).toHaveURL('/login')
  })
})

test.describe('Protected Routes', () => {
  test('should redirect to login when not authenticated', async ({ page }) => {
    await page.goto('/dashboard')
    await expect(page).toHaveURL('/login')
  })

  test('should access dashboard when authenticated', async ({ page, testUser }) => {
    // Simular login (pode usar API ou localStorage)
    await page.goto('/login')
    await page.fill('[data-testid="email"]', testUser.email)
    await page.fill('[data-testid="password"]', 'password123')
    await page.click('[data-testid="submit"]')

    await expect(page).toHaveURL('/dashboard')
    await expect(page.locator('[data-testid="user-name"]')).toContainText(testUser.name)
  })
})
```

## Scripts de Teste

### Package.json Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:unit": "jest tests/unit",
    "test:integration": "jest tests/integration",
    "test:e2e": "playwright test",
    "test:db:setup": "./scripts/test-setup.sh",
    "test:db:teardown": "docker-compose -f docker-compose.test.yml down -v",
    "test:ci": "npm run test:db:setup && npm run test && npm run test:e2e && npm run test:db:teardown"
  }
}
```

### CI/CD Pipeline

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: password
          POSTGRES_DB: test
        options: >
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7
        options: >
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Generate Prisma Client
        run: npx prisma generate

      - name: Run migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test

      - name: Run unit tests
        run: npm run test:unit
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test

      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test
          REDIS_URL: redis://localhost:6379

      - name: Install Playwright
        run: npx playwright install --with-deps

      - name: Run E2E tests
        run: npm run test:e2e
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test

      - name: Upload coverage reports
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info

      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: test-results
          path: |
            test-results/
            playwright-report/
```

## Boas Práticas de Testing

### 1. Estrutura de Testes
- **Arrange**: Preparar dados e estado
- **Act**: Executar a ação sendo testada
- **Assert**: Verificar o resultado

### 2. Isolamento de Testes
- Cada teste deve ser independente
- Limpar dados entre testes
- Usar transações quando possível

### 3. Dados de Teste
- Usar factories para criar dados consistentes
- Evitar dados hardcoded
- Limpar dados após cada teste

### 4. Performance
- Usar banco em memória para testes unitários
- Paralelizar testes quando possível
- Otimizar queries de setup

### 5. Cobertura
- Mirar em 80%+ de cobertura
- Focar em código crítico
- Testar casos edge

---

[🔙 Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>