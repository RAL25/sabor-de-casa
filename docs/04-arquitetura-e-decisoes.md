# 🏛️ 04 - Arquitetura e Decisões Técnicas (ADRs): Sabor de Casa

Este documento registra as decisões arquiteturais fundamentais (ADRs) que estruturam o ecossistema do **Sabor de Casa**. Ele detalha a escolha da topologia de monorepo, isolamento de frontends, estratégias de concorrência, mensageria e persistência de dados.

---

## 1. Visão Geral da Topologia

A aplicação adota o modelo **1 Backend Centralizado (NestJS) + 2 Frontends Especializados (Next.js)** organizados em um **Monorepo gerenciado por Turborepo**:

```text
               ┌──────────────────────────────────────────────┐
               │          PostgreSQL  +  Redis                │
               └──────────────────────┬───────────────────────┘
                                      │
                                      │ (Prisma ORM & BullMQ)
                                      ▼
               ┌──────────────────────────────────────────────┐
               │           Backend API (NestJS)               │
               │  - Autenticação & RBAC (Guards)              │
               │  - Transações SQL com Pessimistic Locking    │
               │  - Gateway WebSocket (Socket.io)             │
               │  - Jobs Assíncronos (Relatórios & DRE)       │
               └──────────────┬────────────────┬──────────────┘
                              │                │
          REST / WebSocket    │                │    REST / WebSocket
                              ▼                ▼
         ┌────────────────────────┐        ┌────────────────────────┐
         │   App Delivery (B2C)   │        │   Admin & KDS (ERP)    │
         │   - Next.js (App Router)        │   - Next.js / Vite     │
         │   - Tailwind CSS       │        │   - Shadcn UI / Grids  │
         │   - Mobile-First & SEO │        │   - Alta Densidade     │
         └────────────────────────┘        └────────────────────────┘
```

Aqui está o conteúdo completo e estruturado para o arquivo **`04-arquitetura-e-decisoes.md`** (_Architecture Decision Records - ADRs_). Você pode salvá-lo diretamente dentro da pasta `docs/` do seu projeto:

---

## 2. Registros de Decisão de Arquitetura (ADRs)

### ADR 01 — Adoção de Monorepo com Turborepo

- **Status:** Aprovado.

- **Contexto:** O ecossistema possui múltiplos pontos de consumo (API Backend, App Delivery e Painel ERP) que compartilham tipagens, DTOs, enums de status e regras de validação.

- **Decisão:** Utilizar o **Turborepo** para orquestrar os pacotes e aplicações no mesmo repositório.

- **Consequências:**
- **Positivas:** Eliminação da duplicidade de código; compartilhamento do pacote `shared-types` com inferência de tipos fim a fim (TypeScript); _build caching_ inteligente que acelera as pipelines de CI/CD.

- **Negativas:** Requer configuração inicial padronizada de _workspaces_ (pnpm ou npm).

---

### ADR 02 — Separação em Dois Frontends Desacoplados

- **Status:** Aprovado.

- **Contexto:** O consumidor final precisa de uma interface rápida e intuitiva para celular, enquanto a gestão da fábrica e cozinha exige painéis com gráficos e tabelas densas.

- **Decisão:** Dividir os clientes em duas aplicações web isoladas: `web-delivery` e `admin-erp`.

- **Consequências:**
- **Positivas:** Otimização de bundle para o cliente (App Delivery leve, sem carregar bibliotecas pesadas de relatórios ou exportação PDF); isolamento seguro das rotas administrativas e maior facilidade de manutenção por domínio.

- **Negativas:** Necessidade de gerenciar dois pontos de entrada e roteamentos separados de deploy.

---

### ADR 03 — Backend Modular com NestJS e Prisma ORM

- **Status:** Aprovado.

- **Contexto:** A API necessita de arquitetura escalável, injeção de dependência nativa e controle estrito de regras de negócio entre os perfis B2B e B2C.

- **Decisão:** Desenvolver a API utilizando **NestJS (TypeScript)** com **Prisma ORM** e banco **PostgreSQL**.

- **Consequências:**
- **Positivas:** Controle de acessos robusto via _Guards_ (`@Roles('ADMIN')`, `@Roles('EMPLOYEE')`); validação centralizada de contratos com `class-validator`; tipagem forte nas consultas relacionais através do Prisma.

- **Negativas:** Curva de aprendizado inicial da estrutura orientada a módulos e decoradores do NestJS.

---

### ADR 04 — Controle de Concorrência de Estoque (Pessimistic Locking)

- **Status:** Aprovado.

- **Contexto:** Em momentos de pico ou campanhas no delivery, múltiplos usuários podem tentar comprar as últimas unidades de pães de queijo simultaneamente, gerando risco de _overbooking_ (venda sem saldo).

- **Decisão:** Implementar **bloqueio pessimista** (`SELECT FOR UPDATE`) encapsulado em transações atômicas no PostgreSQL durante o fluxo de checkout (`OrdersModule`).

- **Consequências:**
- **Positivas:** Garantia de consistência estrita de estoque (ACID); integridade transacional com suporte a _Rollback_ automático em caso de falta de itens.

- **Negativas:** Criação de um breve bloqueio na linha do registro do produto durante a execução da transação (aceitável dado o volume de transações por item).

---

### ADR 05 — Comunicação Bidirecional em Tempo Real (WebSockets / Socket.io)

- **Status:** Aprovado.

- **Contexto:** A esteira de produção da cozinha (KDS), a logística e a tela de tracking do cliente precisam de atualizações instantâneas de status sem recorrer a _polling_ HTTP contínuo.

- **Decisão:** Utilizar o módulo de **Gateways WebSocket do NestJS** integrado com **Socket.io**.

- **Consequências:**
- **Positivas:** Redução de tráfego desnecessário de rede; disparo imediato de alarmes sonoros no KDS e atualização em tempo real da linha do tempo no cliente.

- **Negativas:** Necessidade de gerenciar o ciclo de vida e reconexão de sockets ativos no frontend.

---

### ADR 06 — Processamento Assíncrono com Redis & BullMQ

- **Status:** Aprovado.

- **Contexto:** Rotinas de geração de relatórios contábeis (DRE), exportação de fechamentos de caixa em PDF e consolidação de lotes podem sobrecarregar o _Event Loop_ do Node.js se executadas de forma síncrona.

- **Decisão:** Delegar tarefas pesadas para filas em segundo plano usando **BullMQ** com persistência em memória via **Redis**.

- **Consequências:**
- **Positivas:** Respostas imediatas nos endpoints da API (`202 Accepted`); processamento não bloqueante de relatórios e faturamento.

- **Negativas:** Adição de uma dependência de infraestrutura (serviço do Redis).

---

### ADR 07 — Ambiente Padronizado via Docker e Docker Compose

- **Status:** Aprovado.

- **Contexto:** Facilitar a inicialização de todo o ecossistema (PostgreSQL, Redis e aplicações) em qualquer ambiente de desenvolvimento ou avaliação técnica com um único comando.

- **Decisão:** Configurar um arquivo `docker-compose.yml` na raiz do repositório contendo as variáveis e containers do PostgreSQL 16 e Redis Alpine.

- **Consequências:**
- **Positivas:** Execução determinística sem dependência de instalações manuais na máquina do desenvolvedor; facilidade na execução de testes E2E.

- **Negativas:** Consumo de recursos de virtualização local via Docker Engine.
