<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=header&animation=twinkling"/>

# Comandos Essenciais do Prisma CLI

## Instalação da CLI

```bash
# Instalar globalmente
npm install -g prisma

# Ou usar npx (recomendado)
npx prisma --help
```

## Comandos de Inicialização

### `prisma init`
Inicializa um novo projeto Prisma:

```bash
# Inicializar com PostgreSQL (padrão)
npx prisma init

# Inicializar com provider específico
npx prisma init --datasource-provider mysql
npx prisma init --datasource-provider sqlite
npx prisma init --datasource-provider sqlserver
npx prisma init --datasource-provider mongodb
```

**O que este comando faz:**
- Cria diretório `prisma/`
- Gera arquivo `schema.prisma`
- Cria arquivo `.env` com `DATABASE_URL`

## Comandos de Migration

### `prisma migrate dev`
Cria e aplica migrations em desenvolvimento:

```bash
# Criar migration com nome
npx prisma migrate dev --name add_user_table

# Aplicar migrations pendentes
npx prisma migrate dev

# Resetar banco e aplicar todas as migrations
npx prisma migrate dev --reset
```

### `prisma migrate deploy`
Aplica migrations em produção:

```bash
# Aplicar migrations em produção
npx prisma migrate deploy
```

### `prisma migrate status`
Verifica status das migrations:

```bash
# Ver status das migrations
npx prisma migrate status
```

### `prisma migrate reset`
Reseta o banco de dados:

```bash
# Resetar banco completamente
npx prisma migrate reset

# Resetar sem confirmação
npx prisma migrate reset --force
```

### `prisma migrate resolve`
Marca migration como aplicada:

```bash
# Marcar migration específica como aplicada
npx prisma migrate resolve --applied 20231201000000_init

# Marcar como com falha
npx prisma migrate resolve --rolled-back 20231201000000_init
```

## Comandos de Geração

### `prisma generate`
Gera o Prisma Client:

```bash
# Gerar cliente
npx prisma generate

# Gerar com watch mode
npx prisma generate --watch
```

## Comandos de Banco de Dados

### `prisma db pull`
Gera schema a partir do banco existente:

```bash
# Fazer introspection do banco
npx prisma db pull

# Forçar sobrescrita do schema
npx prisma db pull --force
```

### `prisma db push`
Aplica mudanças do schema diretamente no banco:

```bash
# Aplicar schema no banco (sem migrations)
npx prisma db push

# Forçar aplicação (cuidado!)
npx prisma db push --force-reset

# Aceitar perda de dados
npx prisma db push --accept-data-loss
```

### `prisma db seed`
Executa script de seed:

```bash
# Executar seed
npx prisma db seed
```

**Configuração no package.json:**
```json
{
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```

## Comandos de Validação

### `prisma validate`
Valida o schema:

```bash
# Validar schema
npx prisma validate
```

### `prisma format`
Formata o arquivo schema:

```bash
# Formatar schema.prisma
npx prisma format
```

## Prisma Studio

### `prisma studio`
Abre interface visual para o banco:

```bash
# Abrir Prisma Studio
npx prisma studio

# Especificar porta
npx prisma studio --port 5555

# Especificar browser
npx prisma studio --browser firefox
```

## Comandos de Debug

### Variáveis de Ambiente para Debug

```bash
# Debug de queries
DEBUG="prisma:query" npx prisma studio

# Debug completo
DEBUG="prisma:*" npm run dev

# Debug de migrations
DEBUG="prisma:migrate" npx prisma migrate dev
```

### Logs Detalhados

```bash
# Executar com logs verbosos
npx prisma migrate dev --verbose
npx prisma generate --verbose
```

## Workflows Comuns

### Desenvolvimento Local

```bash
# 1. Modificar schema.prisma
# 2. Criar e aplicar migration
npx prisma migrate dev --name describe_changes

# 3. Gerar cliente (automático com migrate dev)
# npx prisma generate

# 4. Verificar no Studio
npx prisma studio
```

### Prototipagem Rápida

```bash
# 1. Modificar schema.prisma
# 2. Aplicar mudanças diretamente
npx prisma db push

# 3. Gerar cliente
npx prisma generate
```

### Deploy em Produção

```bash
# 1. Aplicar migrations
npx prisma migrate deploy

# 2. Gerar cliente
npx prisma generate

# 3. Executar seed (se necessário)
npx prisma db seed
```

### Banco Existente

```bash
# 1. Fazer introspection
npx prisma db pull

# 2. Gerar cliente
npx prisma generate

# 3. Criar baseline migration
npx prisma migrate dev --name baseline
```

## Flags Úteis

### Flags Globais

| Flag | Descrição |
|------|----------|
| `--help` | Mostra ajuda |
| `--version` | Mostra versão |
| `--schema` | Especifica caminho do schema |
| `--verbose` | Output detalhado |

### Flags de Migration

| Flag | Descrição |
|------|----------|
| `--name` | Nome da migration |
| `--force` | Força execução |
| `--reset` | Reseta banco antes |
| `--accept-data-loss` | Aceita perda de dados |
| `--create-only` | Só cria, não aplica |

### Flags de Generate

| Flag | Descrição |
|------|----------|
| `--watch` | Regenera automaticamente |
| `--data-proxy` | Gera para Data Proxy |

## Scripts NPM Recomendados

Adicione ao seu `package.json`:

```json
{
  "scripts": {
    "db:migrate": "prisma migrate dev",
    "db:deploy": "prisma migrate deploy",
    "db:reset": "prisma migrate reset",
    "db:seed": "prisma db seed",
    "db:studio": "prisma studio",
    "db:generate": "prisma generate",
    "db:pull": "prisma db pull",
    "db:push": "prisma db push",
    "db:validate": "prisma validate",
    "db:format": "prisma format"
  }
}
```

## Troubleshooting

### Problemas Comuns

**Migration falhou:**
```bash
# Ver status
npx prisma migrate status

# Resolver migration
npx prisma migrate resolve --applied MIGRATION_NAME
```

**Schema fora de sincronia:**
```bash
# Resetar e reaplicar
npx prisma migrate reset
npx prisma migrate dev
```

**Cliente desatualizado:**
```bash
# Regenerar cliente
npx prisma generate
```

**Banco não conecta:**
```bash
# Testar conexão
npx prisma db pull
```

### Comandos de Diagnóstico

```bash
# Informações do ambiente
npx prisma version

# Validar configuração
npx prisma validate

# Status das migrations
npx prisma migrate status

# Debug de conexão
DEBUG="prisma:*" npx prisma db pull
```

## Dicas de Produtividade

### 1. Aliases
Crie aliases no seu shell:

```bash
# .bashrc ou .zshrc
alias pm="npx prisma migrate"
alias pg="npx prisma generate"
alias ps="npx prisma studio"
```

### 2. Scripts de Desenvolvimento

```bash
#!/bin/bash
# dev-reset.sh
echo "Resetando banco de desenvolvimento..."
npx prisma migrate reset --force
echo "Executando seed..."
npx prisma db seed
echo "Abrindo Studio..."
npx prisma studio
```

### 3. Hooks de Git

```bash
# .git/hooks/pre-commit
#!/bin/sh
npx prisma validate
npx prisma format
```

---

[Voltar ao índice principal](../README.md)

<img width=100% src="https://capsule-render.vercel.app/api?type=waving&color=2D3748&height=120&section=footer"/>