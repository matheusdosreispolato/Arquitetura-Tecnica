# T-SQL Básico — SELECT, INSERT, UPDATE, DELETE

> Estes comandos funcionam em **ambos os ambientes**. Diferenças específicas do RDS estão marcadas com o ícone ☁️.

---

## 1. Conexão ao Banco

### 🖥️ MSSQL Padrão (instância local ou remota)

```powershell
# Via sqlcmd
sqlcmd -S "SERVIDOR\INSTANCIA" -U "usuario" -P "senha" -d "Banco" -Q "SELECT @@VERSION"

# Autenticação Windows (sem usuário/senha)
sqlcmd -S "SERVIDOR\INSTANCIA" -E -d "Banco"
```

### ☁️ AWS RDS

```powershell
# Endpoint do RDS — sem \INSTANCIA, sempre pela porta 1433
sqlcmd -S "meu-rds.xxxxxxxx.sa-east-1.rds.amazonaws.com,1433" `
       -U "admin" `
       -P "SuaSenha123!" `
       -d "MeuBanco"

# Testar conectividade antes de conectar
Test-NetConnection -ComputerName "meu-rds.xxxxxxxx.sa-east-1.rds.amazonaws.com" -Port 1433
```

> **Diferença:** No RDS não existe o formato `SERVIDOR\INSTANCIA`. O acesso é sempre pelo endpoint DNS + porta 1433.

---

## 2. Banco de Dados — Informações Gerais

```sql
-- Listar bancos de dados
SELECT name, database_id, state_desc, recovery_model_desc
FROM sys.databases ORDER BY name;

-- Banco em uso no momento
SELECT DB_NAME() AS BancoAtual;

-- Trocar de banco
USE MeuBanco;
GO

-- Tamanho dos bancos
SELECT
    DB_NAME(database_id)     AS Banco,
    SUM(size * 8 / 1024)     AS TamanhoMB
FROM sys.master_files
GROUP BY database_id
ORDER BY TamanhoMB DESC;
```

---

## 3. SELECT

```sql
-- Tudo
SELECT * FROM dbo.Clientes;

-- Colunas com alias
SELECT
    ClienteID   AS ID,
    Nome        AS NomeCompleto,
    Email,
    DataCadastro
FROM dbo.Clientes;

-- Filtros WHERE
SELECT * FROM dbo.Clientes
WHERE Ativo = 1 AND Estado = 'SP';

-- Ordenação
SELECT * FROM dbo.Pedidos ORDER BY DataPedido DESC;

-- Paginação com OFFSET/FETCH (SQL Server 2012+)
SELECT * FROM dbo.Pedidos
ORDER BY DataPedido DESC
OFFSET 0 ROWS FETCH NEXT 50 ROWS ONLY;

-- TOP N
SELECT TOP 10 * FROM dbo.Pedidos ORDER BY DataPedido DESC;

-- DISTINCT
SELECT DISTINCT Estado FROM dbo.Clientes;

-- Contagem e agregações
SELECT
    Estado,
    COUNT(*)          AS Total,
    MAX(DataCadastro) AS UltimoCadastro
FROM dbo.Clientes
GROUP BY Estado
HAVING COUNT(*) > 10
ORDER BY Total DESC;

-- INNER JOIN
SELECT
    p.PedidoID,
    c.Nome    AS Cliente,
    p.Total,
    p.DataPedido
FROM dbo.Pedidos p
INNER JOIN dbo.Clientes c ON c.ClienteID = p.ClienteID
WHERE p.DataPedido >= '2024-01-01';

-- LEFT JOIN (inclui registros sem correspondência)
SELECT c.Nome, p.PedidoID
FROM dbo.Clientes c
LEFT JOIN dbo.Pedidos p ON p.ClienteID = c.ClienteID
WHERE p.PedidoID IS NULL;   -- Clientes que nunca compraram

-- Subquery
SELECT * FROM dbo.Clientes
WHERE ClienteID IN (
    SELECT DISTINCT ClienteID FROM dbo.Pedidos
    WHERE DataPedido >= DATEADD(MONTH, -3, GETDATE())
);

-- LIKE
SELECT * FROM dbo.Clientes WHERE Nome LIKE '%Silva%';

-- BETWEEN
SELECT * FROM dbo.Pedidos
WHERE DataPedido BETWEEN '2024-01-01' AND '2024-12-31';

-- CASE WHEN
SELECT
    Nome,
    CASE
        WHEN TotalCompras > 10000 THEN 'Premium'
        WHEN TotalCompras > 5000  THEN 'Ouro'
        ELSE 'Standard'
    END AS Categoria
FROM dbo.Clientes;
```

---

## 4. INSERT

```sql
-- Simples
INSERT INTO dbo.Clientes (Nome, Email, Estado, DataCadastro, Ativo)
VALUES ('João Silva', 'joao@email.com', 'SP', GETDATE(), 1);

-- Múltiplos registros
INSERT INTO dbo.Clientes (Nome, Email, Estado, DataCadastro, Ativo)
VALUES
    ('Maria Souza',  'maria@email.com',  'RJ', GETDATE(), 1),
    ('Carlos Lima',  'carlos@email.com', 'MG', GETDATE(), 1),
    ('Ana Pereira',  'ana@email.com',    'SP', GETDATE(), 0);

-- INSERT com SELECT (cópia de dados)
INSERT INTO dbo.ClientesArquivo (Nome, Email, Estado, DataArquivo)
SELECT Nome, Email, Estado, GETDATE()
FROM dbo.Clientes WHERE Ativo = 0;

-- INSERT com OUTPUT (retorna o ID gerado)
INSERT INTO dbo.Clientes (Nome, Email)
OUTPUT INSERTED.ClienteID, INSERTED.Nome
VALUES ('Teste', 'teste@email.com');
```

---

## 5. UPDATE

```sql
-- Simples
UPDATE dbo.Clientes
SET Ativo = 0
WHERE ClienteID = 42;

-- Múltiplos campos
UPDATE dbo.Clientes
SET
    Email         = 'novo@email.com',
    Estado        = 'SP',
    DataAlteracao = GETDATE()
WHERE ClienteID = 42;

-- UPDATE com JOIN
UPDATE c
SET c.TotalCompras = sub.Total
FROM dbo.Clientes c
INNER JOIN (
    SELECT ClienteID, SUM(Valor) AS Total
    FROM dbo.Pedidos
    GROUP BY ClienteID
) sub ON sub.ClienteID = c.ClienteID;

-- UPDATE com OUTPUT (ver o que foi alterado)
UPDATE dbo.Clientes
SET Ativo = 0
OUTPUT
    DELETED.ClienteID,
    DELETED.Ativo  AS AtivoAntes,
    INSERTED.Ativo AS AtivoDepois
WHERE DataCadastro < '2020-01-01';
```

---

## 6. DELETE

```sql
-- Simples
DELETE FROM dbo.Clientes WHERE ClienteID = 42;

-- Com condição composta
DELETE FROM dbo.Logs
WHERE DataLog < DATEADD(MONTH, -6, GETDATE())
  AND Tipo = 'DEBUG';

-- DELETE com JOIN
DELETE p
FROM dbo.Pedidos p
INNER JOIN dbo.Clientes c ON c.ClienteID = p.ClienteID
WHERE c.Ativo = 0;

-- DELETE com OUTPUT (registra o que foi removido)
DELETE FROM dbo.Clientes
OUTPUT DELETED.*
WHERE Ativo = 0 AND DataCadastro < '2019-01-01';

-- TRUNCATE (remove tudo sem log por linha — muito mais rápido)
-- ATENÇÃO: irreversível, não dispara triggers
TRUNCATE TABLE dbo.LogsTemporarios;
```

---

## 7. Datas e Funções de Tempo

```sql
-- Data e hora atual
SELECT GETDATE(), GETUTCDATE(), SYSDATETIME();

-- Manipulação
SELECT
    DATEADD(DAY,    -7, GETDATE())  AS SemanaPassada,
    DATEADD(MONTH,  -1, GETDATE())  AS MesPassado,
    DATEADD(YEAR,   -1, GETDATE())  AS AnoPassado,
    DATEDIFF(DAY, '2024-01-01', GETDATE()) AS DiasDesde2024,
    FORMAT(GETDATE(), 'yyyy-MM-dd') AS DataFormatada,
    YEAR(GETDATE())   AS Ano,
    MONTH(GETDATE())  AS Mes,
    DAY(GETDATE())    AS Dia;

-- Primeiro e último dia do mês
SELECT
    DATEFROMPARTS(YEAR(GETDATE()), MONTH(GETDATE()), 1) AS PrimeiroDia,
    EOMONTH(GETDATE())                                   AS UltimoDia;
```

---

## 8. Transações com TRY/CATCH

```sql
-- Padrão em ambos os ambientes
BEGIN TRY
    BEGIN TRANSACTION;

        UPDATE dbo.Contas SET Saldo = Saldo - 1000 WHERE ContaID = 1;
        UPDATE dbo.Contas SET Saldo = Saldo + 1000 WHERE ContaID = 2;

    COMMIT TRANSACTION;
    PRINT 'Transferência realizada.';
END TRY
BEGIN CATCH
    ROLLBACK TRANSACTION;
    PRINT 'Erro: ' + ERROR_MESSAGE();
END CATCH;
```

---

## 9. Variáveis e Controle de Fluxo

```sql
-- Variáveis
DECLARE @ClienteID INT = 42;
DECLARE @Nome      NVARCHAR(100);

SELECT @Nome = Nome FROM dbo.Clientes WHERE ClienteID = @ClienteID;
PRINT 'Cliente: ' + ISNULL(@Nome, 'Não encontrado');

-- IF / ELSE
IF EXISTS (SELECT 1 FROM dbo.Clientes WHERE ClienteID = @ClienteID)
    PRINT 'Cliente existe.'
ELSE
    PRINT 'Não encontrado.';

-- WHILE
DECLARE @i INT = 1;
WHILE @i <= 5
BEGIN
    PRINT 'Iteração: ' + CAST(@i AS VARCHAR);
    SET @i = @i + 1;
END
```

---

## 10. Informações do Ambiente

```sql
-- Versão do SQL Server
SELECT @@VERSION;

-- Nome do servidor
SELECT @@SERVERNAME;

-- Usuário logado
SELECT SUSER_NAME() AS Login, USER_NAME() AS UsuarioDB;
```

### ☁️ No RDS, `@@SERVERNAME` retorna o identificador interno da instância — não o endpoint DNS externo. Use o endpoint da sua connection string para conexões externas.
