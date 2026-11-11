
# Financeiro

Sistema de Controle Financeiro Pessoal.

Aplicação web para gestão financeira pessoal, com arquitetura multiusuário, foco em integridade contábil, rastreabilidade, segurança e auditoria.

## Objetivo

Centralizar o controle financeiro em uma única aplicação, permitindo acompanhar receitas, despesas, transferências, contas, cartões de crédito, faturas, parcelas, investimentos, previsões, recorrências, orçamentos e metas.

O sistema deverá permitir rastrear indicadores e saldos até os lançamentos financeiros que os originaram.

Embora o uso inicial seja pessoal, a arquitetura será multiusuário desde o início, com isolamento completo dos dados.

## Documentação oficial

A baseline documental do projeto é a versão 1.0.

| Documento | Descrição |
|---|---|
| [PRD](docs/PRD.md) | Requisitos funcionais, regras de negócio e critérios de aceite |
| [Architecture](docs/ARCHITECTURE.md) | Arquitetura, tecnologias e diretrizes técnicas |
| [ERD](docs/ERD.md) | Modelo conceitual e lógico do banco de dados |

Os documentos devem ser consultados antes de qualquer implementação.

Pendências identificadas nos documentos deverão ser resolvidas antes da implementação das partes afetadas.

## Stack tecnológica

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
- GitHub Actions quando necessário

## Arquitetura

O sistema utilizará arquitetura de monólito modular, com frontend e backend separados.

O PostgreSQL será a autoridade de integridade e concorrência das operações financeiras.

Não serão introduzidos microsserviços, Redis ou infraestrutura de filas no MVP sem justificativa concreta e aprovação.

## Convenções importantes

- Tabelas e colunas em português, utilizando `snake_case`.
- Identificadores primários `BIGINT GENERATED ALWAYS AS IDENTITY`.
- Chaves estrangeiras `BIGINT`.
- Valores monetários em `NUMERIC`, sem ponto flutuante.
- Backend utilizando `Decimal` para cálculos financeiros.
- Valores monetários transportados pela API como strings decimais.
- Moeda explícita, com BRL no MVP.
- Operações financeiras efetivadas e seus históricos são imutáveis.
- Correções realizadas por estornos e compensações.
- Isolamento multiusuário garantido no backend e no banco de dados.

## Estrutura do projeto

```text
financeiro/
├── docs/
│   ├── PRD.md
│   ├── ARCHITECTURE.md
│   └── ERD.md
├── backend/
├── frontend/
├── .env.example
├── .gitignore
└── README.md
```

As pastas de backend e frontend serão estruturadas durante as respectivas etapas de implementação.

## Desenvolvimento

As decisões de implementação deverão respeitar a documentação oficial.

Alterações de arquitetura ou regras de negócio devem ser discutidas e documentadas antes de serem incorporadas.

Migrations, commits, push, merge e operações destrutivas exigem autorização explícita do responsável pelo projeto.
