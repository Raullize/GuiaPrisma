<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# Integração com Frameworks

## Next.js

### Setup Inicial

```bash
# Criar projeto Next.js
npx create-next-app@latest my-app --typescript --tailwind --eslint
cd my-app

# Instalar Prisma
npm install prisma @prisma/client
npm install -D prisma

# Inicializar Prisma
npx prisma init
```

### Configuração do Cliente

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma = globalForPrisma.prisma ??
  new PrismaClient({
    log: ['query'],
  })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

### API Routes

```typescript
// pages/api/users/index.ts (Pages Router)
import type { NextApiRequest, NextApiResponse } from 'next'
import { prisma } from '../../../lib/prisma'

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method === 'GET') {
    try {
      const users = await prisma.user.findMany({
        select: {
          id: true,
          email: true,
          name: true,
          createdAt: true,
        },
      })
      res.status(200).json(users)
    } catch (error) {
      res.status(500).json({ error: 'Erro ao buscar usuários' })
    }
  } else if (req.method === 'POST') {
    try {
      const { email, name } = req.body
      const user = await prisma.user.create({
        data: { email, name },
      })
      res.status(201).json(user)
    } catch (error) {
      res.status(500).json({ error: 'Erro ao criar usuário' })
    }
  } else {
    res.setHeader('Allow', ['GET', 'POST'])
    res.status(405).end(`Method ${req.method} Not Allowed`)
  }
}
```

### App Router (Next.js 13+)

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'
import { z } from 'zod'

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
})

export async function GET() {
  try {
    const users = await prisma.user.findMany({
      select: {
        id: true,
        email: true,
        name: true,
        createdAt: true,
      },
    })
    return NextResponse.json(users)
  } catch (error) {
    return NextResponse.json(
      { error: 'Erro ao buscar usuários' },
      { status: 500 }
    )
  }
}

export async function POST(request: NextRequest) {
  try {
    const body = await request.json()
    const { email, name } = createUserSchema.parse(body)
    
    const user = await prisma.user.create({
      data: { email, name },
    })
    
    return NextResponse.json(user, { status: 201 })
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Dados inválidos', details: error.errors },
        { status: 400 }
      )
    }
    return NextResponse.json(
      { error: 'Erro ao criar usuário' },
      { status: 500 }
    )
  }
}
```

### Server Components

```typescript
// app/users/page.tsx
import { prisma } from '@/lib/prisma'

interface User {
  id: number
  email: string
  name: string
  createdAt: Date
}

async function getUsers(): Promise<User[]> {
  return prisma.user.findMany({
    select: {
      id: true,
      email: true,
      name: true,
      createdAt: true,
    },
    orderBy: {
      createdAt: 'desc',
    },
  })
}

export default async function UsersPage() {
  const users = await getUsers()

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-2xl font-bold mb-6">Usuários</h1>
      <div className="grid gap-4">
        {users.map((user) => (
          <div key={user.id} className="border p-4 rounded-lg">
            <h2 className="font-semibold">{user.name}</h2>
            <p className="text-gray-600">{user.email}</p>
            <p className="text-sm text-gray-500">
              Criado em: {user.createdAt.toLocaleDateString()}
            </p>
          </div>
        ))}
      </div>
    </div>
  )
}
```

### Middleware para Database

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // Adicionar headers de CORS para API routes
  if (request.nextUrl.pathname.startsWith('/api/')) {
    const response = NextResponse.next()
    response.headers.set('Access-Control-Allow-Origin', '*')
    response.headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS')
    response.headers.set('Access-Control-Allow-Headers', 'Content-Type, Authorization')
    return response
  }
}

export const config = {
  matcher: '/api/:path*',
}
```

## Express.js

### Setup Básico

```bash
# Criar projeto
mkdir express-prisma-app
cd express-prisma-app
npm init -y

# Instalar dependências
npm install express cors helmet morgan
npm install -D @types/express @types/cors @types/morgan typescript ts-node nodemon
npm install prisma @prisma/client
npm install zod

# Inicializar Prisma
npx prisma init
```

### Configuração Principal

```typescript
// src/app.ts
import express from 'express'
import cors from 'cors'
import helmet from 'helmet'
import morgan from 'morgan'
import { userRoutes } from './routes/users'
import { postRoutes } from './routes/posts'
import { errorHandler } from './middleware/errorHandler'
import { prisma } from './lib/prisma'

const app = express()

// Middleware
app.use(helmet())
app.use(cors())
app.use(morgan('combined'))
app.use(express.json())
app.use(express.urlencoded({ extended: true }))

// Routes
app.use('/api/users', userRoutes)
app.use('/api/posts', postRoutes)

// Health check
app.get('/health', async (req, res) => {
  try {
    await prisma.$queryRaw`SELECT 1`
    res.status(200).json({ status: 'OK', database: 'Connected' })
  } catch (error) {
    res.status(503).json({ status: 'Error', database: 'Disconnected' })
  }
})

// Error handling
app.use(errorHandler)

// Graceful shutdown
process.on('SIGINT', async () => {
  await prisma.$disconnect()
  process.exit(0)
})

export { app }
```

### Routes com Validação

```typescript
// src/routes/users.ts
import { Router } from 'express'
import { z } from 'zod'
import { prisma } from '../lib/prisma'
import { validateBody } from '../middleware/validation'
import { asyncHandler } from '../utils/asyncHandler'

const router = Router()

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
  age: z.number().min(0).max(120).optional(),
})

const updateUserSchema = createUserSchema.partial()

// GET /api/users
router.get('/', asyncHandler(async (req, res) => {
  const { page = 1, limit = 10, search } = req.query
  
  const where = search ? {
    OR: [
      { name: { contains: search as string, mode: 'insensitive' } },
      { email: { contains: search as string, mode: 'insensitive' } },
    ],
  } : {}
  
  const [users, total] = await Promise.all([
    prisma.user.findMany({
      where,
      skip: (Number(page) - 1) * Number(limit),
      take: Number(limit),
      select: {
        id: true,
        email: true,
        name: true,
        age: true,
        createdAt: true,
      },
      orderBy: { createdAt: 'desc' },
    }),
    prisma.user.count({ where }),
  ])
  
  res.json({
    users,
    pagination: {
      page: Number(page),
      limit: Number(limit),
      total,
      pages: Math.ceil(total / Number(limit)),
    },
  })
}))

// POST /api/users
router.post('/', validateBody(createUserSchema), asyncHandler(async (req, res) => {
  const user = await prisma.user.create({
    data: req.body,
    select: {
      id: true,
      email: true,
      name: true,
      age: true,
      createdAt: true,
    },
  })
  
  res.status(201).json(user)
}))

// GET /api/users/:id
router.get('/:id', asyncHandler(async (req, res) => {
  const { id } = req.params
  
  const user = await prisma.user.findUnique({
    where: { id: Number(id) },
    include: {
      posts: {
        select: {
          id: true,
          title: true,
          createdAt: true,
        },
        orderBy: { createdAt: 'desc' },
      },
    },
  })
  
  if (!user) {
    return res.status(404).json({ error: 'Usuário não encontrado' })
  }
  
  res.json(user)
}))

// PUT /api/users/:id
router.put('/:id', validateBody(updateUserSchema), asyncHandler(async (req, res) => {
  const { id } = req.params
  
  const user = await prisma.user.update({
    where: { id: Number(id) },
    data: req.body,
    select: {
      id: true,
      email: true,
      name: true,
      age: true,
      updatedAt: true,
    },
  })
  
  res.json(user)
}))

// DELETE /api/users/:id
router.delete('/:id', asyncHandler(async (req, res) => {
  const { id } = req.params
  
  await prisma.user.delete({
    where: { id: Number(id) },
  })
  
  res.status(204).send()
}))

export { router as userRoutes }
```

### Middleware de Validação

```typescript
// src/middleware/validation.ts
import { Request, Response, NextFunction } from 'express'
import { z } from 'zod'

export function validateBody(schema: z.ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      req.body = schema.parse(req.body)
      next()
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          error: 'Dados inválidos',
          details: error.errors,
        })
      }
      next(error)
    }
  }
}

export function validateQuery(schema: z.ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      req.query = schema.parse(req.query)
      next()
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          error: 'Parâmetros inválidos',
          details: error.errors,
        })
      }
      next(error)
    }
  }
}
```

## Fastify

### Setup e Configuração

```typescript
// src/app.ts
import Fastify from 'fastify'
import { prisma } from './lib/prisma'
import { userRoutes } from './routes/users'

const fastify = Fastify({
  logger: true,
})

// Registrar plugins
fastify.register(require('@fastify/cors'), {
  origin: true,
})

fastify.register(require('@fastify/helmet'))

// Adicionar Prisma ao contexto
fastify.decorate('prisma', prisma)

// Registrar routes
fastify.register(userRoutes, { prefix: '/api/users' })

// Health check
fastify.get('/health', async (request, reply) => {
  try {
    await prisma.$queryRaw`SELECT 1`
    return { status: 'OK', database: 'Connected' }
  } catch (error) {
    reply.code(503)
    return { status: 'Error', database: 'Disconnected' }
  }
})

// Graceful shutdown
fastify.addHook('onClose', async () => {
  await prisma.$disconnect()
})

export { fastify }
```

### Routes com Schema Validation

```typescript
// src/routes/users.ts
import { FastifyPluginAsync } from 'fastify'

const userRoutes: FastifyPluginAsync = async (fastify) => {
  // Schemas para validação
  const createUserSchema = {
    body: {
      type: 'object',
      required: ['email', 'name'],
      properties: {
        email: { type: 'string', format: 'email' },
        name: { type: 'string', minLength: 1 },
        age: { type: 'number', minimum: 0, maximum: 120 },
      },
    },
  }
  
  const getUserSchema = {
    params: {
      type: 'object',
      required: ['id'],
      properties: {
        id: { type: 'number' },
      },
    },
  }
  
  // GET /users
  fastify.get('/', async (request, reply) => {
    const users = await fastify.prisma.user.findMany({
      select: {
        id: true,
        email: true,
        name: true,
        createdAt: true,
      },
    })
    
    return users
  })
  
  // POST /users
  fastify.post('/', {
    schema: createUserSchema,
  }, async (request, reply) => {
    const user = await fastify.prisma.user.create({
      data: request.body as any,
      select: {
        id: true,
        email: true,
        name: true,
        createdAt: true,
      },
    })
    
    reply.code(201)
    return user
  })
  
  // GET /users/:id
  fastify.get('/:id', {
    schema: getUserSchema,
  }, async (request, reply) => {
    const { id } = request.params as { id: number }
    
    const user = await fastify.prisma.user.findUnique({
      where: { id },
      include: {
        posts: true,
      },
    })
    
    if (!user) {
      reply.code(404)
      return { error: 'Usuário não encontrado' }
    }
    
    return user
  })
}

export { userRoutes }
```

## NestJS

### Setup e Módulos

```bash
# Instalar NestJS CLI
npm i -g @nestjs/cli

# Criar projeto
nest new nest-prisma-app
cd nest-prisma-app

# Instalar Prisma
npm install prisma @prisma/client
npm install -D prisma

# Inicializar Prisma
npx prisma init
```

### Prisma Service

```typescript
// src/prisma/prisma.service.ts
import { Injectable, OnModuleInit } from '@nestjs/common'
import { PrismaClient } from '@prisma/client'

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect()
  }
}
```

### Prisma Module

```typescript
// src/prisma/prisma.module.ts
import { Module } from '@nestjs/common'
import { PrismaService } from './prisma.service'

@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

### User Module Completo

```typescript
// src/users/dto/create-user.dto.ts
import { IsEmail, IsString, IsOptional, IsNumber, Min, Max } from 'class-validator'

export class CreateUserDto {
  @IsEmail()
  email: string

  @IsString()
  name: string

  @IsOptional()
  @IsNumber()
  @Min(0)
  @Max(120)
  age?: number
}

// src/users/dto/update-user.dto.ts
import { PartialType } from '@nestjs/mapped-types'
import { CreateUserDto } from './create-user.dto'

export class UpdateUserDto extends PartialType(CreateUserDto) {}

// src/users/users.service.ts
import { Injectable, NotFoundException } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import { CreateUserDto } from './dto/create-user.dto'
import { UpdateUserDto } from './dto/update-user.dto'

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  async create(createUserDto: CreateUserDto) {
    return this.prisma.user.create({
      data: createUserDto,
      select: {
        id: true,
        email: true,
        name: true,
        age: true,
        createdAt: true,
      },
    })
  }

  async findAll() {
    return this.prisma.user.findMany({
      select: {
        id: true,
        email: true,
        name: true,
        age: true,
        createdAt: true,
      },
    })
  }

  async findOne(id: number) {
    const user = await this.prisma.user.findUnique({
      where: { id },
      include: {
        posts: true,
      },
    })

    if (!user) {
      throw new NotFoundException(`Usuário com ID ${id} não encontrado`)
    }

    return user
  }

  async update(id: number, updateUserDto: UpdateUserDto) {
    try {
      return await this.prisma.user.update({
        where: { id },
        data: updateUserDto,
        select: {
          id: true,
          email: true,
          name: true,
          age: true,
          updatedAt: true,
        },
      })
    } catch (error) {
      throw new NotFoundException(`Usuário com ID ${id} não encontrado`)
    }
  }

  async remove(id: number) {
    try {
      await this.prisma.user.delete({
        where: { id },
      })
    } catch (error) {
      throw new NotFoundException(`Usuário com ID ${id} não encontrado`)
    }
  }
}

// src/users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  Delete,
  ParseIntPipe,
  HttpCode,
  HttpStatus,
} from '@nestjs/common'
import { UsersService } from './users.service'
import { CreateUserDto } from './dto/create-user.dto'
import { UpdateUserDto } from './dto/update-user.dto'

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto)
  }

  @Get()
  findAll() {
    return this.usersService.findAll()
  }

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.findOne(id)
  }

  @Patch(':id')
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateUserDto: UpdateUserDto,
  ) {
    return this.usersService.update(id, updateUserDto)
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.remove(id)
  }
}

// src/users/users.module.ts
import { Module } from '@nestjs/common'
import { UsersService } from './users.service'
import { UsersController } from './users.controller'
import { PrismaModule } from '../prisma/prisma.module'

@Module({
  imports: [PrismaModule],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

## Dicas Gerais de Integração

### 1. Environment Variables

```bash
# .env
DATABASE_URL="postgresql://user:password@localhost:5432/mydb"
NODE_ENV="development"
PORT=3000

# .env.production
DATABASE_URL="postgresql://user:password@prod-host:5432/proddb?sslmode=require"
NODE_ENV="production"
PORT=8080
```

### 2. Docker Configuration

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY prisma ./prisma/
RUN npx prisma generate

COPY . .
RUN npm run build

EXPOSE 3000

CMD ["npm", "run", "start:prod"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/myapp
    depends_on:
      - db
      
  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
      
volumes:
  postgres_data:
```

### 3. CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
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
        
      - name: Run tests
        run: npm test
        
      - name: Build application
        run: npm run build
        
      - name: Deploy to production
        run: |
          npx prisma migrate deploy
          npm run start:prod
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

---

[Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>