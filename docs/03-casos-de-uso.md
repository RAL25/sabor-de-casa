# 📑 03 - Casos de Uso (UC): Sabor de Casa

Este documento especifica os fluxos funcionais e casos de uso do ecossistema **Sabor de Casa**[cite: 1, 4]. A implementação dos _Use Cases_ deve garantir a integridade dos dados por meio de transações atômicas, bloqueios de concorrência e emissão de eventos em tempo real[cite: 2, 4].

---

## 1. Módulo do Cliente e Delivery (B2C & B2B)

### [UC01] Realizar Pedido B2C (Delivery / Varejo)

- **Atores:** Cliente Final (B2C)[cite: 1, 4].
- **Pré-condições:** Usuário autenticado e com endereço cadastrado em Montes Claros/MG[cite: 1, 4].
- **Fluxo Principal:**
  1. O cliente visualiza o cardápio e seleciona o estado do produto: _Assado_ ou _Congelado_[cite: 1].
  2. O cliente adiciona os sabores desejados ao carrinho (sem exigência de quantidade mínima)[cite: 1].
  3. No checkout, informa o CEP para cálculo do frete e insere um cupom de desconto (opcional)[cite: 4].
  4. O cliente seleciona a forma de pagamento (Pix Dinâmico, Cartão in-app ou Pagamento na Entrega)[cite: 4].
  5. O sistema abre uma transação atômica no PostgreSQL com bloqueio pessimista (`SELECT FOR UPDATE`)[cite: 2, 4].
  6. O sistema valida o saldo em estoque, decrementa a quantidade, gera um **PIN de segurança de 4 dígitos** e registra o pedido como `PENDENTE`[cite: 2, 4].
  7. O evento `order:created` é disparado via WebSocket para a cozinha (KDS)[cite: 4].
- **Fluxo de Exceção (Estoque Insuficiente):**
  - Se um ou mais itens selecionados não possuírem saldo suficiente, a transação sofre _Rollback_ completo, o estoque permanece inalterado e o sistema retorna erro `400 Bad Request` informando os itens esgotados[cite: 2].
- **Pós-condições:** Pedido registrado no banco de dados, estoque atualizado e comanda enviada à esteira de produção[cite: 2, 4].

---

### [UC02] Realizar Pedido B2B (Atacado / Revenda)

- **Atores:** Cliente Corporativo / Revendedor (B2B)[cite: 1, 4].
- **Pré-condições:** Usuário autenticado com perfil B2B e CNPJ validado[cite: 1, 4].
- **Fluxo Principal:**
  1. O cliente acessa a área de atacado, onde o catálogo exibe exclusivamente produtos no estado **Congelado**[cite: 1].
  2. O cliente seleciona a quantidade desejada de pacotes, obrigatoriamente em múltiplos de lotes fechados (**múltiplos de 55 unidades**)[cite: 1].
  3. O sistema calcula o valor total com base na tabela de preços de congelados e valida o estoque disponível[cite: 1, 2].
  4. O cliente agenda a data de entrega desejada e confirma o pedido por faturamento PJ.
  5. O pedido é registrado com o identificador `tipo_cliente: B2B` e status `PENDENTE`[cite: 1, 4].
- **Fluxo Alternativo (Quantidade fora do padrão):**
  - Se a quantidade informada não for múltiplo exato de 55 unidades, o sistema bloqueia o avanço e sugere o ajuste para o lote mais próximo[cite: 1].

---

### [UC03] Acompanhar Pedido em Tempo Real

- **Atores:** Cliente Final (B2C)[cite: 1, 4].
- **Pré-condições:** Pedido registrado e com status ativo.
- **Fluxo Principal:**
  1. O cliente acessa a tela de rastreamento do pedido no App Delivery[cite: 4].
  2. A tela estabelece conexão via WebSocket (`EventsModule`)[cite: 4].
  3. A interface atualiza a linha do tempo conforme o status muda (`PENDENTE` $\rightarrow$ `EM_PREPARO` $\rightarrow$ `A_CAMINHO` $\rightarrow$ `ENTREGUE`)[cite: 4].
  4. Quando o status mudar para `A_CAMINHO`, a tela destaca o **PIN de 4 dígitos** que deverá ser informado ao entregador[cite: 4].

---

## 2. Módulo de Cozinha e Operação (KDS & Impressão)

### [UC04] Gerenciar Fila de Pedidos na Cozinha (KDS)

- **Atores:** Operador de Cozinha / Auxiliar de Produção[cite: 1, 4].
- **Pré-condições:** Operador autenticado no painel ERP com permissão `EMPLOYEE` ou `ADMIN`[cite: 1, 4].
- **Fluxo Principal:**
  1. A tela do KDS recebe o evento WebSocket `kitchen:new_order`, emitindo alerta sonoro e posicionando o card na coluna _Novos Pedidos_[cite: 4].
  2. O sistema aciona a impressão automática da comanda de produção em layout térmico de 80mm/58mm via biblioteca ESC/POS[cite: 4].
  3. O operador clica em "Iniciar Preparo", alterando o status para `EM_PREPARO` (colocando no forno para assar ou separando embalagem de congelados)[cite: 1, 4].
  4. Ao finalizar, o operador clica em "Pronto para Retirada", notificando a equipe de logística[cite: 4].

---

## 3. Módulo de Logística e Entregadores

### [UC05] Despachar e Validar Entrega com PIN

- **Atores:** Entregador / Despachante[cite: 1, 4].
- **Pré-condições:** Pedido com status pronto para despacho.
- **Fluxo Principal:**
  1. O despachante atribui o pedido a um entregador disponível[cite: 4].
  2. O entregador abre a rota no módulo de logística, que gera o _deep link_ direto para Google Maps ou Waze com o endereço do cliente[cite: 4].
  3. Ao chegar no endereço, o entregador solicita o PIN de 4 dígitos ao cliente[cite: 4].
  4. O entregador digita o PIN no aplicativo e submete a validação (`POST /entregas/validar-pin`)[cite: 4].
  5. O backend valida a correspondência do hash/PIN; em caso de sucesso, o status do pedido é alterado para `ENTREGUE` e a entrega é finalizada[cite: 4].
- **Fluxo de Exceção (PIN Inválido):**
  - Se o PIN digitado estiver incorreto, o sistema recusa a baixa, emite alerta sonoro de erro e mantém o status `A_CAMINHO`[cite: 4].

---

## 4. Módulo de Fábrica, Estoque e Produção (ERP)

### [UC06] Registrar Lote de Produção (_Make to Stock_)

- **Atores:** Administrador / Gestor de Produção[cite: 1].
- **Pré-condições:** Usuário autenticado com permissão `ADMIN`[cite: 1, 4].
- **Fluxo Principal:**
  1. O gestor seleciona o sabor produzido (ex: Carne Seca) e a quantidade de lotes fabricados[cite: 1].
  2. O sistema consulta a Ficha Técnica (BOM) do produto para calcular o consumo de insumos base (polvilho, queijo, ovos, leite, óleo) e insumo de recheio (carne seca)[cite: 1].
  3. O sistema valida se o estoque de matéria-prima é suficiente para cobrir todos os lotes[cite: 4].
  4. O sistema gera a entrada do lote na tabela `LOTES_PRODUCAO` (incrementando +55 unidades por lote no estoque de congelados) e debita automaticamente os insumos utilizados[cite: 1].
- **Fluxo de Exceção (Falta de Matéria-Prima):**
  - Se o estoque de algum insumo for insuficiente, o sistema bloqueia o registro do lote e destaca os ingredientes faltantes no painel de compras[cite: 4].

---

### [UC07] Pausar Produto / Ajuste Rápido de Cardápio

- **Atores:** Administrador / Gerente de Turno[cite: 1].
- **Fluxo Principal:**
  1. No painel ERP, o gerente alterna o status de um sabor para `isActive = false`[cite: 4].
  2. O backend dispara um evento via WebSocket para todas as instâncias ativas do App Delivery[cite: 4].
  3. O catálogo no app do cliente é atualizado imediatamente, desabilitando o botão de compra para aquele sabor sem necessidade de _refresh_[cite: 2, 4].

---

## 5. Módulo Financeiro e DRE (ERP)

### [UC08] Fechamento Financeiro e Geração de DRE

- **Atores:** Administrador / Gestor Financeiro[cite: 1].
- **Pré-condições:** Período contábil com movimentações registradas[cite: 1].
- **Fluxo Principal:**
  1. O gestor seleciona o mês de competência no módulo financeiro[cite: 1].
  2. A requisição dispara um job em segundo plano processado pelo Redis + BullMQ (`ReportsModule`)[cite: 2, 4].
  3. O job consolida:
     - **Receita Bruta:** Total de vendas B2C e B2B finalizadas no período[cite: 1].
     - **Impostos:** Alíquota do Simples Nacional provisionada sobre o faturamento bruto[cite: 1].
     - **CMV (Custo das Mercadorias Vendidas):** Custo real dos insumos e embalagens baixadas no período[cite: 1].
     - **Despesas Operacionais & Pró-labore:** Gastos fixos e retiradas dos 4 sócios fundadores[cite: 1].
  4. O sistema gera a tabela DRE comparativa e disponibiliza o download do relatório em PDF formatado[cite: 1, 2].
