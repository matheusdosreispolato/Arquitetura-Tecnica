# Padronização de Nomenclaturas — MSSQL

> Guia de padrões de nomes para todos os objetos do banco de dados. Seguir esses padrões garante que qualquer pessoa do time consiga entender o que um objeto faz só pelo nome, sem precisar abrir o código.

---

## Por que padronizar nomes?

Um banco sem padrão parece isso na prática:

```sql
SELECT * FROM tbl1
JOIN USUARIOS_novo u ON u.id = tbl1.idusuario
JOIN Pedido_FINAL pf ON pf.UserId = u.UserId
```

Com padrão:

```sql
SELECT * FROM vendas.Pedidos p
JOIN dbo.Usuarios u       ON u.UsuarioID = p.UsuarioID
JOIN vendas.ItensPedido i ON i.PedidoID  = p.PedidoID
```

---

## 1. Regras Gerais

| Regra | Descrição |
|---|---|
| **PascalCase** | Primeira letra de cada palavra maiúscula: `NomeProduto`, `DataCadastro` |
| **Sem espaços** | Nunca use espaços em nomes: ❌ `Data Cadastro` → ✅ `DataCadastro` |
| **Sem acentos** | Evite acentos em nomes: ❌ `Endereço` → ✅ `Endereco` |
| **Sem abreviações desnecessárias** | ❌ `DtCd`, `NmCli` → ✅ `DataCadastro`, `NomeCliente` |
| **Sem prefixos de tipo** | ❌ `tblClientes`, `intIdade` → ✅ `Clientes`, `Idade` |
| **Idioma único** | Escolha português **ou** inglês e mantenha em todo o banco |
| **Singular para tabelas** | ❌ `Clientes` é aceito, mas a tabela representa a entidade `Cliente` |

---

## 2. Schemas (Namespaces)

Schemas organizam objetos por domínio/área de negócio. Pense neles como "gavetas" do banco.

### Padrão: `lowercase` simples, sem underline

| ✅ Correto | ❌ Errado |
|---|---|
| `vendas` | `Vendas`, `VENDAS`, `vnd` |
| `financeiro` | `fin`, `Financeiro_Novo` |
| `rh` | `RecursosHumanos`, `RH_2024` |
| `logs` | `LOG`, `log_sistema` |
| `dbo` | Reservado para objetos sem schema definido (padrão do SQL Server) |

```sql
-- Criar schemas por domínio
CREATE SCHEMA vendas;
CREATE SCHEMA financeiro;
CREATE SCHEMA rh;
CREATE SCHEMA logs;
CREATE SCHEMA integracao;  -- para tabelas de entrada/saída de outros sistemas
```

---

## 3. Tabelas

### Padrão: `PascalCase`, substantivo no **singular**

| ✅ Correto | ❌ Errado |
|---|---|
| `vendas.Pedido` | `vendas.pedidos`, `vendas.tbl_Pedido`, `vendas.PEDIDOS` |
| `vendas.ItemPedido` | `vendas.Itens_Pedido`, `vendas.tblItem` |
| `dbo.Usuario` | `dbo.USUARIO_NOVO`, `dbo.Usuarios2` |
| `rh.Funcionario` | `rh.func`, `rh.FUNCIONARIOS_FINAL` |
| `logs.AuditoriaAcesso` | `logs.log_acesso`, `logs.tbl_auditoria` |

> **Sobre singular vs plural:** o padrão mais comum no SQL Server é **singular** (`Pedido`, não `Pedidos`), pois a tabela representa a entidade, não a coleção. O mais importante é ser consistente em todo o banco.

```sql
-- Exemplos de tabelas bem nomeadas
CREATE TABLE vendas.Pedido        (PedidoID INT IDENTITY PRIMARY KEY, ...);
CREATE TABLE vendas.ItemPedido    (ItemID   INT IDENTITY PRIMARY KEY, ...);
CREATE TABLE financeiro.NotaFiscal(NotaID   INT IDENTITY PRIMARY KEY, ...);
CREATE TABLE rh.Funcionario       (FuncID   INT IDENTITY PRIMARY KEY, ...);
CREATE TABLE logs.AuditoriaTabela (AudID    BIGINT IDENTITY PRIMARY KEY, ...);
```

---

## 4. Colunas

### Padrão: `PascalCase`, nome descritivo, sem prefixo de tipo

| ✅ Correto | ❌ Errado |
|---|---|
| `ClienteID` | `id`, `ID_CLIENTE`, `intClienteId`, `fk_cliente` |
| `NomeCompleto` | `Nome1`, `NM_COMPLETO`, `nome_completo` |
| `DataCadastro` | `DtCad`, `DATA_CADASTRO`, `dataCadastro` |
| `Ativo` | `ativo_flag`, `FLG_ATIVO`, `is_active` |
| `ValorTotal` | `VL_TOT`, `valor_total`, `TotalVal` |

### Padrões específicos por tipo de coluna

| Tipo de dado | Padrão de nome | Exemplos |
|---|---|---|
| Chave primária | `<NomeTabela>ID` | `PedidoID`, `ClienteID`, `UsuarioID` |
| Chave estrangeira | `<NomeTabelaRef>ID` | `ClienteID`, `ProdutoID`, `VendedorID` |
| Data/Hora | `Data<Evento>` ou `<Evento>Em` | `DataCadastro`, `DataAlteracao`, `DataNascimento` |
| Booleano (BIT) | Adjetivo ou verbo | `Ativo`, `Excluido`, `Aprovado`, `Bloqueado` |
| Valor monetário | `Valor<Descricao>` | `ValorTotal`, `ValorDesconto`, `ValorFrete` |
| Descrição longa | `Descricao`, `Observacao` | `DescricaoProduto`, `ObservacaoPedido` |
| Código externo | `Codigo<Entidade>` | `CodigoProduto`, `CodigoERP`, `CodigoBanco` |

```sql
CREATE TABLE vendas.Pedido (
    PedidoID        INT          NOT NULL IDENTITY(1,1),  -- PK: NomeID
    ClienteID       INT          NOT NULL,                 -- FK: referencia Cliente
    VendedorID      INT              NULL,                 -- FK: referencia Vendedor
    CodigoPedido    VARCHAR(30)  NOT NULL,                 -- código externo
    ValorTotal      DECIMAL(18,2)NOT NULL DEFAULT 0,       -- monetário
    DataPedido      DATETIME2(0) NOT NULL DEFAULT GETUTCDATE(), -- data do evento
    DataFaturamento DATETIME2(0)     NULL,
    Ativo           BIT          NOT NULL DEFAULT 1,       -- booleano
    Observacao      NVARCHAR(MAX)    NULL                  -- texto livre
);
```

---

## 5. Constraints

### Padrão: `<Tipo>_<Tabela>_<Coluna(s)>`

| Tipo | Prefixo | Exemplo |
|---|---|---|
| Primary Key | `PK_` | `PK_Pedido` |
| Foreign Key | `FK_` | `FK_Pedido_Cliente` |
| Unique | `UQ_` | `UQ_Pedido_Codigo` |
| Check | `CK_` | `CK_Pedido_Status` |
| Default | `DF_` | `DF_Pedido_DataPedido` |

```sql
CREATE TABLE vendas.Pedido (
    PedidoID     INT NOT NULL IDENTITY(1,1),
    ClienteID    INT NOT NULL,
    CodigoPedido VARCHAR(30) NOT NULL,
    Status       TINYINT NOT NULL DEFAULT 1,
    DataPedido   DATETIME2(0) NOT NULL DEFAULT GETUTCDATE(),

    CONSTRAINT PK_Pedido            PRIMARY KEY (PedidoID),
    CONSTRAINT FK_Pedido_Cliente    FOREIGN KEY (ClienteID) REFERENCES dbo.Cliente(ClienteID),
    CONSTRAINT UQ_Pedido_Codigo     UNIQUE (CodigoPedido),
    CONSTRAINT CK_Pedido_Status     CHECK (Status BETWEEN 1 AND 4),
    CONSTRAINT DF_Pedido_DataPedido DEFAULT GETUTCDATE() FOR DataPedido
);
```

---

## 6. Índices

### Padrão: `<Tipo>_<Tabela>_<Colunas>`

| Tipo | Prefixo | Quando usar |
|---|---|---|
| Non-clustered | `IX_` | Índice comum de busca |
| Unique | `UX_` | Garante unicidade + performance |
| Clustered | `CX_` | Índice clusterizado (raro, geralmente é a PK) |

```sql
-- Índice simples de busca
CREATE INDEX IX_Pedido_ClienteID
ON vendas.Pedido (ClienteID);

-- Índice composto (ClienteID + data, para relatórios)
CREATE INDEX IX_Pedido_ClienteID_DataPedido
ON vendas.Pedido (ClienteID, DataPedido DESC);

-- Índice único (código externo deve ser único)
CREATE UNIQUE INDEX UX_Pedido_CodigoPedido
ON vendas.Pedido (CodigoPedido);
```

---

## 7. Views

### Padrão: `vw_<NomeDescritivo>` no schema correspondente

| ✅ Correto | ❌ Errado |
|---|---|
| `vendas.vw_PedidoCompleto` | `vendas.PedidoCompleto`, `vw_vendas_pedidos` |
| `financeiro.vw_RelatorioMensal` | `financeiro.rel_mensal`, `VIEW_FIN_MES` |
| `dbo.vw_ClienteAtivo` | `dbo.ClientesAtivos_v2`, `clientes_ativos_view` |

```sql
CREATE OR ALTER VIEW vendas.vw_PedidoCompleto AS
SELECT
    p.PedidoID,
    p.CodigoPedido,
    c.NomeCompleto AS Cliente,
    p.ValorTotal,
    p.DataPedido
FROM vendas.Pedido p
INNER JOIN dbo.Cliente c ON c.ClienteID = p.ClienteID;
GO
```

---

## 8. Stored Procedures

### Padrão: `usp_<Verbo><Entidade>` no schema correspondente

> `usp` = **u**ser **s**tored **p**rocedure (diferencia das procedures do sistema, que começam com `sp_`)

| Verbo | Quando usar | Exemplo |
|---|---|---|
| `Buscar` | SELECT / consulta | `usp_BuscarPedidoPorCliente` |
| `Inserir` | INSERT | `usp_InserirPedido` |
| `Atualizar` | UPDATE | `usp_AtualizarStatusPedido` |
| `Cancelar` | Ação de negócio | `usp_CancelarPedido` |
| `Processar` | Rotina complexa | `usp_ProcessarFaturamento` |
| `Sincronizar` | Integração | `usp_SincronizarClienteERP` |

```sql
-- ✅ Correto
CREATE OR ALTER PROCEDURE vendas.usp_BuscarPedidoPorCliente @ClienteID INT AS ...
CREATE OR ALTER PROCEDURE vendas.usp_CancelarPedido         @PedidoID INT  AS ...
CREATE OR ALTER PROCEDURE financeiro.usp_ProcessarFaturamento @Mes INT, @Ano INT AS ...

-- ❌ Errado
CREATE PROCEDURE sp_pedidos AS ...        -- sp_ é reservado para sistema
CREATE PROCEDURE GetPedidos AS ...        -- sem prefixo, sem schema, inglês misturado
CREATE PROCEDURE proc_cancelar AS ...     -- prefixo genérico
```

---

## 9. Functions

### Padrão: `fn_<NomeDescritivo>` no schema correspondente

```sql
-- Função escalar
CREATE OR ALTER FUNCTION dbo.fn_FormatarCPF (@CPF VARCHAR(11)) RETURNS VARCHAR(14) AS ...
CREATE OR ALTER FUNCTION dbo.fn_CalcularIdade (@DataNascimento DATE) RETURNS INT AS ...

-- Função inline table-valued
CREATE OR ALTER FUNCTION vendas.fn_PedidosPorPeriodo (@Inicio DATE, @Fim DATE)
RETURNS TABLE AS RETURN (...);

-- ❌ Errado
CREATE FUNCTION GetClientes() ...     -- sem prefixo, em inglês
CREATE FUNCTION function_calculo() ... -- nome genérico
```

---

## 10. Triggers

### Padrão: `trg_<Tabela>_<Evento>`

| Evento | Sufixo |
|---|---|
| INSERT | `_Insert` |
| UPDATE | `_Update` |
| DELETE | `_Delete` |
| INSERT + UPDATE | `_InsertUpdate` |
| INSERT + UPDATE + DELETE | `_Auditoria` |

```sql
CREATE OR ALTER TRIGGER trg_Pedido_Auditoria
ON vendas.Pedido AFTER INSERT, UPDATE, DELETE AS ...

CREATE OR ALTER TRIGGER trg_Cliente_Update
ON dbo.Cliente AFTER UPDATE AS ...

-- ❌ Errado
CREATE TRIGGER t1 ON vendas.Pedido ...    -- nome sem significado
CREATE TRIGGER trigger_pedidos ...        -- genérico demais
```

---

## 11. Logins e Usuários de Banco

### Padrão por tipo de uso

| Tipo | Prefixo | Exemplo |
|---|---|---|
| Aplicação (leitura + escrita) | `app_` | `app_vendas`, `app_portal` |
| Serviço / integração | `svc_` | `svc_erp`, `svc_bi`, `svc_monitoramento` |
| Somente leitura | `ro_` | `ro_relatorios`, `ro_analytics` |
| DBA / administração | `dba_` | `dba_suporte`, `dba_deploy` |
| Pessoa física (ambiente dev) | `dev_` | `dev_joao`, `dev_maria` |

```sql
-- Criar login com padrão
CREATE LOGIN app_vendas   WITH PASSWORD = 'App@Vendas2024!';
CREATE LOGIN svc_erp      WITH PASSWORD = 'Svc@Erp2024!';
CREATE LOGIN ro_relatorio WITH PASSWORD = 'Ro@Rel2024!';

-- ❌ Errado
CREATE LOGIN usuario1        WITH PASSWORD = '...'  -- sem contexto
CREATE LOGIN joao_silva      WITH PASSWORD = '...'  -- nome pessoal em produção
CREATE LOGIN admin_app       WITH PASSWORD = '...'  -- "admin" é ambíguo
CREATE LOGIN teste           WITH PASSWORD = '...'  -- NUNCA em produção
```

---

## 12. Roles (Grupos de Permissão)

### Padrão: `role_<dominio>_<nivel>`

| Nível | Significado | Exemplo |
|---|---|---|
| `readonly` | Somente leitura | `role_vendas_readonly` |
| `readwrite` | Leitura e escrita | `role_vendas_readwrite` |
| `execute` | Apenas executar procedures | `role_vendas_execute` |
| `admin` | Controle total no schema | `role_vendas_admin` |

```sql
-- Criar roles
CREATE ROLE role_vendas_readonly;
CREATE ROLE role_vendas_readwrite;
CREATE ROLE role_vendas_execute;
CREATE ROLE role_financeiro_readonly;

-- Conceder permissões às roles
GRANT SELECT                ON SCHEMA::vendas TO role_vendas_readonly;
GRANT SELECT, INSERT, UPDATE ON SCHEMA::vendas TO role_vendas_readwrite;
GRANT EXECUTE               ON SCHEMA::vendas TO role_vendas_execute;

-- Atribuir usuários às roles
ALTER ROLE role_vendas_readonly  ADD MEMBER ro_relatorio;
ALTER ROLE role_vendas_readwrite ADD MEMBER app_vendas;
ALTER ROLE role_vendas_execute   ADD MEMBER svc_erp;

-- ❌ Errado
CREATE ROLE admin         -- muito genérico
CREATE ROLE role1         -- sem significado
CREATE ROLE vendas_users  -- sem indicar nível de acesso
```

---

## 13. Tabela Resumo

```
Objeto          │ Prefixo/Padrão          │ Exemplo
────────────────┼─────────────────────────┼──────────────────────────────
Schema          │ lowercase               │ vendas, financeiro, rh, logs
Tabela          │ PascalCase singular     │ vendas.Pedido
Coluna PK       │ <Tabela>ID              │ PedidoID
Coluna FK       │ <TabelaRef>ID           │ ClienteID
Coluna Data     │ Data<Evento>            │ DataCadastro, DataAlteracao
Coluna Bool     │ Adjetivo                │ Ativo, Aprovado, Excluido
Coluna Valor    │ Valor<Desc>             │ ValorTotal, ValorFrete
PK Constraint   │ PK_<Tabela>             │ PK_Pedido
FK Constraint   │ FK_<Tab>_<TabRef>       │ FK_Pedido_Cliente
Unique          │ UQ_<Tab>_<Col>          │ UQ_Pedido_Codigo
Check           │ CK_<Tab>_<Col>          │ CK_Pedido_Status
Default         │ DF_<Tab>_<Col>          │ DF_Pedido_DataPedido
Índice          │ IX_<Tab>_<Cols>         │ IX_Pedido_ClienteID
Índice Único    │ UX_<Tab>_<Col>          │ UX_Pedido_CodigoPedido
View            │ vw_<Descricao>          │ vendas.vw_PedidoCompleto
Procedure       │ usp_<Verbo><Entidade>   │ vendas.usp_CancelarPedido
Function        │ fn_<Descricao>          │ dbo.fn_FormatarCPF
Trigger         │ trg_<Tab>_<Evento>      │ trg_Pedido_Auditoria
Login Aplicação │ app_<sistema>           │ app_vendas, app_portal
Login Serviço   │ svc_<sistema>           │ svc_erp, svc_bi
Login Readonly  │ ro_<contexto>           │ ro_relatorios
Role            │ role_<dominio>_<nivel>  │ role_vendas_readonly
```
