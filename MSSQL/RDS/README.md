# RDS — AWS RDS para SQL Server

> Esta pasta contém documentação **exclusiva do AWS RDS para SQL Server** — o que muda em relação a uma instalação padrão, limitações, e como realizar operações sem acesso ao console AWS.
>
> Para comandos T-SQL gerais (SELECT, CREATE TABLE, Procedures etc.), consulte a pasta pai [`/MSSQL`](../README.md).

---

## Estrutura desta pasta

```
RDS/
├── README.md                    ← este arquivo
├── 01-conexao-configuracao.md   ← Endpoint, connection strings, sp_configure, Multi-AZ
├── 02-usuarios-permissoes.md    ← Logins, roles, permissões e limitações do user admin
├── 03-backup-restore.md         ← rds_backup_database → S3, restore, scripts PowerShell
└── 04-limitacoes-diferencas.md  ← O que não existe no RDS e como contornar
```

---

## Documentos

| Arquivo | Conteúdo |
|---|---|
| [01 — Conexão e Configuração](./01-conexao-configuracao.md) | Endpoint DNS, formatos de connection string (.NET, JDBC, Python), `sp_configure` no RDS, verificar saúde da instância, Multi-AZ e failover |
| [02 — Usuários e Permissões](./02-usuarios-permissoes.md) | Criar logins SQL, logins Windows (não disponível), o que o usuário `admin` pode e não pode, roles fixas, permissões granulares, perfis prontos |
| [03 — Backup e Restore](./03-backup-restore.md) | `rds_backup_database` (FULL/DIFF/LOG), `rds_restore_database`, monitorar e cancelar tasks, scripts PowerShell de backup automático e restore com monitoramento |
| [04 — Limitações e Diferenças](./04-limitacoes-diferencas.md) | Tabela completa do que muda no RDS, alternativas para `xp_cmdshell`, `BACKUP TO DISK`, `BULK INSERT`, SQL Agent sem `CmdExec`, checklist de migração |

---

## Resumo das Principais Diferenças

| Recurso | 🖥️ MSSQL Padrão | ☁️ AWS RDS |
|---|---|---|
| Backup | `BACKUP DATABASE TO DISK` | `rds_backup_database` → S3 |
| Acesso ao SO | `xp_cmdshell` | **Não disponível** → PowerShell externo |
| Login Windows | Disponível | **Não disponível** via T-SQL |
| `sp_configure` | Todas as opções | Algumas restritas → Parameter Group |
| `DBCC DROPCLEANBUFFERS` | Disponível | **Não disponível** → `FREEPROCCACHE` |
| Criar `sysadmin` | Disponível | **Não disponível** via T-SQL |
| Modo de conexão | `SERVIDOR\INSTANCIA` | Endpoint DNS + porta 1433 |

---

## Pré-requisitos para uso desta pasta

- Acesso ao endpoint RDS via porta 1433 (Security Group liberado)
- Login SQL com permissões adequadas (não é necessário acesso ao console AWS)
- Para backup/restore: Option Group com `SQLSERVER_BACKUP_RESTORE` habilitado
- Para scripts PowerShell: `sqlcmd` e módulo `SqlServer` instalados
