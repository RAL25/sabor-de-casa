# 📄 01 - Visão do Produto: Sabor de Casa

## 1. Contexto e Oportunidade

O **Sabor de Casa** é uma fábrica e distribuidora de pão de queijo gourmet sediada em Montes Claros - MG. O negócio nasceu com a proposta de aliar a tradição da culinária mineira à comodidade do comércio digital e à inovação em sabores artesanais.

A operação comercial atua em duas frentes complementares:

- **B2C (Business to Consumer):** Venda direta ao consumidor final para consumo imediato (pães de queijo assados e quentes) ou estocagem doméstica (pães de queijo congelados) via delivery próprio.
- **B2B (Business to Business):** Fornecimento programado de pães de queijo congelados em grandes volumes para padarias, cafeterias, hotéis e estabelecimentos revendedores da região.

### O Problema

Sistemas de delivery genéricos (como iFood ou Uber Eats) cobram altas taxas de intermediação e não suportam pedidos no modelo B2B (com regras de atacado por lotes), nem se integram nativamente ao controle de matéria-prima, ficha técnica de produção e finanças da fábrica.

### A Solução

Uma plataforma web full-stack desacoplada em duas interfaces compartilhando o mesmo ecossistema:

1. Um **App Web de Delivery (B2C/B2B)** rápido, intuitivo e com foco em conversão.
2. Um **ERP e Painel de Cozinha (KDS)** para gestão de produção em lotes (_Make to Stock_), controle de estoque de insumos e DRE financeiro.

---

## 2. Atores do Sistema

| Ator                          | Canal de Acesso       | Responsabilidades Principais                                                                                                    |
| :---------------------------- | :-------------------- | :------------------------------------------------------------------------------------------------------------------------------ |
| **Cliente Final (B2C)**       | App Delivery          | Navegar pelo cardápio, alternar entre itens _Assados_ e _Congelados_, realizar checkout ágil e rastrear o pedido em tempo real. |
| **Cliente Corporativo (B2B)** | App Delivery / Portal | Realizar pedidos de revenda por CNPJ em múltiplos de lotes fechados (apenas produtos congelados).                               |
| **Operador de Cozinha (KDS)** | Painel ERP (KDS)      | Visualizar comandas em tempo real, gerenciar esteira de preparo (forno/embalagem) e acionar impressões térmicas.                |
| **Entregador / Logística**    | Módulo de Entregas    | Visualizar endereços de entrega com rota integrada (Google Maps/Waze) e validar conclusão via PIN de 4 dígitos.                 |
| **Administrador / Gestor**    | Painel ERP            | Registrar lotes fabricados (55 un/lote), monitorar estoque de matérias-primas, lançar despesas operacionais e auditar o DRE.    |

---

## 3. Escopo do Produto

### O que o sistema FAZ (In Scope):

- **Catálogo com Precificação Dinâmica:** Apresenta os 6 sabores do cardápio (Tradicional, Presunto e Queijo, Bacon, Frango Desfiado, Carne Seca e Goiabada) com variação de preços para produtos assados vs. congelados.
- **Gestão de Lotes de Produção (Make to Stock):** Registro de fabricação baseado em lotes padronizados de 55 unidades, decrementando insumos automaticamente com base na ficha técnica.
- **Esteira de Pedidos em Tempo Real:** Comunicação bidirecional via WebSockets entre cliente, cozinha (KDS) e entregador.
- **Trava Transacional de Estoque:** Controle de concorrência com bloqueio pessimista (_Pessimistic Locking_) no banco de dados para evitar vendas duplicadas sem estoque.
- **Painel Financeiro e DRE:** Consolidação de receitas, custos de mercadorias vendidas (CMV), impostos do Simples Nacional e cálculo de pró-labore sob cenários Otimista e Pessimista.

### O que o sistema NÃO FAZ (Out of Scope - MVP):

- Rastreamento por geolocalização contínua de GPS do entregador em tempo real (utiliza-se atualização por marcos de status e validação de PIN).
- Emissão de Notas Fiscais Eletrônicas (NFe/NFCe) direta com a SEFAZ (o sistema calcula e consolida os impostos para fins de DRE, mas a emissão do XML fiscal é externa).
- Gateway de frete com múltiplas transportadoras terceirizadas (o cálculo é baseado em raio por CEP local de Montes Claros).

---

## 4. Diferenciais Competitivos

1. **Abordagem Híbrida (B2C + B2B):** Capacidade de atender a venda fracionada para a mesa do cliente e a venda em escala para estabelecimentos comerciais na mesma infraestrutura.
2. **Arquitetura de Alta Performance:** Aplicação web modular e responsiva, eliminando fricção de download em lojas de aplicativos e reduzindo custos operacionais com intermediadores.
3. **Visão de Engenharia de Produção:** O software não trata o produto apenas como um item de prateleira, mas modela a ficha técnica de insumos e a capacidade produtiva da cozinha.

---

## 5. Critérios de Sucesso do MVP

- **Tempo de Checkout:** Conclusão de um pedido B2C em menos de 60 segundos.
- **Latência de Comunicação:** Atualização de status entre Checkout, KDS e Delivery com latência inferior a 500ms via WebSockets.
- **Integridade Operacional:** Zero inconsistências de saldo de estoque em compras simultâneas.
- **Confiabilidade Financeira:** Relatório DRE mensal gerado automaticamente refletindo 100% das transações e lotes registrados no período.
