# 📋 02 - Regras de Negócio (RN): Sabor de Casa

Este documento reúne todas as regras e restrições de domínio que regem o ecossistema do **Sabor de Casa**. As regras aqui descritas devem ser obrigatoriamente aplicadas pelo backend (NestJS) e respeitadas por todas as interfaces consumidoras (App Delivery e Painel ERP).

---

## 1. Comercialização e Clientes (B2C e B2B)

### RN01 — Segmentação de Clientes e Estados Permitidos

- **B2C (Consumidor Final):**
  - Pode adquirir pães de queijo no estado **Assado** (pronto para consumo) ou **Congelado** (pacote para preparo doméstico).
  - Não há exigência de quantidade mínima por pedido (venda fracionada/unitária) .
  - Identificação obrigatória por **CPF** no momento do checkout.
- **B2B (Empresas e Revendedores):**
  - Fornecimento exclusivo de produtos no estado **Congelado**.
  - Exige validação de **CNPJ** ativo no cadastro.
  - Vendas realizadas obrigatoriamente em múltiplos fechados de lotes (**múltiplos de 55 unidades**) .

---

## 2. Catálogo e Precificação

### RN02 — Tabela de Preços Diferenciada por Estado

O sistema deve aplicar o valor unitário baseado estritamente na escolha do cliente entre o produto _Assado_ ou _Congelado_, conforme a tabela oficial do negócio:

| Sabor                              | Preço Congelado (Un.) | Preço Assado (Un.) |
| :--------------------------------- | :-------------------- | :----------------- |
| **Tradicional**                    | R$ 1,25               | R$ 2,80            |
| **Recheado com Presunto e Queijo** | R$ 1,60               | R$ 3,20            |
| **Recheado com Bacon**             | R$ 2,20               | R$ 4,00            |
| **Recheado com Frango Desfiado**   | R$ 1,40               | R$ 3,00            |
| **Recheado com Carne Seca**        | R$ 1,60               | R$ 3,30            |
| **Recheado com Goiabada**          | R$ 1,40               | R$ 2,95            |

> **Nota de Integridade:** O valor unitário e subtotal são calculados e validados exclusivamente pelo backend com base no banco de dados. Qualquer preço enviado pelo payload do frontend que divirja do cadastro será ignorado.

---

## 3. Gestão de Estoque, Produção e Concorrência

### RN03 — Produção Padronizada por Lotes (_Make to Stock_)

- A fabricação opera sob o modelo _Make to Stock_ (produção antecipada para estoque).
- Todo lote cadastrado no ERP possui tamanho fixo padrão de **55 unidades**.
- A confirmação de registro de um lote incrementa automaticamente o saldo do produto no estoque de congelados em +55 unidades.
- É obrigatório vincular o identificador do operador/sócio responsável pela produção do lote para fins de rastreabilidade.

### RN04 — Baixa Automática de Matérias-Primas (Ficha Técnica / BOM)

- Cada lote registrado consome quantidades parametrizadas de insumos base (polvilho doce, queijo, leite, ovos, óleo, farinha de trigo e sal) e recheios específicos (bacon, frango, carne seca, presunto/mussarela ou goiabada).
- O sistema não permite a entrada de um lote se o saldo de qualquer um dos insumos da ficha técnica for insuficiente, disparando alerta de reposição.

### RN05 — Trava de Concorrência no Checkout (_Pessimistic Locking_)

- Ao submeter um pedido, o backend deve abrir uma transação atômica e aplicar bloqueio pessimista (`SELECT FOR UPDATE`) nas linhas dos produtos selecionados .
- Se o estoque disponível for menor que a quantidade requisitada, a transação deve ser abortada via _Rollback_, retornando status HTTP `400 Bad Request` com mensagem explicativa .

---

## 4. Checkout, Cupons e Validação de Entrega

### RN06 — Regras de Desconto e Cupons Promocionais

- Cada cupom possui código identificador único, data de expiração, valor mínimo de pedido e tipo de abatimento (percentual ou fixo em reais).
- O desconto aplica-se estritamente sobre o subtotal dos produtos, não incidindo sobre a taxa de frete.
- Não é permitido o uso cumulativo de mais de um cupom por checkout.

### RN07 — Confirmação Segura de Entrega (PIN de 4 Dígitos)

- Todo pedido despachado gera um código aleatório de 4 dígitos numéricos (**PIN**) visível apenas no app do cliente.
- A conclusão da entrega no módulo do entregador só ocorre mediante a digitação correta desse PIN, alterando o status do pedido para `CONCLUIDO`.

---

## 5. Módulo Financeiro e DRE (Demonstração do Resultado)

### RN08 — Lançamento e Fechamento Contábil (Simples Nacional)

- **Regime Tributário:** Cálculo de provisão de impostos baseado na alíquota simplificada do Simples Nacional sobre a receita bruta.
- **Custo das Mercadorias Vendidas (CMV):** Calculado pelo custo dos insumos baixados pelas fichas técnicas somado ao material de embalagem utilizado.
- **Distribuição de Resultados / Pró-labore:** O sistema deve calcular o pró-labore mensal dos sócios e gerar a DRE consolidada comparando a meta com os cenários:
  - **Cenário Otimista:** Produção diária de 2 lotes congelados + 1 lote assado por sabor.
  - **Cenário Pessimista:** Produção de 1 lote semanal por sabor.
