# 🗺️ 07 - Roadmap de Desenvolvimento e Entregas do MVP: Sabor de Casa

Este documento estabelece o plano de execução e o cronograma de implementação modular para o ecossistema **Sabor de Casa**. O desenvolvimento é dividido em 7 fases iterativas, priorizando a integridade do banco de dados, regras de concorrência, comunicação em tempo real e experiência de uso dos módulos B2C, B2B e ERP.

---

## 📅 Visão Geral das Fases

```text
[Fase 1: Fundação & Banco] ➔ [Fase 2: Core API & Auth] ➔ [Fase 3: Real-Time & Logística]
                                                                   │
┌──────────────────────────────────────────────────────────────────┘
▼
[Fase 4: Frontend Delivery] ➔ [Fase 5: Frontend ERP/KDS] ➔ [Fase 6: Financeiro/DRE] ➔ [Fase 7: Qualidade & Deploy]

```

---

## 🚀 Detalhamento das Fases de Implementação

### 🧱 Fase 1: Fundação do Monorepo e Banco de Dados

**Objetivo:** Configurar o ambiente padronizado via Turborepo, Docker e a camada de persistência com o Prisma ORM.

- [ ] **Configuração do Monorepo:**
- Inicializar workspace Turborepo (`apps/api`, `apps/web-delivery`, `apps/admin-erp`, `packages/database`, `packages/shared-types`).

- Configurar scripts de build, lint e pipeline de cache do Turborepo.

- [ ] **Infraestrutura Local (Docker):**
- Criar o `docker-compose.yml` contendo os serviços do **PostgreSQL 16** e **Redis Alpine**.

- [ ] **Modelagem & Migrations (Prisma):**
- Configurar o `schema.prisma` consolidado no pacote `packages/database`.

- Gerar e rodar as migrations iniciais de criação de tabelas, enums e índices relacionais.

- [ ] **Seeds Iniciais:**
- Script para popular os 6 sabores de pão de queijo (com preços congelado/assado), insumos de matéria-prima e o usuário administrador inicial.

---

### ⚙️ Fase 2: Core Backend, Autenticação e Checkout Seguro

**Objetivo:** Desenvolver o núcleo transacional da API em NestJS com validação de dados e trava de concorrência.

- [ ] **Autenticação & RBAC (`AuthModule`):**
- Implementar autenticação via JWT com hash de senha seguro via `bcrypt`.

- Criar _Guards_ de autorização para isolar papéis (`ADMIN`, `EMPLOYEE`, `CUSTOMER`).

- [ ] **Módulo de Catálogo (`CatalogModule`):**
- Endpoints `GET /products` e `PATCH /products/:id/toggle-status` (pausa de itens).

- [ ] **Módulo de Checkout Concorrente (`OrdersModule`):**
- Implementar regras de checkout para consumidores B2C (unitário) e empresas B2B (múltiplos de 55 unidades).

- Implementar transação atômica com **bloqueio pessimista** (`SELECT FOR UPDATE`) para impedir venda sem estoque.

- Lógica de cálculo de frete por CEP e aplicação de cupons de desconto.

---

### ⚡ Fase 3: Comunicação em Tempo Real, KDS e Logística

**Objetivo:** Estabelecer a infraestrutura de WebSockets para a cozinha e o fluxo de entregas com validação por PIN.

- [ ] **Gateway WebSockets (`EventsModule`):**
- Configurar o Socket.io Gateway no NestJS com autenticação via handshake JWT.

- Implementar canais e salas isoladas para clientes, despachantes e cozinha.

- [ ] **Módulo de Cozinha (`KitchenModule` / KDS):**
- Disparo do evento `kitchen:new_order` ao confirmar novos pedidos.

- Máquina de estados da esteira de preparo (`PENDENTE` $\rightarrow$ `EM_PREPARO` $\rightarrow$ `A_CAMINHO`).

- Geração de layout de impressão térmica formatado em padrão ESC/POS (80mm/58mm).

- [ ] **Módulo de Logística (`LogisticsModule`):**
- Geração automática do PIN de 4 dígitos na criação do pedido.

- Endpoint `POST /deliveries/validate-pin` para validação e baixa de entregas.

---

### 📱 Fase 4: Frontend do App de Delivery (B2C & B2B)

**Objetivo:** Construir a loja virtual responsiva, rápida e focada em conversão utilizando Next.js e Tailwind CSS.

- [ ] **Catálogo Interativo:**
- Seletor de estado do produto (_Assado_ vs. _Congelado_) recalculando preços instantaneamente.

- Grid de produtos com fotos, modal de customização de pratos e busca textual.

- [ ] **Carrinho & Checkout Ágil:**
- Validação de documento (CPF para B2C e CNPJ para B2B).

- Busca automática de logradouro via API de CEP (ViaCEP).

- Integração de pagamento via Pix com QR Code dinâmico e opção de Copia e Cola.

- [ ] **Linha do Tempo em Tempo Real (Tracking):**
- Conexão via WebSocket para atualizar o status do pedido sem recarregar a página.

- Exibição destacada do PIN de segurança quando o status mudar para `A_CAMINHO`.

---

### 🖥️ Fase 5: Frontend do Painel ERP e Operação de Fábrica

**Objetivo:** Criar o painel administrativo para a gestão de lotes de fabricação, estoque de matérias-primas e KDS.

- [ ] **Painel KDS Operacional:**
- Visão Kanban da esteira de pedidos com alerta sonoro automático para novos itens.

- Botão de avanço rápido de etapas e acionamento de impressão de comandas.

- [ ] **Módulo de Fábrica (_Make to Stock_):**
- Formulário para registro de lotes de produção (55 unidades por lote).

- Abatimento automático no estoque de insumos com base na Ficha Técnica (BOM).

- [ ] **Gestão de Cardápio:**
- Tabela com switch para pausar sabores esgotados em tempo real via WebSocket.

---

### 📊 Fase 6: Módulo Financeiro, Relatórios e DRE

**Objetivo:** Implementar o fechamento contábil e a geração assíncrona de relatórios gerenciais.

- [ ] **Fila Assíncrona com Redis & BullMQ (`ReportsModule`):**
- Configuração de filas para execução de tarefas pesadas em background.

- [ ] **Consolidação de DRE (Simples Nacional):**
- Cálculo de Receita Bruta, CMV (insumos e embalagens) e provisão de impostos do Simples Nacional.

- Registro de despesas operacionais e cálculo do pró-labore dos sócios.

- Comparativo dos resultados sob os cenários **Otimista** vs. **Pessimista**.

- [ ] **Exportação de Relatórios:**
- Geração de arquivos PDF formatados para fechamento de caixa diário e DRE mensal.

---

### 🧪 Fase 7: Qualidade, Testes, Documentação e Deploy

**Objetivo:** Blindar a aplicação com testes automatizados, documentação interativa e facilidade de execução.

- [ ] **Automação de Testes (Jest):**
- Testes unitários para regras de negócio críticas (bloqueio de venda sem estoque, cálculo de múltiplos B2B e baixa de insumos).

- Testes de integração E2E para o fluxo completo de checkout e validação de PIN.

- [ ] **Documentação de API (Swagger):**
- Configuração do `@nestjs/swagger` expondo schemas e endpoints para testes diretos via browser.

- [ ] **Apresentação de Portfólio no GitHub:**
- Finalização do `README.md` principal com arquitetura, diagramas Mermaid, instruções de execução via `docker-compose up` e capturas de tela/GIFs do sistema em funcionamento.

---

## 🎯 Critérios de Aceite para Conclusão do MVP (Definition of Done)

1. **Execução em 1 Comando:** O projeto deve subir banco de dados, Redis, API e frontends via Docker Compose sem falhas de inicialização.

2. **Consistência de Estoque:** Nenhuma transação concorrente de checkout deve permitir saldo negativo em produtos.

3. **Sincronização em Tempo Real:** Atualizações de pedidos no ERP/KDS devem refletir no app do cliente em menos de 500ms.

4. **Confiabilidade de Lotes:** Todo lote fabricado deve incrementar exatamente 55 unidades no produto e abater a matéria-prima correspondente.
