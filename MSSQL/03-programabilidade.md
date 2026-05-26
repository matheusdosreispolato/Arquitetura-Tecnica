# T-SQL Programabilidade — Procedures, Functions, Triggers

> Todos os objetos abaixo funcionam em **ambos os ambientes**. Diferenças do RDS marcadas com ☁️.

---

## Resumo das diferenças no RDS

| Recurso | 🖥️ MSSQL Padrão | ☁️ AWS RDS |
|---|---|---|
| `xp_cmdshell` dentro de Procedure | Disponível (se habilitado) | **Bloqueado** — use PowerShell externo |
| CLR Procedures / Functions | Disponível | **Bloqueado** |
| Triggers CLR | Disponível | **Bloqueado** |
| Tabelas temporárias globais `##` | Disponível | Funciona, mas perdidas em failover Multi-AZ |
| SQL Dinâmico com `EXEC @str` | Disponível | Disponível (preferir `sp_executesql`) |
| Procedures do sistema (`sp_`) | Acesso total | Algumas restritas (ex: `sp_configure` limitado) |

---

## 1. Stored Procedures

```sql
-- Procedure com parâmetros de entrada e saída
CREATE OR ALTER PROCEDURE vendas.usp_BuscarPedidosCliente
    @ClienteID    INT,
    @DataInicio   DATE           = NULL,
    @DataFim      DATE           = NULL,
    @TotalPedidos INT            OUTPUT,
    @ValorTotal   DECIMAL(18,2)  OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SET @DataInicio = ISNULL(@DataInicio, '1900-01-01');
    SET @DataFim    = ISNULL(@DataFim,    '2099-12-31');

    SELECT
        p.PedidoID,
        p.NumeroPedido,
        p.Status,
        p.Valor,
        p.DataPedido
    FROM vendas.Pedidos p
    WHERE p.ClienteID   = @ClienteID
      AND p.DataPedido BETWEEN @DataInicio AND @DataFim
    ORDER BY p.DataPedido DESC;

    SELECT
        @TotalPedidos = COUNT(*),
        @ValorTotal   = SUM(Valor)
    FROM vendas.Pedidos
    WHERE ClienteID   = @ClienteID
      AND DataPedido BETWEEN @DataInicio AND @DataFim;
END;
GO

-- Executar
DECLARE @Total INT, @Valor DECIMAL(18,2);
EXEC vendas.usp_BuscarPedidosCliente
    @ClienteID    = 1,
    @DataInicio   = '2024-01-01',
    @TotalPedidos = @Total OUTPUT,
    @ValorTotal   = @Valor OUTPUT;

SELECT @Total AS TotalPedidos, @Valor AS ValorTotal;
GO
```

```sql
-- Procedure com TRY/CATCH e transação
CREATE OR ALTER PROCEDURE vendas.usp_CancelarPedido
    @PedidoID   INT,
    @Motivo     NVARCHAR(500)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

            IF NOT EXISTS (SELECT 1 FROM vendas.Pedidos WHERE PedidoID = @PedidoID AND Status = 1)
            BEGIN
                RAISERROR('Pedido não encontrado ou não está aberto.', 16, 1);
                RETURN;
            END

            UPDATE vendas.Pedidos
            SET
                Status        = 3,
                Observacoes   = ISNULL(Observacoes,'') + CHAR(13)+CHAR(10) + 'Cancelado: ' + @Motivo,
                DataAlteracao = GETUTCDATE()
            WHERE PedidoID = @PedidoID;

        COMMIT TRANSACTION;
        SELECT 'Pedido cancelado com sucesso.' AS Resultado;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        SELECT
            ERROR_NUMBER()  AS Codigo,
            ERROR_MESSAGE() AS Mensagem,
            ERROR_LINE()    AS Linha;
    END CATCH;
END;
GO

-- Executar
EXEC vendas.usp_CancelarPedido @PedidoID = 10, @Motivo = 'Solicitação do cliente';
```

> ☁️ **AWS RDS:** `xp_cmdshell` dentro de procedures está bloqueado. Se sua procedure precisava chamar um executável ou script externo, substitua por um script PowerShell externo que invoca a procedure via `Invoke-Sqlcmd`. CLR Procedures também não estão disponíveis no RDS.

---

## 2. Funções

```sql
-- Função escalar
CREATE OR ALTER FUNCTION dbo.fn_FormatarCPF (@CPF VARCHAR(11))
RETURNS VARCHAR(14)
AS
BEGIN
    RETURN STUFF(STUFF(STUFF(@CPF, 4, 0, '.'), 8, 0, '.'), 12, 0, '-');
END;
GO
-- Uso: SELECT dbo.fn_FormatarCPF('12345678901') → '123.456.789-01'

-- Função inline Table-Valued (melhor performance que multi-statement)
CREATE OR ALTER FUNCTION vendas.fn_PedidosPorCliente (@ClienteID INT)
RETURNS TABLE
AS
RETURN (
    SELECT
        p.PedidoID,
        p.NumeroPedido,
        p.Valor,
        p.DataPedido,
        p.Status
    FROM vendas.Pedidos p
    WHERE p.ClienteID = @ClienteID
);
GO

-- Uso
SELECT * FROM vendas.fn_PedidosPorCliente(1) WHERE Status = 1;
```

> ☁️ **AWS RDS:** Funções T-SQL (escalares e inline TVF) funcionam normalmente. CLR Functions (escritas em C#/VB.NET) estão bloqueadas — reescreva a lógica em T-SQL puro ou mova para a camada de aplicação.

---

## 3. Triggers

```sql
-- Trigger de auditoria INSERT / UPDATE / DELETE
CREATE TABLE logs.AuditoriaTabelas (
    AuditoriaID  BIGINT       NOT NULL IDENTITY(1,1),
    Tabela       VARCHAR(128) NOT NULL,
    Operacao     CHAR(1)      NOT NULL,   -- I, U, D
    UsuarioDB    VARCHAR(128) NOT NULL DEFAULT SUSER_SNAME(),
    DataHora     DATETIME2(3) NOT NULL DEFAULT SYSDATETIME(),
    DadosAntes   NVARCHAR(MAX)    NULL,
    DadosDepois  NVARCHAR(MAX)    NULL,
    CONSTRAINT PK_Auditoria PRIMARY KEY CLUSTERED (AuditoriaID)
);
GO

CREATE OR ALTER TRIGGER trg_Clientes_Auditoria
ON vendas.Clientes
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    -- INSERT
    IF EXISTS (SELECT 1 FROM inserted) AND NOT EXISTS (SELECT 1 FROM deleted)
        INSERT INTO logs.AuditoriaTabelas (Tabela, Operacao, DadosDepois)
        SELECT 'vendas.Clientes', 'I', (SELECT * FROM inserted FOR JSON AUTO);

    -- DELETE
    IF EXISTS (SELECT 1 FROM deleted) AND NOT EXISTS (SELECT 1 FROM inserted)
        INSERT INTO logs.AuditoriaTabelas (Tabela, Operacao, DadosAntes)
        SELECT 'vendas.Clientes', 'D', (SELECT * FROM deleted FOR JSON AUTO);

    -- UPDATE
    IF EXISTS (SELECT 1 FROM inserted) AND EXISTS (SELECT 1 FROM deleted)
        INSERT INTO logs.AuditoriaTabelas (Tabela, Operacao, DadosAntes, DadosDepois)
        SELECT 'vendas.Clientes', 'U',
               (SELECT * FROM deleted  FOR JSON AUTO),
               (SELECT * FROM inserted FOR JSON AUTO);
END;
GO
```

> ☁️ **AWS RDS:** Triggers DML (`AFTER`, `INSTEAD OF`) funcionam normalmente. Triggers CLR (código .NET no trigger) estão bloqueados. Para auditoria de dados sensíveis no RDS, prefira triggers T-SQL com gravação em tabela de log — como o exemplo acima.

---

## 4. Tabelas Temporárias e Table Variables

```sql
-- Tabela temporária local (# — escopo da sessão)
CREATE TABLE #PedidosTemp (
    PedidoID  INT,
    ClienteID INT,
    Valor     DECIMAL(18,2)
);

INSERT INTO #PedidosTemp
SELECT PedidoID, ClienteID, Valor FROM vendas.Pedidos
WHERE DataPedido >= DATEADD(MONTH, -1, GETDATE());

SELECT * FROM #PedidosTemp;
DROP TABLE IF EXISTS #PedidosTemp;

-- Table variable (escopo da variável — sem estatísticas — ideal para < 1.000 linhas)
DECLARE @tbl TABLE (
    ID   INT,
    Nome NVARCHAR(100)
);
INSERT INTO @tbl VALUES (1,'Alpha'),(2,'Beta'),(3,'Gamma');
SELECT * FROM @tbl;
```

### ☁️ AWS RDS — Tabelas temporárias globais (`##`)

```sql
-- Tabelas temporárias globais (## — visíveis para todas as sessões)
-- funcionam no RDS, mas use com cautela:
-- em instâncias Multi-AZ um failover destrói as ##temp tables.
CREATE TABLE ##CacheShared (
    Chave VARCHAR(100) PRIMARY KEY,
    Valor NVARCHAR(MAX)
);
```

---

## 5. JSON (SQL Server 2016+ / RDS 2016+)

```sql
-- Gerar JSON de query
SELECT ClienteID, Nome, Email
FROM vendas.Clientes WHERE Ativo = 1
FOR JSON PATH, ROOT('clientes');

-- Ler valores de JSON
DECLARE @json NVARCHAR(MAX) = '{"nome":"João","email":"joao@test.com","ativo":true}';

SELECT
    JSON_VALUE(@json, '$.nome')  AS Nome,
    JSON_VALUE(@json, '$.email') AS Email;

-- Expandir array JSON em linhas
DECLARE @pedidos NVARCHAR(MAX) = '[
    {"id":1,"valor":100.50},
    {"id":2,"valor":250.00}
]';

SELECT id, valor
FROM OPENJSON(@pedidos)
WITH (
    id    INT           '$.id',
    valor DECIMAL(18,2) '$.valor'
);

-- Coluna JSON com validação
ALTER TABLE vendas.Pedidos ADD MetadadosJson NVARCHAR(MAX) NULL
    CONSTRAINT CK_Pedidos_JSON CHECK (ISJSON(MetadadosJson) = 1);
```

---

## 6. SQL Dinâmico

```sql
-- Executar string como SQL (usar com cuidado — risco de SQL Injection)
DECLARE @tabela   NVARCHAR(128) = 'vendas.Clientes';
DECLARE @coluna   NVARCHAR(128) = 'Estado';
DECLARE @filtro   NVARCHAR(10)  = 'SP';
DECLARE @sql      NVARCHAR(MAX);

SET @sql = N'SELECT * FROM ' + QUOTENAME(@tabela) + ' WHERE ' + QUOTENAME(@coluna) + ' = @val';

EXEC sp_executesql @sql, N'@val NVARCHAR(10)', @val = @filtro;

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

> ☁️ **AWS RDS:** SQL Dinâmico funciona normalmente. Prefira sempre `sp_executesql` com parâmetros em vez de concatenação de string — além de prevenir SQL Injection, o RDS (assim como qualquer SQL Server) reutiliza melhor os planos de execução com queries parametrizadas.
