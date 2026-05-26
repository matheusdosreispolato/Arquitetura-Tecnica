# RDL — Introdução e Estrutura

> **RDL** (Report Definition Language) é o formato de arquivo usado pelo **SQL Server Reporting Services (SSRS)** e pelo **Report Builder**. É um XML que descreve tudo do relatório: de onde vêm os dados, como são exibidos, como são formatados e como o usuário interage.

---

## 1. O que é o Report Builder?

O **Report Builder** é a ferramenta visual da Microsoft para criar relatórios `.rdl`. Ele funciona de forma independente do Visual Studio e é voltado para analistas e usuários de negócio.

| Ferramenta | Público | Onde rodar |
|---|---|---|
| **Report Builder** | Analistas, DBA, usuários avançados | Desktop standalone |
| **SSRS (Visual Studio)** | Desenvolvedores | Visual Studio / SSDT |
| **Power BI Report Builder** | Analistas + Power BI | Publicação no Power BI Premium |

Todos geram arquivos `.rdl` com a mesma estrutura.

---

## 2. Estrutura do Arquivo RDL (XML)

Um `.rdl` é um XML. Ao abrir no Notepad ou VS Code, você verá algo assim:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Report xmlns="http://schemas.microsoft.com/sqlserver/reporting/2016/01/reportdefinition">

  <!-- 1. Data Sources — de onde vêm os dados -->
  <DataSources> ... </DataSources>

  <!-- 2. Datasets — as queries que alimentam o relatório -->
  <DataSets> ... </DataSets>

  <!-- 3. Parâmetros do relatório — filtros do usuário -->
  <ReportParameters> ... </ReportParameters>

  <!-- 4. Corpo do relatório — tabelas, gráficos, imagens -->
  <Body>
    <ReportItems> ... </ReportItems>
  </Body>

  <!-- 5. Cabeçalho e rodapé de página -->
  <PageHeader> ... </PageHeader>
  <PageFooter> ... </PageFooter>

  <!-- 6. Configurações de página (tamanho, margens) -->
  <Page> ... </Page>

</Report>
```

---

## 3. Componentes Principais

### DataSource — A Fonte de Dados

Define a conexão com o banco. Em produção, normalmente aponta para uma **conexão compartilhada** no servidor SSRS.

```xml
<DataSources>
  <DataSource Name="DS_Vendas">
    <ConnectionProperties>
      <DataProvider>SQL</DataProvider>
      <ConnectString>
        Data Source=meu-rds.xxxx.sa-east-1.rds.amazonaws.com;
        Initial Catalog=MeuBanco;
        User ID=ro_relatorio;
        Password=SenhaAqui
      </ConnectString>
    </ConnectionProperties>
  </DataSource>
</DataSources>
```

> **Boas práticas:** em produção, use **Shared Data Source** (conexão compartilhada no servidor), não embuta a connection string no `.rdl`. Assim, se o servidor mudar, você altera em um lugar só.

---

### Dataset — A Query de Dados

Cada dataset é uma query que retorna dados para o relatório. Um relatório pode ter vários datasets.

```xml
<DataSets>
  <DataSet Name="DS_Pedidos">
    <Query>
      <DataSourceName>DS_Vendas</DataSourceName>
      <CommandText>
        SELECT
            p.PedidoID,
            p.CodigoPedido,
            c.NomeCompleto AS Cliente,
            p.ValorTotal,
            p.DataPedido
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
    <Fields>
      <Field Name="PedidoID">    <DataField>PedidoID</DataField>    </Field>
      <Field Name="CodigoPedido"><DataField>CodigoPedido</DataField></Field>
      <Field Name="Cliente">     <DataField>Cliente</DataField>     </Field>
      <Field Name="ValorTotal">  <DataField>ValorTotal</DataField>  </Field>
      <Field Name="DataPedido">  <DataField>DataPedido</DataField>  </Field>
    </Fields>
  </DataSet>
</DataSets>
```

---

### ReportParameter — Parâmetros do Usuário

Filtros que o usuário preenche ao executar o relatório (datas, status, filial, etc.).

```xml
<ReportParameters>
  <ReportParameter Name="DataInicio">
    <DataType>DateTime</DataType>
    <DefaultValue>
      <Values><Value>=DateAdd("d", -30, Today())</Value></Values>
    </DefaultValue>
    <Prompt>Data Início</Prompt>
  </ReportParameter>

  <ReportParameter Name="DataFim">
    <DataType>DateTime</DataType>
    <DefaultValue>
      <Values><Value>=Today()</Value></Values>
    </DefaultValue>
    <Prompt>Data Fim</Prompt>
  </ReportParameter>
</ReportParameters>
```

---

## 4. Objetos Visuais (ReportItems)

Dentro do `<Body>` ficam todos os elementos visuais:

| Objeto | Tag XML | Para que serve |
|---|---|---|
| **Tablix** | `<Tablix>` | Tabelas, matrizes e listas — o mais usado |
| **Textbox** | `<Textbox>` | Texto fixo ou expressão isolada |
| **Image** | `<Image>` | Imagem estática ou dinâmica (logo, foto) |
| **Chart** | `<Chart>` | Gráficos (barras, linhas, pizza, etc.) |
| **Subreport** | `<Subreport>` | Relatório dentro de relatório |
| **Line** | `<Line>` | Linha decorativa |
| **Rectangle** | `<Rectangle>` | Contêiner / agrupamento visual |

### Tablix — A estrutura mais usada

O Tablix é o coração do Report Builder. Ele representa tabelas simples, matrizes cruzadas e listas:

```xml
<Tablix Name="tblPedidos">
  <TablixBody>
    <TablixColumns>
      <TablixColumn><Width>3cm</Width></TablixColumn>
      <TablixColumn><Width>5cm</Width></TablixColumn>
      <TablixColumn><Width>3cm</Width></TablixColumn>
    </TablixColumns>
    <TablixRows>
      <!-- Linha de cabeçalho -->
      <TablixRow>
        <TablixCells>
          <TablixCell><CellContents>
            <Textbox Name="hdrCodigo">
              <Paragraphs><Paragraph><TextRuns><TextRun>
                <Value>Código</Value>
                <Style><FontWeight>Bold</FontWeight></Style>
              </TextRun></TextRuns></Paragraph></Paragraphs>
            </Textbox>
          </CellContents></TablixCell>
          <!-- ... mais células -->
        </TablixCells>
      </TablixRow>
      <!-- Linha de dados (se repete para cada registro) -->
      <TablixRow>
        <TablixCells>
          <TablixCell><CellContents>
            <Textbox Name="txtCodigo">
              <Paragraphs><Paragraph><TextRuns><TextRun>
                <Value>=Fields!CodigoPedido.Value</Value>
              </TextRun></TextRuns></Paragraph></Paragraphs>
            </Textbox>
          </CellContents></TablixCell>
        </TablixCells>
      </TablixRow>
    </TablixRows>
  </TablixBody>
  <DataSetName>DS_Pedidos</DataSetName>
</Tablix>
```

---

## 5. Tipos de Dados nos Parâmetros

| DataType RDL | Tipo equivalente | Exemplo de valor |
|---|---|---|
| `String` | Texto | `"São Paulo"` |
| `Integer` | Número inteiro | `42` |
| `Float` | Decimal | `3.14` |
| `DateTime` | Data e hora | `#2024-01-15#` |
| `Boolean` | Verdadeiro/Falso | `True` |

---

## 6. Fluxo de Funcionamento

```
Usuário preenche parâmetros
        ↓
Report Builder monta a query com os parâmetros
        ↓
Query é enviada ao SQL Server (RDS ou padrão)
        ↓
SQL Server retorna o dataset
        ↓
Report Builder aplica expressões, formatação e grupos
        ↓
Relatório é renderizado (PDF, Excel, HTML, Word)
```

---

## 7. Formatos de Exportação Disponíveis

| Formato | Extensão | Quando usar |
|---|---|---|
| PDF | `.pdf` | Impressão, envio por e-mail |
| Excel | `.xlsx` | Análise posterior, tabelas dinâmicas |
| Word | `.docx` | Documentos formais |
| CSV | `.csv` | Integração com outros sistemas |
| XML | `.xml` | Integração, dados estruturados |
| HTML | `.html` | Visualização web |
| TIFF | `.tif` | Arquivamento de imagem |

---

## 8. Extensões de Arquivo

| Extensão | Descrição |
|---|---|
| `.rdl` | Report Definition — relatório completo |
| `.rds` | Report Data Source — fonte de dados compartilhada |
| `.rsd` | Report Shared Dataset — dataset compartilhado |
| `.rdlc` | RDL Client — relatório para uso local (sem servidor SSRS) |
