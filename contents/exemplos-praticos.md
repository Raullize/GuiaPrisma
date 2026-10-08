<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# Exemplos Práticos: Projetos Completos

## Projeto 1: Blog API com Next.js

### Estrutura do Projeto

```
blog-api/
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── src/
│   ├── lib/
│   │   ├── prisma.ts
│   │   └── validations.ts
│   ├── pages/
│   │   └── api/
│   │       ├── posts/
│   │       ├── users/
│   │       └── auth/
│   └── types/
├── package.json
└── .env
```

### Schema do Blog

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  avatar    String?
  bio       String?
  role      Role     @default(USER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  posts    Post[]
  comments Comment[]
  likes    Like[]

  @@map("users")
}

model Category {
  id          Int    @id @default(autoincrement())
  name        String @unique
  slug        String @unique
  description String?
  color       String @default("#6366f1")

  posts Post[]

  @@map("categories")
}

model Post {
  id          Int         @id @default(autoincrement())
  title       String
  slug        String      @unique
  content     String
  excerpt     String?
  coverImage  String?
  published   Boolean     @default(false)
  publishedAt DateTime?
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt
  viewCount   Int         @default(0)

  authorId   Int
  author     User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  categoryId Int
  category   Category @relation(fields: [categoryId], references: [id])

  comments Comment[]
  likes    Like[]
  tags     TagOnPost[]

  @@map("posts")
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  color String @default("#64748b")

  posts TagOnPost[]

  @@map("tags")
}

model TagOnPost {
  postId Int
  tagId  Int

  post Post @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag  Tag  @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([postId, tagId])
  @@map("tags_on_posts")
}

model Comment {
  id        Int      @id @default(autoincrement())
  content   String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  authorId Int
  author   User @relation(fields: [authorId], references: [id], onDelete: Cascade)
  postId   Int
  post     Post @relation(fields: [postId], references: [id], onDelete: Cascade)

  parentId Int?
  parent   Comment?  @relation("CommentReplies", fields: [parentId], references: [id])
  replies  Comment[] @relation("CommentReplies")

  @@map("comments")
}

model Like {
  id        Int      @id @default(autoincrement())
  createdAt DateTime @default(now())

  userId Int
  user   User @relation(fields: [userId], references: [id], onDelete: Cascade)
  postId Int
  post   Post @relation(fields: [postId], references: [id], onDelete: Cascade)

  @@unique([userId, postId])
  @@map("likes")
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

### API Routes - Posts

```typescript
// src/pages/api/posts/index.ts
import type { NextApiRequest, NextApiResponse } from 'next'
import { prisma } from '../../../lib/prisma'
import { z } from 'zod'

const createPostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
  excerpt: z.string().max(300).optional(),
  coverImage: z.string().url().optional(),
  categoryId: z.number().int().positive(),
  tags: z.array(z.string()).optional(),
  published: z.boolean().default(false),
})

const querySchema = z.object({
  page: z.string().transform(Number).default('1'),
  limit: z.string().transform(Number).default('10'),
  category: z.string().optional(),
  tag: z.string().optional(),
  search: z.string().optional(),
  published: z.string().transform(Boolean).optional(),
})

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method === 'GET') {
    try {
      const { page, limit, category, tag, search, published } = querySchema.parse(req.query)
      
      const where: any = {}
      
      if (published !== undefined) {
        where.published = published
      }
      
      if (category) {
        where.category = {
          slug: category
        }
      }
      
      if (tag) {
        where.tags = {
          some: {
            tag: {
              name: tag
            }
          }
        }
      }
      
      if (search) {
        where.OR = [
          { title: { contains: search, mode: 'insensitive' } },
          { content: { contains: search, mode: 'insensitive' } },
          { excerpt: { contains: search, mode: 'insensitive' } },
        ]
      }
      
      const [posts, total] = await Promise.all([
        prisma.post.findMany({
          where,
          skip: (page - 1) * limit,
          take: limit,
          include: {
            author: {
              select: {
                id: true,
                name: true,
                avatar: true,
              },
            },
            category: {
              select: {
                id: true,
                name: true,
                slug: true,
                color: true,
              },
            },
            tags: {
              include: {
                tag: {
                  select: {
                    id: true,
                    name: true,
                    color: true,
                  },
                },
              },
            },
            _count: {
              select: {
                comments: true,
                likes: true,
              },
            },
          },
          orderBy: {
            createdAt: 'desc',
          },
        }),
        prisma.post.count({ where }),
      ])
      
      res.json({
        posts: posts.map(post => ({
          ...post,
          tags: post.tags.map(t => t.tag),
        })),
        pagination: {
          page,
          limit,
          total,
          pages: Math.ceil(total / limit),
        },
      })
    } catch (error) {
      console.error('Error fetching posts:', error)
      res.status(500).json({ error: 'Erro interno do servidor' })
    }
  } else if (req.method === 'POST') {
    try {
      const data = createPostSchema.parse(req.body)
      const { tags, ...postData } = data
      
      // Gerar slug único
      const baseSlug = data.title
        .toLowerCase()
        .replace(/[^a-z0-9]+/g, '-')
        .replace(/^-|-$/g, '')
      
      let slug = baseSlug
      let counter = 1
      
      while (await prisma.post.findUnique({ where: { slug } })) {
        slug = `${baseSlug}-${counter}`
        counter++
      }
      
      const post = await prisma.post.create({
        data: {
          ...postData,
          slug,
          authorId: 1, // TODO: Pegar do token de autenticação
          publishedAt: data.published ? new Date() : null,
          tags: tags ? {
            create: tags.map(tagName => ({
              tag: {
                connectOrCreate: {
                  where: { name: tagName },
                  create: { name: tagName },
                },
              },
            })),
          } : undefined,
        },
        include: {
          author: {
            select: {
              id: true,
              name: true,
              avatar: true,
            },
          },
          category: true,
          tags: {
            include: {
              tag: true,
            },
          },
        },
      })
      
      res.status(201).json({
        ...post,
        tags: post.tags.map(t => t.tag),
      })
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          error: 'Dados inválidos',
          details: error.errors,
        })
      }
      console.error('Error creating post:', error)
      res.status(500).json({ error: 'Erro interno do servidor' })
    }
  } else {
    res.setHeader('Allow', ['GET', 'POST'])
    res.status(405).end(`Method ${req.method} Not Allowed`)
  }
}
```

```typescript
// src/pages/api/posts/[slug].ts
import type { NextApiRequest, NextApiResponse } from 'next'
import { prisma } from '../../../lib/prisma'

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  const { slug } = req.query
  
  if (req.method === 'GET') {
    try {
      const post = await prisma.post.findUnique({
        where: { slug: slug as string },
        include: {
          author: {
            select: {
              id: true,
              name: true,
              avatar: true,
              bio: true,
            },
          },
          category: true,
          tags: {
            include: {
              tag: true,
            },
          },
          comments: {
            where: {
              parentId: null, // Apenas comentários principais
            },
            include: {
              author: {
                select: {
                  id: true,
                  name: true,
                  avatar: true,
                },
              },
              replies: {
                include: {
                  author: {
                    select: {
                      id: true,
                      name: true,
                      avatar: true,
                    },
                  },
                },
                orderBy: {
                  createdAt: 'asc',
                },
              },
            },
            orderBy: {
              createdAt: 'desc',
            },
          },
          _count: {
            select: {
              likes: true,
              comments: true,
            },
          },
        },
      })
      
      if (!post) {
        return res.status(404).json({ error: 'Post não encontrado' })
      }
      
      // Incrementar visualizações
      await prisma.post.update({
        where: { id: post.id },
        data: {
          viewCount: {
            increment: 1,
          },
        },
      })
      
      res.json({
        ...post,
        tags: post.tags.map(t => t.tag),
      })
    } catch (error) {
      console.error('Error fetching post:', error)
      res.status(500).json({ error: 'Erro interno do servidor' })
    }
  } else {
    res.setHeader('Allow', ['GET'])
    res.status(405).end(`Method ${req.method} Not Allowed`)
  }
}
```

## Projeto 2: E-commerce API com Express

### Schema do E-commerce

```prisma
// prisma/schema.prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  phone     String?
  avatar    String?
  role      UserRole @default(CUSTOMER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  addresses Address[]
  orders    Order[]
  cart      CartItem[]
  reviews   Review[]
  wishlist  WishlistItem[]

  @@map("users")
}

model Category {
  id          Int    @id @default(autoincrement())
  name        String @unique
  slug        String @unique
  description String?
  image       String?
  parentId    Int?
  parent      Category?  @relation("CategoryHierarchy", fields: [parentId], references: [id])
  children    Category[] @relation("CategoryHierarchy")

  products Product[]

  @@map("categories")
}

model Product {
  id          Int     @id @default(autoincrement())
  name        String
  slug        String  @unique
  description String?
  price       Decimal @db.Decimal(10, 2)
  comparePrice Decimal? @db.Decimal(10, 2)
  sku         String  @unique
  stock       Int     @default(0)
  weight      Decimal? @db.Decimal(8, 2)
  dimensions  String?
  active      Boolean @default(true)
  featured    Boolean @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  categoryId Int
  category   Category @relation(fields: [categoryId], references: [id])

  images       ProductImage[]
  variants     ProductVariant[]
  orderItems   OrderItem[]
  cartItems    CartItem[]
  reviews      Review[]
  wishlistItems WishlistItem[]

  @@map("products")
}

model ProductImage {
  id        Int    @id @default(autoincrement())
  url       String
  alt       String?
  order     Int    @default(0)
  productId Int
  product   Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@map("product_images")
}

model ProductVariant {
  id        Int     @id @default(autoincrement())
  name      String  // Ex: "Cor", "Tamanho"
  value     String  // Ex: "Azul", "M"
  price     Decimal? @db.Decimal(10, 2)
  stock     Int     @default(0)
  sku       String  @unique
  productId Int
  product   Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@map("product_variants")
}

model Address {
  id           Int     @id @default(autoincrement())
  street       String
  number       String
  complement   String?
  neighborhood String
  city         String
  state        String
  zipCode      String
  country      String  @default("Brasil")
  isDefault    Boolean @default(false)

  userId Int
  user   User @relation(fields: [userId], references: [id], onDelete: Cascade)

  orders Order[]

  @@map("addresses")
}

model Order {
  id            Int         @id @default(autoincrement())
  orderNumber   String      @unique
  status        OrderStatus @default(PENDING)
  total         Decimal     @db.Decimal(10, 2)
  subtotal      Decimal     @db.Decimal(10, 2)
  shipping      Decimal     @db.Decimal(10, 2)
  tax           Decimal     @db.Decimal(10, 2)
  discount      Decimal     @default(0) @db.Decimal(10, 2)
  paymentMethod String?
  paymentStatus PaymentStatus @default(PENDING)
  notes         String?
  createdAt     DateTime    @default(now())
  updatedAt     DateTime    @updatedAt

  userId    Int
  user      User    @relation(fields: [userId], references: [id])
  addressId Int
  address   Address @relation(fields: [addressId], references: [id])

  items OrderItem[]

  @@map("orders")
}

model OrderItem {
  id       Int     @id @default(autoincrement())
  quantity Int
  price    Decimal @db.Decimal(10, 2)
  total    Decimal @db.Decimal(10, 2)

  orderId   Int
  order     Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  productId Int
  product   Product @relation(fields: [productId], references: [id])

  @@map("order_items")
}

model CartItem {
  id        Int      @id @default(autoincrement())
  quantity  Int
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  userId    Int
  user      User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  productId Int
  product   Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@unique([userId, productId])
  @@map("cart_items")
}

model Review {
  id        Int      @id @default(autoincrement())
  rating    Int      // 1-5
  title     String?
  comment   String?
  verified  Boolean  @default(false)
  createdAt DateTime @default(now())

  userId    Int
  user      User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  productId Int
  product   Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@unique([userId, productId])
  @@map("reviews")
}

model WishlistItem {
  id        Int      @id @default(autoincrement())
  createdAt DateTime @default(now())

  userId    Int
  user      User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  productId Int
  product   Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@unique([userId, productId])
  @@map("wishlist_items")
}

enum UserRole {
  CUSTOMER
  ADMIN
  MANAGER
}

enum OrderStatus {
  PENDING
  CONFIRMED
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
  REFUNDED
}

enum PaymentStatus {
  PENDING
  PAID
  FAILED
  REFUNDED
}
```

### Service Layer - Products

```typescript
// src/services/productService.ts
import { prisma } from '../lib/prisma'
import { Prisma } from '@prisma/client'

export class ProductService {
  async findMany(params: {
    page?: number
    limit?: number
    category?: string
    search?: string
    minPrice?: number
    maxPrice?: number
    featured?: boolean
    sortBy?: 'name' | 'price' | 'created' | 'popularity'
    sortOrder?: 'asc' | 'desc'
  }) {
    const {
      page = 1,
      limit = 20,
      category,
      search,
      minPrice,
      maxPrice,
      featured,
      sortBy = 'created',
      sortOrder = 'desc',
    } = params

    const where: Prisma.ProductWhereInput = {
      active: true,
    }

    if (category) {
      where.category = {
        slug: category,
      }
    }

    if (search) {
      where.OR = [
        { name: { contains: search, mode: 'insensitive' } },
        { description: { contains: search, mode: 'insensitive' } },
        { sku: { contains: search, mode: 'insensitive' } },
      ]
    }

    if (minPrice !== undefined || maxPrice !== undefined) {
      where.price = {}
      if (minPrice !== undefined) where.price.gte = minPrice
      if (maxPrice !== undefined) where.price.lte = maxPrice
    }

    if (featured !== undefined) {
      where.featured = featured
    }

    const orderBy: Prisma.ProductOrderByWithRelationInput = {}
    switch (sortBy) {
      case 'name':
        orderBy.name = sortOrder
        break
      case 'price':
        orderBy.price = sortOrder
        break
      case 'created':
        orderBy.createdAt = sortOrder
        break
      case 'popularity':
        // Ordenar por número de pedidos
        orderBy.orderItems = {
          _count: sortOrder,
        }
        break
    }

    const [products, total] = await Promise.all([
      prisma.product.findMany({
        where,
        skip: (page - 1) * limit,
        take: limit,
        include: {
          category: {
            select: {
              id: true,
              name: true,
              slug: true,
            },
          },
          images: {
            orderBy: {
              order: 'asc',
            },
            take: 1,
          },
          variants: {
            select: {
              id: true,
              name: true,
              value: true,
              price: true,
            },
          },
          _count: {
            select: {
              reviews: true,
              orderItems: true,
            },
          },
        },
        orderBy,
      }),
      prisma.product.count({ where }),
    ])

    // Calcular rating médio
    const productsWithRating = await Promise.all(
      products.map(async (product) => {
        const avgRating = await prisma.review.aggregate({
          where: { productId: product.id },
          _avg: { rating: true },
        })

        return {
          ...product,
          averageRating: avgRating._avg.rating || 0,
          reviewCount: product._count.reviews,
          salesCount: product._count.orderItems,
        }
      })
    )

    return {
      products: productsWithRating,
      pagination: {
        page,
        limit,
        total,
        pages: Math.ceil(total / limit),
      },
    }
  }

  async findBySlug(slug: string) {
    const product = await prisma.product.findUnique({
      where: { slug },
      include: {
        category: {
          include: {
            parent: true,
          },
        },
        images: {
          orderBy: {
            order: 'asc',
          },
        },
        variants: true,
        reviews: {
          include: {
            user: {
              select: {
                id: true,
                name: true,
                avatar: true,
              },
            },
          },
          orderBy: {
            createdAt: 'desc',
          },
        },
        _count: {
          select: {
            reviews: true,
            orderItems: true,
          },
        },
      },
    })

    if (!product) {
      return null
    }

    // Calcular estatísticas de reviews
    const reviewStats = await prisma.review.groupBy({
      by: ['rating'],
      where: { productId: product.id },
      _count: { rating: true },
    })

    const avgRating = await prisma.review.aggregate({
      where: { productId: product.id },
      _avg: { rating: true },
    })

    // Produtos relacionados
    const relatedProducts = await prisma.product.findMany({
      where: {
        categoryId: product.categoryId,
        id: { not: product.id },
        active: true,
      },
      take: 4,
      include: {
        images: {
          take: 1,
          orderBy: { order: 'asc' },
        },
      },
    })

    return {
      ...product,
      averageRating: avgRating._avg.rating || 0,
      reviewStats,
      relatedProducts,
    }
  }

  async create(data: Prisma.ProductCreateInput) {
    return prisma.product.create({
      data,
      include: {
        category: true,
        images: true,
        variants: true,
      },
    })
  }

  async updateStock(productId: number, quantity: number) {
    return prisma.product.update({
      where: { id: productId },
      data: {
        stock: {
          increment: quantity,
        },
      },
    })
  }
}
```

### Controller - Cart

```typescript
// src/controllers/cartController.ts
import { Request, Response } from 'express'
import { prisma } from '../lib/prisma'
import { z } from 'zod'

const addToCartSchema = z.object({
  productId: z.number().int().positive(),
  quantity: z.number().int().positive().max(10),
})

const updateCartSchema = z.object({
  quantity: z.number().int().min(0).max(10),
})

export class CartController {
  async getCart(req: Request, res: Response) {
    try {
      const userId = req.user.id // Assumindo middleware de auth

      const cartItems = await prisma.cartItem.findMany({
        where: { userId },
        include: {
          product: {
            include: {
              images: {
                take: 1,
                orderBy: { order: 'asc' },
              },
              variants: true,
            },
          },
        },
      })

      const total = cartItems.reduce(
        (sum, item) => sum + Number(item.product.price) * item.quantity,
        0
      )

      res.json({
        items: cartItems,
        total,
        itemCount: cartItems.reduce((sum, item) => sum + item.quantity, 0),
      })
    } catch (error) {
      console.error('Error fetching cart:', error)
      res.status(500).json({ error: 'Erro interno do servidor' })
    }
  }

  async addToCart(req: Request, res: Response) {
    try {
      const userId = req.user.id
      const { productId, quantity } = addToCartSchema.parse(req.body)

      // Verificar se produto existe e tem estoque
      const product = await prisma.product.findUnique({
        where: { id: productId },
      })

      if (!product) {
        return res.status(404).json({ error: 'Produto não encontrado' })
      }

      if (!product.active) {
        return res.status(400).json({ error: 'Produto não disponível' })
      }

      if (product.stock < quantity) {
        return res.status(400).json({
          error: 'Estoque insuficiente',
          available: product.stock,
        })
      }

      // Adicionar ou atualizar item no carrinho
      const cartItem = await prisma.cartItem.upsert({
        where: {
          userId_productId: {
            userId,
            productId,
          },
        },
        update: {
          quantity: {
            increment: quantity,
          },
        },
        create: {
          userId,
          productId,
          quantity,
        },
        include: {
          product: {
            include: {
              images: {
                take: 1,
                orderBy: { order: 'asc' },
              },
            },
          },
        },
      })

      res.status(201).json(cartItem)
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          error: 'Dados inválidos',
          details: error.errors,
        })
      }
      console.error('Error adding to cart:', error)
      res.status(500).json({ error: 'Erro interno do servidor' })
    }
  }

  async updateCartItem(req: Request, res: Response) {
    try {
      const userId = req.user.id
      const { id } = req.params
      const { quantity } = updateCartSchema.parse(req.body)

      if (quantity === 0) {
        // Remover item do carrinho
        await prisma.cartItem.delete({
          where: {
            id: Number(id),
            userId, // Garantir que o usuário só pode modificar seus próprios itens
          },
        })
        return res.status(204).send()
      }

      // Verificar estoque
      const cartItem = await prisma.cartItem.findFirst({
        where: {
          id: Number(id),
          userId,
        },
        include: {
          product: true,
        },
      })

      if (!cartItem) {
        return res.status(404).json({ error: 'Item não encontrado' })
      }

      if (cartItem.product.stock < quantity) {
        return res.status(400).json({
          error: 'Estoque insuficiente',
          available: cartItem.product.stock,
        })
      }

      const updatedItem = await prisma.cartItem.update({
        where: { id: Number(id) },
        data: { quantity },
        include: {
          product: {
            include: {
              images: {
                take: 1,
                orderBy: { order: 'asc' },
              },
            },
          },
        },
      })

      res.json(updatedItem)
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          error: 'Dados inválidos',
          details: error.errors,
        })
      }
      console.error('Error updating cart item:', error)
      res.status(500).json({ error: 'Erro interno do servidor' })
    }
  }

  async clearCart(req: Request, res: Response) {
    try {
      const userId = req.user.id

      await prisma.cartItem.deleteMany({
        where: { userId },
      })

      res.status(204).send()
    } catch (error) {
      console.error('Error clearing cart:', error)
      res.status(500).json({ error: 'Erro interno do servidor' })
    }
  }
}
```

## Projeto 3: Sistema de Reservas com NestJS

### Schema de Reservas

```prisma
model Hotel {
  id          Int      @id @default(autoincrement())
  name        String
  description String?
  address     String
  city        String
  state       String
  country     String
  zipCode     String
  phone       String?
  email       String?
  website     String?
  rating      Decimal? @db.Decimal(2, 1)
  amenities   String[] // ["wifi", "pool", "gym"]
  images      String[]
  active      Boolean  @default(true)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  rooms        Room[]
  reservations Reservation[]
  reviews      HotelReview[]

  @@map("hotels")
}

model RoomType {
  id          Int     @id @default(autoincrement())
  name        String  // "Standard", "Deluxe", "Suite"
  description String?
  maxGuests   Int
  bedType     String  // "Single", "Double", "Queen", "King"
  size        Int?    // em m²
  amenities   String[] // ["tv", "minibar", "balcony"]

  rooms Room[]

  @@map("room_types")
}

model Room {
  id         Int     @id @default(autoincrement())
  number     String
  floor      Int?
  pricePerNight Decimal @db.Decimal(10, 2)
  active     Boolean @default(true)

  hotelId    Int
  hotel      Hotel    @relation(fields: [hotelId], references: [id])
  roomTypeId Int
  roomType   RoomType @relation(fields: [roomTypeId], references: [id])

  reservations ReservationRoom[]

  @@unique([hotelId, number])
  @@map("rooms")
}

model Guest {
  id        Int      @id @default(autoincrement())
  firstName String
  lastName  String
  email     String   @unique
  phone     String?
  document  String?  // CPF, Passport, etc.
  birthDate DateTime?
  nationality String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  reservations Reservation[]

  @@map("guests")
}

model Reservation {
  id          Int               @id @default(autoincrement())
  checkIn     DateTime
  checkOut    DateTime
  guests      Int
  status      ReservationStatus @default(PENDING)
  totalAmount Decimal           @db.Decimal(10, 2)
  paidAmount  Decimal           @default(0) @db.Decimal(10, 2)
  notes       String?
  createdAt   DateTime          @default(now())
  updatedAt   DateTime          @updatedAt

  guestId Int
  guest   Guest @relation(fields: [guestId], references: [id])
  hotelId Int
  hotel   Hotel @relation(fields: [hotelId], references: [id])

  rooms    ReservationRoom[]
  payments Payment[]

  @@map("reservations")
}

model ReservationRoom {
  id            Int @id @default(autoincrement())
  pricePerNight Decimal @db.Decimal(10, 2)
  nights        Int
  subtotal      Decimal @db.Decimal(10, 2)

  reservationId Int
  reservation   Reservation @relation(fields: [reservationId], references: [id], onDelete: Cascade)
  roomId        Int
  room          Room        @relation(fields: [roomId], references: [id])

  @@map("reservation_rooms")
}

model Payment {
  id            Int           @id @default(autoincrement())
  amount        Decimal       @db.Decimal(10, 2)
  method        PaymentMethod
  status        PaymentStatus @default(PENDING)
  transactionId String?
  processedAt   DateTime?
  createdAt     DateTime      @default(now())

  reservationId Int
  reservation   Reservation @relation(fields: [reservationId], references: [id])

  @@map("payments")
}

model HotelReview {
  id        Int      @id @default(autoincrement())
  rating    Int      // 1-5
  title     String?
  comment   String?
  createdAt DateTime @default(now())

  guestId Int
  guest   Guest @relation(fields: [guestId], references: [id])
  hotelId Int
  hotel   Hotel @relation(fields: [hotelId], references: [id])

  @@unique([guestId, hotelId])
  @@map("hotel_reviews")
}

enum ReservationStatus {
  PENDING
  CONFIRMED
  CHECKED_IN
  CHECKED_OUT
  CANCELLED
  NO_SHOW
}

enum PaymentMethod {
  CREDIT_CARD
  DEBIT_CARD
  PIX
  BANK_TRANSFER
  CASH
}

enum PaymentStatus {
  PENDING
  PAID
  FAILED
  REFUNDED
}
```

### Service - Availability

```typescript
// src/reservations/availability.service.ts
import { Injectable } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'
import { addDays, format } from 'date-fns'

@Injectable()
export class AvailabilityService {
  constructor(private prisma: PrismaService) {}

  async checkAvailability(params: {
    hotelId: number
    checkIn: Date
    checkOut: Date
    guests: number
    roomTypeId?: number
  }) {
    const { hotelId, checkIn, checkOut, guests, roomTypeId } = params

    // Buscar todos os quartos do hotel
    const rooms = await this.prisma.room.findMany({
      where: {
        hotelId,
        active: true,
        roomType: {
          maxGuests: {
            gte: guests,
          },
          ...(roomTypeId && { id: roomTypeId }),
        },
      },
      include: {
        roomType: true,
        reservations: {
          where: {
            reservation: {
              status: {
                in: ['CONFIRMED', 'CHECKED_IN'],
              },
              OR: [
                {
                  checkIn: {
                    lt: checkOut,
                  },
                  checkOut: {
                    gt: checkIn,
                  },
                },
              ],
            },
          },
          include: {
            reservation: true,
          },
        },
      },
    })

    // Filtrar quartos disponíveis
    const availableRooms = rooms.filter(room => {
      return room.reservations.length === 0
    })

    // Agrupar por tipo de quarto
    const roomsByType = availableRooms.reduce((acc, room) => {
      const typeId = room.roomType.id
      if (!acc[typeId]) {
        acc[typeId] = {
          roomType: room.roomType,
          rooms: [],
          minPrice: room.pricePerNight,
          maxPrice: room.pricePerNight,
        }
      }
      acc[typeId].rooms.push(room)
      acc[typeId].minPrice = acc[typeId].minPrice < room.pricePerNight 
        ? acc[typeId].minPrice 
        : room.pricePerNight
      acc[typeId].maxPrice = acc[typeId].maxPrice > room.pricePerNight 
        ? acc[typeId].maxPrice 
        : room.pricePerNight
      return acc
    }, {})

    return Object.values(roomsByType)
  }

  async getCalendarAvailability(hotelId: number, month: number, year: number) {
    const startDate = new Date(year, month - 1, 1)
    const endDate = new Date(year, month, 0)

    const reservations = await this.prisma.reservation.findMany({
      where: {
        hotelId,
        status: {
          in: ['CONFIRMED', 'CHECKED_IN'],
        },
        OR: [
          {
            checkIn: {
              gte: startDate,
              lte: endDate,
            },
          },
          {
            checkOut: {
              gte: startDate,
              lte: endDate,
            },
          },
          {
            checkIn: {
              lt: startDate,
            },
            checkOut: {
              gt: endDate,
            },
          },
        ],
      },
      include: {
        rooms: {
          include: {
            room: {
              include: {
                roomType: true,
              },
            },
          },
        },
      },
    })

    const totalRooms = await this.prisma.room.count({
      where: {
        hotelId,
        active: true,
      },
    })

    // Calcular ocupação por dia
    const calendar = []
    let currentDate = new Date(startDate)

    while (currentDate <= endDate) {
      const occupiedRooms = reservations.filter(reservation => {
        return currentDate >= reservation.checkIn && currentDate < reservation.checkOut
      }).reduce((sum, reservation) => sum + reservation.rooms.length, 0)

      calendar.push({
        date: format(currentDate, 'yyyy-MM-dd'),
        occupiedRooms,
        availableRooms: totalRooms - occupiedRooms,
        occupancyRate: (occupiedRooms / totalRooms) * 100,
      })

      currentDate = addDays(currentDate, 1)
    }

    return calendar
  }
}
```

---

[Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>