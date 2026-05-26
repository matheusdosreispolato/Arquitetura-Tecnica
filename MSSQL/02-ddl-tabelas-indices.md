# T-SQL DDL — Tabelas, Índices e Constraints

> Comandos DDL (Data Definition Language) válidos em **ambos os ambientes**. Diferenças do RDS marcadas com ☁️.

---

## 1. Banco de Dados

### 🖥️ MSSQL Padrão

```sql
CREATE DATABASE MeuBanco
COLLATE Latin1_General_CI_AI;
GO

-- Configurações recomendadas
ALTER DATABASE MeuBanco SET RECOVERY FULL;
ALTER DATABASE MeuBanco SET AUTO_CLOSE OFF;
ALTER DATABASE MeuBanco SET AUTO_SHRINK OFF;         -- NUNCA habilite em produção
ALTER DATABASE MeuBanco SET READ_COMMITTED_SNAPSHOT ON;
GO
```

### ☁️ AWS RDS

```sql
-- O banco principal é criado via console/CLI ao provisionar o RDS.
-- Para bancos adicionais, use CREATE DATABASE normalmente:
CREATE DATABASE MeuBancoDB
COLLATE Latin1_General_CI_AI;
GO

-- Configurações via ALTER DATABASE funcionam normalmente no RDS
ALTER DATABASE MeuBancoDB SET RECOVERY FULL;
ALTER DATABASE MeuBancoDB SET READ_COMMITTED_SNAPSHOT ON;
GO

-- Verificar configurações
SELECT name, recovery_model_desc, is_read_committed_snapshot_on,
       compatibility_level, collation_name
FROM sys.databases WHERE name = 'MeuBancoDB';
```

---

## 2. Schemas

```sql
-- Criar schemas para organização (funciona igual nos dois ambientes)
CREATE SCHEMA vendas;
GO
CREATE SCHEMA financeiro;
GO
CREATE SCHEMA logs;
GO

-- Listar schemas
SELECT schema_id, name FROM sys.schemas ORDER BY name;

-- Mover tabela para outro schema
ALTER SCHEMA vendas TRANSFER dbo.Pedidos;
```

---

## 3. CREATE TABLE

```sql
-- Tabela completa com tipos, constraints e colunas computadas
CREATE TABLE vendas.Clientes (
    ClienteID       INT             NOT NULL IDENTITY(1,1),
    Codigo          VARCHAR(20)     NOT NULL,
    Nome            NVARCHAR(200)   NOT NULL,
    Email           NVARCHAR(150)       NULL,
    Telefone        VARCHAR(20)         NULL,
    Estado          CHAR(2)         NOT NULL DEFAULT 'SP',
    Ativo           BIT             NOT NULL DEFAULT 1,
    DataCadastro    DATETIME2(0)    NOT NULL DEFAULT GETUTCDATE(),
    DataAlteracao   DATETIME2(0)        NULL,
    CONSTRAINT PK_Clientes          PRIMARY KEY CLUSTERED (ClienteID),
    CONSTRAINT UQ_Clientes_Codigo   UNIQUE (Codigo),
    CONSTRAINT UQ_Clientes_Email    UNIQUE (Email),
    CONSTRAINT CK_Clientes_Estado   CHECK (Estado IN ('SP','RJ','MG','RS','BA','PR','SC'))
);
GO

-- Tabela com chave estrangeira
CREATE TABLE vendas.Pedidos (
    PedidoID        INT             NOT NULL IDENTITY(1,1),
    ClienteID       INT             NOT NULL,
    NumeroPedido    VARCHAR(30)     NOT NULL,
    Status          TINYINT         NOT NULL DEFAULT 1,
    Valor           DECIMAL(18,2)   NOT NULL DEFAULT 0,
    DataPedido      DATETIME2(0)    NOT NULL DEFAULT GETUTCDATE(),
    DataFaturamento DATETIME2(0)        NULL,
    Observacoes     NVARCHAR(MAX)       NULL,
    CONSTRAINT PK_Pedidos           PRIMARY KEY CLUSTERED (PedidoID),
    CONSTRAINT UQ_Pedidos_Numero    UNIQUE (NumeroPedido),
    CONSTRAINT FK_Pedidos_Clientes  FOREIGN KEY (ClienteID)
        REFERENCES vendas.Clientes (ClienteID)
        ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT CK_Pedidos_Status    CHECK (Status BETWEEN 1 AND 3),
    CONSTRAINT CK_Pedidos_Valor     CHECK (Valor >= 0)
);
GO

-- Tabela com coluna computada (PERSISTED — armazenada fisicamente)
CREATE TABLE vendas.ItensPedido (
    ItemID          INT             NOT NULL IDENTITY(1,1),
    PedidoID        INT             NOT NULL,
    ProdutoCodigo   VARCHAR(50)     NOT NULL,
    Descricao       NVARCHAR(300)   NOT NULL,
    Quantidade      INT             NOT NULL DEFAULT 1,
    PrecoUnitario   DECIMAL(18,4)   NOT NULL,
    Desconto        DECIMAL(5,2)    NOT NULL DEFAULT 0,
    Total AS (Quantidade * PrecoUnitario * (1 - Desconto/100)) PERSISTED,
    CONSTRAINT PK_ItensPedido       PRIMARY KEY CLUSTERED (ItemID),
    CONSTRAINT FK_Itens_Pedidos     FOREIGN KEY (PedidoID)
        REFERENCES vendas.Pedidos (PedidoID) ON DELETE CASCADE
);
GO
```

---

## 4. ALTER TABLE

```sql
-- Adicionar coluna
ALTER TABLE vendas.Clientes ADD CPF VARCHAR(14) NULL;

-- Modificar tipo
ALTER TABLE vendas.Clientes ALTER COLUMN Telefone VARCHAR(30) NULL;

-- Renomear coluna (via system stored procedure)
EXEC sp_rename 'vendas.Clientes.CPF', 'DocumentoFederal', 'COLUMN';

-- Remover coluna
ALTER TABLE vendas.Clientes DROP COLUMN Telefone;

-- Adicionar DEFAULT
ALTER TABLE vendas.Pedidos
ADD CONSTRAINT DF_Pedidos_DataPedido DEFAULT GETUTCDATE() FOR DataPedido;

-- Remover constraint
ALTER TABLE vendas.Pedidos DROP CONSTRAINT DF_Pedidos_DataPedido;

-- Adicionar FK depois da criação
ALTER TABLE vendas.ItensPedido
ADD CONSTRAINT FK_Itens_Pedidos
    FOREIGN KEY (PedidoID) REFERENCES vendas.Pedidos(PedidoID);

-- Desabilitar FK temporariamente (ex: carga de dados)
ALTER TABLE vendas.ItensPedido NOCHECK CONSTRAINT FK_Itens_Pedidos;
-- ... carga ...
ALTER TABLE vendas.ItensPedido WITH CHECK CHECK CONSTRAINT FK_Itens_Pedidos;
```

---

## 5. DROP — Remover Objetos

```sql
-- Remover tabela com segurança (na ordem correta por FK)
DROP TABLE IF EXISTS vendas.ItensPedido;
DROP TABLE IF EXISTS vendas.Pedidos;
DROP TABLE IF EXISTS vendas.Clientes;

-- Remover índice
DROP INDEX IF EXISTS IX_Pedidos_ClienteData ON vendas.Pedidos;

-- Remover schema
DROP SCHEMA IF EXISTS vendas;
```

### 🖥️ MSSQL Padrão — Remover banco

```sql
-- Requer que não haja conexões ativas
ALTER DATABASE MeuBanco SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
DROP DATABASE MeuBanco;
```

### ☁️ AWS RDS — Remover banco

```sql
-- No RDS, a mesma abordagem funciona, mas o banco master
-- (criado no provisionamento) não pode ser dropado via T-SQL.
-- Bancos adicionais criados pelo usuário podem ser removidos normalmente.
ALTER DATABASE MeuBancoDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
DROP DATABASE MeuBancoDB;
```

---

## 6. Índices

```sql
-- Índice não-clusterizado simples
CREATE NONCLUSTERED INDEX IX_Clientes_Nome
ON vendas.Clientes (Nome ASC);

-- Índice composto
CREATE NONCLUSTERED INDEX IX_Pedidos_ClienteData
ON vendas.Pedidos (ClienteID, DataPedido DESC);

-- Índice com colunas incluídas (evita key lookup)
CREATE NONCLUSTERED INDEX IX_Pedidos_Status_Inc
ON vendas.Pedidos (Status, DataPedido DESC)
INCLUDE (ClienteID, NumeroPedido, Valor);

-- Índice filtrado (partial index)
CREATE NONCLUSTERED INDEX IX_Pedidos_Abertos
ON vendas.Pedidos (DataPedido DESC)
WHERE Status = 1;

-- Índice único com filtro de NULL
CREATE UNIQUE NONCLUSTERED INDEX UX_Clientes_Email
ON vendas.Clientes (Email)
WHERE Email IS NOT NULL;
```

### 🖥️ MSSQL Padrão — Manutenção de índices

```sql
-- Rebuild com ONLINE (Enterprise Edition)
ALTER INDEX ALL ON vendas.Pedidos REBUILD WITH (ONLINE = ON);

-- Reorganize (operação leve, sempre online)
ALTER INDEX IX_Pedidos_ClienteData ON vendas.Pedidos REORGANIZE;
```

### ☁️ AWS RDS — Manutenção de índices

```sql
-- No RDS Standard/Developer Edition, ONLINE = ON não está disponível.
-- Use REBUILD sem ONLINE (gera bloqueio temporário) em janela de manutenção:
ALTER INDEX ALL ON vendas.Pedidos REBUILD;

-- Ou REORGANIZE (sem bloqueio, mas menos efetivo)
ALTER INDEX ALL ON vendas.Pedidos REORGANIZE;

-- Verificar fragmentação antes de decidir
SELECT
    OBJECT_NAME(ips.object_id)           AS Tabela,
    i.name                                AS Indice,
    ips.avg_fragmentation_in_percent      AS Fragmentacao,
    ips.page_count                        AS Paginas
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
INNER JOIN sys.indexes i
    ON i.object_id = ips.object_id AND i.index_id = ips.index_id
WHERE ips.avg_fragmentation_in_percent > 10
  AND ips.page_count > 100
ORDER BY ips.avg_fragmentation_in_percent DESC;
-- Regra: < 30% → REORGANIZE | > 30% → REBUILD
```

---

## 7. Views

```sql
-- View simples (funciona igual nos dois ambientes)
CREATE OR ALTER VIEW vendas.vw_PedidosCompletos AS
SELECT
    p.PedidoID,
    p.NumeroPedido,
    c.Nome        AS Cliente,
    c.Estado,
    CASE p.Status
        WHEN 1 THEN 'Aberto'
        WHEN 2 THEN 'Faturado'
        WHEN 3 THEN 'Cancelado'
    END           AS StatusDescricao,
    p.Valor,
    p.DataPedido
FROM vendas.Pedidos p
INNER JOIN vendas.Clientes c ON c.ClienteID = p.ClienteID;
GO

-- View indexada / materializada (funciona nos dois ambientes)
CREATE OR ALTER VIEW vendas.vw_TotaisCliente
WITH SCHEMABINDING AS
SELECT
    p.ClienteID,
    COUNT_BIG(*) AS TotalPedidos,
    SUM(p.Valor) AS ValorTotal
FROM vendas.Pedidos p
GROUP BY p.ClienteID;
GO

CREATE UNIQUE CLUSTERED INDEX IX_vw_TotaisCliente
ON vendas.vw_TotaisCliente (ClienteID);
GO
```

---

## 8. Consultas de Metadados

```sql
-- Todas as tabelas do banco
SELECT TABLE_SCHEMA, TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE = 'BASE TABLE'
ORDER BY TABLE_SCHEMA, TABLE_NAME;

-- Colunas de uma tabela
SELECT COLUMN_NAME, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH,
       IS_NULLABLE, COLUMN_DEFAULT
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'vendas' AND TABLE_NAME = 'Clientes'
ORDER BY ORDINAL_POSITION;

-- Todas as constraints
SELECT tc.CONSTRAINT_NAME, tc.CONSTRAINT_TYPE, kcu.COLUMN_NAME
FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS tc
JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE kcu
    ON kcu.CONSTRAINT_NAME = tc.CONSTRAINT_NAME
WHERE tc.TABLE_SCHEMA = 'vendas' AND tc.TABLE_NAME = 'Pedidos';

-- Índices de uma tabela
SELECT
    i.name       AS Indice,
    i.type_desc,
    i.is_unique,
    STRING_AGG(c.name, ', ') WITHIN GROUP (ORDER BY ic.key_ordinal) AS Colunas
FROM sys.indexes i
JOIN sys.index_columns ic
    ON ic.object_id = i.object_id AND ic.index_id = i.index_id
JOIN sys.columns c
    ON c.object_id = ic.object_id AND c.column_id = ic.column_id
WHERE i.object_id = OBJECT_ID('vendas.Pedidos')
  AND ic.is_included_column = 0
GROUP BY i.name, i.type_desc, i.is_unique;
```
