<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# Como Instalar e Configurar o Prisma

## Pré-requisitos

Antes de começar, certifique-se de ter:
- **Node.js** (versão 14.17.0 ou superior)
- **npm** ou **yarn** instalado
- Um banco de dados (PostgreSQL, MySQL, SQLite, SQL Server ou MongoDB)

## Instalação

### 1. Inicializar Projeto Node.js

```bash
# Criar novo projeto
mkdir meu-projeto-prisma
cd meu-projeto-prisma
npm init -y
```

### 2. Instalar Prisma

```bash
# Instalar Prisma CLI como dependência de desenvolvimento
npm install prisma --save-dev

# Instalar Prisma Client
npm install @prisma/client
```

### 3. Inicializar Prisma

```bash
# Inicializar configuração do Prisma
npx prisma init
```

Este comando criará:
- `prisma/` - Diretório com arquivos do Prisma
- `prisma/schema.prisma` - Schema principal
- `.env` - Variáveis de ambiente

## Configuração do Banco de Dados

### Configurar Variáveis de Ambiente

Edite o arquivo `.env` criado:

```env
# PostgreSQL
DATABASE_URL="postgresql://usuario:senha@localhost:5432/meudb?schema=public"

# MySQL
DATABASE_URL="mysql://usuario:senha@localhost:3306/meudb"

# SQLite
DATABASE_URL="file:./dev.db"

# SQL Server
DATABASE_URL="sqlserver://localhost:1433;database=meudb;user=usuario;password=senha;"
```

### Configurar Schema

Edite `prisma/schema.prisma`:

```prisma
// Gerador do cliente
generator client {
  provider = "prisma-client-js"
}

// Configuração do banco
datasource db {
  provider = "postgresql" // ou "mysql", "sqlite", "sqlserver"
  url      = env("DATABASE_URL")
}

// Exemplo de modelo
model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

## Configurações por Banco de Dados

### PostgreSQL

**Instalação do PostgreSQL:**
```bash
# Ubuntu/Debian
sudo apt-get install postgresql postgresql-contrib

# macOS (com Homebrew)
brew install postgresql

# Windows: Baixar do site oficial
```

**Configuração:**
```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

### MySQL

**Configuração:**
```prisma
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}
```

### SQLite (Ideal para desenvolvimento)

**Configuração:**
```prisma
datasource db {
  provider = "sqlite"
  url      = "file:./dev.db"
}
```

### MongoDB

**Configuração:**
```prisma
datasource db {
  provider = "mongodb"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
  previewFeatures = ["mongoDb"]
}
```

## Primeira Migration

```bash
# Criar e aplicar primeira migration
npx prisma migrate dev --name init
```

Este comando:
1. Cria uma migration baseada no schema
2. Aplica a migration no banco
3. Gera o Prisma Client

## Gerar Prisma Client

```bash
# Gerar cliente (necessário após mudanças no schema)
npx prisma generate
```

## Estrutura de Arquivos Final

```
meu-projeto-prisma/
├── prisma/
│   ├── migrations/
│   │   └── 20231201000000_init/
│   │       └── migration.sql
│   └── schema.prisma
├── node_modules/
├── .env
├── package.json
└── package-lock.json
```

## Configurações Avançadas

### Múltiplos Schemas (PostgreSQL)

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  schemas  = ["public", "admin"]
}

model User {
  id   Int    @id @default(autoincrement())
  name String
  
  @@schema("public")
}
```

### Configuração de Output

```prisma
generator client {
  provider = "prisma-client-js"
  output   = "./generated/client"
}
```

### Preview Features

```prisma
generator client {
  provider = "prisma-client-js"
  previewFeatures = ["fullTextSearch", "metrics"]
}
```

## Verificação da Instalação

```bash
# Verificar status da conexão
npx prisma db pull

# Abrir Prisma Studio
npx prisma studio
```

## Troubleshooting Comum

### Erro de Conexão
- Verificar se o banco está rodando
- Conferir credenciais no `.env`
- Testar conectividade de rede

### Erro de Permissões
- Verificar permissões do usuário do banco
- Conferir se o usuário pode criar/modificar tabelas

### Erro de SSL (PostgreSQL)
```env
DATABASE_URL="postgresql://user:pass@host:5432/db?sslmode=require"
```

---

[Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>