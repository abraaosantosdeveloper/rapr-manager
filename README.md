## 💼 RaprManager

Gestão de estoque e operações para pequenas lojas — simples, rastreável e sem planilhas.

> Substitua Excel por um sistema com controle real de produtos, movimentações e usuários.

## Objetivo do Produto

O RaprManager substitui o uso de planilhas por uma aplicação web com:

- controle de estoque com rastreabilidade
- cadastro de produtos e gestão de ciclo de vida
- registro auditável de movimentações (entrada e saída)
- organização de recursos internos por loja
- base estruturada para relatórios e insights

Escopo atual: operação de loja única por tenant, com plano mensal único e arquitetura monolítica modular.

## Como funciona (fluxo real)

1. Admin cria conta e registra a loja
2. Sistema cria um tenant isolado
3. Admin adiciona funcionários
4. Funcionários cadastram produtos
5. Movimentações são registradas (entrada/saída)
6. Tudo fica rastreável e centralizado
7. Admin acompanha e toma decisões

## ⚡ Quick Start (5 minutos)

```bash
git clone <repo>
cd raprmanager
npm install
cp .env.example .env
npx prisma migrate dev
npm run dev
```

## Stack Técnica

- Runtime/backend: Node.js + TypeScript
- Framework HTTP: Fastify
- ORM: Prisma
- Banco de dados: PostgreSQL (Railway)
- Pagamentos: Stripe (assinatura mensal única)
- Deploy: Vercel (API) + Railway (PostgreSQL)
- Arquitetura: monólito modular

## Arquitetura de Alto Nível

O sistema é organizado por domínios dentro de um único backend. Cada domínio expõe rotas, regras de negócio e acesso a dados de forma desacoplada internamente, mas com deploy único.

Princípios arquiteturais:

- isolamento multi-tenant obrigatório em toda consulta
- regras de negócio centralizadas em services
- controllers focados em protocolo HTTP
- repositories focados em persistência
- validação explícita de entrada e autorização por papel

## Multi-Tenancy

Modelo de isolamento: `tenant_id`.

Regras operacionais:

- cada loja corresponde a um tenant
- todo usuário pertence a um tenant
- entidades de negócio carregam `tenant_id`
- qualquer listagem/consulta/escrita deve filtrar por `tenant_id`
- acesso cruzado entre tenants é bloqueado por design

Recomendação de implementação:

- injetar `tenant_id` no contexto autenticado (ex.: `request.user.tenantId`)
- recusar requisições sem contexto de tenant válido
- em operações de escrita, validar ownership do recurso antes de alterar

Exemplo de filtro obrigatório no repository:

```ts
await prisma.product.findMany({
	where: { tenantId: auth.tenantId },
	orderBy: { createdAt: 'desc' },
})
```

### Exemplo: Product

```ts
{
  id: string
  name: string
  quantity: number
  tenantId: string
  createdAt: Date
}
```

## Controle de Acesso (RBAC)

Papéis suportados:

- `admin` (dono da loja)
- `user` (funcionário)

Matriz de permissões mínima:

| Ação | admin | user |
| --- | --- | --- |
| Criar usuários | Sim | Não |
| Visualizar dados administrativos | Sim | Não |
| Acessar relatórios | Sim | Não |
| Cadastrar produtos | Sim | Sim |
| Registrar movimentações | Sim | Sim |

Diretriz de implementação:

- autorização em middleware por rota sensível
- autorização contextual em service para regras específicas

## Módulos Principais

- Autenticação: login, emissão/validação de token, contexto de tenant
- Gestão de usuários: criação e manutenção de usuários por admin
- Produtos: cadastro, atualização e consulta de catálogo interno
- Movimentações: entrada/saída de estoque com trilha de auditoria
- Relatórios (futuro): consolidação analítica e visão financeira

## Decisões Arquiteturais

- Monolito modular ao invés de microservices → reduz complexidade inicial
- Multi-tenant por `tenant_id` → isolamento simples e eficiente
- Stripe para billing → evita reinventar pagamentos
- Prisma → velocidade de desenvolvimento com segurança

## Regras de Negócio Críticas

- Nenhuma query roda sem `tenant_id`
- Estoque não é atualizado diretamente → sempre via movimentação
- Acesso depende de assinatura ativa

## Modelagem Conceitual

Entidades centrais e papel no domínio:

- `Tenant` (Store): unidade de negócio isolada; raiz do escopo multi-tenant
- `User`: identidade autenticável vinculada a um `Tenant` e a um papel (`admin`/`user`)
- `Product`: item controlado em estoque dentro de um tenant
- `Movement`: evento de alteração de estoque (entrada/saída), associado a produto, usuário e tenant
- `Subscription`: estado de cobrança do tenant (plano, status, ciclo, referências Stripe)

Relacionamentos esperados (visão resumida):

- Tenant 1:N Users
- Tenant 1:N Products
- Tenant 1:N Movements
- Product 1:N Movements
- Tenant 1:1 Subscription (modelo atual de plano único)

## Estrutura de Pastas (Referência)

```txt
src/
	routes/
	controllers/
	services/
	repositories/
	models/
	middleware/
```

Responsabilidades:

- `src/routes`: mapeamento de endpoints, versionamento e composição de middlewares
- `src/controllers`: tradução HTTP -> caso de uso (parse, chamada de service, resposta)
- `src/services`: regras de negócio, validações de domínio e orquestração entre componentes
- `src/repositories`: leitura/escrita no banco via Prisma com filtros de tenant
- `src/models`: tipos de domínio, DTOs e contratos internos
- `src/middleware`: autenticação, autorização, validação e cross-cutting concerns

Regra de separação:

- controller não acessa Prisma diretamente
- repository não contém regra de autorização
- service aplica invariantes de domínio e permissões contextuais

## Fluxo de Pagamento (Stripe)

Fluxo operacional:

1. usuário cria conta e tenant
2. backend cria sessão de checkout no Stripe para o plano mensal
3. usuário conclui o pagamento no Stripe Checkout
4. Stripe envia webhook de confirmação (ex.: `checkout.session.completed`)
5. backend valida assinatura do webhook e ativa a assinatura do tenant
6. tenant passa a ter acesso pleno aos módulos pagos

Pontos críticos:

- nunca ativar tenant apenas por callback do frontend
- status de assinatura deve ser atualizado exclusivamente por webhook assinado
- registrar `stripeCustomerId`, `stripeSubscriptionId` e status local da assinatura

## Como Rodar o Projeto

### Pré-requisitos

- Node.js 20+
- npm 10+
- PostgreSQL (local ou Railway)
- Docker (opcional, para banco local)

### 1) Instalação

```bash
npm install
```

Se o projeto ainda não estiver com as dependências base para a stack definida, instalar:

```bash
npm install fastify @prisma/client stripe zod dotenv
npm install -D typescript tsx @types/node prisma
```

### 2) Variáveis de ambiente

Criar `.env` na raiz:

```env
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DB?schema=public"
STRIPE_SECRET_KEY="sk_test_xxx"
STRIPE_WEBHOOK_SECRET="whsec_xxx"
JWT_SECRET="change_me"
PORT=3000
```

Exemplo mínimo exigido:

```env
DATABASE_URL=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
```

### 3) Prisma

```bash
npx prisma generate
npx prisma migrate dev
```

### 4) Scripts de execução (recomendado)

Adicionar no `package.json`:

```json
{
	"scripts": {
		"dev": "tsx watch src/server.ts",
		"build": "tsc -p tsconfig.json",
		"start": "node dist/server.js",
		"prisma:generate": "prisma generate",
		"prisma:migrate": "prisma migrate dev"
	}
}
```

### 5) Rodar em desenvolvimento

```bash
npm run dev
```

### 6) Build e execução de produção

```bash
npm run build
npm run start
```

## Deploy

### API (Vercel)

- publicar backend Node.js
- configurar variáveis de ambiente de produção
- garantir endpoint público para webhook Stripe

### Banco (Railway)

- provisionar PostgreSQL
- usar `DATABASE_URL` do ambiente Railway
- aplicar migrations no ambiente alvo

## Boas Práticas do Projeto

- Isolamento por tenant é obrigatório em todas as consultas e mutações
- Regras de negócio ficam em `services`, não em `controllers`
- Inputs devem ser validados na borda (schema validation)
- Fluxo de pagamento deve depender de webhooks assinados do Stripe
- Logs devem conter contexto (`tenant_id`, `user_id`, `request_id`) para auditoria
- Mudanças de estoque devem ser modeladas como eventos (`Movement`), não apenas update de saldo

## Segurança e Confiabilidade

- hash de senha com algoritmo forte (ex.: bcrypt/argon2)
- tokens com expiração curta e rotação quando aplicável
- limitação de taxa em rotas críticas (auth/webhook)
- idempotência para processamento de webhooks
- trilha de auditoria para operações sensíveis

## Roadmap

- dashboard analítico operacional por tenant
- relatórios financeiros (margem, giro, ruptura)
- alertas inteligentes (baixo estoque, variação anômala)
- suporte a múltiplas lojas por usuário (multi-store account)

## Critérios de Pronto (MVP)

- autenticação com RBAC funcional
- CRUD de produtos com escopo de tenant
- movimentações de estoque com histórico consultável
- assinatura ativa via Stripe webhook
- bloqueio de acesso para tenant sem assinatura válida

## Licença

Definir política de licenciamento antes de distribuição pública.
