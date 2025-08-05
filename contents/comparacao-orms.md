<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# ⚖️ Comparação com Outros ORMs

## Prisma vs Sequelize

### Definição de Modelos

**Prisma:**
```prisma
// schema.prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id       Int    @id @default(autoincrement())
  title    String
  content  String
  authorId Int
  author   User   @relation(fields: [authorId], references: [id])
}
```

**Sequelize:**
```javascript
// models/User.js
const { DataTypes } = require('sequelize')

module.exports = (sequelize) => {
  const User = sequelize.define('User', {
    id: {
      type: DataTypes.INTEGER,
      primaryKey: true,
      autoIncrement: true,
    },
    email: {
      type: DataTypes.STRING,
      unique: true,
      allowNull: false,
    },
    name: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    createdAt: {
      type: DataTypes.DATE,
      defaultValue: DataTypes.NOW,
    },
  })

  User.associate = (models) => {
    User.hasMany(models.Post, {
      foreignKey: 'authorId',
      as: 'posts',
    })
  }

  return User
}

// models/Post.js
module.exports = (sequelize) => {
  const Post = sequelize.define('Post', {
    id: {
      type: DataTypes.INTEGER,
      primaryKey: true,
      autoIncrement: true,
    },
    title: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    content: {
      type: DataTypes.TEXT,
      allowNull: false,
    },
    authorId: {
      type: DataTypes.INTEGER,
      allowNull: false,
    },
  })

  Post.associate = (models) => {
    Post.belongsTo(models.User, {
      foreignKey: 'authorId',
      as: 'author',
    })
  }

  return Post
}
```

### Queries e Operações

**Prisma:**
```typescript
// Buscar usuários com posts
const users = await prisma.user.findMany({
  include: {
    posts: {
      select: {
        id: true,
        title: true,
      },
    },
  },
})

// Criar usuário com posts
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'João Silva',
    posts: {
      create: [
        {
          title: 'Primeiro Post',
          content: 'Conteúdo do post...',
        },
      ],
    },
  },
  include: {
    posts: true,
  },
})

// Transação
const result = await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({
    data: { email: 'test@example.com', name: 'Test' },
  })
  
  const post = await tx.post.create({
    data: {
      title: 'Test Post',
      content: 'Content',
      authorId: user.id,
    },
  })
  
  return { user, post }
})
```

**Sequelize:**
```javascript
// Buscar usuários com posts
const users = await User.findAll({
  include: [
    {
      model: Post,
      as: 'posts',
      attributes: ['id', 'title'],
    },
  ],
})

// Criar usuário com posts (mais complexo)
const user = await User.create({
  email: 'user@example.com',
  name: 'João Silva',
}, {
  include: [
    {
      model: Post,
      as: 'posts',
    },
  ],
})

// Criar posts separadamente
const post = await Post.create({
  title: 'Primeiro Post',
  content: 'Conteúdo do post...',
  authorId: user.id,
})

// Transação
const result = await sequelize.transaction(async (t) => {
  const user = await User.create({
    email: 'test@example.com',
    name: 'Test',
  }, { transaction: t })
  
  const post = await Post.create({
    title: 'Test Post',
    content: 'Content',
    authorId: user.id,
  }, { transaction: t })
  
  return { user, post }
})
```

### Migrations

**Prisma:**
```bash
# Gerar migration automaticamente
npx prisma migrate dev --name add_user_posts

# Aplicar em produção
npx prisma migrate deploy
```

**Sequelize:**
```bash
# Criar migration manualmente
npx sequelize-cli migration:generate --name add-user-posts

# Editar arquivo de migration
# migrations/20231201000000-add-user-posts.js
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('Users', {
      id: {
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
        type: Sequelize.INTEGER,
      },
      email: {
        type: Sequelize.STRING,
        unique: true,
        allowNull: false,
      },
      // ... outros campos
    })
  },
  
  down: async (queryInterface, Sequelize) => {
    await queryInterface.dropTable('Users')
  },
}

# Aplicar migration
npx sequelize-cli db:migrate
```

### Vantagens e Desvantagens

| Aspecto | Prisma | Sequelize |
|---------|--------|----------|
| **Type Safety** | ✅ Nativo | ❌ Requer configuração extra |
| **Schema Definition** | ✅ Declarativo | ❌ Imperativo |
| **Auto-completion** | ✅ Excelente | ⚠️ Limitado |
| **Migrations** | ✅ Automáticas | ❌ Manuais |
| **Query Builder** | ✅ Intuitivo | ⚠️ Verboso |
| **Performance** | ✅ Otimizado | ⚠️ Depende da configuração |
| **Comunidade** | ⚠️ Crescendo | ✅ Estabelecida |
| **Flexibilidade** | ⚠️ Opinionated | ✅ Muito flexível |

## Prisma vs TypeORM

### Definição de Entidades

**Prisma:**
```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  profile   Profile?
  posts     Post[]
  createdAt DateTime @default(now())
}

model Profile {
  id     Int    @id @default(autoincrement())
  bio    String
  avatar String?
  userId Int    @unique
  user   User   @relation(fields: [userId], references: [id])
}
```

**TypeORM:**
```typescript
// entities/User.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  OneToOne,
  OneToMany,
  CreateDateColumn,
} from 'typeorm'
import { Profile } from './Profile'
import { Post } from './Post'

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number

  @Column({ unique: true })
  email: string

  @Column()
  name: string

  @OneToOne(() => Profile, profile => profile.user)
  profile: Profile

  @OneToMany(() => Post, post => post.author)
  posts: Post[]

  @CreateDateColumn()
  createdAt: Date
}

// entities/Profile.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  OneToOne,
  JoinColumn,
} from 'typeorm'
import { User } from './User'

@Entity('profiles')
export class Profile {
  @PrimaryGeneratedColumn()
  id: number

  @Column()
  bio: string

  @Column({ nullable: true })
  avatar: string

  @Column()
  userId: number

  @OneToOne(() => User, user => user.profile)
  @JoinColumn()
  user: User
}
```

### Repository Pattern

**Prisma:**
```typescript
// services/userService.ts
import { prisma } from '../lib/prisma'

export class UserService {
  async findUserWithProfile(id: number) {
    return prisma.user.findUnique({
      where: { id },
      include: {
        profile: true,
        posts: {
          select: {
            id: true,
            title: true,
          },
        },
      },
    })
  }

  async createUserWithProfile(userData: any) {
    return prisma.user.create({
      data: {
        email: userData.email,
        name: userData.name,
        profile: {
          create: {
            bio: userData.bio,
            avatar: userData.avatar,
          },
        },
      },
      include: {
        profile: true,
      },
    })
  }
}
```

**TypeORM:**
```typescript
// repositories/userRepository.ts
import { Repository } from 'typeorm'
import { AppDataSource } from '../data-source'
import { User } from '../entities/User'
import { Profile } from '../entities/Profile'

export class UserRepository {
  private userRepo: Repository<User>
  private profileRepo: Repository<Profile>

  constructor() {
    this.userRepo = AppDataSource.getRepository(User)
    this.profileRepo = AppDataSource.getRepository(Profile)
  }

  async findUserWithProfile(id: number) {
    return this.userRepo.findOne({
      where: { id },
      relations: {
        profile: true,
        posts: true,
      },
      select: {
        id: true,
        email: true,
        name: true,
        profile: true,
        posts: {
          id: true,
          title: true,
        },
      },
    })
  }

  async createUserWithProfile(userData: any) {
    const queryRunner = AppDataSource.createQueryRunner()
    await queryRunner.connect()
    await queryRunner.startTransaction()

    try {
      const user = this.userRepo.create({
        email: userData.email,
        name: userData.name,
      })
      const savedUser = await queryRunner.manager.save(user)

      const profile = this.profileRepo.create({
        bio: userData.bio,
        avatar: userData.avatar,
        userId: savedUser.id,
      })
      await queryRunner.manager.save(profile)

      await queryRunner.commitTransaction()
      
      return this.findUserWithProfile(savedUser.id)
    } catch (error) {
      await queryRunner.rollbackTransaction()
      throw error
    } finally {
      await queryRunner.release()
    }
  }
}
```

### Query Builder

**Prisma:**
```typescript
// Queries complexas
const users = await prisma.user.findMany({
  where: {
    posts: {
      some: {
        createdAt: {
          gte: new Date('2023-01-01'),
        },
        published: true,
      },
    },
  },
  include: {
    _count: {
      select: {
        posts: true,
      },
    },
  },
  orderBy: {
    posts: {
      _count: 'desc',
    },
  },
})

// Agregações
const stats = await prisma.post.aggregate({
  _count: {
    id: true,
  },
  _avg: {
    viewCount: true,
  },
  where: {
    published: true,
  },
})
```

**TypeORM:**
```typescript
// Query Builder
const users = await userRepo
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.posts', 'post')
  .where('post.createdAt >= :date', { date: new Date('2023-01-01') })
  .andWhere('post.published = :published', { published: true })
  .loadRelationCountAndMap('user.postCount', 'user.posts')
  .orderBy('user.postCount', 'DESC')
  .getMany()

// Agregações
const stats = await postRepo
  .createQueryBuilder('post')
  .select('COUNT(post.id)', 'count')
  .addSelect('AVG(post.viewCount)', 'avgViews')
  .where('post.published = :published', { published: true })
  .getRawOne()
```

### Comparação Detalhada

| Aspecto | Prisma | TypeORM |
|---------|--------|----------|
| **Decorators** | ❌ Não usa | ✅ Usa extensivamente |
| **Active Record** | ❌ Não suporta | ✅ Suporta |
| **Data Mapper** | ✅ Padrão | ✅ Suporta |
| **Raw SQL** | ✅ Suporte completo | ✅ Suporte completo |
| **Migrations** | ✅ Automáticas | ⚠️ Semi-automáticas |
| **Type Safety** | ✅ 100% type-safe | ⚠️ Boa, mas não perfeita |
| **Learning Curve** | ✅ Mais fácil | ⚠️ Mais complexo |
| **Flexibilidade** | ⚠️ Menos flexível | ✅ Muito flexível |

## Prisma vs Drizzle ORM

### Definição de Schema

**Prisma:**
```prisma
model User {
  id    Int    @id @default(autoincrement())
  email String @unique
  name  String
  posts Post[]
}

model Post {
  id       Int    @id @default(autoincrement())
  title    String
  content  String
  authorId Int
  author   User   @relation(fields: [authorId], references: [id])
}
```

**Drizzle:**
```typescript
// schema.ts
import { pgTable, serial, text, integer } from 'drizzle-orm/pg-core'
import { relations } from 'drizzle-orm'

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  email: text('email').unique().notNull(),
  name: text('name').notNull(),
})

export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: text('title').notNull(),
  content: text('content').notNull(),
  authorId: integer('author_id').notNull(),
})

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}))

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
}))
```

### Queries

**Prisma:**
```typescript
const users = await prisma.user.findMany({
  include: {
    posts: true,
  },
})

const user = await prisma.user.create({
  data: {
    email: 'test@example.com',
    name: 'Test User',
    posts: {
      create: {
        title: 'First Post',
        content: 'Content here...',
      },
    },
  },
})
```

**Drizzle:**
```typescript
import { db } from './db'
import { users, posts } from './schema'
import { eq } from 'drizzle-orm'

const usersWithPosts = await db.query.users.findMany({
  with: {
    posts: true,
  },
})

// Inserção mais verbosa
const [user] = await db.insert(users).values({
  email: 'test@example.com',
  name: 'Test User',
}).returning()

const [post] = await db.insert(posts).values({
  title: 'First Post',
  content: 'Content here...',
  authorId: user.id,
}).returning()
```

### Comparação

| Aspecto | Prisma | Drizzle |
|---------|--------|---------|
| **Bundle Size** | ⚠️ Maior | ✅ Menor |
| **Performance** | ✅ Boa | ✅ Excelente |
| **Type Safety** | ✅ Excelente | ✅ Excelente |
| **SQL-like** | ❌ Abstração alta | ✅ Próximo ao SQL |
| **Migrations** | ✅ Automáticas | ⚠️ Manuais |
| **Ecosystem** | ✅ Maduro | ⚠️ Novo |
| **Learning Curve** | ✅ Fácil | ⚠️ Médio |

## Prisma vs Mongoose (MongoDB)

### Schema Definition

**Prisma:**
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "mongodb"
  url      = env("DATABASE_URL")
}

model User {
  id       String @id @default(auto()) @map("_id") @db.ObjectId
  email    String @unique
  name     String
  profile  Profile?
  posts    Post[]
  postIds  String[] @db.ObjectId
}

model Profile {
  id     String @id @default(auto()) @map("_id") @db.ObjectId
  bio    String
  avatar String?
  user   User   @relation(fields: [userId], references: [id])
  userId String @unique @db.ObjectId
}

model Post {
  id       String @id @default(auto()) @map("_id") @db.ObjectId
  title    String
  content  String
  author   User   @relation(fields: [authorId], references: [id])
  authorId String @db.ObjectId
}
```

**Mongoose:**
```javascript
// models/User.js
const mongoose = require('mongoose')

const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
  },
  name: {
    type: String,
    required: true,
  },
  profile: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Profile',
  },
  posts: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Post',
  }],
}, {
  timestamps: true,
})

module.exports = mongoose.model('User', userSchema)

// models/Profile.js
const profileSchema = new mongoose.Schema({
  bio: {
    type: String,
    required: true,
  },
  avatar: String,
  user: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true,
  },
})

module.exports = mongoose.model('Profile', profileSchema)
```

### Queries

**Prisma:**
```typescript
// Buscar com relacionamentos
const users = await prisma.user.findMany({
  include: {
    profile: true,
    posts: {
      select: {
        id: true,
        title: true,
      },
    },
  },
})

// Criar com relacionamentos
const user = await prisma.user.create({
  data: {
    email: 'test@example.com',
    name: 'Test User',
    profile: {
      create: {
        bio: 'Test bio',
      },
    },
    posts: {
      create: [
        {
          title: 'First Post',
          content: 'Content...',
        },
      ],
    },
  },
})
```

**Mongoose:**
```javascript
// Buscar com relacionamentos
const users = await User.find()
  .populate('profile')
  .populate('posts', 'id title')
  .exec()

// Criar com relacionamentos (mais complexo)
const user = new User({
  email: 'test@example.com',
  name: 'Test User',
})
const savedUser = await user.save()

const profile = new Profile({
  bio: 'Test bio',
  user: savedUser._id,
})
const savedProfile = await profile.save()

const post = new Post({
  title: 'First Post',
  content: 'Content...',
  author: savedUser._id,
})
const savedPost = await post.save()

// Atualizar referências
savedUser.profile = savedProfile._id
savedUser.posts.push(savedPost._id)
await savedUser.save()
```

## Resumo das Comparações

### Quando Usar Prisma

✅ **Ideal para:**
- Projetos TypeScript
- Equipes que valorizam type safety
- Desenvolvimento rápido
- Projetos que precisam de migrations automáticas
- APIs REST/GraphQL modernas
- Projetos com relacionamentos complexos

### Quando Considerar Alternativas

⚠️ **Sequelize se:**
- Projeto JavaScript puro
- Necessita máxima flexibilidade
- Equipe já experiente com Sequelize
- Projeto legado

⚠️ **TypeORM se:**
- Usa decorators extensivamente
- Necessita Active Record pattern
- Projeto complexo com muita customização
- Equipe experiente com Java/C# (similar ao Entity Framework)

⚠️ **Drizzle se:**
- Performance é crítica
- Bundle size é importante
- Prefere SQL-like syntax
- Projeto edge computing

⚠️ **Mongoose se:**
- Usa MongoDB exclusivamente
- Necessita flexibilidade de schema
- Projeto JavaScript puro
- Equipe experiente com MongoDB

### Tabela Comparativa Final

| Critério | Prisma | Sequelize | TypeORM | Drizzle | Mongoose |
|----------|--------|-----------|---------|---------|----------|
| **Type Safety** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Developer Experience** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Performance** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Flexibilidade** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Comunidade** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Migrations** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **Learning Curve** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |

---

[🔙 Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>