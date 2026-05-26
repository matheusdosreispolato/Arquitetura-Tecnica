# AWS RDS MSSQL — Limitações e Diferenças em Relação ao MSSQL Padrão

> Guia rápido do que funciona diferente, o que não existe no RDS e como contornar cada situação sem acesso ao console AWS.

---

## 1. Tabela Geral de Diferenças

| Recurso | 🖥️ MSSQL Padrão | ☁️ AWS RDS | Alternativa no RDS |
|---|---|---|---|
| `xp_cmdshell` | Disponível | **Não disponível** | PowerShell externo |
| Acesso ao sistema de arquivos | `BULK INSERT`, `OPENROWSET` com disco local | **Restrito** | S3 + `rds_backup_database` |
| Backup para disco local | `BACKUP TO DISK` | **Não disponível** | `rds_backup_database` → S3 |
| SQL Server Agent | Disponível completo | Disponível (limitado) | msdb jobs funcionam |
| Login Windows (AD local) | Disponível | **Não disponível** | AWS Directory Service |
| `DBCC DROPCLEANBUFFERS` | Disponível | **Não disponível** | `DBCC FREEPROCCACHE` |
| Trace Flags (`-T`) | Na inicialização do serviço | Via Parameter Group AWS | Parameter Group |
| `sp_configure` completo | Todas as opções | Algumas restritas | Parameter Group AWS |
| Criar `sysadmin` | Disponível | **Não disponível via T-SQL** | Só via console AWS |
| Múltiplas instâncias | `SERVIDOR\INSTANCIA` | Uma instância por RDS | Criar outro RDS |
| Arquivo de dados (MDF/LDF) | Controle total do caminho | Gerenciado pela AWS | Não configurável |
| `ONLINE = ON` no Rebuild | Enterprise Edition | Apenas Enterprise RDS | Planejar janela de manutenção |
| `RESTORE ... STOPAT` | Disponível | Não direto via T-SQL | Snapshot automatico do RDS |
| Linked Servers | Disponível | **Limitado** | Alternativas abaixo |

---

## 2. O que NÃO existe no RDS e como contornar

### ❌ `xp_cmdshell`

```sql
-- MSSQL Padrão:
EXEC xp_cmdshell 'dir C:\Backups';  -- NÃO funciona no RDS

-- ✅ Alternativa no RDS: use PowerShell externamente
# Via PowerShell com sqlcmd
Invoke-Sqlcmd -ServerInstance $endpoint -Username $user -Password $pass `
    -Query "SELECT * FROM sys.databases"
```

### ❌ `BACKUP DATABASE TO DISK`

```sql
-- MSSQL Padrão:
BACKUP DATABASE MeuBanco TO DISK = 'C:\Backups\backup.bak';  -- NÃO no RDS

-- ✅ Alternativa no RDS:
EXEC msdb.dbo.rds_backup_database
    @source_db_name      = 'MeuBanco',
    @s3_arn_to_backup_to = 'arn:aws:s3:::meu-bucket/backup.bak',
    @type                = 'FULL';
```

### ❌ `BULK INSERT` de arquivo local

```sql
-- MSSQL Padrão:
BULK INSERT dbo.Clientes FROM 'C:\Data\clientes.csv'
WITH (FIELDTERMINATOR=',', ROWTERMINATOR='\n');  -- NÃO no RDS

-- ✅ Alternativa 1: Inserir via aplicação/script PowerShell
# Ler CSV em PowerShell e usar Invoke-Sqlcmd com INSERT

-- ✅ Alternativa 2: OPENROWSET com S3 (requer configuração de linked server S3)
-- Veja seção 5 deste documento.
```

### ❌ `DBCC DROPCLEANBUFFERS`

```sql
-- NÃO disponível no RDS
-- ✅ Use apenas FREEPROCCACHE para limpar planos de execução:
DBCC FREEPROCCACHE;
```

### ❌ Criar login `sysadmin`

```sql
-- MSSQL Padrão:
CREATE LOGIN super_user WITH PASSWORD = 'senha';
EXEC sp_addsrvrolemember 'super_user', 'sysadmin';  -- NÃO no RDS

-- ✅ No RDS, use db_owner no banco e não sysadmin no servidor.
-- Para acesso equivalente a sysadmin, configure via console AWS.
```

---

## 3. SQL Server Agent no RDS

O SQL Agent **existe** no RDS, mas com algumas restrições:

```sql
-- Criar job via T-SQL (funciona no RDS)
USE msdb;
GO

EXEC sp_add_job
    @job_name = 'Backup_Diario_MeuBanco';

EXEC sp_add_jobstep
    @job_name    = 'Backup_Diario_MeuBanco',
    @step_name   = 'Executar_Backup',
    @subsystem   = 'TSQL',
    @command     = N'
EXEC msdb.dbo.rds_backup_database
    @source_db_name       = ''MeuBanco'',
    @s3_arn_to_backup_to  = ''arn:aws:s3:::meu-bucket/backups/auto_backup.bak'',
    @type                 = ''FULL'',
    @overwrite_s3_backup_file = 1;',
    @database_name = 'msdb';

EXEC sp_add_schedule
    @schedule_name      = 'TodosDias2h',
    @freq_type          = 4,    -- Diário
    @freq_interval      = 1,
    @active_start_time  = 020000;  -- 02:00:00

EXEC sp_attach_schedule
    @job_name      = 'Backup_Diario_MeuBanco',
    @schedule_name = 'TodosDias2h';

EXEC sp_add_jobserver
    @job_name = 'Backup_Diario_MeuBanco';
GO

-- Verificar jobs existentes
SELECT job_id, name, enabled, description
FROM msdb.dbo.sysjobs ORDER BY name;

-- Executar job manualmente
EXEC msdb.dbo.sp_start_job @job_name = 'Backup_Diario_MeuBanco';

-- Histórico de execuções
SELECT
    j.name                      AS Job,
    h.run_date,
    h.run_time,
    h.run_status,               -- 1 = Sucesso, 0 = Falha
    h.run_duration,
    h.message
FROM msdb.dbo.sysjobhistory h
INNER JOIN msdb.dbo.sysjobs j ON j.job_id = h.job_id
ORDER BY h.run_date DESC, h.run_time DESC;
```

> **Restrição no RDS:** Passos do tipo `CmdExec` (sistema operacional) **não funcionam** no Agent do RDS — apenas `TSQL`, `SSIS`, `SSAS`.

---

## 4. Linked Servers no RDS

```sql
-- Linked Server para outro SQL Server (funciona no RDS)
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

> **Atenção no RDS:** Linked servers para providers OLE DB externos podem ter restrições. Linked servers para outros SQL Servers geralmente funcionam, mas depende do Security Group da VPC liberar o tráfego.

---

## 5. Importar Dados de CSV sem BULK INSERT (Alternativa PowerShell)

```powershell
# ============================================================
# Importar CSV para tabela SQL via PowerShell
# Substitui BULK INSERT em ambientes RDS
# ============================================================
param(
    [string]$Endpoint   = "meu-rds.xxxx.sa-east-1.rds.amazonaws.com,1433",
    [string]$Usuario    = "admin",
    [string]$Senha      = "SuaSenha123!",
    [string]$Banco      = "MeuBanco",
    [string]$Tabela     = "dbo.Clientes",
    [string]$ArquivoCSV = "C:\Temp\clientes.csv"
)

Import-Module SqlServer

# Ler CSV
$dados = Import-Csv -Path $ArquivoCSV -Delimiter ','

Write-Host "Importando $($dados.Count) registros para $Tabela..."

# Usar SqlBulkCopy para performance
$connStr = "Server=$Endpoint;Database=$Banco;User Id=$Usuario;Password=$Senha;TrustServerCertificate=True;"
$conn    = New-Object System.Data.SqlClient.SqlConnection($connStr)
$conn.Open()

$bulkCopy = New-Object System.Data.SqlClient.SqlBulkCopy($conn)
$bulkCopy.DestinationTableName = $Tabela
$bulkCopy.BatchSize = 1000

# Converter para DataTable
$dt = New-Object System.Data.DataTable
$dados[0].PSObject.Properties.Name | ForEach-Object { [void]$dt.Columns.Add($_) }
$dados | ForEach-Object {
    $row = $dt.NewRow()
    $_.PSObject.Properties | ForEach-Object { $row[$_.Name] = $_.Value }
    $dt.Rows.Add($row)
}

$bulkCopy.WriteToServer($dt)
$conn.Close()

Write-Host "Importacao concluida!" -ForegroundColor Green
```

---

## 6. Checklist: Antes de Migrar para o RDS

```
[ ] Scripts que usam xp_cmdshell → migrar para PowerShell externo
[ ] Scripts BACKUP TO DISK → migrar para rds_backup_database → S3
[ ] BULK INSERT de arquivo local → migrar para PowerShell + SqlBulkCopy
[ ] Jobs com passos CmdExec → reescrever como TSQL ou Lambda/PowerShell externo
[ ] Logins Windows → avaliar AWS Directory Service
[ ] Linked Servers → validar conectividade via VPC Security Group
[ ] DBCC DROPCLEANBUFFERS → remover ou substituir por FREEPROCCACHE
[ ] Scripts que assumem caminho de arquivo (MDF/LDF) → remover dependências de path
[ ] Trace Flags → verificar equivalente em Parameter Group
```
