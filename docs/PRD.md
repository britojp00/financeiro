
# PRD — Sistema de Controle Financeiro Pessoal

**Versão:** 1.0
**Status:** Baseline documental; requisitos sujeitos às pendências identificadas
**Data:** 10/11/2026

> Esta versão consolida os requisitos funcionais e as decisões de negócio do projeto. O PRD v0.3 foi previamente aprovado; a revisão v0.4 incorporou decisões posteriores, especialmente o detalhamento do módulo de cartões. A versão documental 1.0 passa a ser a referência oficial do projeto.

## 1. Visão e objetivos

Aplicação web de controle financeiro pessoal, com foco em integridade contábil, rastreabilidade, acompanhamento de saldos e patrimônio e auditoria dos gastos desde indicadores até os lançamentos de origem. Uso inicialmente pessoal, mas com arquitetura multiusuário e isolamento completo desde o início.

O sistema deverá registrar receitas, despesas, transferências, ajustes, previsões, recorrências, cartões, faturas, parcelas, investimentos, orçamentos e metas. Dashboard e relatórios devem permitir navegar dos indicadores até os registros que os compõem.

## 2. Escopo do MVP

### 2.1 Usuários e cadastros
- Autenticação e isolamento por usuário.
- Contas bancárias/dinheiro disponível e instituições.
- Categorias e subcategorias em tabelas distintas.
- Cartões de crédito e investimentos.
- Orçamentos e metas.

### 2.2 Movimentações e livro-razão
- `movimentacoes` representa o evento de negócio.
- `lancamentos_financeiros` representa seus efeitos concretos sobre contas, investimentos ou obrigações de cartão.
- Uma movimentação pode gerar N lançamentos; todos são persistidos atomicamente.
- Transferência gera uma movimentação e lançamentos de saída/entrada.
- Ajustes de saldo são movimentações justificadas, nunca edição direta do saldo.
- Operações efetivadas são imutáveis; correções ocorrem por estornos/compensações vinculados.

### 2.3 Previsões e recorrências
- Previstos e efetivados são separados.
- A previsão original é preservada; efetivação cria movimentação real vinculada.
- Recorrências podem ser editadas, pausadas e encerradas; ocorrência individual pode ser alterada sem alterar a série.
- Agendamento simples com persistência e idempotência no PostgreSQL. Sem infraestrutura de filas no MVP.

### 2.4 Cartões, compras e faturas
- `cartoes` é entidade própria, separada de `contas`.
- `cartoes.conta_id` é opcional e não determina a conta usada para pagar a fatura.
- Cada cartão possui dia de fechamento e dia de vencimento.
- Cada fatura armazena suas próprias datas de ciclo, fechamento e vencimento.
- Compra à vista gera uma movimentação e uma parcela (1/1). Compra parcelada gera uma movimentação pelo total e N parcelas individuais.
- Soma das parcelas corresponde ao total da compra; distribuição de centavos deve ser determinística.
- Cada parcela pode ser vinculada a no máximo um item de fatura.
- `itens_faturas` reúne parcelas cobradas e itens independentes: juros, multas, anuidades, tarifas, descontos, créditos e ajustes.
- Valor de item é positivo; `natureza` define acréscimo ou redução.
- A obrigação total da compra no cartão é reconhecida no momento da compra. A inclusão posterior das parcelas nas faturas não pode duplicar o efeito contábil.
- Fatura é agrupador de cobrança, não uma segunda despesa.

### 2.5 Pagamentos e estornos de faturas
- Uma fatura pode receber vários pagamentos, parciais ou integrais.
- Cada registro de `pagamentos_faturas` pertence a uma fatura e corresponde a exatamente uma movimentação.
- A conta pagadora é identificada pelos lançamentos financeiros da movimentação, sem duplicar `conta_id` em `pagamentos_faturas`.
- Pagamento reduz saldo da conta e obrigação do cartão; não gera nova despesa.
- Um pagamento pode ter vários estornos parciais.
- Cada `estornos_pagamentos_faturas` corresponde a uma movimentação de estorno.
- A soma dos estornos efetivados não pode exceder o valor efetivamente pago.
- Estornos recompõem a conta e restabelecem a obrigação, com lançamentos inversos.
- Pagamentos e estornos efetivados são imutáveis; correções exigem compensações.

### 2.6 Cancelamentos e devoluções de compras
- Cancelamento/devolução integral ou parcial é operação compensatória vinculada à compra original.
- Compra e parcelas originais são preservadas.
- Para parcelas futuras ainda não faturadas, compensar da última para a primeira.
- Excedente é tratado conforme crédito efetivamente concedido pela operadora ou reembolso recebido.
- Parcelas faturadas e não pagas podem receber crédito nas respectivas faturas.
- Parcelas pagas são tratadas por crédito futuro ou entrada em conta, conforme destino real do reembolso.
- A compensação não pode reduzir a obrigação duas vezes.
- Deve existir rastreabilidade entre compra, cancelamento, parcelas afetadas, créditos e lançamentos.

### 2.7 Investimentos
- `investimentos` é tabela própria, obrigatoriamente vinculada a uma conta; uma conta pode ter vários investimentos.
- Aportes e resgates são transferências internas, não despesas/receitas comuns.
- Rendimentos são reconhecidos separadamente.
- Projeções são distintas de rendimentos efetivos; histórico de taxas e avaliações é preservado.
- Acompanhamento inicial dos cofrinhos Banco do Brasil e Mercado Pago, sem presumir que tenham a mesma modalidade.

### 2.8 Dashboard e auditoria
- Dinheiro disponível, fluxo de caixa e patrimônio total.
- Patrimônio inclui contas, investimentos e obrigações/dívidas.
- Auditoria por período, categoria, subcategoria, estabelecimento, conta, cartão e demais dimensões.
- Gráficos e indicadores devem permitir chegar aos lançamentos de origem.

## 3. Regras transversais

1. Valores monetários em `NUMERIC`, nunca ponto flutuante.
2. Moeda explícita; MVP em BRL, estrutura preparada para outras moedas.
3. Transferências não são receitas/despesas.
4. Aportes/resgates são transferências entre ativos; rendimentos são separados.
5. Saldo deriva dos lançamentos válidos; não é editado diretamente.
6. Ajustes exigem justificativa e trilha de auditoria.
7. Operações efetivadas são imutáveis; correções geram eventos compensatórios.
8. Previsões não alteram saldo efetivo.
9. Operações críticas são atômicas e idempotentes.
10. PostgreSQL é a autoridade de integridade e concorrência; não usar locks em memória.

## 4. Convenções obrigatórias

- Tabelas e colunas em português, `snake_case`.
- PK: `BIGINT GENERATED ALWAYS AS IDENTITY`; FKs `BIGINT`; sem UUID para PK/FK.
- `situacao CHAR(1)` com códigos documentados e `CHECK` por domínio.
- Exclusão lógica: `data_excluido TIMESTAMPTZ NULL`, apenas para cadastros aplicáveis.
- Não duplicar situação operacional e exclusão lógica.
- Não excluir logicamente operações financeiras efetivadas, lançamentos, pagamentos, estornos ou histórico.
- Timestamps com fuso (`TIMESTAMPTZ`).
- Categorias e subcategorias em tabelas separadas.

## 5. Stack aprovada

**Frontend:** React, TypeScript, Vite, Tailwind CSS, shadcn/ui, TanStack Query, React Hook Form, Zod e Recharts.
**Backend:** Python 3.13+, FastAPI, Pydantic, SQLAlchemy 2, Alembic, uv, Pytest e Ruff.
**Banco/infra:** PostgreSQL, Docker e Docker Compose. Git/GitHub; GitHub Actions quando necessário.

Arquitetura de monólito modular. Redis, filas e microsserviços não entram no MVP sem justificativa concreta e aprovação.

## 6. Fora do MVP

Integrações bancárias automáticas, importação automática de extratos, antecipação avançada de parcelas, carteira ampla de ativos/corretoras, indicadores econômicos externos, IA, microsserviços e filas sem necessidade comprovada.

## 7. Critérios de aceite

- Transferência gera uma movimentação com efeitos de débito/crédito.
- Compra parcelada aparece uma vez na auditoria de despesas.
- Soma das parcelas equivale ao total da compra.
- Parcela não pode aparecer em mais de um item de fatura.
- Pagamento parcial mantém saldo correto e não duplica despesa.
- Estornos parciais/múltiplos não excedem o pagamento original.
- Cancelamento preserva compra e parcelas originais.
- Compensações não duplicam redução de obrigação.
- Alterar configuração do cartão não recalcula faturas antigas.
- Usuário não acessa dados de outro usuário.

## 8. Processo de evolução e pendências

Este PRD é a referência oficial de requisitos. Decisões aprovadas não devem ser reabertas sem necessidade ou solicitação. Mudanças futuras exigem nova revisão e histórico.

Pendências antes de implementação do ERD: fechar constraints multiusuário, estados/transições, alocação física de cancelamentos, regra de compras processadas com atraso, calendário para dias inexistentes, distribuição de centavos, modalidades reais dos investimentos e autenticação/recuperação de conta.

Nenhum agente pode alterar arquivos, executar migrations, commits, push, merge ou operações destrutivas sem autorização explícita do usuário.