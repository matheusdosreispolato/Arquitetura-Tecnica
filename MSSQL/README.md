# MSSQL — Documentação T-SQL e AWS RDS

> Documentação completa de SQL Server: comandos T-SQL válidos em qualquer instalação, boas práticas e conteúdo específico do AWS RDS na subpasta `/RDS`.

---

## Estrutura desta pasta

```
MSSQL/
├── README.md                        ← este arquivo
├── 01-basico-dml.md                 ← SELECT, INSERT, UPDATE, DELETE
├── 02-ddl-tabelas-indices.md        ← CREATE TABLE, ALTER, índices, constraints
├── 03-programabilidade.md           ← Procedures, Functions, Triggers, JSON
├── 04-avancado-performance.md       ← CTEs, Window Functions, monitoramento
├── 09-padronizacao-nomenclaturas.md ← Padrões de nomes para todos os objetos
├── 10-joins-views-procedures.md     ← JOINs explicados + View vs Procedure
└── RDS/                             ← Específico do AWS RDS para SQL Server
    ├── README.md
    ├── 01-conexao-configuracao.md
    ├── 02-usuarios-permissoes.md
    ├── 03-backup-restore.md
    └── 04-limitacoes-diferencas.md
```

---

## T-SQL — Funciona em MSSQL Padrão e AWS RDS

| Arquivo | Conteúdo |
|---|---|
| [01 — Básico / DML](./01-basico-dml.md) | Conexão, SELECT, INSERT, UPDATE, DELETE, transações, datas, variáveis |
| [02 — DDL: Tabelas e Índices](./02-ddl-tabelas-indices.md) | CREATE TABLE, ALTER, DROP, índices (tipos, rebuild, fragmentação), views |
| [03 — Programabilidade](./03-programabilidade.md) | Stored Procedures, Functions, Triggers, JSON, SQL Dinâmico, tabelas temporárias |
| [04 — Avançado e Performance](./04-avancado-performance.md) | CTEs recursivas, Window Functions, PIVOT dinâmico, queries custosas, bloqueios, missing indexes |

---

## Boas Práticas

| Arquivo | Conteúdo |
|---|---|
| [09 — Padronização de Nomenclaturas](./09-padronizacao-nomenclaturas.md) | Padrões de nome para tabelas, colunas, schemas, logins, roles, procedures, views, índices, constraints |
| [10 — JOINs, Views e Procedures](./10-joins-views-procedures.md) | INNER/LEFT/RIGHT/FULL JOIN (para leigos e técnicos), quando usar View vs Procedure vs Function |

---

## AWS RDS — Conteúdo Específico

Tudo que é diferente ou exclusivo do AWS RDS para SQL Server está em [`/RDS`](./RDS/README.md):

- Conexão por endpoint DNS, connection strings, sp_configure no RDS
- Logins, roles e permissões no RDS
- Backup e Restore para S3 via `rds_backup_database`
- Limitações: xp_cmdshell, BULK INSERT, SQL Agent — o que muda e alternativas

---

## Marcadores usados nos documentos

| Marcador | Significado |
|---|---|
| `🖥️ MSSQL Padrão` | Funciona em qualquer SQL Server (on-premises, VM, Docker) |
| `☁️ AWS RDS` | Comportamento diferente ou exclusivo do RDS |
| *(sem marcador)* | Funciona igual nos dois ambientes |
