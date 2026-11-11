
# Architecture — Sistema de Controle Financeiro Pessoal

**Versão:** 1.0
**Status:** Baseline documental; diretrizes arquiteturais consolidadas, com pendências técnicas identificadas
**Alinhamento:** PRD v1.0 e ERD v1.0

## 1. Objetivo e arquitetura

Aplicação financeira multiusuário, inicialmente de uso pessoal, com prioridade para integridade contábil, rastreabilidade, segurança e auditoria.

A arquitetura será um monólito modular, com frontend e backend separados.

Não serão introduzidos microsserviços, Redis ou sistemas de filas no MVP sem justificativa concreta e aprovação.

## 2. Stack tecnológica

### Frontend
- React
- TypeScript
- Vite
- Tailwind CSS
- shadcn/ui
- TanStack Query
- React Hook Form
- Zod
- Recharts

### Backend
- Python 3.13+
- FastAPI
- Pydantic
- SQLAlchemy 2
- Alembic
- uv
- Pytest
- Ruff

### Banco e infraestrutura
- PostgreSQL
- Docker
- Docker Compose
- Git e GitHub
- GitHub Actions quando necessário.

## 3. Módulos de domínio

- Usuários e autenticação.
- Instituições e cadastros.
- Contas.
- Movimentações e livro-razão.
- Cartões, parcelas e faturas.
- Pagamentos e estornos de faturas.
- Investimentos.
- Previsões e recorrências.
- Orçamentos e metas.
- Dashboard e relatórios.

Cada módulo deverá possuir responsabilidades claras, sem abstrações desnecessárias.

## 4. Convenções de dados

- Tabelas e colunas em português, utilizando `snake_case`.
- PK: `BIGINT GENERATED ALWAYS AS IDENTITY`.
- FKs: `BIGINT`.
- Não utilizar UUID para PK ou FK.
- Valores monetários em `NUMERIC`, nunca `FLOAT`, `REAL` ou ponto flutuante da linguagem.
- Moeda explícita, com BRL no MVP.
- Datas e horários com `TIMESTAMPTZ`; datas de calendário com `DATE`.
- Situação com `CHAR(1)`, códigos documentados e constraints `CHECK`.
- Exclusão lógica por `data_excluido TIMESTAMPTZ NULL`, somente para cadastros aplicáveis.
- Operações financeiras efetivadas e históricos não podem ser excluídos logicamente.

## 5. Núcleo financeiro

### 5.1 movimentacoes

Representa o evento de negócio, como:
- Receita.
- Despesa.
- Transferência.
- Ajuste.
- Compra.
- Pagamento.
- Estorno.
- Compensação.

### 5.2 lancamentos_financeiros

Representa os efeitos concretos de uma movimentação sobre os elementos financeiros.

Uma movimentação pode gerar vários lançamentos, todos persistidos atomicamente.

Destinos permitidos:

1. Conta: somente `conta_id`.
2. Investimento: `conta_id` e `investimento_id`, obrigatoriamente correspondentes.
3. Cartão: somente `cartao_id`.

O relacionamento entre investimento e conta deverá ser protegido por FK composta.

O saldo será derivado dos lançamentos. Ajustes exigem movimentações justificadas.

## 6. Cartões, compras, parcelas e faturas

### 6.1 Cartões

`cartoes` é uma entidade própria, separada de `contas`.

O campo `conta_id` é opcional e representa uma associação cadastral, não a conta obrigatoriamente utilizada para pagar a fatura.

Cada cartão possui:
- Dia de fechamento.
- Dia de vencimento.
- Limite, quando aplicável.
- Situação operacional.

### 6.2 Faturas

Cada fatura armazena:
- Cartão.
- Data inicial do ciclo.
- Data de fechamento.
- Data de vencimento.
- Situação.

As datas ficam registradas na própria fatura para preservar o histórico, mesmo quando a configuração do cartão mudar.

**Regra de fechamento aprovada:**

- Compras realizadas até o dia anterior ao fechamento entram no ciclo que está fechando.
- Compras realizadas no próprio dia do fechamento entram no próximo ciclo.

A data da compra e, quando disponível, a data de processamento devem ser preservadas.

### 6.3 Compras e parcelas

Uma compra representa uma única movimentação financeira.

- Compra à vista: uma movimentação e uma parcela (1/1).
- Compra parcelada: uma movimentação pelo valor total e N registros em `parcelas`.
- A soma das parcelas deve corresponder ao valor total da compra.
- A distribuição de centavos deverá ser determinística.

A obrigação total da compra no cartão é reconhecida no momento da compra.

A inclusão posterior das parcelas nas faturas serve para organizar a cobrança, sem gerar novamente a obrigação no livro-razão.

### 6.4 Itens de fatura

`itens_faturas` representa os componentes cobrados ou creditados em uma fatura.

Pode referenciar:
- Uma parcela de compra.
- Juros.
- Multas.
- Anuidades.
- Tarifas.
- Descontos.
- Créditos.
- Ajustes.

Uma parcela poderá aparecer em no máximo um item de fatura.

Os valores serão positivos, com a natureza definindo se representam acréscimo ou redução.

A fatura será um agrupador de cobrança, não uma nova despesa.

Encargos e créditos que alterem a obrigação deverão produzir efeitos correspondentes no livro-razão exatamente uma vez.

### 6.5 Pagamentos de faturas

A tabela `pagamentos_faturas` relaciona uma fatura a uma movimentação de pagamento.

Relacionamentos:
- Uma fatura pode ter vários pagamentos.
- Cada pagamento pertence a uma única fatura.
- Cada pagamento corresponde a exatamente uma movimentação.
- Cada movimentação de pagamento corresponde a um único registro em `pagamentos_faturas`.

A conta pagadora será identificada pelos lançamentos financeiros da movimentação.

Não será duplicado `conta_id` na tabela `pagamentos_faturas`.

O pagamento reduz o saldo da conta e a obrigação do cartão, sem gerar uma nova despesa.

Pagamentos parciais são permitidos.

### 6.6 Estornos de pagamentos

A tabela `estornos_pagamentos_faturas` registra estornos parciais ou integrais de pagamentos.

Relacionamentos:
- Um pagamento pode possuir vários estornos.
- Cada estorno pertence a um pagamento.
- Cada estorno corresponde a uma movimentação própria.

A soma dos estornos efetivados não pode exceder o valor efetivamente pago.

O estorno produz lançamentos inversos aos do pagamento:
- Recomposição do saldo da conta.
- Restabelecimento da obrigação do cartão.

O pagamento original e os estornos efetivados são imutáveis.

Correções devem ocorrer por novas operações compensatórias.

## 7. Cancelamentos e devoluções de compras

Cancelamentos e devoluções serão operações compensatórias vinculadas à compra original.

Regras:
- A compra original permanece preservada.
- As parcelas originais permanecem registradas.
- Parcelas futuras ainda não faturadas são compensadas da última para a primeira.
- Valores excedentes seguem o crédito efetivamente concedido pela operadora ou o reembolso recebido.
- Parcelas faturadas e não pagas podem receber créditos nas respectivas faturas.
- Parcelas pagas são tratadas por crédito futuro ou entrada em conta, conforme o destino real.

Uma compensação nunca poderá reduzir a obrigação duas vezes.

O modelo físico deverá permitir rastrear a compra, o cancelamento, as parcelas afetadas, os créditos e os lançamentos correspondentes.

A estrutura definitiva de alocação das compensações por parcela ainda depende de validação do ERD.

## 8. Investimentos

`investimentos` será uma tabela própria, separada de `contas`.

Cada investimento possui obrigatoriamente uma conta vinculada.

Uma conta pode possuir vários investimentos.

Aportes:
- Saída da conta.
- Entrada no investimento.

Resgates:
- Saída do investimento.
- Entrada na conta.

Rendimentos são reconhecidos separadamente.

Projeções e estimativas não serão confundidas com rendimentos efetivamente recebidos.

O histórico de taxas, avaliações e movimentações deverá ser preservado.

## 9. Previsões e recorrências

Previsões não alteram saldos efetivos.

A efetivação cria uma movimentação real vinculada à previsão original.

Recorrências podem ser editadas, pausadas e encerradas. Uma ocorrência individual poderá ser alterada sem modificar a série.

O processamento será realizado por agendador simples, com persistência e idempotência no PostgreSQL.

Não haverá infraestrutura de filas no MVP.

## 10. Transações e concorrência

- Operações financeiras e seus lançamentos devem ser atômicos.
- PostgreSQL será a autoridade de integridade e concorrência.
- Constraints e transações protegerão invariantes.
- Operações concorrentes que alterem limites monetários deverão utilizar bloqueios ou isolamento adequado.
- Idempotência deverá ser persistida.
- Locks em memória não serão utilizados como mecanismo de integridade.

## 11. Imutabilidade e auditoria

Operações efetivadas não podem ser atualizadas ou apagadas.

Correções devem gerar novas operações de estorno ou compensação vinculadas às originais.

Cadastros podem utilizar exclusão lógica quando aplicável, mas isso nunca poderá ocultar o histórico financeiro.

## 12. Índices e desempenho

Os índices serão definidos conforme padrões de consulta e volume esperado.

Considerar:
- FKs utilizadas em joins.
- Filtros por usuário, situação e período.
- Ordenações cronológicas.
- Restrições de unicidade.
- Índices compostos para consultas multiusuário.
- Índices parciais para cadastros ativos quando úteis.

Evitar indexação indiscriminada.

Validar consultas representativas com `EXPLAIN (ANALYZE, BUFFERS)`.

## 13. Segurança multiusuário

O `usuario_id` será obtido da identidade autenticada no backend.

O backend não deve confiar no identificador enviado diretamente pelo frontend.

Consultas e comandos deverão respeitar o isolamento por usuário.

FKs compostas e constraints deverão impedir associações entre registros pertencentes a usuários diferentes.

## 14. Testes

Cobrir:
- Regras de valores monetários.
- Compras à vista e parceladas.
- Distribuição de centavos.
- Fechamento de faturas.
- Processamento tardio de compras.
- Encargos e créditos.
- Pagamentos parciais e múltiplos.
- Estornos parciais e múltiplos.
- Cancelamentos e devoluções.
- Concorrência.
- Idempotência.
- Integridade entre contas e investimentos.
- Isolamento multiusuário.
- Imutabilidade e auditoria.

## 15. Pendências

Antes da implementação definitiva:
- Fechar ERD físico e constraints.
- Definir estados e transições.
- Formalizar cálculo do saldo da fatura sem duplicidade.
- Fechar estrutura de cancelamentos e alocações por parcela.
- Definir processamento tardio e calendário de fechamento.
- Definir distribuição de centavos.
- Confirmar modalidades reais dos investimentos.