# RDL — Datasets, Parâmetros e Filtros

> Como conectar o relatório ao banco, criar queries com parâmetros e configurar filtros dinâmicos para o usuário.

---

## 1. DataSource — Tipos de Conexão

### Conexão embutida (Embedded)

A string de conexão fica dentro do `.rdl`. Funciona, mas não recomendado para produção.

```xml
<DataSources>
  <DataSource Name="DS_Banco">
    <ConnectionProperties>
      <DataProvider>SQL</DataProvider>
      <ConnectString>
        Data Source=meu-rds.xxxx.sa-east-1.rds.amazonaws.com,1433;
        Initial Catalog=MeuBanco;
        User ID=ro_relatorio;
        Password=Senha123!;
        Encrypt=True;
        TrustServerCertificate=True
      </ConnectString>
    </ConnectionProperties>
  </DataSource>
</DataSources>
```

### Conexão compartilhada (Shared DataSource — recomendado)

Referencia um `.rds` publicado no servidor SSRS. Uma mudança no servidor basta para todos os relatórios.

```xml
<DataSources>
  <DataSource Name="DS_Banco">
    <DataSourceReference>/Datasources/DS_Producao</DataSourceReference>
  </DataSource>
</DataSources>
```

### Providers disponíveis

| Provider | Banco de dados |
|---|---|
| `SQL` | SQL Server / Azure SQL / RDS MSSQL |
| `OLEDB` | OLE DB genérico |
| `ODBC` | ODBC genérico |
| `Oracle` | Oracle Database |
| `OLEDB-MD` | Cubo OLAP / Analysis Services |

---

## 2. Dataset — Queries

### Dataset simples

```xml
<DataSets>
  <DataSet Name="DS_Pedidos">
    <Query>
      <DataSourceName>DS_Banco</DataSourceName>
      <CommandType>Text</CommandType>
      <CommandText>
        SELECT
            p.PedidoID,
            p.CodigoPedido,
            c.NomeCompleto  AS Cliente,
            c.Estado,
            p.ValorTotal,
            p.DataPedido,
            p.Status
        FROM vendas.Pedido p
        INNER JOIN dbo.Cliente c ON c.ClienteID = p.ClienteID
        WHERE p.DataPedido BETWEEN @DataInicio AND @DataFim
        ORDER BY p.DataPedido DESC
      </CommandText>
      <QueryParameters>
        <QueryParameter Name="@DataInicio">
          <Value>=Parameters!DataInicio.Value</Value>
        </QueryParameter>
        <QueryParameter Name="@DataFim">
          <Value>=Parameters!DataFim.Value</Value>
        </QueryParameter>
      </QueryParameters>
    </Query>
  </DataSet>
</DataSets>
```

### Dataset chamando Stored Procedure

```xml
<DataSet Name="DS_RelatorioVendas">
  <Query>
    <DataSourceName>DS_Banco</DataSourceName>
    <CommandType>StoredProcedure</CommandType>
    <CommandText>vendas.usp_RelatorioVendasMensal</CommandText>
    <QueryParameters>
      <QueryParameter Name="@Mes">
        <Value>=Parameters!Mes.Value</Value>
      </QueryParameter>
      <QueryParameter Name="@Ano">
        <Value>=Parameters!Ano.Value</Value>
      </QueryParameter>
      <QueryParameter Name="@Estado">
        <Value>=Parameters!Estado.Value</Value>
      </QueryParameter>
    </QueryParameters>
  </Query>
</DataSet>
```

> **Boas práticas:** prefira chamar **Stored Procedures** nos datasets. Isso mantém a lógica SQL no banco, facilita manutenção e melhora a segurança.

---

## 3. Múltiplos Datasets no Mesmo Relatório

Um relatório pode ter vários datasets para propósitos diferentes:

```
DS_Cabecalho    → dados do cliente / empresa (um registro)
DS_Pedidos      → lista de pedidos (múltiplos registros)
DS_Totais       → totalizadores por mês (para o gráfico)
DS_Parametros   → alimenta listas de parâmetros (estados, status)
```

```xml
<!-- Dataset para preencher o parâmetro de Estado -->
<DataSet Name="DS_Estados">
  <Query>
    <DataSourceName>DS_Banco</DataSourceName>
    <CommandText>
      SELECT DISTINCT Estado FROM dbo.Cliente ORDER BY Estado
    </CommandText>
  </Query>
</DataSet>

<!-- Dataset para preencher status -->
<DataSet Name="DS_StatusPedido">
  <Query>
    <DataSourceName>DS_Banco</DataSourceName>
    <CommandText>
      SELECT StatusID, Descricao
      FROM dbo.TipoStatus
      WHERE Tabela = 'Pedido'
      ORDER BY StatusID
    </CommandText>
  </Query>
</DataSet>
```

---

## 4. Parâmetros — Tipos e Configurações

### Parâmetro de Data

```xml
<ReportParameter Name="DataInicio">
  <DataType>DateTime</DataType>
  <Nullable>false</Nullable>
  <DefaultValue>
    <!-- Padrão: primeiro dia do mês atual -->
    <Values>
      <Value>=DateSerial(Year(Today()), Month(Today()), 1)</Value>
    </Values>
  </DefaultValue>
  <Prompt>Data Início</Prompt>
  <AllowBlank>false</AllowBlank>
</ReportParameter>

<ReportParameter Name="DataFim">
  <DataType>DateTime</DataType>
  <DefaultValue>
    <Values><Value>=Today()</Value></Values>
  </DefaultValue>
  <Prompt>Data Fim</Prompt>
</ReportParameter>
```

### Parâmetro de Lista (dropdown)

```xml
<!-- Parâmetro que mostra lista de estados -->
<ReportParameter Name="Estado">
  <DataType>String</DataType>
  <Nullable>true</Nullable>
  <DefaultValue>
    <Values><Value>Todos</Value></Values>
  </DefaultValue>
  <Prompt>Estado</Prompt>
  <ValidValues>
    <!-- Alimentado pelo dataset DS_Estados -->
    <DataSetReference>
      <DataSetName>DS_Estados</DataSetName>
      <ValueField>Estado</ValueField>   <!-- valor enviado à query -->
      <LabelField>Estado</LabelField>   <!-- texto exibido ao usuário -->
    </DataSetReference>
  </ValidValues>
</ReportParameter>
```

### Parâmetro com valores fixos

```xml
<ReportParameter Name="Status">
  <DataType>Integer</DataType>
  <Prompt>Status do Pedido</Prompt>
  <ValidValues>
    <ParameterValues>
      <ParameterValue>
        <Value>0</Value>
        <Label>Todos</Label>
      </ParameterValue>
      <ParameterValue>
        <Value>1</Value>
        <Label>Aberto</Label>
      </ParameterValue>
      <ParameterValue>
        <Value>2</Value>
        <Label>Faturado</Label>
      </ParameterValue>
      <ParameterValue>
        <Value>3</Value>
        <Label>Cancelado</Label>
      </ParameterValue>
    </ParameterValues>
  </ValidValues>
  <DefaultValue>
    <Values><Value>0</Value></Values>
  </DefaultValue>
</ReportParameter>
```

### Parâmetro Multi-Valor (seleção múltipla)

```xml
<ReportParameter Name="Estados">
  <DataType>String</DataType>
  <MultiValue>true</MultiValue>   <!-- habilita seleção múltipla -->
  <Prompt>Estados</Prompt>
  <ValidValues>
    <DataSetReference>
      <DataSetName>DS_Estados</DataSetName>
      <ValueField>Estado</ValueField>
      <LabelField>Estado</LabelField>
    </DataSetReference>
  </ValidValues>
</ReportParameter>
```

**Na query T-SQL**, use `IN` com uma tabela de parâmetros:

```sql
-- Tratamento de multi-valor no T-SQL com SSRS
WHERE Estado IN (@Estados)
-- SSRS converte automaticamente o array em lista para o IN

-- Alternativa mais robusta via procedure:
CREATE PROCEDURE vendas.usp_PedidosPorEstado
    @Estados NVARCHAR(MAX)   -- recebe "SP,RJ,MG"
AS
BEGIN
    SELECT p.*, c.Estado
    FROM vendas.Pedido p
    INNER JOIN dbo.Cliente c ON c.ClienteID = p.ClienteID
    WHERE c.Estado IN (
        SELECT value FROM STRING_SPLIT(@Estados, ',')
    )
END
```

---

## 5. Parâmetros em Cascata

Parâmetros em cascata: o valor de um filtra as opções do próximo (ex: Região → Estado → Cidade).

```
[Parâmetro 1: Regiao]  →  alimenta  →  [DS_Estados filtrando por @Regiao]
                                              ↓
                                    [Parâmetro 2: Estado]  →  alimenta  →  [DS_Cidades]
                                                                                  ↓
                                                                         [Parâmetro 3: Cidade]
```

```xml
<!-- Dataset de estados filtra pela região selecionada -->
<DataSet Name="DS_EstadosPorRegiao">
  <Query>
    <CommandText>
      SELECT EstadoID, NomeEstado
      FROM dbo.Estado
      WHERE RegiaoID = @Regiao
      ORDER BY NomeEstado
    </CommandText>
    <QueryParameters>
      <QueryParameter Name="@Regiao">
        <Value>=Parameters!Regiao.Value</Value>
      </QueryParameter>
    </QueryParameters>
  </Query>
</DataSet>

<!-- Parâmetro Estado depende do Parâmetro Regiao -->
<ReportParameter Name="Estado">
  <ValidValues>
    <DataSetReference>
      <DataSetName>DS_EstadosPorRegiao</DataSetName>
      <ValueField>EstadoID</ValueField>
      <LabelField>NomeEstado</LabelField>
    </DataSetReference>
  </ValidValues>
</ReportParameter>
```

> **Atenção:** a ordem dos parâmetros no XML importa. O parâmetro que depende de outro deve vir **depois** no XML.

---

## 6. Filtros no Dataset vs Filtros na Query

| | Filtro na Query (WHERE) | Filtro no Dataset (Filters) |
|---|---|---|
| **Onde é executado** | No SQL Server | No Report Builder |
| **Performance** | ✅ Melhor — filtra no banco | ❌ Pior — traz tudo e filtra na memória |
| **Quando usar** | Sempre que possível | Quando a query não pode ser alterada |
| **Funciona com parâmetros** | ✅ | ✅ |

```xml
<!-- Filtro no Dataset (use só quando necessário) -->
<DataSet Name="DS_Pedidos">
  <Filters>
    <Filter>
      <FilterExpression>=Fields!Status.Value</FilterExpression>
      <Operator>Equal</Operator>
      <FilterValues>
        <FilterValue>=Parameters!Status.Value</FilterValue>
      </FilterValues>
    </Filter>
    <Filter>
      <FilterExpression>=Fields!ValorTotal.Value</FilterExpression>
      <Operator>GreaterThan</Operator>
      <FilterValues>
        <FilterValue>0</FilterValue>
      </FilterValues>
    </Filter>
  </Filters>
</DataSet>
```

### Operadores de Filtro disponíveis

| Operador | Significado |
|---|---|
| `Equal` | `=` |
| `NotEqual` | `<>` |
| `GreaterThan` | `>` |
| `GreaterThanOrEqual` | `>=` |
| `LessThan` | `<` |
| `LessThanOrEqual` | `<=` |
| `Like` | Contém padrão (usa `%`) |
| `In` | Está na lista |
| `Between` | Entre dois valores |
| `TopN` | Top N registros |
| `BottomN` | Bottom N registros |
| `TopPercent` | Top N% |

---

## 7. Parâmetros Ocultos (Hidden)

Parâmetros ocultos não são exibidos ao usuário, mas podem receber valores via URL ou código.

```xml
<ReportParameter Name="EmpresaID">
  <DataType>Integer</DataType>
  <Hidden>true</Hidden>   <!-- não aparece para o usuário -->
  <DefaultValue>
    <Values><Value>1</Value></Values>
  </DefaultValue>
</ReportParameter>
```

**Uso via URL** (para embutir relatório com parâmetros pré-definidos):

```
https://servidor/ReportServer/Pages/ReportViewer.aspx
  ?/Relatorios/PedidosMensal
  &DataInicio=2024-01-01
  &DataFim=2024-01-31
  &Estado=SP
  &rs:Format=PDF
```
