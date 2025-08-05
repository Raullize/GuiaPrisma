<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# 🔍 Conceitos Fundamentais do Prisma

## O que é um ORM?

ORM (Object-Relational Mapping) é uma técnica de programação que permite mapear dados entre sistemas incompatíveis usando linguagens de programação orientadas a objetos. Em termos simples, um ORM atua como uma ponte entre seu código JavaScript/TypeScript e o banco de dados relacional.

## Arquitetura do Prisma

O Prisma é composto por três componentes principais:

### 1. **Prisma Schema**
- Arquivo de configuração central (`schema.prisma`)
- Define modelos de dados, conexões de banco e geradores
- Linguagem declarativa própria do Prisma

### 2. **Prisma Client**
- Cliente auto-gerado e type-safe
- Fornece uma API intuitiva para queries
- Gerado automaticamente baseado no schema

### 3. **Prisma CLI**
- Ferramenta de linha de comando
- Gerencia migrations, gera o client, e muito mais
- Essencial para o workflow de desenvolvimento

## Principais Conceitos

### **Models (Modelos)**
Representam tabelas do banco de dados no schema do Prisma:

```prisma
model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
  posts Post[]
}
```

### **Fields (Campos)**
Propriedades dos modelos que correspondem às colunas da tabela:
- **Scalar fields**: String, Int, Boolean, DateTime, etc.
- **Relation fields**: Referências a outros modelos

### **Attributes (Atributos)**
Modificadores que adicionam metadados aos campos:
- `@id`: Campo de chave primária
- `@unique`: Campo único
- `@default()`: Valor padrão
- `@map()`: Mapeia para nome diferente no banco

### **Relations (Relacionamentos)**
Conexões entre modelos:
- **One-to-One**: Um usuário tem um perfil
- **One-to-Many**: Um usuário tem muitos posts
- **Many-to-Many**: Posts têm muitas categorias

### **Datasource**
Configura a conexão com o banco de dados:

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

### **Generator**
Configura o que será gerado pelo Prisma:

```prisma
generator client {
  provider = "prisma-client-js"
}
```

## Fluxo de Trabalho Típico

1. **Definir Schema**: Criar/modificar `schema.prisma`
2. **Gerar Migration**: `npx prisma migrate dev`
3. **Gerar Client**: `npx prisma generate`
4. **Usar Client**: Importar e usar em sua aplicação

## Vantagens do Prisma

✅ **Type Safety**: Queries completamente tipadas
✅ **Auto-completion**: IntelliSense completo
✅ **Migrations**: Versionamento automático do schema
✅ **Introspection**: Gera schema a partir de banco existente
✅ **Multi-database**: Suporte a vários SGBDs
✅ **Performance**: Queries otimizadas automaticamente

## Comparação: SQL vs Prisma

**SQL Tradicional:**
```sql
SELECT u.name, p.title 
FROM users u 
JOIN posts p ON u.id = p.user_id 
WHERE u.email = 'user@example.com';
```

**Prisma Client:**
```typescript
const userWithPosts = await prisma.user.findUnique({
  where: { email: 'user@example.com' },
  include: { posts: true }
});
```

---

[🔙 Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>