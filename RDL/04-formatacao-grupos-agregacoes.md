# RDL — Formatação, Grupos e Agregações

> Como organizar dados em grupos, calcular totais e subtotais, aplicar formatação condicional e controlar visibilidade de elementos.

---

## 1. Formatação Condicional

Toda propriedade visual de um objeto pode receber uma expressão. Na interface do Report Builder, clique com botão direito na célula → **Propriedades da Caixa de Texto** → aba **Fonte** ou **Preenchimento**.

### Cor de fundo por condição

```vb
' Propriedade: BackgroundColor da célula

' Semáforo por status
=Switch(
    Fields!Status.Value = 1, "LightBlue",
    Fields!Status.Value = 2, "LightGreen",
    Fields!Status.Value = 3, "LightCoral",
    True, "White"
)

' Alerta por valor
=IIF(Fields!ValorTotal.Value < 0, "#FFCCCC", "Transparent")

' Linha zebrada (linhas alternadas em cinza claro)
=IIF(RowNumber(Nothing) Mod 2 = 0, "#F5F5F5", "White")
```

### Cor de texto

```vb
' Propriedade: Color do texto

=IIF(Fields!Variacao.Value < 0, "Red", "DarkGreen")

=Switch(
    Fields!Saldo.Value < 0,    "Red",
    Fields!Saldo.Value = 0,    "DimGray",
    Fields!Saldo.Value > 0,    "DarkGreen",
    True,                      "Black"
)
```

### Negrito e Itálico

```vb
' Propriedade: FontWeight
=IIF(Fields!Status.Value = 3, "Bold", "Normal")

' Propriedade: FontStyle
=IIF(Fields!Excluido.Value = True, "Italic", "Normal")
```

### Decoração de texto (tachado para itens cancelados)

```vb
' Propriedade: TextDecoration
=IIF(Fields!Status.Value = 3, "LineThrough", "None")
' Opções: None, Underline, Overline, LineThrough
```

---

## 2. Visibilidade Condicional

### Ocultar linha completa

```xml
<!-- Propriedade Hidden de uma TablixRow -->
<Visibility>
  <Hidden>=IIF(Fields!Ativo.Value = False, True, False)</Hidden>
</Visibility>

<!-- Ocultar linhas com valor zero -->
<Visibility>
  <Hidden>=IIF(Fields!ValorTotal.Value = 0, True, False)</Hidden>
</Visibility>
```

### Mostrar/Ocultar coluna inteira via parâmetro

```xml
<!-- A coluna "CPF" só aparece se o parâmetro MostrarCPF = True -->
<Visibility>
  <Hidden>=NOT Parameters!MostrarCPF.Value</Hidden>
</Visibility>
```

### Seção expansível (drill-down) — clique para expandir

```xml
<!-- A linha de detalhe começa oculta e expande ao clicar no grupo pai -->
<Visibility>
  <Hidden>true</Hidden>
  <ToggleItem>txtNomeGrupo</ToggleItem>  <!-- ID do objeto que controla o toggle -->
</Visibility>
```

---

## 3. Grupos no Tablix

Grupos organizam os dados por uma coluna (ex: por cliente, por mês, por estado) e permitem calcular subtotais.

### Estrutura visual de um Tablix com grupo

```
┌──────────────────────────────────────────────────────┐
│  CABEÇALHO DO RELATÓRIO (fora do Tablix)             │
├──────────────────────────────────────────────────────┤
│  CABEÇALHO DO TABLIX (linha fixa — nomes de colunas) │
├──────────────────────────────────────────────────────┤
│  CABEÇALHO DO GRUPO (ex: nome do cliente)            │
│    LINHA DE DETALHE (se repete por registro)         │
│    LINHA DE DETALHE                                  │
│  RODAPÉ DO GRUPO (subtotal do cliente)               │
├──────────────────────────────────────────────────────┤
│  CABEÇALHO DO GRUPO (próximo cliente)                │
│    LINHA DE DETALHE                                  │
│  RODAPÉ DO GRUPO                                     │
├──────────────────────────────────────────────────────┤
│  RODAPÉ DO TABLIX (total geral)                      │
└──────────────────────────────────────────────────────┘
```

### Grupo definido no XML

```xml
<TablixRowHierarchy>
  <TablixMembers>
    <!-- Grupo por Cliente -->
    <TablixMember>
      <Group Name="GrupoCliente">
        <GroupExpressions>
          <GroupExpression>=Fields!ClienteID.Value</GroupExpression>
        </GroupExpressions>
      </Group>
      <SortExpressions>
        <SortExpression>
          <Value>=Fields!NomeCompleto.Value</Value>
          <Direction>Ascending</Direction>
        </SortExpression>
      </SortExpressions>
      <TablixMembers>
        <!-- Linha de detalhe (dentro do grupo) -->
        <TablixMember>
          <DataGroupName>GrupoCliente</DataGroupName>
        </TablixMember>
      </TablixMembers>
    </TablixMember>
  </TablixMembers>
</TablixRowHierarchy>
```

### Múltiplos níveis de agrupamento

```
Grupo Nível 1: Estado
  Grupo Nível 2: Cliente
    Linha de detalhe: Pedido
  Subtotal Cliente
Subtotal Estado
Total Geral
```

```xml
<Group Name="GrupoEstado">
  <GroupExpressions>
    <GroupExpression>=Fields!Estado.Value</GroupExpression>
  </GroupExpressions>
  <!-- Dentro deste grupo, há outro grupo -->
</Group>
```

---

## 4. Agregações por Escopo

O escopo define **onde** a função de agregação conta/soma.

```vb
' Sem escopo → agrega no escopo atual (grupo ou tabela inteira)
=Sum(Fields!ValorTotal.Value)

' Com escopo de dataset → soma todos os registros do dataset
=Sum(Fields!ValorTotal.Value, "DS_Pedidos")

' Com escopo de grupo → soma dentro do grupo especificado
=Sum(Fields!ValorTotal.Value, "GrupoCliente")
=Sum(Fields!ValorTotal.Value, "GrupoEstado")

' Percentual do item sobre o total do grupo pai
=Fields!ValorTotal.Value / Sum(Fields!ValorTotal.Value, "GrupoCliente")

' Percentual do item sobre o total geral
=Fields!ValorTotal.Value / Sum(Fields!ValorTotal.Value, "DS_Pedidos")
```

### Exemplos práticos de agregação

```vb
' Total de pedidos
=Count(Fields!PedidoID.Value)

' Clientes únicos
=CountDistinct(Fields!ClienteID.Value)

' Ticket médio
=Sum(Fields!ValorTotal.Value) / Count(Fields!PedidoID.Value)

' Ticket médio protegido contra divisão por zero
=IIF(Count(Fields!PedidoID.Value) = 0, 0,
     Sum(Fields!ValorTotal.Value) / Count(Fields!PedidoID.Value))

' Maior pedido do grupo
=Max(Fields!ValorTotal.Value)

' Data do pedido mais recente
=Max(Fields!DataPedido.Value)

' Percentual de cancelados
=Count(IIF(Fields!Status.Value = 3, 1, Nothing)) /
 Count(Fields!PedidoID.Value) * 100
```

---

## 5. Ordenação

### Ordenação simples no Tablix

```xml
<SortExpressions>
  <SortExpression>
    <Value>=Fields!DataPedido.Value</Value>
    <Direction>Descending</Direction>   <!-- Ascending ou Descending -->
  </SortExpression>
  <SortExpression>
    <Value>=Fields!NomeCompleto.Value</Value>
    <Direction>Ascending</Direction>
  </SortExpression>
</SortExpressions>
```

### Ordenação interativa (usuário clica no cabeçalho)

```xml
<!-- Na célula de cabeçalho da coluna -->
<Textbox Name="hdrValorTotal">
  <UserSort>
    <SortExpression>=Fields!ValorTotal.Value</SortExpression>
    <SortExpressionScope>DS_Pedidos</SortExpressionScope>
  </UserSort>
</Textbox>
```

---

## 6. Numeração de Linhas e Ranking

```vb
' Número da linha no relatório inteiro (reinicia a cada execução)
=RowNumber(Nothing)

' Número da linha dentro do grupo atual
=RowNumber("GrupoCliente")

' Ranking por valor (maior primeiro)
=Rank(Fields!ValorTotal.Value, "DS_Pedidos", True)
' True = decrescente (maior = rank 1)
' False = crescente (menor = rank 1)
```

---

## 7. Paginação

### Quebra de página por grupo

```xml
<!-- Adicionar quebra de página no início de cada grupo -->
<Group Name="GrupoCliente">
  <PageBreak>
    <BreakLocation>Start</BreakLocation>
    <!-- Opções: Start, End, StartAndEnd, Between, None -->
  </PageBreak>
</Group>
```

### Reiniciar numeração de página por grupo

```xml
<Group Name="GrupoCliente">
  <NewPage>Next</NewPage>         <!-- pula para próxima página -->
  <ResetPageNumber>true</ResetPageNumber>  <!-- reinicia contagem -->
</Group>
```

### Manter grupo junto (evita quebra no meio do grupo)

```xml
<KeepTogether>true</KeepTogether>
```

---

## 8. Expressões de Formatação de Valor

```vb
' Moeda em Real (usando Format)
=Format(Fields!ValorTotal.Value, "R$ #,##0.00")
' Resultado: R$ 1.234,56

' Número com separador de milhar
=Format(Fields!Quantidade.Value, "#,##0")
' Resultado: 1.234

' Porcentagem
=Format(Fields!Percentual.Value / 100, "0.0%")
' Resultado: 15,3%

' Mostrar zero como traço
=IIF(Fields!ValorTotal.Value = 0, "-",
     Format(Fields!ValorTotal.Value, "R$ #,##0.00"))

' Mostrar NULL como "N/A"
=IIF(IsNothing(Fields!Observacao.Value), "N/A", Fields!Observacao.Value)
```

---

## 9. Imagem Dinâmica (logo por empresa)

```xml
<Image Name="imgLogo">
  <Source>Database</Source>  <!-- External, Embedded, Database -->
  <Value>=Fields!LogoEmpresa.Value</Value>
  <MIMEType>image/png</MIMEType>
  <Sizing>FitProportional</Sizing>
</Image>
```

Para logo externa (URL):

```xml
<Image Name="imgLogo">
  <Source>External</Source>
  <Value>="https://meu-bucket.s3.sa-east-1.amazonaws.com/logo.png"</Value>
  <Sizing>FitProportional</Sizing>
</Image>
```

---

## 10. Gráfico Simples — Configuração no XML

```xml
<Chart Name="GraficoVendasMes">
  <DataSetName>DS_VendasPorMes</DataSetName>
  <ChartSeriesCollection>
    <ChartSeries Name="SerieVendas">
      <ChartDataPoints>
        <ChartDataPoint>
          <ChartDataPointValues>
            <Y>=Sum(Fields!ValorTotal.Value)</Y>
          </ChartDataPointValues>
        </ChartDataPoint>
      </ChartDataPoints>
    </ChartSeries>
  </ChartSeriesCollection>
  <ChartCategoryHierarchy>
    <ChartMembers>
      <ChartMember>
        <Group Name="GrupoMes">
          <GroupExpressions>
            <GroupExpression>=Fields!MesAno.Value</GroupExpression>
          </GroupExpressions>
        </Group>
      </ChartMember>
    </ChartMembers>
  </ChartCategoryHierarchy>
</Chart>
```

---

## 11. Tabela de Referência Rápida — Expressões de Formatação

| Objetivo | Expressão |
|---|---|
| Cor zebrada | `=IIF(RowNumber(Nothing) Mod 2=0, "#F5F5F5","White")` |
| Ocultar linha zero | `=IIF(Fields!Valor.Value=0, True, False)` |
| Negrito no total | `=IIF(Fields!Tipo.Value="TOTAL","Bold","Normal")` |
| Status com cor | `=Switch(Fields!S.Value=1,"LightBlue", Fields!S.Value=2,"LightGreen",True,"White")` |
| Percentual do total | `=Fields!V.Value / Sum(Fields!V.Value,"DS") * 100` |
| Número de página | `="Pág. " & Globals!PageNumber & "/" & Globals!TotalPages` |
| Data de emissão | `=Format(Globals!ExecutionTime,"dd/MM/yyyy HH:mm")` |
| NULL como traço | `=IIF(IsNothing(Fields!V.Value),"-",Format(Fields!V.Value,"N2"))` |
| Diferença de dias | `=DateDiff("d",Fields!DataInicio.Value,Fields!DataFim.Value)` |
