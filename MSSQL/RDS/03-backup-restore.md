# Backup e Restore no AWS RDS para SQL Server

> No RDS existem **dois sistemas de backup independentes** que coexistem: o backup gerenciado pela AWS (snapshots automáticos e manuais, configurados via CLI) e o backup nativo do SQL Server via `rds_backup_database` direcionado ao S3. Entender os dois é essencial para montar uma estratégia completa de recuperação.

---

## Os dois níveis

```
☁️ Nível RDS (AWS CLI)         → snapshots gerenciados pela AWS,
                                   janela de backup automático,
                                   retenção, restore de snapshot,
                                   point-in-time recovery via snapshot

🗄️ Nível SQL Server (T-SQL)   → rds_backup_database (FULL/DIFF/LOG),
                                   rds_restore_database, rds_restore_log,
                                   rds_finish_restore, rds_fn_task_status
```

> **Regra prática:**
> - Use o **Nível RDS** para garantia de sobrevivência da instância (DR, point-in-time do RDS, criar nova instância).
> - Use o **Nível SQL Server** para exportar e importar bancos individuais entre instâncias ou entre ambientes (homologação → produção, migração).

---

## 1. Comparação entre os dois sistemas

| Operação | ☁️ Nível RDS (CLI) | 🗄️ Nível SQL Server (T-SQL) |
|---|---|---|
| Backup automático | Sim (snapshot diário) | Não automático (exige Agent ou script) |
| Backup manual | `aws rds create-db-snapshot` | `rds_backup_database @type='FULL'` |
| Destino | Gerenciado pela AWS (transparente) | ARN do S3 (bucket explícito) |
| Granularidade | Instância inteira | Banco individual |
| Point-in-time | Sim, via `restore-db-instance-to-point-in-time` | Não (exige FULL + LOG manual) |
| Restore cria nova instância | Sim (sempre nova instância RDS) | Não (cria novo banco na mesma instância) |
| Monitoramento | `aws rds describe-db-snapshots` | `rds_fn_task_status` |
| Retenção configurável | Sim, 1–35 dias | Controlado pelo usuário no S3 |

---

## 2. Backup Automático e Retenção

### ☁️ Nível RDS — AWS CLI

O RDS realiza snapshots automáticos diários dentro da janela configurada e mantém os backups pelo período de retenção definido.

```powershell
# Verificar configuração atual de backup automático
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].{
        BackupWindow:PreferredBackupWindow,
        RetencaoDias:BackupRetentionPeriod,
        LatestBackup:LatestRestorableTime
    }" `
    --output table

# Configurar janela de backup (horário UTC) e retenção (dias)
aws rds modify-db-instance `
    --db-instance-identifier minha-instancia-rds `
    --preferred-backup-window "03:00-04:00" `
    --backup-retention-period 7 `
    --apply-immediately

# Desativar backups automáticos (retenção = 0, não recomendado)
aws rds modify-db-instance `
    --db-instance-identifier minha-instancia-rds `
    --backup-retention-period 0 `
    --apply-immediately
```

### 🗄️ Nível SQL Server — T-SQL

```sql
-- Verificar o recovery model dos bancos (impacta backup de log)
SELECT name, recovery_model_desc, log_reuse_wait_desc
FROM sys.databases
ORDER BY name;

-- O backup automático do RDS sempre usa FULL recovery model para os bancos de sistema.
-- Para bancos de usuário, o recovery model é configurado explicitamente:
ALTER DATABASE MeuBanco SET RECOVERY FULL;
```

---

## 3. Snapshots Manuais

### ☁️ Nível RDS — AWS CLI

Snapshot captura **a instância inteira**. É independente dos backups via `rds_backup_database`.

```powershell
# Criar snapshot manual
aws rds create-db-snapshot `
    --db-instance-identifier minha-instancia-rds `
    --db-snapshot-identifier snapshot-meubanco-20240615

# Aguardar o snapshot ficar disponível
aws rds wait db-snapshot-available `
    --db-snapshot-identifier snapshot-meubanco-20240615

Write-Host "Snapshot disponivel."

# Listar todos os snapshots da instância
aws rds describe-db-snapshots `
    --db-instance-identifier minha-instancia-rds `
    --query "DBSnapshots[*].{
        ID:DBSnapshotIdentifier,
        Tipo:SnapshotType,
        Status:Status,
        Criado:SnapshotCreateTime,
        TamanhoGB:AllocatedStorage
    }" `
    --output table

# Ver apenas snapshots automáticos
aws rds describe-db-snapshots `
    --db-instance-identifier minha-instancia-rds `
    --snapshot-type automated `
    --output table

# Deletar snapshot manual
aws rds delete-db-snapshot `
    --db-snapshot-identifier snapshot-meubanco-20240615
```

---

## 4. Restore a partir de Snapshot

### ☁️ Nível RDS — AWS CLI

O restore de snapshot **cria uma nova instância RDS** — nunca sobrescreve a instância de origem.

```powershell
# Restore de snapshot (cria nova instância RDS)
aws rds restore-db-instance-from-db-snapshot `
    --db-instance-identifier minha-instancia-restaurada `
    --db-snapshot-identifier snapshot-meubanco-20240615 `
    --db-instance-class db.t3.medium

# Aguardar a nova instância ficar disponível
aws rds wait db-instance-available `
    --db-instance-identifier minha-instancia-restaurada

# Verificar endpoint da instância restaurada
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-restaurada `
    --query "DBInstances[0].Endpoint" `
    --output table

# Point-in-time restore (qualquer ponto dentro da janela de retenção)
aws rds restore-db-instance-to-point-in-time `
    --source-db-instance-identifier minha-instancia-rds `
    --target-db-instance-identifier minha-instancia-pit `
    --restore-time "2024-06-15T14:30:00Z" `
    --db-instance-class db.t3.medium

# Ver qual é o ponto mais recente disponível para restore
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].LatestRestorableTime" `
    --output text
```

---

## 5. Backup de Banco Individual via T-SQL

Este mecanismo **não usa snapshots**. O arquivo `.bak` vai diretamente para o S3 configurado na Option Group.

> **Pré-requisito:** A Option Group do RDS deve ter `SQLSERVER_BACKUP_RESTORE` habilitado com uma IAM Role com acesso ao bucket S3.

### 🗄️ Nível SQL Server — T-SQL

```sql
-- Verificar se o recurso está disponível
SELECT OBJECT_ID('msdb.dbo.rds_backup_database') AS ExisteSP;
-- NULL = opção SQLSERVER_BACKUP_RESTORE não habilitada na Option Group

-- Verificar tasks recentes (backup/restore)
SELECT
    task_id,
    task_type,
    database_name,
    [% complete]    AS Percentual,
    lifecycle       AS Status,
    task_info,
    last_updated,
    created_at,
    S3_object_arn
FROM msdb.dbo.rds_fn_task_status(NULL, 0)
ORDER BY created_at DESC;
```

### ☁️ Nível RDS — AWS CLI (verificar Option Group)

```powershell
# Verificar se SQLSERVER_BACKUP_RESTORE está na Option Group da instância
$og = (aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].OptionGroupMemberships[0].OptionGroupName" `
    --output text)

aws rds describe-option-groups `
    --option-group-name $og `
    --query "OptionGroupsList[0].Options[?OptionName=='SQLSERVER_BACKUP_RESTORE']" `
    --output table
```

---

## 6. BACKUP FULL

### 🖥️ MSSQL Padrão — T-SQL

```sql
-- Backup para disco local
BACKUP DATABASE MeuBanco
TO DISK = 'C:\Backups\MeuBanco_FULL.bak'
WITH COMPRESSION, STATS = 10, CHECKSUM;

-- Backup dividido em múltiplos arquivos
BACKUP DATABASE MeuBanco
TO DISK = 'C:\Backups\MeuBanco_1.bak',
   DISK = 'C:\Backups\MeuBanco_2.bak'
WITH COMPRESSION, STATS = 5;
```

### 🗄️ Nível SQL Server (RDS) — T-SQL

```sql
-- Backup FULL para S3
EXEC msdb.dbo.rds_backup_database
    @source_db_name       = 'MeuBanco',
    @s3_arn_to_backup_to  = 'arn:aws:s3:::meu-bucket/backups/MeuBanco_FULL.bak',
    @type                 = 'FULL',
    @overwrite_s3_backup_file = 1,   -- 1 = sobrescreve se já existir
    @backup_compression   = 1;        -- 1 = compressão habilitada

-- Backup com nome dinâmico por data e hora
DECLARE @s3Path NVARCHAR(500);
SET @s3Path = 'arn:aws:s3:::meu-bucket/backups/MeuBanco_FULL_'
              + FORMAT(GETDATE(), 'yyyyMMdd_HHmmss') + '.bak';

EXEC msdb.dbo.rds_backup_database
    @source_db_name       = 'MeuBanco',
    @s3_arn_to_backup_to  = @s3Path,
    @type                 = 'FULL',
    @overwrite_s3_backup_file = 1,
    @backup_compression   = 1;
```

---

## 7. BACKUP DIFFERENTIAL

### 🖥️ MSSQL Padrão — T-SQL

```sql
BACKUP DATABASE MeuBanco
TO DISK = 'C:\Backups\MeuBanco_DIFF.bak'
WITH DIFFERENTIAL, COMPRESSION, STATS = 10;
```

### 🗄️ Nível SQL Server (RDS) — T-SQL

```sql
EXEC msdb.dbo.rds_backup_database
    @source_db_name       = 'MeuBanco',
    @s3_arn_to_backup_to  = 'arn:aws:s3:::meu-bucket/backups/MeuBanco_DIFF.bak',
    @type                 = 'DIFFERENTIAL',
    @overwrite_s3_backup_file = 1,
    @backup_compression   = 1;
```

---

## 8. BACKUP DE LOG DE TRANSAÇÃO

### 🖥️ MSSQL Padrão — T-SQL

```sql
-- Verificar e ajustar recovery model
SELECT name, recovery_model_desc FROM sys.databases WHERE name = 'MeuBanco';
ALTER DATABASE MeuBanco SET RECOVERY FULL;

-- Backup de log
BACKUP LOG MeuBanco
TO DISK = 'C:\Backups\MeuBanco_LOG.bak'
WITH COMPRESSION, STATS = 10;
```

### 🗄️ Nível SQL Server (RDS) — T-SQL

```sql
-- Verificar e ajustar recovery model
ALTER DATABASE MeuBanco SET RECOVERY FULL;

-- Backup de log para S3
EXEC msdb.dbo.rds_backup_database
    @source_db_name       = 'MeuBanco',
    @s3_arn_to_backup_to  = 'arn:aws:s3:::meu-bucket/logs/MeuBanco_LOG.bak',
    @type                 = 'LOG',
    @overwrite_s3_backup_file = 1,
    @backup_compression   = 1;
```

---

## 9. RESTORE de Banco Individual

### 🖥️ MSSQL Padrão — T-SQL

```sql
-- Restore FULL simples
RESTORE DATABASE MeuBancoRestore
FROM DISK = 'C:\Backups\MeuBanco_FULL.bak'
WITH MOVE 'MeuBanco'      TO 'C:\Data\MeuBancoRestore.mdf',
     MOVE 'MeuBanco_log'  TO 'C:\Log\MeuBancoRestore_log.ldf',
     REPLACE, STATS = 10;

-- Restore FULL + DIFFERENTIAL + LOG (Point-in-Time)
RESTORE DATABASE MeuBancoRestore
FROM DISK = 'C:\Backups\MeuBanco_FULL.bak'
WITH NORECOVERY, REPLACE, STATS = 10;

RESTORE DATABASE MeuBancoRestore
FROM DISK = 'C:\Backups\MeuBanco_DIFF.bak'
WITH NORECOVERY;

RESTORE LOG MeuBancoRestore
FROM DISK = 'C:\Backups\MeuBanco_LOG.bak'
WITH RECOVERY, STOPAT = '2024-06-15 14:30:00';
```

### 🗄️ Nível SQL Server (RDS) — T-SQL

```sql
-- Restore FULL simples (banco é criado automaticamente)
EXEC msdb.dbo.rds_restore_database
    @restore_db_name        = 'MeuBancoRestore',
    @s3_arn_to_restore_from = 'arn:aws:s3:::meu-bucket/backups/MeuBanco_FULL.bak';

-- Acompanhar o progresso
EXEC msdb.dbo.rds_task_status @db_name = 'MeuBancoRestore';

-- Restore FULL com NORECOVERY (para aplicar LOG depois)
EXEC msdb.dbo.rds_restore_database
    @restore_db_name        = 'MeuBancoRestore',
    @s3_arn_to_restore_from = 'arn:aws:s3:::meu-bucket/backups/MeuBanco_FULL.bak',
    @with_norecovery        = 1;

-- Aguardar o passo anterior antes de continuar (checar lifecycle = SUCCESS)
SELECT TOP 1 lifecycle, [% complete]
FROM msdb.dbo.rds_fn_task_status('MeuBancoRestore', 0)
ORDER BY created_at DESC;

-- Aplicar LOG com NORECOVERY
EXEC msdb.dbo.rds_restore_log
    @restore_db_name        = 'MeuBancoRestore',
    @s3_arn_to_restore_from = 'arn:aws:s3:::meu-bucket/logs/MeuBanco_LOG.bak',
    @with_norecovery        = 1;

-- Finalizar o restore e colocar o banco online
EXEC msdb.dbo.rds_finish_restore @db_name = 'MeuBancoRestore';
```

---

## 10. Monitorar e Cancelar Tasks de Backup/Restore

### 🗄️ Nível SQL Server — T-SQL

```sql
-- Todas as tasks com detalhes completos
SELECT
    task_id,
    task_type,
    database_name,
    [% complete]    AS Percentual,
    lifecycle       AS Status,   -- SUCCESS, IN_PROGRESS, ERROR, CANCEL_REQUESTED
    task_info,                   -- Mensagem de erro se houver
    last_updated,
    created_at,
    S3_object_arn
FROM msdb.dbo.rds_fn_task_status(NULL, 0)
ORDER BY created_at DESC;

-- Filtrar por banco específico
EXEC msdb.dbo.rds_task_status @db_name = 'MeuBanco';

-- Cancelar task em andamento
EXEC msdb.dbo.rds_cancel_task @task_id = 123;
-- Substitua 123 pelo task_id obtido na consulta acima
```

### ☁️ Nível RDS — AWS CLI

```powershell
# Ver eventos recentes (inclui início/fim de backups automáticos)
aws rds describe-events `
    --source-identifier minha-instancia-rds `
    --source-type db-instance `
    --duration 1440 `
    --output table
# --duration em minutos (1440 = últimas 24 horas)

# Ver status atual da instância (confirma se há manutenção ou backup em curso)
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].{Status:DBInstanceStatus, BackupWindow:PreferredBackupWindow}" `
    --output table
```

---

## 11. Automação via PowerShell — Backup com Monitoramento

### 🗄️ Nível SQL Server — PowerShell (invoca T-SQL)

```powershell
# ============================================================
# Backup RDS → S3 com monitoramento automático de progresso
# ============================================================
param(
    [string]$Endpoint = "meu-rds.xxxx.sa-east-1.rds.amazonaws.com,1433",
    [string]$Usuario  = "admin",
    [string]$Senha    = "SuaSenha123!",
    [string]$Banco    = "MeuBanco",
    [string]$S3Bucket = "meu-bucket",
    [string]$S3Pasta  = "backups"
)

$timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$dia       = (Get-Date).DayOfWeek
$tipo      = if ($dia -eq 'Sunday') { 'FULL' } else { 'DIFFERENTIAL' }
$s3ARN     = "arn:aws:s3:::$S3Bucket/$S3Pasta/${Banco}_${tipo}_${timestamp}.bak"

Write-Host "========================================" -ForegroundColor Cyan
Write-Host " Banco  : $Banco"
Write-Host " Tipo   : $tipo"
Write-Host " Destino: $s3ARN"
Write-Host "========================================" -ForegroundColor Cyan

# Iniciar backup
$query = @"
EXEC msdb.dbo.rds_backup_database
    @source_db_name       = '$Banco',
    @s3_arn_to_backup_to  = '$s3ARN',
    @type                 = '$tipo',
    @overwrite_s3_backup_file = 1,
    @backup_compression   = 1;
"@

try {
    Invoke-Sqlcmd -ServerInstance $Endpoint -Username $Usuario -Password $Senha `
                  -Query $query -QueryTimeout 60
    Write-Host "[$(Get-Date -f HH:mm:ss)] Backup iniciado com sucesso." -ForegroundColor Green
} catch {
    Write-Error "Falha ao iniciar backup: $_"; exit 1
}

# Monitorar progresso a cada 30 segundos (máx. 1 hora)
$maxTentativas = 120; $tentativa = 0; $status = ""

do {
    Start-Sleep -Seconds 30
    $tentativa++

    $r = Invoke-Sqlcmd `
        -ServerInstance $Endpoint -Username $Usuario -Password $Senha `
        -Query "SELECT TOP 1 lifecycle, [% complete], task_info
                FROM msdb.dbo.rds_fn_task_status('$Banco', 0)
                WHERE task_type = 'BACKUP'
                ORDER BY created_at DESC"

    $status = $r.lifecycle
    Write-Host "[$(Get-Date -f HH:mm:ss)] $status | $($r.'% complete')%"

} while ($status -notin @('SUCCESS', 'ERROR') -and $tentativa -lt $maxTentativas)

if ($status -eq 'SUCCESS') {
    Write-Host "[$(Get-Date -f HH:mm:ss)] Backup concluido com sucesso!" -ForegroundColor Green
} else {
    Write-Error "Backup falhou. Info: $($r.task_info)"; exit 1
}
```

---

## 12. Automação via PowerShell — Restore com Monitoramento

### 🗄️ Nível SQL Server — PowerShell (invoca T-SQL)

```powershell
# ============================================================
# Restore de banco RDS a partir de arquivo .bak no S3
# ============================================================
param(
    [string]$Endpoint     = "meu-rds.xxxx.sa-east-1.rds.amazonaws.com,1433",
    [string]$Usuario      = "admin",
    [string]$Senha        = "SuaSenha123!",
    [string]$BancoDestino = "MeuBancoRestaurado",
    [string]$S3ARN        = "arn:aws:s3:::meu-bucket/backups/MeuBanco_FULL_20240101.bak"
)

Write-Host "Iniciando restore de $BancoDestino a partir de $S3ARN" -ForegroundColor Cyan

Invoke-Sqlcmd -ServerInstance $Endpoint -Username $Usuario -Password $Senha `
    -Query "EXEC msdb.dbo.rds_restore_database
                @restore_db_name        = '$BancoDestino',
                @s3_arn_to_restore_from = '$S3ARN';" `
    -QueryTimeout 60

# Monitorar (máx. 2 horas)
$maxTentativas = 240; $tentativa = 0; $status = ""

do {
    Start-Sleep -Seconds 30; $tentativa++
    $r = Invoke-Sqlcmd -ServerInstance $Endpoint -Username $Usuario -Password $Senha `
         -Query "SELECT TOP 1 lifecycle, [% complete]
                 FROM msdb.dbo.rds_fn_task_status('$BancoDestino', 0)
                 WHERE task_type = 'RESTORE'
                 ORDER BY created_at DESC"
    $status = $r.lifecycle
    Write-Host "[$(Get-Date -f HH:mm:ss)] $status | $($r.'% complete')%"
} while ($status -notin @('SUCCESS', 'ERROR') -and $tentativa -lt $maxTentativas)

if ($status -eq 'SUCCESS') {
    Write-Host "Restore concluido com sucesso!" -ForegroundColor Green
} else {
    Write-Error "Restore falhou."; exit 1
}
```

---

## 13. Estratégia de Backup Recomendada

| Cenário | Solução |
|---|---|
| Recuperação de desastre (instância RDS inteira) | ☁️ Snapshot automático diário + retenção de 7 dias |
| Antes de uma alteração de risco | ☁️ `aws rds create-db-snapshot` (snapshot manual) |
| Exportar banco para outra instância RDS | 🗄️ `rds_backup_database FULL` → S3 → `rds_restore_database` |
| Migração homologação → produção | 🗄️ `rds_backup_database FULL` no banco de origem → restore no destino |
| Backup frequente com point-in-time granular | 🗄️ `FULL` semanal + `DIFFERENTIAL` diário + `LOG` a cada hora |
| Conferir se os backups existem | ☁️ `aws rds describe-db-snapshots` + 🗄️ `rds_fn_task_status` |
