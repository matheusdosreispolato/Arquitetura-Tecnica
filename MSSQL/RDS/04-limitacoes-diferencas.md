# Limitações e Diferenças do AWS RDS para SQL Server

> Guia de tudo que funciona diferente, o que não existe no RDS e como contornar cada situação — sem acesso ao console AWS. Para cada limitação, é indicado se o impacto ou a alternativa é no **nível RDS (AWS CLI)** ou no **nível SQL Server (T-SQL)**.

---

## Os dois níveis

```
☁️ Nível RDS (AWS CLI)       → configurações de instância, Parameter Group,
                                Option Group, Security Group, reinicialização

🗄️ Nível SQL Server (T-SQL) → o que o T-SQL pode ou não executar dentro
                                do banco, stored procedures disponíveis,
                                permissões, objetos de sistema
```

---

## 1. Tabela Geral de Diferenças

| Recurso | 🖥️ MSSQL Padrão | ☁️ Nível RDS | 🗄️ Nível SQL Server (RDS) |
|---|---|---|---|
| `xp_cmdshell` | Disponível | Não gerenciável via CLI | **Bloqueado** — não executa |
| `BACKUP TO DISK` | Disponível | Não se aplica | **Bloqueado** — use `rds_backup_database` |
| `BULK INSERT` de arquivo local | Disponível | Não se aplica | **Bloqueado** — sem acesso a disco |
| `DBCC DROPCLEANBUFFERS` | Disponível | Não se aplica | **Bloqueado** |
| Criar login `sysadmin` | Disponível | Gerenciado pela AWS | **Bloqueado** via T-SQL |
| SQL Server Agent | Disponível completo | Status via `describe-db-instances` | Disponível (sem `CmdExec`) |
| Linked Servers | Disponível | Requer Security Group aberto | Disponível (com restrições) |
| Logins Windows (AD local) | Disponível | Requer AWS Directory Service | Não disponível via T-SQL puro |
| Trace Flags na inicialização | Parâmetro de serviço | Via Parameter Group | Não configurável por T-SQL |
| `sp_configure` completo | Todas as opções | Algumas via Parameter Group | Parte disponível, parte bloqueada |
| Arquivo MDF/LDF customizado | Controle total do caminho | Gerenciado pela AWS | Não configurável |
| Múltiplas instâncias (`INSTANCIA\NOME`) | Disponível | Uma instância por RDS | Não disponível |
| `RESTORE ... STOPAT` | Disponível | Point-in-time via snapshot RDS | Não disponível diretamente |
| `ONLINE = ON` no Rebuild de índice | Enterprise Edition | Apenas Enterprise RDS | Exige edição correta |

---

## 2. xp_cmdshell

### 🗄️ Nível SQL Server — T-SQL

```sql
-- MSSQL Padrão: habilitar e usar xp_cmdshell
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
EXEC xp_cmdshell 'dir C:\Backups';

-- ❌ No RDS: xp_cmdshell não pode ser habilitado — retorna erro de permissão
-- mesmo com o usuário master (admin), não é possível ativar xp_cmdshell no RDS
```

### ☁️ Nível RDS — Alternativa via PowerShell externo

```powershell
# Em vez de xp_cmdshell, use PowerShell no host externo para
# executar comandos de sistema operacional e interagir com o banco via sqlcmd

# Exemplo: listar arquivos de backup no S3 (equivale a "dir" de uma pasta de backup)
aws s3 ls s3://meu-bucket/backups/ --human-readable

# Exemplo: executar query no banco via PowerShell externo
Invoke-Sqlcmd `
    -ServerInstance "meu-rds.xxxx.sa-east-1.rds.amazonaws.com,1433" `
    -Username "admin" `
    -Password "SuaSenha123!" `
    -Query "SELECT TOP 10 * FROM vendas.Pedido WHERE PedidoID > 1000"
```

---

## 3. BACKUP TO DISK

### 🗄️ Nível SQL Server — T-SQL

```sql
-- MSSQL Padrão:
BACKUP DATABASE MeuBanco TO DISK = 'C:\Backups\MeuBanco.bak';  -- ❌ Não funciona no RDS

-- ✅ No RDS — use rds_backup_database direcionando para o S3:
EXEC msdb.dbo.rds_backup_database
    @source_db_name       = 'MeuBanco',
    @s3_arn_to_backup_to  = 'arn:aws:s3:::meu-bucket/backups/MeuBanco_FULL.bak',
    @type                 = 'FULL',
    @overwrite_s3_backup_file = 1,
    @backup_compression   = 1;
```

### ☁️ Nível RDS — Snapshots como alternativa complementar

```powershell
# ✅ Snapshot manual da instância inteira (nível RDS, não requer Option Group)
aws rds create-db-snapshot `
    --db-instance-identifier minha-instancia-rds `
    --db-snapshot-identifier backup-manual-20240615

# Ver todos os snapshots disponíveis
aws rds describe-db-snapshots `
    --db-instance-identifier minha-instancia-rds `
    --output table
```

> **Diferença importante:** `rds_backup_database` exporta um banco individual para S3 (`.bak` portátil). O snapshot RDS captura a instância inteira e só pode ser restaurado como uma nova instância RDS — não é um `.bak` transferível.

---

## 4. BULK INSERT de Arquivo Local

### 🗄️ Nível SQL Server — T-SQL

```sql
-- MSSQL Padrão:
BULK INSERT dbo.Clientes
FROM 'C:\Data\clientes.csv'
WITH (FIELDTERMINATOR=',', ROWTERMINATOR='\n');  -- ❌ Não funciona no RDS

-- ✅ Alternativa via T-SQL no RDS: INSERT linha a linha ou via aplicação
-- Para volumes pequenos, use INSERT direto:
INSERT INTO dbo.Clientes (Nome, Email) VALUES ('João Silva', 'joao@empresa.com');
```

### ☁️ Nível RDS — Alternativa via PowerShell externo (SqlBulkCopy)

```powershell
# ✅ Substituto do BULK INSERT no RDS: PowerShell com SqlBulkCopy
param(
    [string]$Endpoint   = "meu-rds.xxxx.sa-east-1.rds.amazonaws.com,1433",
    [string]$Usuario    = "admin",
    [string]$Senha      = "SuaSenha123!",
    [string]$Banco      = "MeuBanco",
    [string]$Tabela     = "dbo.Clientes",
    [string]$ArquivoCSV = "C:\Temp\clientes.csv"
)

Import-Module SqlServer

$dados   = Import-Csv -Path $ArquivoCSV -Delimiter ','
$connStr = "Server=$Endpoint;Database=$Banco;User Id=$Usuario;Password=$Senha;TrustServerCertificate=True;"
$conn    = New-Object System.Data.SqlClient.SqlConnection($connStr)
$conn.Open()

$bulkCopy = New-Object System.Data.SqlClient.SqlBulkCopy($conn)
$bulkCopy.DestinationTableName = $Tabela
$bulkCopy.BatchSize = 1000

$dt = New-Object System.Data.DataTable
$dados[0].PSObject.Properties.Name | ForEach-Object { [void]$dt.Columns.Add($_) }
$dados | ForEach-Object {
    $row = $dt.NewRow()
    $_.PSObject.Properties | ForEach-Object { $row[$_.Name] = $_.Value }
    $dt.Rows.Add($row)
}

$bulkCopy.WriteToServer($dt)
$conn.Close()
Write-Host "Importacao concluida: $($dados.Count) registros." -ForegroundColor Green
```

---

## 5. DBCC Bloqueados no RDS

### 🗄️ Nível SQL Server — T-SQL

```sql
-- ❌ Não disponível no RDS:
DBCC DROPCLEANBUFFERS;     -- limpa buffer pool (bloqueado)
DBCC SHRINKFILE(...);      -- em algumas versões do RDS (verificar)

-- ✅ Disponível no RDS:
DBCC FREEPROCCACHE;        -- limpa cache de planos de execução
DBCC CHECKDB('MeuBanco');  -- verificar integridade do banco
DBCC SHOW_STATISTICS('vendas.Pedido', 'IX_Pedido_ClienteID');
DBCC SQLPERF(LOGSPACE);    -- uso do log de transações
```

---

## 6. Criar Login sysadmin

### 🗄️ Nível SQL Server — T-SQL

```sql
-- MSSQL Padrão:
CREATE LOGIN super_admin WITH PASSWORD = 'Senha@123!';
EXEC sp_addsrvrolemember 'super_admin', 'sysadmin';  -- ❌ Bloqueado no RDS

-- ✅ No RDS, use db_owner por banco em vez de sysadmin no servidor:
CREATE LOGIN dba_suporte WITH PASSWORD = 'Dba@Sup2024!';
GO
USE MeuBanco;
CREATE USER dba_suporte FOR LOGIN dba_suporte;
ALTER ROLE db_owner ADD MEMBER dba_suporte;
GO

-- Verificar permissões de servidor do usuário atual
SELECT * FROM fn_my_permissions(NULL, 'SERVER');
```

### ☁️ Nível RDS — Verificar o usuário master

```powershell
# O usuário master é o único com poderes próximos a sysadmin.
# Verificar qual é o usuário master da instância:
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].MasterUsername" `
    --output text

# Redefinir senha do master se necessário:
aws rds modify-db-instance `
    --db-instance-identifier minha-instancia-rds `
    --master-user-password "NovaSenha@Segura2024!" `
    --apply-immediately
```

> **Limitação do master no RDS:** o usuário master tem poderes equivalentes a `sysadmin`, mas não pode adicionar outros logins à role `sysadmin`, não acessa o sistema operacional e não pode usar `xp_cmdshell`.

---

## 7. SQL Server Agent

O SQL Agent **existe** no RDS, mas com restrições específicas.

### 🗄️ Nível SQL Server — T-SQL

```sql
-- ✅ Criar job via T-SQL (funciona no RDS — apenas passos TSQL, SSIS, SSAS)
USE msdb;
GO

EXEC sp_add_job @job_name = 'Backup_Diario_MeuBanco';

EXEC sp_add_jobstep
    @job_name    = 'Backup_Diario_MeuBanco',
    @step_name   = 'Executar_Backup',
    @subsystem   = 'TSQL',           -- ✅ TSQL funciona | ❌ CmdExec NÃO funciona
    @command     = N'
EXEC msdb.dbo.rds_backup_database
    @source_db_name       = ''MeuBanco'',
    @s3_arn_to_backup_to  = ''arn:aws:s3:::meu-bucket/backups/auto_backup.bak'',
    @type                 = ''FULL'',
    @overwrite_s3_backup_file = 1;',
    @database_name = 'msdb';

EXEC sp_add_schedule
    @schedule_name      = 'TodosDias2h',
    @freq_type          = 4,        -- 4 = Diário
    @freq_interval      = 1,
    @active_start_time  = 020000;   -- 02:00:00

EXEC sp_attach_schedule
    @job_name      = 'Backup_Diario_MeuBanco',
    @schedule_name = 'TodosDias2h';

EXEC sp_add_jobserver @job_name = 'Backup_Diario_MeuBanco';
GO

-- Verificar jobs existentes
SELECT job_id, name, enabled, description FROM msdb.dbo.sysjobs ORDER BY name;

-- Executar job manualmente
EXEC msdb.dbo.sp_start_job @job_name = 'Backup_Diario_MeuBanco';

-- Histórico de execuções
SELECT
    j.name                  AS Job,
    h.run_date,
    h.run_time,
    h.run_status,           -- 1 = Sucesso, 0 = Falha
    h.run_duration,
    h.message
FROM msdb.dbo.sysjobhistory h
JOIN msdb.dbo.sysjobs j ON j.job_id = h.job_id
ORDER BY h.run_date DESC, h.run_time DESC;
```

### ☁️ Nível RDS — Verificar status do Agent via CLI

```powershell
# Verificar se o SQL Agent está rodando (eventos de reinício do Agent aparecem aqui)
aws rds describe-events `
    --source-identifier minha-instancia-rds `
    --source-type db-instance `
    --duration 1440 `
    --output table
```

> **Restrição no RDS:** Passos do tipo `CmdExec` (comandos de SO) **não funcionam** no Agent do RDS. Use apenas `TSQL`. Para automações que precisariam de `CmdExec`, substitua por scripts PowerShell externos agendados no Task Scheduler do servidor local.

---

## 8. Linked Servers

### 🗄️ Nível SQL Server — T-SQL

```sql
-- ✅ Linked Server para outro SQL Server (geralmente funciona no RDS)
EXEC sp_addlinkedserver
    @server     = 'ServidorRemoto',
    @srvproduct = '',
    @provider   = 'SQLOLEDB',
    @datasrc    = 'outro-servidor.exemplo.com,1433';

EXEC sp_addlinkedsrvlogin
    @rmtsrvname  = 'ServidorRemoto',
    @useself     = 'FALSE',
    @rmtuser     = 'usuario_remoto',
    @rmtpassword = 'senha_remota';

-- Testar
SELECT * FROM ServidorRemoto.MeuBanco.dbo.Tabela;

-- Remover linked server
EXEC sp_dropserver 'ServidorRemoto', 'droplogins';
```

### ☁️ Nível RDS — Liberar tráfego no Security Group

```powershell
# O linked server só funciona se o Security Group da VPC do RDS
# permitir tráfego de saída para a porta 1433 do servidor remoto.

# Verificar os Security Groups da instância RDS:
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].VpcSecurityGroups" `
    --output table

# Adicionar regra de saída no Security Group (substitua sg-xxxxxxxx e o IP/CIDR):
aws ec2 authorize-security-group-egress `
    --group-id sg-xxxxxxxx `
    --protocol tcp `
    --port 1433 `
    --cidr 10.0.0.0/16
```

---

## 9. Logins Windows (Active Directory)

### 🗄️ Nível SQL Server — T-SQL

```sql
-- MSSQL Padrão:
CREATE LOGIN [DOMINIO\usuario] FROM WINDOWS;  -- ❌ Não disponível via T-SQL no RDS

-- ✅ No RDS sem AD: use apenas logins SQL (com senha)
CREATE LOGIN app_usuario WITH PASSWORD = 'App@Usuario2024!';
```

### ☁️ Nível RDS — Configurar AWS Directory Service (requer ajuste na instância)

```powershell
# Para habilitar autenticação Windows no RDS, a instância deve estar
# associada a um AWS Managed Microsoft AD.

# Verificar se a instância está associada a um domínio:
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].DomainMemberships" `
    --output table

# Associar instância a um domínio existente (requer Directory ID e IAM Role):
aws rds modify-db-instance `
    --db-instance-identifier minha-instancia-rds `
    --domain "d-xxxxxxxxxx" `
    --domain-iam-role-name "rds-directory-service-role" `
    --apply-immediately
```

---

## 10. Trace Flags e sp_configure Restritos

### ☁️ Nível RDS — AWS CLI (Parameter Group)

Parâmetros que normalmente seriam definidos via `sp_configure` ou trace flags de inicialização são gerenciados via Parameter Group no RDS.

```powershell
# Listar Parameter Group associado à instância
$pg = (aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].DBParameterGroups[0].DBParameterGroupName" `
    --output text)

# Ver todos os parâmetros do grupo
aws rds describe-db-parameters `
    --db-parameter-group-name $pg `
    --output table

# Alterar parâmetro equivalente a trace flag / sp_configure
aws rds modify-db-parameter-group `
    --db-parameter-group-name $pg `
    --parameters "ParameterName=max degree of parallelism,ParameterValue=4,ApplyMethod=immediate"

aws rds modify-db-parameter-group `
    --db-parameter-group-name $pg `
    --parameters "ParameterName=optimize for ad hoc workloads,ParameterValue=1,ApplyMethod=immediate"

# Reiniciar a instância para aplicar parâmetros com ApplyMethod=pending-reboot
aws rds reboot-db-instance `
    --db-instance-identifier minha-instancia-rds

aws rds wait db-instance-available `
    --db-instance-identifier minha-instancia-rds
```

### 🗄️ Nível SQL Server — T-SQL (o que ainda funciona via sp_configure)

```sql
-- Verificar configurações atuais (as que o RDS permite)
SELECT name, value, value_in_use, description
FROM sys.configurations
WHERE name IN (
    'max degree of parallelism',
    'cost threshold for parallelism',
    'optimize for ad hoc workloads',
    'max server memory (MB)',
    'fill factor (%)'
)
ORDER BY name;

-- ✅ Alterar via sp_configure (parâmetros permitidos no RDS)
EXEC sp_configure 'max degree of parallelism', 4;   RECONFIGURE;
EXEC sp_configure 'cost threshold for parallelism', 25; RECONFIGURE;
EXEC sp_configure 'optimize for ad hoc workloads', 1;  RECONFIGURE;

-- ❌ Tentativa de alterar parâmetros bloqueados retorna erro de permissão
-- EXEC sp_configure 'show advanced options', 1;  -- restrito no RDS
```

---

## 11. Restore Point-in-Time

### ☁️ Nível RDS — AWS CLI

O RDS mantém backups automáticos contínuos dentro da janela de retenção configurada, permitindo restore para qualquer ponto desse período.

```powershell
# Ver o ponto mais recente disponível para restore
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].LatestRestorableTime" `
    --output text

# Restore para um ponto específico no tempo (cria nova instância)
aws rds restore-db-instance-to-point-in-time `
    --source-db-instance-identifier minha-instancia-rds `
    --target-db-instance-identifier minha-instancia-pit `
    --restore-time "2024-06-15T14:30:00Z" `
    --db-instance-class db.t3.medium

# Aguardar a nova instância ficar disponível
aws rds wait db-instance-available `
    --db-instance-identifier minha-instancia-pit
```

### 🗄️ Nível SQL Server — T-SQL

```sql
-- ❌ RESTORE ... STOPAT não está disponível diretamente no RDS.
-- A alternativa via T-SQL é fazer backup de LOG frequente e restaurar manualmente:

-- 1. Backup FULL
EXEC msdb.dbo.rds_backup_database
    @source_db_name = 'MeuBanco',
    @s3_arn_to_backup_to = 'arn:aws:s3:::meu-bucket/MeuBanco_FULL.bak',
    @type = 'FULL';

-- 2. Backups de LOG periódicos (a cada hora, por exemplo)
EXEC msdb.dbo.rds_backup_database
    @source_db_name = 'MeuBanco',
    @s3_arn_to_backup_to = 'arn:aws:s3:::meu-bucket/logs/MeuBanco_LOG_14h.bak',
    @type = 'LOG';

-- 3. Para point-in-time manual: restore FULL com NORECOVERY + aplicar LOGs em sequência
EXEC msdb.dbo.rds_restore_database
    @restore_db_name = 'MeuBancoPIT',
    @s3_arn_to_restore_from = 'arn:aws:s3:::meu-bucket/MeuBanco_FULL.bak',
    @with_norecovery = 1;

EXEC msdb.dbo.rds_restore_log
    @restore_db_name = 'MeuBancoPIT',
    @s3_arn_to_restore_from = 'arn:aws:s3:::meu-bucket/logs/MeuBanco_LOG_14h.bak',
    @with_norecovery = 1;

EXEC msdb.dbo.rds_finish_restore @db_name = 'MeuBancoPIT';
```

---

## 12. Checklist: Antes de Migrar para o RDS

```
NÍVEL SQL SERVER (T-SQL):
[ ] Scripts que usam xp_cmdshell → migrar para PowerShell externo
[ ] BACKUP TO DISK → migrar para rds_backup_database → S3
[ ] BULK INSERT de arquivo local → migrar para PowerShell + SqlBulkCopy
[ ] DBCC DROPCLEANBUFFERS → remover ou substituir por FREEPROCCACHE
[ ] Scripts que assumem caminho de arquivo (MDF/LDF) → remover dependências de path
[ ] Jobs com passos CmdExec → reescrever como TSQL ou script externo agendado
[ ] Logins Windows locais → avaliar AWS Directory Service
[ ] RESTORE ... STOPAT → usar backup de LOG frequente + restore em etapas
[ ] sp_addsrvrolemember com 'sysadmin' → substituir por db_owner por banco

NÍVEL RDS (AWS CLI / infraestrutura):
[ ] Verificar Option Group: SQLSERVER_BACKUP_RESTORE habilitado para rds_backup_database
[ ] Verificar Parameter Group: parâmetros como MAXDOP, fill factor, optimize for ad hoc
[ ] Verificar Security Groups: portas abertas para Linked Servers e conexões externas
[ ] Configurar janela de backup automático e retenção adequada
[ ] Avaliar Multi-AZ se alta disponibilidade for requisito
[ ] Trace Flags → verificar equivalente em Parameter Group
```
