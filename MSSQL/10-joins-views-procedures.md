# JOINs, Views e Stored Procedures — Guia Conceitual e Prático

> Este documento explica os conceitos de forma acessível para **leigos** e também traz a visão **técnica** com exemplos práticos em T-SQL.

---

## Parte 1 — JOINs: Combinando Tabelas

### O que é um JOIN?

Um JOIN é a forma de **combinar dados de duas ou mais tabelas** em uma única consulta.

Pense assim: você tem uma agenda de contatos (tabela `Cliente`) e um caderno de pedidos (tabela `Pedido`). Para ver "qual cliente fez qual pedido", você precisa cruzar as duas — isso é um JOIN.

---

### INNER JOIN — "Só quem está nos dois lados"

#### Para leigos

Imagine uma **lista de convidados de uma festa** e uma **lista de pessoas que compareceram**. O INNER JOIN te mostra só quem estava nas **duas listas** — ou seja, foi convidado **e** compareceu. Quem foi convidado mas não veio fica de fora. Quem veio sem convite também fica de fora.

```
Lista de Convidados    Lista de Presença
─────────────────      ─────────────────
Ana                    Ana          ← aparece nos dois → INNER JOIN inclui
Carlos                 Carlos       ← aparece nos dois → INNER JOIN inclui
Mariana                              ← só nos convidados → NÃO aparece
                       Pedro         ← só na presença  → NÃO aparece
```

**Resultado do INNER JOIN:** Ana, Carlos

#### Para técnicos

Retorna apenas as linhas que têm **correspondência em ambas as tabelas**. Linhas sem par são descartadas.

```sql
-- Pedidos com seus clientes (somente pedidos que têm cliente cadastrado)
SELECT
    p.PedidoID,
    p.CodigoPedido,
    c.NomeCompleto AS Cliente,
    p.ValorTotal
FROM vendas.Pedido p
INNER JOIN dbo.Cliente c ON c.ClienteID = p.ClienteID;

-- Se um pedido tiver ClienteID = 99 e não existir Cliente com ID 99,
-- esse pedido NÃO aparece no resultado.
```

**Use quando:** você só quer registros que tenham par nos dois lados. A ausência de correspondência significa dado inválido ou incompleto.

---

### LEFT JOIN — "Tudo do lado esquerdo, mesmo sem par"

#### Para leigos

Agora imagine que você quer a **lista de todos os clientes**, e para cada um, ver se eles fizeram algum pedido. Alguns clientes nunca fizeram pedido — mas você ainda quer que eles apareçam na lista, só com o pedido em branco.

O LEFT JOIN garante que **todo mundo do lado esquerdo aparece**, mesmo que não tenha nada do lado direito.

```
Clientes (esquerda)    Pedidos (direita)
───────────────────    ─────────────────
Ana        ───────────► Pedido #1     ← tem par → aparece com dados do pedido
Carlos     ───────────► Pedido #2     ← tem par → aparece com dados do pedido
Mariana    ──────────── (sem pedido)  ← SEM par → aparece com pedido NULL
```

**Resultado do LEFT JOIN:** Ana + Pedido #1 | Carlos + Pedido #2 | Mariana + NULL

#### Para técnicos

Retorna **todas as linhas da tabela da esquerda** (a do FROM) e as linhas correspondentes da direita. Onde não há correspondência, os campos da direita vêm como `NULL`.

```sql
-- Todos os clientes, com ou sem pedidos
SELECT
    c.NomeCompleto AS Cliente,
    p.PedidoID,
    p.CodigoPedido,
    p.ValorTotal
FROM dbo.Cliente c
LEFT JOIN vendas.Pedido p ON p.ClienteID = c.ClienteID;
-- Clientes sem pedido aparecem com PedidoID = NULL

-- Encontrar clientes que NUNCA fizeram pedido
SELECT c.ClienteID, c.NomeCompleto
FROM dbo.Cliente c
LEFT JOIN vendas.Pedido p ON p.ClienteID = c.ClienteID
WHERE p.PedidoID IS NULL;  -- filtra só os que não têm par
```

**Use quando:** você quer a lista completa do lado esquerdo independente de haver correspondência no lado direito.

---

### Comparação visual INNER vs LEFT

```
Tabela A (Clientes)        Tabela B (Pedidos)
┌─────────────────┐        ┌──────────────────┐
│ ID │ Nome       │        │ PedidoID│ClienteID│
├────┼────────────┤        ├─────────┼─────────┤
│  1 │ Ana        │──────► │    101  │    1    │
│  2 │ Carlos     │──────► │    102  │    2    │
│  3 │ Mariana    │  ✗     │    103  │    1    │
└─────────────────┘        └──────────────────┘

INNER JOIN → Ana(101), Ana(103), Carlos(102)   [Mariana não aparece]
LEFT JOIN  → Ana(101), Ana(103), Carlos(102), Mariana(NULL)  [Mariana aparece com NULL]
```

---

### RIGHT JOIN — "Tudo do lado direito, mesmo sem par"

#### Para leigos

É o inverso do LEFT JOIN. Você garante que **tudo do lado direito aparece**, mesmo sem par no lado esquerdo. Na prática, é pouco usado — prefere-se inverter a ordem das tabelas e usar LEFT JOIN, que é mais legível.

```sql
-- Equivalente: todos os pedidos, mesmo sem cliente cadastrado
SELECT c.NomeCompleto, p.PedidoID, p.ValorTotal
FROM dbo.Cliente c
RIGHT JOIN vendas.Pedido p ON p.ClienteID = c.ClienteID;

-- Mesmo resultado, mais legível com LEFT JOIN invertido:
SELECT c.NomeCompleto, p.PedidoID, p.ValorTotal
FROM vendas.Pedido p
LEFT JOIN dbo.Cliente c ON c.ClienteID = p.ClienteID;
```

**Dica prática:** prefira sempre LEFT JOIN e ajuste a ordem das tabelas. RIGHT JOIN deixa o código mais difícil de ler.

---

### FULL OUTER JOIN — "Tudo dos dois lados"

#### Para leigos

Mostra **absolutamente todo mundo**, dos dois lados, com ou sem par. Se não tem par, vem NULL do lado que falta.

```
Clientes               Pedidos
───────────────        ───────────────────
Ana      ─────────────► Pedido #1    ← par → aparece completo
Carlos   ─────────────► Pedido #2    ← par → aparece completo
Mariana  ── (sem pedido) NULL        ← só cliente → aparece com pedido NULL
         Pedido #99 ── (sem cliente) ← só pedido  → aparece com cliente NULL
```

```sql
SELECT c.NomeCompleto, p.PedidoID, p.ValorTotal
FROM dbo.Cliente c
FULL OUTER JOIN vendas.Pedido p ON p.ClienteID = c.ClienteID;
```

**Use quando:** você quer identificar registros órfãos dos dois lados — ex: pedidos sem cliente E clientes sem pedido, numa única consulta de diagnóstico.

---

### CROSS JOIN — "Todos com todos"

#### Para leigos

Pega cada item de uma lista e combina com cada item da outra. Se você tem 3 cores e 4 tamanhos de camiseta, o CROSS JOIN gera as 12 combinações possíveis.

```sql
-- Gerar todas as combinações de cor + tamanho
SELECT cor.Nome AS Cor, tam.Descricao AS Tamanho
FROM dbo.CorProduto cor
CROSS JOIN dbo.TamanhoProduto tam;
```

**Use quando:** você precisa de combinações cartesianas — tabelas de variações de produto, calendários, matrizes de preços. **Evite** em tabelas grandes — 1.000 linhas × 1.000 linhas = 1.000.000 linhas no resultado.

---

### Tabela de Decisão: Qual JOIN usar?

| Pergunta | JOIN recomendado |
|---|---|
| "Quero só registros com correspondência nos dois lados" | `INNER JOIN` |
| "Quero todos da tabela principal, mesmo sem par" | `LEFT JOIN` |
| "Quero encontrar registros sem par (órfãos)" | `LEFT JOIN WHERE direita.id IS NULL` |
| "Quero tudo dos dois lados, com ou sem par" | `FULL OUTER JOIN` |
| "Quero todas as combinações possíveis" | `CROSS JOIN` |

---

## Parte 2 — Views: Quando Usar?

### O que é uma View?

Uma View é uma **consulta salva com um nome**. Ela não armazena dados — toda vez que você a usa, ela executa o SELECT por baixo dos panos. Pense nela como um "atalho" ou um "apelido" para uma consulta complexa.

#### Para leigos

Imagine que toda manhã você precisa de um relatório que cruza três tabelas, filtra pelos últimos 30 dias e formata datas. Em vez de escrever essa consulta de 40 linhas toda vez, você cria uma View com o nome `vw_RelatorioVendasMensal` e simplesmente faz `SELECT * FROM vw_RelatorioVendasMensal`. A complexidade fica escondida dentro da view.

### Quando criar uma View?

✅ **Crie uma View quando:**

- A mesma consulta complexa é usada em vários lugares
- Você quer **simplificar** o acesso dos desenvolvedores/analistas a dados de múltiplas tabelas
- Você quer **controlar o que o usuário vê** (ex: uma view que esconde colunas sensíveis como `Senha` ou `CPF`)
- Você quer dar um **nome de negócio** a uma consulta técnica
- Ferramentas de BI/relatório (Power BI, Excel) precisam de uma "tabela" simples para conectar

❌ **Não use View quando:**

- Você precisa **receber parâmetros** (use Function ou Procedure)
- Você precisa **modificar dados** (INSERT/UPDATE/DELETE — use Procedure)
- O resultado precisa ser **calculado de forma dinâmica** por múltiplos critérios
- A consulta é usada uma única vez

```sql
-- ✅ Boa View: simplifica acesso a dados de múltiplas tabelas
CREATE OR ALTER VIEW vendas.vw_PedidoCompleto AS
SELECT
    p.PedidoID,
    p.CodigoPedido,
    c.NomeCompleto    AS Cliente,
    c.Estado,
    v.NomeCompleto    AS Vendedor,
    p.ValorTotal,
    p.DataPedido,
    CASE p.Status
        WHEN 1 THEN 'Aberto'
        WHEN 2 THEN 'Faturado'
        WHEN 3 THEN 'Cancelado'
    END AS StatusDescricao
FROM vendas.Pedido p
INNER JOIN dbo.Cliente  c ON c.ClienteID  = p.ClienteID
LEFT  JOIN rh.Funcionario v ON v.FuncID   = p.VendedorID;
GO

-- Uso simples pelo analista
SELECT * FROM vendas.vw_PedidoCompleto
WHERE Estado = 'SP' AND DataPedido >= '2024-01-01';

-- ✅ View de segurança: esconde dados sensíveis
CREATE OR ALTER VIEW dbo.vw_ClientePublico AS
SELECT ClienteID, NomeCompleto, Estado, DataCadastro, Ativo
FROM dbo.Cliente;
-- CPF, Senha e Telefone ficam ocultos nesta view
GO
```

---

## Parte 3 — Stored Procedures: Quando Usar?

### O que é uma Stored Procedure?

Uma Stored Procedure é um **programa armazenado no banco de dados** que executa uma sequência de operações. Ela pode receber parâmetros, tomar decisões, modificar dados e retornar resultados.

#### Para leigos

Se uma View é um "atalho para consultar", uma Procedure é um "botão de ação". Você aperta o botão "Cancelar Pedido" passando o número do pedido, e a procedure faz tudo: verifica se pode cancelar, atualiza o status, registra no log, e te devolve se deu certo ou não. Tudo em um único comando.

### Quando criar uma Stored Procedure?

✅ **Crie uma Procedure quando:**

- Você precisa **modificar dados** (INSERT, UPDATE, DELETE) com lógica de negócio
- A operação tem **múltiplos passos** que devem rodar juntos (transação)
- Você precisa de **parâmetros de entrada e/ou saída**
- Há **regras de negócio** que precisam ser validadas antes de agir
- Você quer **centralizar a lógica** para não duplicar em várias aplicações
- Precisa de **tratamento de erros** (TRY/CATCH)

❌ **Não use Procedure quando:**

- Você só precisa consultar dados simples (use SELECT ou View)
- A lógica é tão simples que um UPDATE direto resolve
- A operação precisa ser usada em um JOIN (use Function)

```sql
-- ✅ Boa Procedure: ação de negócio com validação, transação e retorno
CREATE OR ALTER PROCEDURE vendas.usp_CancelarPedido
    @PedidoID  INT,
    @Motivo    NVARCHAR(500),
    @Resultado NVARCHAR(200) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- Validação
        IF NOT EXISTS (SELECT 1 FROM vendas.Pedido WHERE PedidoID = @PedidoID)
        BEGIN
            SET @Resultado = 'Erro: Pedido não encontrado.';
            ROLLBACK; RETURN;
        END

        IF NOT EXISTS (SELECT 1 FROM vendas.Pedido WHERE PedidoID = @PedidoID AND Status = 1)
        BEGIN
            SET @Resultado = 'Erro: Pedido não está em aberto.';
            ROLLBACK; RETURN;
        END

        -- Ação
        UPDATE vendas.Pedido
        SET Status      = 3,
            Observacao  = ISNULL(Observacao,'') + ' | Cancelado: ' + @Motivo,
            DataAlteracao = GETUTCDATE()
        WHERE PedidoID = @PedidoID;

        COMMIT TRANSACTION;
        SET @Resultado = 'Pedido cancelado com sucesso.';
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK;
        SET @Resultado = 'Erro interno: ' + ERROR_MESSAGE();
    END CATCH;
END;
GO

-- Chamada
DECLARE @msg NVARCHAR(200);
EXEC vendas.usp_CancelarPedido
    @PedidoID  = 42,
    @Motivo    = 'Solicitação do cliente',
    @Resultado = @msg OUTPUT;

SELECT @msg;
```

---

## Parte 4 — Comparativo: View vs Procedure vs Function

| | View | Stored Procedure | Function (inline) |
|---|---|---|---|
| **Objetivo** | Simplificar consulta | Executar ação | Calcular valor ou retornar tabela |
| **Modifica dados?** | ❌ (em geral) | ✅ | ❌ |
| **Recebe parâmetros?** | ❌ | ✅ | ✅ |
| **Retorna tabela?** | ✅ | ✅ (via SELECT) | ✅ (inline TVF) |
| **Retorna valor único?** | ❌ | Via OUTPUT | ✅ (scalar) |
| **Pode usar em JOIN?** | ✅ | ❌ | ✅ |
| **Tem TRY/CATCH?** | ❌ | ✅ | ❌ |
| **Tem transação?** | ❌ | ✅ | ❌ |
| **Quando usar** | Consultas reutilizáveis | Ações de negócio | Cálculos e filtros reutilizáveis |

### Resumo em uma frase

- **View** → "Deixa eu ver esses dados de forma simples"
- **Procedure** → "Faz essa operação por mim, com segurança"
- **Function** → "Calcula isso para mim e deixa eu usar no SELECT"

```sql
-- Os três juntos em um exemplo real:

-- Function: calcula desconto
CREATE OR ALTER FUNCTION dbo.fn_CalcularDesconto (@ValorTotal DECIMAL(18,2), @TipoCliente TINYINT)
RETURNS DECIMAL(18,2) AS
BEGIN
    RETURN CASE @TipoCliente
        WHEN 3 THEN @ValorTotal * 0.15  -- Premium: 15%
        WHEN 2 THEN @ValorTotal * 0.10  -- Ouro: 10%
        ELSE         @ValorTotal * 0    -- Standard: 0%
    END
END;
GO

-- View: usa a function para simplificar consulta
CREATE OR ALTER VIEW vendas.vw_PedidoComDesconto AS
SELECT
    p.PedidoID,
    c.NomeCompleto AS Cliente,
    p.ValorTotal,
    dbo.fn_CalcularDesconto(p.ValorTotal, c.TipoCliente) AS ValorDesconto,
    p.ValorTotal - dbo.fn_CalcularDesconto(p.ValorTotal, c.TipoCliente) AS ValorFinal
FROM vendas.Pedido p
INNER JOIN dbo.Cliente c ON c.ClienteID = p.ClienteID;
GO

-- Procedure: aplica o desconto e persiste
CREATE OR ALTER PROCEDURE vendas.usp_AplicarDesconto @PedidoID INT AS
BEGIN
    UPDATE p
    SET p.ValorFinal = p.ValorTotal - dbo.fn_CalcularDesconto(p.ValorTotal, c.TipoCliente)
    FROM vendas.Pedido p
    INNER JOIN dbo.Cliente c ON c.ClienteID = p.ClienteID
    WHERE p.PedidoID = @PedidoID;
END;
GO
```
