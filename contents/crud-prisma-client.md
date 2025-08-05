<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# 🧑‍💻 CRUD com Prisma Client

## Configuração Inicial

### Instalação e Setup

```typescript
// Instalar Prisma Client
npm install @prisma/client

// Gerar cliente
npx prisma generate
```

### Importação e Inicialização

```typescript
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

// Com configurações
const prisma = new PrismaClient({
  log: ['query', 'info', 'warn', 'error'],
  errorFormat: 'pretty',
})

// Fechar conexão (importante!)
process.on('beforeExit', async () => {
  await prisma.$disconnect()
})
```

## CREATE - Criando Registros

### Criar Registro Simples

```typescript
// Criar usuário
const user = await prisma.user.create({
  data: {
    email: 'john@example.com',
    name: 'John Doe',
    age: 30
  }
})

console.log(user)
// { id: 1, email: 'john@example.com', name: 'John Doe', age: 30, createdAt: ... }
```

### Criar com Relacionamentos

```typescript
// Criar usuário com posts
const userWithPosts = await prisma.user.create({
  data: {
    email: 'jane@example.com',
    name: 'Jane Smith',
    posts: {
      create: [
        {
          title: 'Meu primeiro post',
          content: 'Conteúdo do post...'
        },
        {
          title: 'Segundo post',
          content: 'Mais conteúdo...'
        }
      ]
    }
  },
  include: {
    posts: true
  }
})
```

### Criar Múltiplos Registros

```typescript
// Criar múltiplos usuários
const users = await prisma.user.createMany({
  data: [
    { email: 'user1@example.com', name: 'User 1' },
    { email: 'user2@example.com', name: 'User 2' },
    { email: 'user3@example.com', name: 'User 3' }
  ]
})

console.log(`${users.count} usuários criados`)
```

### Upsert (Create ou Update)

```typescript
// Criar se não existir, atualizar se existir
const user = await prisma.user.upsert({
  where: {
    email: 'john@example.com'
  },
  update: {
    name: 'John Updated'
  },
  create: {
    email: 'john@example.com',
    name: 'John Doe'
  }
})
```

## READ - Lendo Registros

### Buscar Todos

```typescript
// Buscar todos os usuários
const users = await prisma.user.findMany()

// Com limite
const users = await prisma.user.findMany({
  take: 10
})

// Com paginação
const users = await prisma.user.findMany({
  skip: 20,
  take: 10
})
```

### Buscar por ID

```typescript
// Buscar por ID único
const user = await prisma.user.findUnique({
  where: {
    id: 1
  }
})

// Buscar por campo único
const user = await prisma.user.findUnique({
  where: {
    email: 'john@example.com'
  }
})
```

### Buscar Primeiro

```typescript
// Buscar primeiro registro que atende critério
const user = await prisma.user.findFirst({
  where: {
    age: {
      gte: 18
    }
  },
  orderBy: {
    createdAt: 'desc'
  }
})
```

### Filtros e Condições

```typescript
// Filtros básicos
const users = await prisma.user.findMany({
  where: {
    age: 25,                    // Igual a
    name: 'John',              // Igual a
    email: {
      contains: '@gmail.com'    // Contém
    }
  }
})

// Operadores de comparação
const users = await prisma.user.findMany({
  where: {
    age: {
      gte: 18,        // Maior ou igual
      lte: 65,        // Menor ou igual
      not: 25         // Diferente de
    },
    email: {
      startsWith: 'john',     // Começa com
      endsWith: '.com',       // Termina com
      contains: 'example'     // Contém
    }
  }
})

// Operadores lógicos
const users = await prisma.user.findMany({
  where: {
    OR: [
      { age: { lt: 18 } },
      { age: { gt: 65 } }
    ],
    AND: [
      { email: { contains: '@' } },
      { name: { not: null } }
    ]
  }
})
```

### Incluir Relacionamentos

```typescript
// Incluir posts do usuário
const userWithPosts = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: true
  }
})

// Incluir relacionamentos aninhados
const userWithPostsAndComments = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: {
      include: {
        comments: true
      }
    }
  }
})

// Filtrar relacionamentos incluídos
const userWithPublishedPosts = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: {
      where: {
        published: true
      }
    }
  }
})
```

### Select Campos Específicos

```typescript
// Selecionar apenas campos específicos
const users = await prisma.user.findMany({
  select: {
    id: true,
    email: true,
    name: true
    // age não será retornado
  }
})

// Combinar select com relacionamentos
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    posts: {
      select: {
        id: true,
        title: true
      }
    }
  }
})
```

### Ordenação

```typescript
// Ordenação simples
const users = await prisma.user.findMany({
  orderBy: {
    name: 'asc'
  }
})

// Ordenação múltipla
const users = await prisma.user.findMany({
  orderBy: [
    { age: 'desc' },
    { name: 'asc' }
  ]
})

// Ordenação por relacionamento
const users = await prisma.user.findMany({
  orderBy: {
    posts: {
      _count: 'desc'
    }
  }
})
```

## UPDATE - Atualizando Registros

### Atualizar Registro Único

```typescript
// Atualizar usuário por ID
const updatedUser = await prisma.user.update({
  where: {
    id: 1
  },
  data: {
    name: 'John Updated',
    age: 31
  }
})
```

### Atualizar Múltiplos Registros

```typescript
// Atualizar múltiplos usuários
const result = await prisma.user.updateMany({
  where: {
    age: {
      lt: 18
    }
  },
  data: {
    status: 'MINOR'
  }
})

console.log(`${result.count} usuários atualizados`)
```

### Operações Atômicas

```typescript
// Incrementar/decrementar
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    age: {
      increment: 1
    },
    loginCount: {
      increment: 1
    }
  }
})

// Operações em arrays (PostgreSQL)
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    tags: {
      push: 'new-tag'
    }
  }
})
```

### Atualizar Relacionamentos

```typescript
// Conectar a registros existentes
const post = await prisma.post.update({
  where: { id: 1 },
  data: {
    categories: {
      connect: [
        { id: 1 },
        { id: 2 }
      ]
    }
  }
})

// Desconectar relacionamentos
const post = await prisma.post.update({
  where: { id: 1 },
  data: {
    categories: {
      disconnect: [
        { id: 1 }
      ]
    }
  }
})

// Criar novos relacionamentos
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    posts: {
      create: {
        title: 'Novo post',
        content: 'Conteúdo...'
      }
    }
  }
})
```

## DELETE - Deletando Registros

### Deletar Registro Único

```typescript
// Deletar usuário por ID
const deletedUser = await prisma.user.delete({
  where: {
    id: 1
  }
})
```

### Deletar Múltiplos Registros

```typescript
// Deletar múltiplos usuários
const result = await prisma.user.deleteMany({
  where: {
    age: {
      lt: 18
    }
  }
})

console.log(`${result.count} usuários deletados`)

// Deletar todos (cuidado!)
const result = await prisma.user.deleteMany({})
```

## Transações

### Transação Simples

```typescript
// Transação com múltiplas operações
const result = await prisma.$transaction([
  prisma.user.create({
    data: {
      email: 'user1@example.com',
      name: 'User 1'
    }
  }),
  prisma.user.create({
    data: {
      email: 'user2@example.com',
      name: 'User 2'
    }
  })
])
```

### Transação Interativa

```typescript
// Transação com lógica condicional
const result = await prisma.$transaction(async (tx) => {
  // Buscar usuário
  const user = await tx.user.findUnique({
    where: { id: 1 }
  })

  if (!user) {
    throw new Error('Usuário não encontrado')
  }

  // Atualizar saldo
  const updatedUser = await tx.user.update({
    where: { id: 1 },
    data: {
      balance: {
        decrement: 100
      }
    }
  })

  // Criar registro de transação
  const transaction = await tx.transaction.create({
    data: {
      userId: 1,
      amount: -100,
      type: 'DEBIT'
    }
  })

  return { user: updatedUser, transaction }
})
```

## Agregações e Contadores

### Contar Registros

```typescript
// Contar todos os usuários
const userCount = await prisma.user.count()

// Contar com filtros
const adultCount = await prisma.user.count({
  where: {
    age: {
      gte: 18
    }
  }
})
```

### Agregações

```typescript
// Agregações numéricas
const stats = await prisma.user.aggregate({
  _avg: {
    age: true
  },
  _sum: {
    balance: true
  },
  _min: {
    age: true
  },
  _max: {
    age: true
  },
  _count: {
    id: true
  }
})

console.log(stats)
// {
//   _avg: { age: 32.5 },
//   _sum: { balance: 15000 },
//   _min: { age: 18 },
//   _max: { age: 65 },
//   _count: { id: 100 }
// }
```

### Group By

```typescript
// Agrupar por campo
const groupedUsers = await prisma.user.groupBy({
  by: ['status'],
  _count: {
    id: true
  },
  _avg: {
    age: true
  }
})

// Resultado:
// [
//   { status: 'ACTIVE', _count: { id: 80 }, _avg: { age: 35 } },
//   { status: 'INACTIVE', _count: { id: 20 }, _avg: { age: 28 } }
// ]
```

## Queries Raw

### SQL Raw

```typescript
// Query SQL direta
const users = await prisma.$queryRaw`
  SELECT * FROM "User" 
  WHERE age > ${25} 
  ORDER BY name
`

// Query com tipagem
interface UserResult {
  id: number
  name: string
  email: string
}

const users: UserResult[] = await prisma.$queryRaw`
  SELECT id, name, email FROM "User"
`
```

### Execute Raw

```typescript
// Executar comando SQL
const result = await prisma.$executeRaw`
  UPDATE "User" 
  SET "lastLogin" = NOW() 
  WHERE id = ${userId}
`

console.log(`${result} linhas afetadas`)
```

## Middleware e Hooks

### Middleware

```typescript
// Middleware para logging
prisma.$use(async (params, next) => {
  const before = Date.now()
  const result = await next(params)
  const after = Date.now()
  
  console.log(`Query ${params.model}.${params.action} took ${after - before}ms`)
  
  return result
})

// Middleware para soft delete
prisma.$use(async (params, next) => {
  if (params.action === 'delete') {
    params.action = 'update'
    params.args['data'] = { deleted: true }
  }
  
  if (params.action === 'deleteMany') {
    params.action = 'updateMany'
    if (params.args.data != undefined) {
      params.args.data['deleted'] = true
    } else {
      params.args['data'] = { deleted: true }
    }
  }
  
  return next(params)
})
```

## Tratamento de Erros

```typescript
import { PrismaClientKnownRequestError } from '@prisma/client/runtime/library'

try {
  const user = await prisma.user.create({
    data: {
      email: 'duplicate@example.com'
    }
  })
} catch (error) {
  if (error instanceof PrismaClientKnownRequestError) {
    if (error.code === 'P2002') {
      console.log('Email já existe!')
    }
  }
  throw error
}
```

## Boas Práticas

### 1. Singleton Pattern

```typescript
// lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma = globalForPrisma.prisma ?? new PrismaClient()

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

### 2. Type Safety

```typescript
// Usar tipos gerados
import type { User, Post } from '@prisma/client'

// Tipos com relacionamentos
type UserWithPosts = User & {
  posts: Post[]
}

// Tipos parciais
type CreateUserData = Pick<User, 'email' | 'name'>
```

### 3. Validação

```typescript
import { z } from 'zod'

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
  age: z.number().min(0).max(120)
})

async function createUser(data: unknown) {
  const validatedData = createUserSchema.parse(data)
  
  return prisma.user.create({
    data: validatedData
  })
}
```

---

[🔙 Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>