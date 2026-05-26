# Conexão e Configuração do AWS RDS para SQL Server

> Dois níveis de operação coexistem: o **nível RDS** (infraestrutura AWS, sem console) e o **nível SQL Server** (dentro do banco). Este documento cobre os dois, com scripts separados para cada um.

---

## Os dois níveis

```
☁️ Nível RDS (AWS CLI)       → descrever instância, alterar parâmetros,
                                reiniciar, verificar status, events log

🗄️ Nível SQL Server (T-SQL) → conectar, configurar sp_configure,
                                monitorar sessions, verificar saúde interna
```

---

## 1. Descobrir o Endpoint da Instância

### ☁️ Nível RDS — AWS CLI

```powershell
# Listar todas as instâncias RDS da conta
aws rds describe-db-instances `
    --query "DBInstances[*].{ID:DBInstanceIdentifier, Engine:Engine, Status:DBInstanceStatus, Endpoint:Endpoint.Address, Port:Endpoint.Port}" `
    --output table

# Endpoint de uma instância específica
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].Endpoint" `
    --output table

# Resultado esperado:
# Address: minha-instancia.xxxx.sa-east-1.rds.amazonaws.com
# Port:    1433
```

### 🗄️ Nível SQL Server — T-SQL

```sql
-- Confirmar o servidor ao qual você está conectado
SELECT @@SERVERNAME AS ServidorInterno, @@VERSION AS Versao;

-- No RDS, @@SERVERNAME retorna um ID interno — não o endpoint DNS externo.
-- Use o endpoint da AWS CLI acima para conexões externas.
```

---

## 2. Testar Conectividade (antes de conectar)

### 🗄️ Nível SQL Server — PowerShell local (fora do banco)

```powershell
# Testar se a porta 1433 está acessível
Test-NetConnection `
    -ComputerName "minha-instancia.xxxx.sa-east-1.rds.amazonaws.com" `
    -Port 1433

# Resultado esperado: TcpTestSucceeded : True

# Resolver o DNS do endpoint
Resolve-DnsName "minha-instancia.xxxx.sa-east-1.rds.amazonaws.com"

# Testar conexão real com sqlcmd
sqlcmd `
    -S "minha-instancia.xxxx.sa-east-1.rds.amazonaws.com,1433" `
    -U "admin" `
    -P "SuaSenha!" `
    -Q "SELECT @@VERSION, GETDATE() AS Agora"
```

---

## 3. Strings de Conexão por Linguagem

### 🗄️ Nível SQL Server — Aplicações

```
# .NET / C# (SqlConnection)
Server=minha-instancia.xxxx.sa-east-1.rds.amazonaws.com,1433;
Database=MeuBanco;
User Id=app_vendas;
Password=App@Vendas2024!;
Encrypt=True;
TrustServerCertificate=True;
Connection Timeout=30;

# JDBC (Java)
jdbc:sqlserver://minha-instancia.xxxx.sa-east-1.rds.amazonaws.com:1433;
databaseName=MeuBanco;
user=app_vendas;
password=App@Vendas2024!;
encrypt=true;
trustServerCertificate=true;

# Python (pyodbc)
DRIVER={ODBC Driver 18 for SQL Server};
SERVER=minha-instancia.xxxx.sa-east-1.rds.amazonaws.com,1433;
DATABASE=MeuBanco;
UID=app_vendas;
PWD=App@Vendas2024!;
TrustServerCertificate=yes;

# PowerShell (Invoke-Sqlcmd)
Invoke-Sqlcmd `
    -ServerInstance "minha-instancia.xxxx.sa-east-1.rds.amazonaws.com,1433" `
    -Username "app_vendas" `
    -Password "App@Vendas2024!" `
    -Database "MeuBanco" `
    -Query "SELECT TOP 1 * FROM vendas.Pedido"
```

---

## 4. Informações da Instância

### ☁️ Nível RDS — AWS CLI

```powershell
# Visão geral completa da instância
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --output json

# Campos mais relevantes (filtrado)
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].{
        Status:DBInstanceStatus,
        Engine:Engine,
        EngineVersion:EngineVersion,
        Classe:DBInstanceClass,
        StorageGB:AllocatedStorage,
        MultiAZ:MultiAZ,
        Endpoint:Endpoint.Address,
        Porta:Endpoint.Port,
        Manutencao:PreferredMaintenanceWindow,
        BackupWindow:PreferredBackupWindow,
        RetencaoBackup:BackupRetentionPeriod
    }" `
    --output table

# Verificar Parameter Group aplicado
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].DBParameterGroups" `
    --output table

# Verificar Option Group (ex: SQLSERVER_BACKUP_RESTORE)
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].OptionGroupMemberships" `
    --output table
```

### 🗄️ Nível SQL Server — T-SQL

```sql
-- Versão e edição do SQL Server na instância RDS
SELECT
    SERVERPROPERTY('Edition')           AS Edicao,
    SERVERPROPERTY('ProductVersion')    AS Versao,
    SERVERPROPERTY('ProductLevel')      AS PatchLevel,
    SERVERPROPERTY('EngineEdition')     AS EngineEdition;
-- EngineEdition = 5 confirma que é RDS/Azure

-- Tempo online da instância
SELECT
    sqlserver_start_time,
    DATEDIFF(HOUR, sqlserver_start_time, GETDATE()) AS HorasOnline
FROM sys.dm_os_sys_info;

-- Bancos de dados na instância
SELECT name, database_id, state_desc, recovery_model_desc
FROM sys.databases
ORDER BY name;
```

---

## 5. Configurações do Servidor

### ☁️ Nível RDS — AWS CLI (Parameter Group)

Configurações que **não podem** ser alteradas via `sp_configure` diretamente são gerenciadas via Parameter Group:

```powershell
# Listar parâmetros do Parameter Group associado à instância
$pg = (aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].DBParameterGroups[0].DBParameterGroupName" `
    --output text)

aws rds describe-db-parameters `
    --db-parameter-group-name $pg `
    --query "Parameters[?ApplyType=='dynamic'].[ParameterName,ParameterValue,Description]" `
    --output table

# Alterar um parâmetro no Parameter Group
aws rds modify-db-parameter-group `
    --db-parameter-group-name $pg `
    --parameters "ParameterName=max degree of parallelism,ParameterValue=4,ApplyMethod=immediate"

# Parâmetros comuns do Parameter Group para SQL Server:
# max degree of parallelism    → grau de paralelismo
# cost threshold for parallelism
# optimize for ad hoc workloads
# fill factor (%)
# rds.force_ssl                → forçar SSL nas conexões
```

### 🗄️ Nível SQL Server — T-SQL (sp_configure)

```sql
-- Verificar configurações atuais
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

-- Alterar o que é permitido via sp_configure no RDS
EXEC sp_configure 'max degree of parallelism', 4;
RECONFIGURE;

EXEC sp_configure 'cost threshold for parallelism', 25;
RECONFIGURE;

EXEC sp_configure 'optimize for ad hoc workloads', 1;
RECONFIGURE;

-- Limpar cache de planos de execução
DBCC FREEPROCCACHE;
```

---

## 6. Manutenção e Reinicialização

### ☁️ Nível RDS — AWS CLI

```powershell
# Reiniciar a instância (causa ~60s de indisponibilidade)
aws rds reboot-db-instance `
    --db-instance-identifier minha-instancia-rds

# Reiniciar com failover (Multi-AZ — promove a réplica secundária)
aws rds reboot-db-instance `
    --db-instance-identifier minha-instancia-rds `
    --force-failover

# Aguardar a instância voltar ao status available
aws rds wait db-instance-available `
    --db-instance-identifier minha-instancia-rds

Write-Host "Instancia disponivel novamente."

# Ver janela de manutenção configurada
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].PreferredMaintenanceWindow" `
    --output text

# Alterar janela de manutenção
aws rds modify-db-instance `
    --db-instance-identifier minha-instancia-rds `
    --preferred-maintenance-window "sun:03:00-sun:04:00" `
    --apply-immediately
```

### 🗄️ Nível SQL Server — T-SQL

```sql
-- Verificar se há uma manutenção em progresso (sessões do sistema)
SELECT session_id, status, login_name, program_name, host_name
FROM sys.dm_exec_sessions
WHERE is_user_process = 0   -- sessões internas do SQL Server
ORDER BY session_id;

-- Verificar conexões ativas antes de uma manutenção
SELECT
    s.session_id,
    s.login_name,
    s.status,
    s.host_name,
    s.program_name,
    s.login_time
FROM sys.dm_exec_sessions s
WHERE s.is_user_process = 1
ORDER BY s.login_time;
```

---

## 7. Monitoramento de Saúde

### ☁️ Nível RDS — AWS CLI

```powershell
# Eventos recentes da instância (erros, manutenções, failovers)
aws rds describe-events `
    --source-identifier minha-instancia-rds `
    --source-type db-instance `
    --duration 1440 `
    --output table
# --duration em minutos (1440 = últimas 24 horas)

# Logs disponíveis na instância
aws rds describe-db-log-files `
    --db-instance-identifier minha-instancia-rds `
    --output table

# Baixar um arquivo de log
aws rds download-db-log-file-portion `
    --db-instance-identifier minha-instancia-rds `
    --log-file-name "error/mssql-error-2024-06-15.log" `
    --output text > "C:\Temp\rds-error.log"
```

### 🗄️ Nível SQL Server — T-SQL

```sql
-- Uso de memória
SELECT
    physical_memory_in_use_kb / 1024    AS MemoriaUsadaMB,
    memory_utilization_percentage
FROM sys.dm_os_process_memory;

-- I/O por banco de dados
SELECT
    DB_NAME(vfs.database_id)        AS Banco,
    SUM(vfs.io_stall_read_ms)       AS ReadStallMs,
    SUM(vfs.io_stall_write_ms)      AS WriteStallMs,
    SUM(vfs.num_of_reads)           AS TotalLeituras,
    SUM(vfs.num_of_writes)          AS TotalEscritas
FROM sys.dm_io_virtual_file_stats(NULL, NULL) vfs
GROUP BY vfs.database_id
ORDER BY ReadStallMs + WriteStallMs DESC;

-- Conexões por banco
SELECT DB_NAME(dbid) AS Banco, COUNT(*) AS Conexoes
FROM sys.sysprocesses
WHERE dbid > 0
GROUP BY dbid
ORDER BY Conexoes DESC;

-- Verificar erros no log do SQL Server (últimas 500 linhas)
EXEC sp_readerrorlog 0, 1, NULL, NULL;
```

---

## 8. Multi-AZ e Failover

### ☁️ Nível RDS — AWS CLI

```powershell
# Verificar se Multi-AZ está habilitado
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].MultiAZ" `
    --output text

# Habilitar Multi-AZ
aws rds modify-db-instance `
    --db-instance-identifier minha-instancia-rds `
    --multi-az `
    --apply-immediately

# Forçar failover manual (testa a réplica)
aws rds reboot-db-instance `
    --db-instance-identifier minha-instancia-rds `
    --force-failover
```

### 🗄️ Nível SQL Server — T-SQL (pós-failover)

```sql
-- Verificar identidade interna após um failover
SELECT @@SERVERNAME AS ServidorAtual, GETDATE() AS Horario;

-- O endpoint DNS externo permanece o mesmo após failover.
-- O @@SERVERNAME interno pode mudar — não use para identificar o servidor.

-- Verificar se há tabelas temporárias globais (são perdidas no failover)
SELECT name FROM tempdb.sys.objects WHERE name LIKE '##%';
```

> **Impacto do failover Multi-AZ:**
> - Tempo típico: 60–120 segundos
> - O endpoint DNS permanece o mesmo (aplicações reconectam automaticamente)
> - Tabelas temporárias globais (`##`) são perdidas
> - Transações em aberto são revertidas
