<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# 🎨 Relacionamentos e Associações

## Tipos de Relacionamentos

O Prisma suporta três tipos principais de relacionamentos:
- **One-to-One (1:1)**: Um registro relaciona-se com exatamente um outro
- **One-to-Many (1:N)**: Um registro relaciona-se com vários outros
- **Many-to-Many (N:N)**: Vários registros relacionam-se com vários outros

## One-to-One (1:1)

### Definição no Schema

```prisma
model User {
  id      Int      @id @default(autoincrement())
  email   String   @unique
  profile Profile? // Relação opcional
}

model Profile {
  id     Int    @id @default(autoincrement())
  bio    String
  avatar String?
  userId Int    @unique // Chave estrangeira única
  user   User   @relation(fields: [userId], references: [id])
}
```

### Operações CRUD

```typescript
// Criar usuário com perfil
const userWithProfile = await prisma.user.create({
  data: {
    email: 'john@example.com',
    profile: {
      create: {
        bio: 'Desenvolvedor apaixonado por tecnologia',
        avatar: 'https://example.com/avatar.jpg'
      }
    }
  },
  include: {
    profile: true
  }
})

// Buscar usuário com perfil
const user = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    profile: true
  }
})

// Atualizar perfil
const updatedUser = await prisma.user.update({
  where: { id: 1 },
  data: {
    profile: {
      update: {
        bio: 'Nova biografia'
      }
    }
  }
})

// Conectar perfil existente
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    profile: {
      connect: { id: 5 }
    }
  }
})
```

## One-to-Many (1:N)

### Definição no Schema

```prisma
model User {
  id    Int    @id @default(autoincrement())
  email String @unique
  name  String
  posts Post[] // Um usuário tem muitos posts
}

model Post {
  id       Int    @id @default(autoincrement())
  title    String
  content  String
  authorId Int    // Chave estrangeira
  author   User   @relation(fields: [authorId], references: [id])
}
```

### Operações CRUD

```typescript
// Criar usuário com posts
const userWithPosts = await prisma.user.create({
  data: {
    email: 'jane@example.com',
    name: 'Jane Doe',
    posts: {
      create: [
        {
          title: 'Primeiro Post',
          content: 'Conteúdo do primeiro post...'
        },
        {
          title: 'Segundo Post',
          content: 'Conteúdo do segundo post...'
        }
      ]
    }
  },
  include: {
    posts: true
  }
})

// Buscar usuário com posts
const userWithPosts = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: {
      orderBy: {
        createdAt: 'desc'
      }
    }
  }
})

// Adicionar post a usuário existente
const newPost = await prisma.post.create({
  data: {
    title: 'Novo Post',
    content: 'Conteúdo...',
    authorId: 1
  }
})

// Ou usando nested write
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    posts: {
      create: {
        title: 'Post via nested write',
        content: 'Conteúdo...'
      }
    }
  }
})
```

## Many-to-Many (N:N)

### Relacionamento Implícito

```prisma
model Post {
  id         Int        @id @default(autoincrement())
  title      String
  content    String
  categories Category[] // Muitos posts têm muitas categorias
}

model Category {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[] // Muitas categorias têm muitos posts
}
```

### Relacionamento Explícito (com tabela intermediária)

```prisma
model Post {
  id               Int               @id @default(autoincrement())
  title            String
  content          String
  postCategories   PostCategory[]
}

model Category {
  id               Int               @id @default(autoincrement())
  name             String            @unique
  postCategories   PostCategory[]
}

model PostCategory {
  id         Int      @id @default(autoincrement())
  postId     Int
  categoryId Int
  assignedAt DateTime @default(now())
  assignedBy String?
  
  post     Post     @relation(fields: [postId], references: [id])
  category Category @relation(fields: [categoryId], references: [id])
  
  @@unique([postId, categoryId])
}
```

### Operações CRUD - Relacionamento Implícito

```typescript
// Criar post com categorias
const postWithCategories = await prisma.post.create({
  data: {
    title: 'Post sobre Prisma',
    content: 'Conteúdo sobre Prisma...',
    categories: {
      create: [
        { name: 'Tecnologia' },
        { name: 'Banco de Dados' }
      ]
    }
  },
  include: {
    categories: true
  }
})

// Conectar a categorias existentes
const post = await prisma.post.create({
  data: {
    title: 'Outro Post',
    content: 'Conteúdo...',
    categories: {
      connect: [
        { id: 1 },
        { id: 2 }
      ]
    }
  }
})

// Atualizar relacionamentos
const updatedPost = await prisma.post.update({
  where: { id: 1 },
  data: {
    categories: {
      connect: [{ id: 3 }],    // Adicionar categoria
      disconnect: [{ id: 1 }]  // Remover categoria
    }
  }
})

// Substituir todas as categorias
const post = await prisma.post.update({
  where: { id: 1 },
  data: {
    categories: {
      set: [
        { id: 2 },
        { id: 3 }
      ]
    }
  }
})
```

### Operações CRUD - Relacionamento Explícito

```typescript
// Criar relacionamento com dados extras
const postCategory = await prisma.postCategory.create({
  data: {
    postId: 1,
    categoryId: 2,
    assignedBy: 'admin@example.com'
  },
  include: {
    post: true,
    category: true
  }
})

// Buscar posts com categorias e metadados
const postsWithCategories = await prisma.post.findMany({
  include: {
    postCategories: {
      include: {
        category: true
      }
    }
  }
})
```

## Relacionamentos Complexos

### Self-Relations (Auto-relacionamentos)

```prisma
model User {
  id        Int    @id @default(autoincrement())
  email     String @unique
  name      String
  
  // Auto-relacionamento para hierarquia
  managerId Int?
  manager   User?  @relation("UserManager", fields: [managerId], references: [id])
  employees User[] @relation("UserManager")
  
  // Auto-relacionamento para amizades
  friendships Friendship[] @relation("UserFriendships")
  friendOf    Friendship[] @relation("FriendOf")
}

model Friendship {
  id       Int      @id @default(autoincrement())
  userId   Int
  friendId Int
  status   String   // 'PENDING', 'ACCEPTED', 'BLOCKED'
  createdAt DateTime @default(now())
  
  user   User @relation("UserFriendships", fields: [userId], references: [id])
  friend User @relation("FriendOf", fields: [friendId], references: [id])
  
  @@unique([userId, friendId])
}
```

### Relacionamentos Polimórficos (Simulados)

```prisma
model Comment {
  id          Int     @id @default(autoincrement())
  content     String
  authorId    Int
  
  // Campos para polimorfismo
  commentableType String // 'POST' ou 'VIDEO'
  commentableId   Int
  
  author User @relation(fields: [authorId], references: [id])
  
  @@index([commentableType, commentableId])
}

model Post {
  id      Int    @id @default(autoincrement())
  title   String
  content String
}

model Video {
  id       Int    @id @default(autoincrement())
  title    String
  url      String
  duration Int
}
```

## Queries Avançadas com Relacionamentos

### Filtrar por Relacionamentos

```typescript
// Usuários que têm posts
const usersWithPosts = await prisma.user.findMany({
  where: {
    posts: {
      some: {} // Pelo menos um post
    }
  }
})

// Usuários sem posts
const usersWithoutPosts = await prisma.user.findMany({
  where: {
    posts: {
      none: {} // Nenhum post
    }
  }
})

// Usuários com posts publicados
const usersWithPublishedPosts = await prisma.user.findMany({
  where: {
    posts: {
      some: {
        published: true
      }
    }
  }
})

// Posts de uma categoria específica
const techPosts = await prisma.post.findMany({
  where: {
    categories: {
      some: {
        name: 'Tecnologia'
      }
    }
  }
})
```

### Contadores e Agregações

```typescript
// Contar posts por usuário
const usersWithPostCount = await prisma.user.findMany({
  include: {
    _count: {
      select: {
        posts: true
      }
    }
  }
})

// Usuários ordenados por número de posts
const usersByPostCount = await prisma.user.findMany({
  orderBy: {
    posts: {
      _count: 'desc'
    }
  },
  include: {
    _count: {
      select: {
        posts: true
      }
    }
  }
})

// Agregações em relacionamentos
const categoryStats = await prisma.category.findMany({
  include: {
    _count: {
      select: {
        posts: true
      }
    }
  }
})
```

### Queries Aninhadas Profundas

```typescript
// Buscar usuários com posts e comentários
const usersWithPostsAndComments = await prisma.user.findMany({
  include: {
    posts: {
      include: {
        comments: {
          include: {
            author: {
              select: {
                name: true,
                email: true
              }
            }
          }
        }
      }
    }
  }
})

// Limitar resultados aninhados
const usersWithRecentPosts = await prisma.user.findMany({
  include: {
    posts: {
      take: 5,
      orderBy: {
        createdAt: 'desc'
      },
      where: {
        published: true
      }
    }
  }
})
```

## Constraints e Integridade

### Cascade Delete

```prisma
model User {
  id    Int    @id @default(autoincrement())
  email String @unique
  posts Post[]
}

model Post {
  id       Int  @id @default(autoincrement())
  title    String
  authorId Int
  author   User @relation(fields: [authorId], references: [id], onDelete: Cascade)
}
```

### Restrict Delete

```prisma
model Category {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[]
}

model Post {
  id         Int      @id @default(autoincrement())
  title      String
  categoryId Int
  category   Category @relation(fields: [categoryId], references: [id], onDelete: Restrict)
}
```

### Set Null

```prisma
model Post {
  id       Int   @id @default(autoincrement())
  title    String
  authorId Int?
  author   User? @relation(fields: [authorId], references: [id], onDelete: SetNull)
}
```

## Boas Práticas

### 1. Nomenclatura Consistente

```prisma
// ✅ Bom: nomes claros e consistentes
model User {
  id    Int    @id @default(autoincrement())
  posts Post[] // Plural para array
}

model Post {
  id       Int  @id @default(autoincrement())
  authorId Int  // Sufixo Id para chave estrangeira
  author   User @relation(fields: [authorId], references: [id])
}

// ❌ Ruim: nomes inconsistentes
model User {
  id   Int     @id @default(autoincrement())
  post Post[]  // Deveria ser posts
}

model Post {
  id     Int  @id @default(autoincrement())
  userId Int  // Inconsistente com authorId
  user   User @relation(fields: [userId], references: [id])
}
```

### 2. Índices em Chaves Estrangeiras

```prisma
model Post {
  id       Int  @id @default(autoincrement())
  title    String
  authorId Int
  author   User @relation(fields: [authorId], references: [id])
  
  @@index([authorId]) // Índice para performance
}
```

### 3. Validação de Relacionamentos

```typescript
// Verificar se relacionamento existe antes de criar
async function createPost(authorId: number, postData: any) {
  const author = await prisma.user.findUnique({
    where: { id: authorId }
  })
  
  if (!author) {
    throw new Error('Autor não encontrado')
  }
  
  return prisma.post.create({
    data: {
      ...postData,
      authorId
    }
  })
}
```

### 4. Lazy Loading vs Eager Loading

```typescript
// ✅ Bom: carregar apenas quando necessário
const user = await prisma.user.findUnique({
  where: { id: 1 }
})

// Carregar posts apenas se necessário
if (needsPosts) {
  const userWithPosts = await prisma.user.findUnique({
    where: { id: 1 },
    include: { posts: true }
  })
}

// ❌ Ruim: sempre carregar tudo
const user = await prisma.user.findUnique({
  where: { id: 1 },
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

### 5. Transações para Operações Complexas

```typescript
// Criar usuário com perfil e posts em transação
const result = await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({
    data: {
      email: 'user@example.com',
      name: 'User Name'
    }
  })
  
  const profile = await tx.profile.create({
    data: {
      bio: 'User bio',
      userId: user.id
    }
  })
  
  const posts = await tx.post.createMany({
    data: [
      { title: 'Post 1', content: 'Content 1', authorId: user.id },
      { title: 'Post 2', content: 'Content 2', authorId: user.id }
    ]
  })
  
  return { user, profile, posts }
})
```

---

[🔙 Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>