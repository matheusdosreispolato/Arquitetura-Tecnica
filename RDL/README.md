# RDL — Report Builder / SSRS

> Documentação de sintaxe e boas práticas para criação de relatórios `.rdl` com o **Microsoft Report Builder** e **SQL Server Reporting Services (SSRS)**.

---

## Documentos

| # | Arquivo | Conteúdo |
|---|---|---|
| 01 | [Introdução e Estrutura](./01-introducao-estrutura.md) | O que é RDL, anatomia do XML, DataSource, Dataset, parâmetros, objetos visuais, tipos de exportação |
| 02 | [Expressões e Sintaxe](./02-expressoes-sintaxe.md) | Sintaxe VB.NET, IIF, Switch, funções de texto/data/número, agregações, armadilhas de NULL |
| 03 | [Datasets e Parâmetros](./03-datasets-parametros.md) | Conexões, queries, stored procedures, parâmetros em cascata, multi-valor, filtros |
| 04 | [Formatação, Grupos e Agregações](./04-formatacao-grupos-agregacoes.md) | Cores condicionais, visibilidade, grupos, subtotais, ranking, paginação, gráficos |

---

## Referência Rápida

### Sintaxe de expressão

```vb
' Sempre começa com =
' Linguagem: VB.NET (não C#)
' Concatenação: & (não +)

=Fields!NomeCampo.Value
=Parameters!NomeParam.Value
=Globals!PageNumber
```

### Condicionais

```vb
' Dois caminhos
=IIF(condicao, valor_verdadeiro, valor_falso)

' Múltiplos caminhos
=Switch(cond1, val1, cond2, val2, True, valPadrao)
```

### Datas

```vb
=Today()                                      ' data atual
=Format(Fields!Data.Value, "dd/MM/yyyy")      ' formatar
=DateAdd("m", -1, Today())                    ' subtrair 1 mês
=DateDiff("d", Fields!Inicio.Value, Today())  ' diferença em dias
```

### Números

```vb
=Format(Fields!Valor.Value, "R$ #,##0.00")   ' moeda
=Format(Fields!Perc.Value,  "0.0%")          ' percentual
=Round(Fields!Valor.Value, 2)                 ' arredondar
```

### Agregações

```vb
=Sum(Fields!Valor.Value)                      ' soma no escopo atual
=Sum(Fields!Valor.Value, "GrupoCliente")      ' soma no grupo
=Count(Fields!ID.Value)                       ' contagem
=CountDistinct(Fields!ClienteID.Value)        ' valores únicos
=Avg(Fields!Valor.Value)                      ' média
=Max(Fields!Data.Value)                       ' maior valor
```

### Formatação condicional

```vb
' Cor de fundo
=IIF(Fields!Valor.Value < 0, "LightCoral", "White")

' Zebra
=IIF(RowNumber(Nothing) Mod 2 = 0, "#F5F5F5", "White")

' Ocultar linha
=IIF(Fields!Valor.Value = 0, True, False)
```
