<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# 📂 Modelagem de Dados com Prisma Schema

## Introdução ao Schema

O arquivo `schema.prisma` é o coração do Prisma. Ele define:
- Estrutura dos dados (modelos)
- Relacionamentos entre entidades
- Configurações do banco de dados
- Geradores de código

## Estrutura Básica do Schema

```prisma
// Gerador do cliente
generator client {
  provider = "prisma-client-js"
}

// Configuração do banco
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Modelos de dados
model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
}
```

## Tipos de Dados

### Tipos Escalares

| Prisma Type | PostgreSQL | MySQL | SQLite | Descrição |
|-------------|------------|-------|--------|-----------|
| `String` | `text` | `varchar` | `text` | Texto |
| `Boolean` | `boolean` | `tinyint(1)` | `integer` | Verdadeiro/Falso |
| `Int` | `integer` | `int` | `integer` | Número inteiro |
| `BigInt` | `bigint` | `bigint` | `integer` | Inteiro grande |
| `Float` | `real` | `float` | `real` | Número decimal |
| `Decimal` | `decimal` | `decimal` | `text` | Decimal preciso |
| `DateTime` | `timestamp` | `datetime` | `text` | Data e hora |
| `Json` | `jsonb` | `json` | `text` | Dados JSON |
| `Bytes` | `bytea` | `longblob` | `blob` | Dados binários |

### Tipos Opcionais

```prisma
model User {
  id       Int      @id @default(autoincrement())
  email    String   @unique
  name     String?  // Campo opcional
  age      Int?     // Idade opcional
  bio      String?  // Biografia opcional
}
```

## Atributos de Campo

### @id - Chave Primária

```prisma
model User {
  id    Int    @id @default(autoincrement())
  uuid  String @id @default(uuid()) // UUID como ID
}

// Chave primária composta
model UserRole {
  userId Int
  roleId Int
  
  @@id([userId, roleId])
}
```

### @unique - Campos Únicos

```prisma
model User {
  id       Int    @id @default(autoincrement())
  email    String @unique
  username String @unique
  
  // Constraint única composta
  @@unique([email, username])
}
```

### @default - Valores Padrão

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
  role      Role     @default(USER)
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

### @updatedAt - Timestamp Automático

```prisma
model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt // Atualizado automaticamente
}
```

### @map - Mapeamento de Nomes

```prisma
model User {
  id        Int    @id @default(autoincrement())
  firstName String @map("first_name") // Mapeia para coluna first_name
  lastName  String @map("last_name")
  
  @@map("users") // Mapeia para tabela users
}
```

## Enums

```prisma
enum Status {
  DRAFT
  PUBLISHED
  ARCHIVED
}

enum UserRole {
  USER
  ADMIN
  MODERATOR
}

model Post {
  id     Int    @id @default(autoincrement())
  title  String
  status Status @default(DRAFT)
}

model User {
  id   Int      @id @default(autoincrement())
  role UserRole @default(USER)
}
```

## Relacionamentos

### One-to-One (1:1)

```prisma
model User {
  id      Int      @id @default(autoincrement())
  email   String   @unique
  profile Profile? // Relação opcional
}

model Profile {
  id     Int    @id @default(autoincrement())
  bio    String
  userId Int    @unique
  user   User   @relation(fields: [userId], references: [id])
}
```

### One-to-Many (1:N)

```prisma
model User {
  id    Int    @id @default(autoincrement())
  email String @unique
  posts Post[] // Um usuário tem muitos posts
}

model Post {
  id       Int    @id @default(autoincrement())
  title    String
  authorId Int
  author   User   @relation(fields: [authorId], references: [id])
}
```

### Many-to-Many (N:N)

```prisma
model Post {
  id         Int        @id @default(autoincrement())
  title      String
  categories Category[] // Muitos posts têm muitas categorias
}

model Category {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[] // Muitas categorias têm muitos posts
}
```

### Many-to-Many com Tabela Intermediária

```prisma
model User {
  id        Int         @id @default(autoincrement())
  email     String      @unique
  userRoles UserRole[]
}

model Role {
  id        Int         @id @default(autoincrement())
  name      String      @unique
  userRoles UserRole[]
}

model UserRole {
  id       Int      @id @default(autoincrement())
  userId   Int
  roleId   Int
  assignedAt DateTime @default(now())
  
  user User @relation(fields: [userId], references: [id])
  role Role @relation(fields: [roleId], references: [id])
  
  @@unique([userId, roleId])
}
```

## Índices

### Índices Simples

```prisma
model User {
  id    Int    @id @default(autoincrement())
  email String @unique
  name  String
  
  @@index([name]) // Índice no campo name
}
```

### Índices Compostos

```prisma
model Post {
  id        Int      @id @default(autoincrement())
  title     String
  authorId  Int
  status    Status
  createdAt DateTime @default(now())
  
  @@index([authorId, status]) // Índice composto
  @@index([createdAt(sort: Desc)]) // Índice com ordenação
}
```

## Constraints

### Check Constraints

```prisma
model User {
  id  Int    @id @default(autoincrement())
  age Int
  
  @@check(age >= 0, name: "age_check")
}
```

### Foreign Key Constraints

```prisma
model Post {
  id       Int  @id @default(autoincrement())
  authorId Int
  author   User @relation(fields: [authorId], references: [id], onDelete: Cascade)
}
```

**Opções de onDelete:**
- `Cascade`: Deleta registros relacionados
- `Restrict`: Impede deleção se há relacionados
- `SetNull`: Define como null
- `SetDefault`: Define valor padrão
- `NoAction`: Nenhuma ação

## Exemplo Completo: Blog

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Role {
  USER
  ADMIN
  MODERATOR
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

model User {
  id        Int       @id @default(autoincrement())
  email     String    @unique
  username  String    @unique
  name      String?
  role      Role      @default(USER)
  isActive  Boolean   @default(true)
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  
  // Relacionamentos
  posts     Post[]
  comments  Comment[]
  profile   Profile?
  
  @@map("users")
}

model Profile {
  id       Int     @id @default(autoincrement())
  bio      String?
  website  String?
  location String?
  userId   Int     @unique
  user     User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("profiles")
}

model Category {
  id    Int    @id @default(autoincrement())
  name  String @unique
  slug  String @unique
  posts Post[]
  
  @@map("categories")
}

model Post {
  id          Int        @id @default(autoincrement())
  title       String
  slug        String     @unique
  content     String
  excerpt     String?
  status      PostStatus @default(DRAFT)
  publishedAt DateTime?
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt
  
  // Relacionamentos
  authorId   Int
  author     User        @relation(fields: [authorId], references: [id])
  comments   Comment[]
  categories Category[]
  
  @@index([authorId])
  @@index([status])
  @@index([publishedAt])
  @@map("posts")
}

model Comment {
  id        Int      @id @default(autoincrement())
  content   String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  // Relacionamentos
  authorId Int
  author   User @relation(fields: [authorId], references: [id])
  postId   Int
  post     Post @relation(fields: [postId], references: [id], onDelete: Cascade)
  
  @@index([postId])
  @@index([authorId])
  @@map("comments")
}
```

## Boas Práticas

### 1. Nomenclatura
- Use PascalCase para modelos: `User`, `BlogPost`
- Use camelCase para campos: `firstName`, `createdAt`
- Use UPPER_CASE para enums: `PUBLISHED`, `DRAFT`

### 2. Relacionamentos
- Sempre defina relacionamentos bidirecionais
- Use `onDelete` apropriado para integridade
- Considere performance ao criar índices

### 3. Campos Obrigatórios
- Use `createdAt` e `updatedAt` em entidades principais
- Considere campos de auditoria quando necessário
- Use enums para campos com valores limitados

### 4. Validações
- Use `@unique` para campos que devem ser únicos
- Implemente constraints no nível do banco
- Considere validações adicionais na aplicação

---

[🔙 Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>