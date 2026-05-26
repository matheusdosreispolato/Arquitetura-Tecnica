# Arquitetura Técnica — Documentação

> **Sem acesso ao console AWS** — todos os scripts funcionam via sqlcmd, SSMS, PowerShell e AWS CLI  
> **Última atualização:** 2026-05-25

---

## Estrutura Geral

```
Arquitetura Tecnica/
├── AGENTS.md          ← guia para IAs: convenções, como contribuir, restrições do ambiente
├── README.md          ← este arquivo
│
├── MSSQL/             ← SQL Server: T-SQL geral + boas práticas + subpasta RDS
│   └── RDS/           ← específico do AWS RDS para SQL Server
│
├── RDL/               ← Report Builder / SSRS (.rdl)
└── S3/                ← Amazon S3 via CLI e PowerShell
```

> Para orientações sobre como atualizar ou expandir esta documentação, leia [`AGENTS.md`](./AGENTS.md).

---

## MSSQL — [`/MSSQL`](./MSSQL/README.md)

### T-SQL Geral — funciona em MSSQL padrão e AWS RDS

| Arquivo | Conteúdo |
|---|---|
| [01 — Básico / DML](./MSSQL/01-basico-dml.md) | SELECT, INSERT, UPDATE, DELETE, transações, datas, variáveis |
| [02 — DDL: Tabelas e Índices](./MSSQL/02-ddl-tabelas-indices.md) | CREATE TABLE, ALTER, DROP, índices, views, metadados |
| [03 — Programabilidade](./MSSQL/03-programabilidade.md) | Procedures, Functions, Triggers, JSON, SQL Dinâmico |
| [04 — Avançado e Performance](./MSSQL/04-avancado-performance.md) | CTEs, Window Functions, PIVOT, queries custosas, bloqueios |

### Boas Práticas

| Arquivo | Conteúdo |
|---|---|
| [09 — Padronização de Nomenclaturas](./MSSQL/09-padronizacao-nomenclaturas.md) | Schemas, tabelas, colunas, constraints, índices, views, procedures, logins, roles |
| [10 — JOINs, Views e Procedures](./MSSQL/10-joins-views-procedures.md) | INNER/LEFT/RIGHT/FULL JOIN para leigos e técnicos; quando usar View vs Procedure vs Function |

### AWS RDS — [`/MSSQL/RDS`](./MSSQL/RDS/README.md)

| Arquivo | Conteúdo |
|---|---|
| [01 — Conexão e Configuração](./MSSQL/RDS/01-conexao-configuracao.md) | Endpoint DNS, connection strings, sp_configure, Multi-AZ, failover |
| [02 — Usuários e Permissões](./MSSQL/RDS/02-usuarios-permissoes.md) | Logins SQL, roles, permissões, limitações do admin, perfis prontos |
| [03 — Backup e Restore](./MSSQL/RDS/03-backup-restore.md) | rds_backup_database (FULL/DIFF/LOG), restore, monitoramento, PowerShell |
| [04 — Limitações e Diferenças](./MSSQL/RDS/04-limitacoes-diferencas.md) | xp_cmdshell, BULK INSERT, SQL Agent, checklist de migração |

---

## RDL — [`/RDL`](./RDL/README.md)

| Arquivo | Conteúdo |
|---|---|
| [01 — Introdução e Estrutura](./RDL/01-introducao-estrutura.md) | O que é RDL, anatomia do XML, DataSource, Dataset, objetos visuais |
| [02 — Expressões e Sintaxe](./RDL/02-expressoes-sintaxe.md) | VB.NET, IIF, Switch, funções de texto/data/número, agregações, NULL |
| [03 — Datasets e Parâmetros](./RDL/03-datasets-parametros.md) | Conexões, queries, stored procedures, parâmetros em cascata, multi-valor |
| [04 — Formatação, Grupos e Agregações](./RDL/04-formatacao-grupos-agregacoes.md) | Cores condicionais, grupos, subtotais, ranking, paginação, gráficos |

---

## S3 — [`/S3`](./S3/README.md)

| Arquivo | Conteúdo |
|---|---|
| [01 — Configuração](./S3/01-configuracao.md) | AWS CLI, módulo PowerShell, credenciais, perfis, regiões |
| [02 — Operações Básicas](./S3/02-operacoes-basicas.md) | Listar, upload, download, copiar, deletar, URL pré-assinada |
| [03 — Automação e Scripts](./S3/03-automacao-scripts.md) | Limpeza por retenção, inventário, pipeline backup→S3→limpeza, DR entre buckets |

---

## Marcadores usados nos documentos

| Marcador | Significado |
|---|---|
| `🖥️ MSSQL Padrão` | Funciona em qualquer SQL Server (on-premises, VM, Docker) |
| `☁️ AWS RDS` | Específico para o RDS — comportamento diferente ou exclusivo |
| *(sem marcador)* | Funciona igual nos dois ambientes |

---

## Pré-requisitos

| Ferramenta | Uso | Instalação |
|---|---|---|
| **sqlcmd** | Executar T-SQL no terminal | `winget install Microsoft.SqlCmd` |
| **SSMS** / **Azure Data Studio** | Interface visual para o banco | [aka.ms/ssms](https://aka.ms/ssms) |
| **AWS CLI** | Gerenciar S3 | `winget install Amazon.AWSCLI` |
| **AWS.Tools.S3** | S3 via PowerShell | `Install-Module AWS.Tools.S3` |
| **SqlServer (PS)** | Queries via PowerShell | `Install-Module SqlServer` |
