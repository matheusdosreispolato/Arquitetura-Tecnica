# T-SQL Avançado — CTEs, Window Functions, Performance

> Scripts avançados válidos em **ambos os ambientes**. Diferenças do RDS marcadas com ☁️.

---

## 1. CTEs (Common Table Expressions)

```sql
-- CTE simples
WITH ClientesAtivos AS (
    SELECT ClienteID, Nome, Estado
    FROM vendas.Clientes WHERE Ativo = 1
)
SELECT * FROM ClientesAtivos WHERE Estado = 'SP';

-- CTE recursiva (hierarquia / organograma)
WITH Hierarquia AS (
    SELECT FuncionarioID, Nome, GerenteID, 0 AS Nivel
    FROM rh.Funcionarios WHERE GerenteID IS NULL   -- âncora

    UNION ALL

    SELECT f.FuncionarioID, f.Nome, f.GerenteID, h.Nivel + 1
    FROM rh.Funcionarios f
    INNER JOIN Hierarquia h ON h.FuncionarioID = f.GerenteID
)
SELECT
    FuncionarioID,
    REPLICATE('  ', Nivel) + Nome AS Arvore,
    Nivel
FROM Hierarquia
ORDER BY Nivel, Nome;

-- Múltiplas CTEs encadeadas
WITH
Pedidos_2024 AS (
    SELECT ClienteID, SUM(Valor) AS Total2024
    FROM vendas.Pedidos WHERE YEAR(DataPedido) = 2024
    GROUP BY ClienteID
),
Pedidos_2025 AS (
    SELECT ClienteID, SUM(Valor) AS Total2025
    FROM vendas.Pedidos WHERE YEAR(DataPedido) = 2025
    GROUP BY ClienteID
)
SELECT
    c.Nome,
    ISNULL(p24.Total2024, 0)                                   AS Total2024,
    ISNULL(p25.Total2025, 0)                                   AS Total2025,
    ISNULL(p25.Total2025, 0) - ISNULL(p24.Total2024, 0)       AS Variacao
FROM vendas.Clientes c
LEFT JOIN Pedidos_2024 p24 ON p24.ClienteID = c.ClienteID
LEFT JOIN Pedidos_2025 p25 ON p25.ClienteID = c.ClienteID;
```

---

## 2. Window Functions

```sql
-- ROW_NUMBER por partição
SELECT ClienteID, PedidoID, Valor,
    ROW_NUMBER() OVER (PARTITION BY ClienteID ORDER BY DataPedido DESC) AS Ranking
FROM vendas.Pedidos;

-- Último pedido de cada cliente
SELECT * FROM (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY ClienteID ORDER BY DataPedido DESC) AS rn
    FROM vendas.Pedidos
) x WHERE rn = 1;

-- RANK, DENSE_RANK e NTILE
SELECT
    ClienteID,
    SUM(Valor)                                       AS TotalGasto,
    RANK()       OVER (ORDER BY SUM(Valor) DESC)     AS Rank,
    DENSE_RANK() OVER (ORDER BY SUM(Valor) DESC)     AS DenseRank,
    NTILE(4)     OVER (ORDER BY SUM(Valor) DESC)     AS Quartil
FROM vendas.Pedidos
GROUP BY ClienteID;

-- LAG / LEAD (comparar com linha anterior/próxima)
SELECT
    DataPedido,
    Valor,
    LAG(Valor)  OVER (ORDER BY DataPedido) AS ValorAnterior,
    LEAD(Valor) OVER (ORDER BY DataPedido) AS ValorProximo,
    Valor - LAG(Valor) OVER (ORDER BY DataPedido) AS Variacao
FROM vendas.Pedidos WHERE ClienteID = 1;

-- Running total (acumulado)
SELECT
    DataPedido, Valor,
    SUM(Valor) OVER (
        ORDER BY DataPedido
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS Acumulado
FROM vendas.Pedidos WHERE ClienteID = 1;

-- Média móvel dos últimos 3
SELECT
    DataPedido, Valor,
    AVG(Valor) OVER (
        ORDER BY DataPedido
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS MediaMovel3
FROM vendas.Pedidos WHERE ClienteID = 1;
```

---

## 3. PIVOT e UNPIVOT

```sql
-- PIVOT estático
SELECT *
FROM (
    SELECT YEAR(DataPedido) AS Ano, MONTH(DataPedido) AS Mes, Valor
    FROM vendas.Pedidos
) src
PIVOT (
    SUM(Valor) FOR Mes IN ([1],[2],[3],[4],[5],[6],[7],[8],[9],[10],[11],[12])
) pvt
ORDER BY Ano;

-- PIVOT dinâmico (colunas geradas em runtime)
DECLARE @cols  NVARCHAR(MAX);
DECLARE @query NVARCHAR(MAX);

SELECT @cols = STRING_AGG(QUOTENAME(Estado), ',')
FROM (SELECT DISTINCT Estado FROM vendas.Clientes) x;

SET @query = N'
SELECT Ano, ' + @cols + '
FROM (
    SELECT YEAR(p.DataPedido) AS Ano, c.Estado, p.Valor
    FROM vendas.Pedidos p
    INNER JOIN vendas.Clientes c ON c.ClienteID = p.ClienteID
) src
PIVOT (SUM(Valor) FOR Estado IN (' + @cols + ')) pvt
ORDER BY Ano;';

EXEC sp_executesql @query;
```

---

## 4. Monitoramento de Performance

### 🖥️ MSSQL Padrão e ☁️ AWS RDS (ambos suportam as views abaixo)

```sql
-- Queries mais custosas no cache de plano
SELECT TOP 20
    qs.total_worker_time / qs.execution_count    AS AvgCPU_us,
    qs.total_logical_reads / qs.execution_count  AS AvgReads,
    qs.execution_count,
    qs.total_elapsed_time / qs.execution_count   AS AvgElapsed_us,
    SUBSTRING(st.text,
        (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1
    ) AS QueryText
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY AvgCPU_us DESC;

-- Conexões ativas e queries em execução
SELECT
    r.session_id,
    r.status,
    r.blocking_session_id    AS BloqueadoPor,
    r.wait_type,
    r.wait_time / 1000.0     AS WaitSeg,
    r.total_elapsed_time / 1000.0 AS TempoSeg,
    DB_NAME(r.database_id)   AS Banco,
    t.text                   AS QueryAtual,
    s.login_name,
    s.host_name
FROM sys.dm_exec_requests r
INNER JOIN sys.dm_exec_sessions s ON s.session_id = r.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id > 50
ORDER BY r.total_elapsed_time DESC;

-- Bloqueios ativos
SELECT
    blocked.session_id                         AS SessionBloqueada,
    bt.text                                    AS QueryBloqueada,
    blocker.session_id                         AS SessionBloqueia,
    bkt.text                                   AS QueryBloqueia,
    blocked.wait_type,
    blocked.wait_time / 1000.0                 AS EsperandoSeg
FROM sys.dm_exec_requests blocked
INNER JOIN sys.dm_exec_requests blocker
    ON blocker.session_id = blocked.blocking_session_id
CROSS APPLY sys.dm_exec_sql_text(blocked.sql_handle) bt
CROSS APPLY sys.dm_exec_sql_text(blocker.sql_handle) bkt
WHERE blocked.blocking_session_id > 0;

-- Missing indexes (índices recomendados pelo SQL Server)
SELECT TOP 20
    ROUND(migs.avg_total_user_cost * migs.avg_user_impact *
          (migs.user_seeks + migs.user_scans), 0)    AS ImpactoEstimado,
    mid.statement                                     AS Tabela,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    migs.user_seeks,
    migs.user_scans
FROM sys.dm_db_missing_index_groups mig
INNER JOIN sys.dm_db_missing_index_group_stats migs
    ON migs.group_handle = mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details mid
    ON mid.index_handle = mig.index_handle
WHERE mid.database_id = DB_ID()
ORDER BY ImpactoEstimado DESC;

-- Índices não usados (candidatos à remoção)
SELECT
    OBJECT_NAME(i.object_id) AS Tabela,
    i.name                   AS Indice,
    i.type_desc,
    ISNULL(ius.user_seeks, 0)   AS Seeks,
    ISNULL(ius.user_scans, 0)   AS Scans,
    ISNULL(ius.user_lookups, 0) AS Lookups,
    ISNULL(ius.user_updates, 0) AS Updates
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats ius
    ON ius.object_id = i.object_id
    AND ius.index_id = i.index_id
    AND ius.database_id = DB_ID()
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
  AND i.index_id > 1
ORDER BY ISNULL(ius.user_seeks,0) + ISNULL(ius.user_scans,0) ASC;

-- Espaço por tabela
SELECT
    s.name                            AS Schema,
    t.name                            AS Tabela,
    p.rows                            AS TotalLinhas,
    SUM(a.total_pages) * 8 / 1024    AS TotalMB,
    SUM(a.used_pages)  * 8 / 1024    AS UsadoMB
FROM sys.tables t
INNER JOIN sys.schemas     s ON s.schema_id = t.schema_id
INNER JOIN sys.indexes     i ON i.object_id = t.object_id
INNER JOIN sys.partitions  p ON p.object_id = t.object_id AND p.index_id = i.index_id
INNER JOIN sys.allocation_units a ON a.container_id = p.partition_id
WHERE t.is_ms_shipped = 0
GROUP BY s.name, t.name, p.rows
ORDER BY TotalMB DESC;
```

---

## 5. Configurações do Servidor

### 🖥️ MSSQL Padrão

```sql
-- Ver e alterar configurações
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;

EXEC sp_configure 'max degree of parallelism', 4;
RECONFIGURE;

EXEC sp_configure 'cost threshold for parallelism', 25;
RECONFIGURE;

EXEC sp_configure 'max server memory (MB)', 12288;
RECONFIGURE;

-- Limpar cache de planos (use com cuidado em produção)
DBCC FREEPROCCACHE;

-- Verificar configurações atuais
SELECT name, value, value_in_use FROM sys.configurations ORDER BY name;
```

### ☁️ AWS RDS

```sql
-- No RDS, sp_configure está disponível MAS com restrições.
-- Algumas opções não podem ser alteradas diretamente.
-- As configurações avançadas são feitas via Parameter Group no console/CLI da AWS.

-- Porém, estas funcionam no RDS via T-SQL:
EXEC sp_configure 'max degree of parallelism', 4;
RECONFIGURE;

EXEC sp_configure 'cost threshold for parallelism', 25;
RECONFIGURE;

-- Limpar cache de planos (disponível no RDS)
DBCC FREEPROCCACHE;

-- Verificar configurações
SELECT name, value_in_use, description
FROM sys.configurations
WHERE name IN (
    'max degree of parallelism',
    'cost threshold for parallelism',
    'max server memory (MB)',
    'optimize for ad hoc workloads'
)
ORDER BY name;

-- DBCC DROPCLEANBUFFERS NÃO está disponível no RDS
-- Use DBCC FREEPROCCACHE para limpar o plano cache quando necessário
```

---

## 6. Integridade do Banco

```sql
-- Verificar integridade completa
DBCC CHECKDB ('MeuBanco') WITH NO_INFOMSGS, ALL_ERRORMSGS;

-- Verificação rápida (somente física — muito mais rápido)
DBCC CHECKDB ('MeuBanco') WITH PHYSICAL_ONLY;

-- Verificar tabela específica
DBCC CHECKTABLE ('vendas.Pedidos') WITH NO_INFOMSGS;
```

### ☁️ AWS RDS — DBCC

```sql
-- No RDS, DBCC CHECKDB funciona normalmente.
-- DBCC DROPCLEANBUFFERS não está disponível.
-- Para verificação de integridade em grandes bancos, prefira PHYSICAL_ONLY
-- para minimizar impacto em produção.

DBCC CHECKDB ('MeuBancoDB') WITH PHYSICAL_ONLY, NO_INFOMSGS;
```

---

## 7. Transaction Log

### 🖥️ MSSQL Padrão

```sql
-- Ver tamanho e uso do log
SELECT
    name           AS Arquivo,
    size * 8 / 1024 AS TamanhoMB,
    FILEPROPERTY(name, 'SpaceUsed') * 8 / 1024 AS UsadoMB
FROM sys.database_files WHERE type = 1;

-- Shrink do log após backup
BACKUP LOG MeuBanco TO DISK = 'NUL';  -- descarta log (somente dev/teste!)
DBCC SHRINKFILE (MeuBanco_log, 256);   -- reduz para ~256MB
```

### ☁️ AWS RDS

```sql
-- No RDS, backup de log para NUL não é permitido.
-- Para reduzir o log, faça um backup real para S3 antes:
EXEC msdb.dbo.rds_backup_database
    @source_db_name      = 'MeuBancoDB',
    @s3_arn_to_backup_to = 'arn:aws:s3:::meu-bucket/logs/log_backup.bak',
    @type                = 'LOG';

-- Verificar tamanho do log no RDS
SELECT
    DB_NAME(database_id) AS Banco,
    name                  AS Arquivo,
    type_desc,
    size * 8 / 1024       AS TamanhoMB
FROM sys.master_files WHERE type = 1
ORDER BY size DESC;
```
