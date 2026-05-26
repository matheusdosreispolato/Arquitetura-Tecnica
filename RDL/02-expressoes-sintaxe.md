# RDL — Expressões e Sintaxe

> Expressões no Report Builder são escritas em **VB.NET** (Visual Basic .NET) e sempre começam com `=`. Elas são usadas em valores de campos, formatações, visibilidade, cores e praticamente qualquer propriedade de um objeto.

---

## 1. Regra de Ouro

```
= <expressão VB.NET>
```

- Começa com `=`
- É VB.NET, não C# (use `&` para concatenar strings, não `+`)
- Aspas duplas dentro da expressão: use duas aspas `""` para representar uma `"`
- Sem ponto-e-vírgula no final

```vb
' ✅ Correto
="Olá, " & Fields!NomeCompleto.Value

' ❌ Errado (C# style)
="Olá, " + Fields!NomeCompleto.Value
```

---

## 2. Referências Fundamentais

### Campos do Dataset

```vb
=Fields!NomeDocampo.Value

' Exemplos
=Fields!NomeCompleto.Value
=Fields!ValorTotal.Value
=Fields!DataPedido.Value
=Fields!CodigoPedido.Value

' Acessar campo de um dataset específico (quando há múltiplos)
=Fields!ValorTotal.Value, "DS_Pedidos"
```

### Parâmetros do Relatório

```vb
=Parameters!NomeParametro.Value

' Exemplos
=Parameters!DataInicio.Value
=Parameters!Estado.Value

' Parâmetro multi-valor: juntar em string
=Join(Parameters!Estados.Value, ", ")

' Verificar se parâmetro é nulo
=IIF(Parameters!Filtro.Value Is Nothing, "Todos", Parameters!Filtro.Value)
```

### Variáveis de Relatório / Grupo

```vb
=Variables!MinhaVariavel.Value

' Globals (informações do próprio relatório)
=Globals!ReportName        ' Nome do relatório
=Globals!PageNumber        ' Número da página atual
=Globals!TotalPages        ' Total de páginas
=Globals!ExecutionTime     ' Data/hora de execução
=Globals!ReportServerUrl   ' URL do servidor
```

---

## 3. Operadores

### Comparação

```vb
=  igual
<> diferente
>  maior que
<  menor que
>= maior ou igual
<= menor ou igual
```

### Lógicos

```vb
And    ' E lógico
Or     ' OU lógico
Not    ' Negação
AndAlso  ' E lógico com curto-circuito (mais eficiente)
OrElse   ' OU lógico com curto-circuito
```

### Concatenação de strings

```vb
' Use & para strings (nunca + como em C#)
="Pedido: " & Fields!CodigoPedido.Value & " - " & Fields!NomeCompleto.Value

' Tratar NULL na concatenação
="Pedido: " & IIF(IsNothing(Fields!CodigoPedido.Value), "", Fields!CodigoPedido.Value)
```

---

## 4. Condicionais

### IIF — If Inline (equivalente ao IF ternário)

```vb
' Sintaxe: IIF(condicao, valor_se_verdadeiro, valor_se_falso)

=IIF(Fields!Status.Value = 1, "Aberto", "Fechado")

=IIF(Fields!ValorTotal.Value > 1000, "Alto", "Normal")

=IIF(Fields!Ativo.Value = True, "Ativo", "Inativo")

' Aninhado (evite mais de 2 níveis — use Switch)
=IIF(Fields!Status.Value = 1, "Aberto",
  IIF(Fields!Status.Value = 2, "Faturado",
    IIF(Fields!Status.Value = 3, "Cancelado", "Desconhecido")))
```

### Switch — Melhor para múltiplas condições

```vb
' Sintaxe: Switch(cond1, val1, cond2, val2, ..., True, valPadrao)
' O "True" no final é o caso padrão (else)

=Switch(
    Fields!Status.Value = 1, "Aberto",
    Fields!Status.Value = 2, "Faturado",
    Fields!Status.Value = 3, "Cancelado",
    Fields!Status.Value = 4, "Devolvido",
    True,                    "Desconhecido"
)

' Classificação por faixa de valor
=Switch(
    Fields!ValorTotal.Value > 10000, "Premium",
    Fields!ValorTotal.Value > 5000,  "Ouro",
    Fields!ValorTotal.Value > 1000,  "Prata",
    True,                            "Standard"
)
```

### Choose — Selecionar por índice numérico

```vb
' Sintaxe: Choose(indice, opcao1, opcao2, opcao3, ...)
' indice começa em 1

=Choose(Fields!DiaSemana.Value, "Dom","Seg","Ter","Qua","Qui","Sex","Sáb")
```

---

## 5. Funções de Texto

```vb
' Comprimento
=Len(Fields!NomeCompleto.Value)

' Maiúsculo / Minúsculo
=UCase(Fields!Nome.Value)     ' NOME COMPLETO
=LCase(Fields!Nome.Value)     ' nome completo

' Primeira letra maiúscula (não nativo — use StrConv)
=StrConv(Fields!Nome.Value, vbProperCase)  ' Nome Completo

' Recortar string
=Left(Fields!CodigoPedido.Value, 3)          ' 3 primeiros chars
=Right(Fields!CodigoPedido.Value, 4)         ' 4 últimos chars
=Mid(Fields!CodigoPedido.Value, 2, 5)        ' do char 2, pega 5

' Remover espaços
=Trim(Fields!Nome.Value)      ' espaços das duas pontas
=LTrim(Fields!Nome.Value)     ' espaço do início
=RTrim(Fields!Nome.Value)     ' espaço do fim

' Substituir
=Replace(Fields!Telefone.Value, "-", "")     ' remove hífens
=Replace(Fields!CPF.Value, ".", "")          ' remove pontos

' Verificar se contém
=InStr(Fields!Email.Value, "@") > 0          ' True se tem @

' Posição de um trecho
=InStr(Fields!Texto.Value, "palavra")        ' posição da ocorrência

' Converter para string
=CStr(Fields!ValorTotal.Value)
=CStr(Fields!PedidoID.Value)

' Verificar nulo ou vazio
=IsNothing(Fields!Observacao.Value)          ' True se NULL
=Fields!Observacao.Value = ""                ' True se vazio
=String.IsNullOrEmpty(Fields!Obs.Value)      ' True se NULL ou ""
```

---

## 6. Funções de Data e Hora

```vb
' Data e hora atual
=Now()          ' data + hora atual
=Today()        ' somente data atual
=Globals!ExecutionTime  ' momento da execução do relatório

' Extrair partes
=Year(Fields!DataPedido.Value)         ' 2024
=Month(Fields!DataPedido.Value)        ' 6
=Day(Fields!DataPedido.Value)          ' 15
=Hour(Fields!DataHora.Value)           ' 14
=Minute(Fields!DataHora.Value)         ' 30
=Weekday(Fields!DataPedido.Value)      ' 1=Domingo ... 7=Sábado

' Nome do dia/mês
=WeekdayName(Weekday(Fields!DataPedido.Value))       ' "Segunda-feira"
=MonthName(Month(Fields!DataPedido.Value))            ' "Junho"

' Adicionar tempo
=DateAdd("d",  7,  Fields!DataPedido.Value)   ' +7 dias
=DateAdd("m", -1,  Today())                   ' -1 mês
=DateAdd("yyyy", 1, Today())                  ' +1 ano
=DateAdd("h",  8,  Fields!DataHora.Value)     ' +8 horas

' Diferença entre datas
=DateDiff("d",  Fields!DataPedido.Value, Today())   ' diferença em dias
=DateDiff("m",  Fields!DataCadastro.Value, Today()) ' em meses
=DateDiff("yyyy", Fields!DataNasc.Value, Today())   ' em anos (idade)

' Formatar data
=Format(Fields!DataPedido.Value, "dd/MM/yyyy")           ' 15/06/2024
=Format(Fields!DataPedido.Value, "dd/MM/yyyy HH:mm")     ' 15/06/2024 14:30
=Format(Fields!DataPedido.Value, "MMMM yyyy")            ' Junho 2024
=Format(Fields!DataPedido.Value, "dd 'de' MMMM 'de' yyyy") ' 15 de Junho de 2024

' Primeiro e último dia do mês
=DateSerial(Year(Today()), Month(Today()), 1)             ' primeiro dia do mês
=DateSerial(Year(Today()), Month(Today()) + 1, 0)         ' último dia do mês
```

---

## 7. Funções Numéricas e de Formatação

```vb
' Formatação de números
=Format(Fields!ValorTotal.Value, "N2")          ' 1.234,56
=Format(Fields!ValorTotal.Value, "C2")          ' R$ 1.234,56  (moeda da cultura)
=Format(Fields!Percentual.Value, "P1")          ' 12,5%
=Format(Fields!Quantidade.Value, "N0")          ' 1.234 (sem decimais)

' Arredondamento
=Round(Fields!ValorTotal.Value, 2)              ' 2 casas decimais
=Math.Floor(Fields!Valor.Value)                 ' arredonda para baixo
=Math.Ceiling(Fields!Valor.Value)               ' arredonda para cima

' Valor absoluto
=Math.Abs(Fields!Variacao.Value)

' Mínimo e máximo entre dois valores
=IIF(Fields!ValorA.Value > Fields!ValorB.Value, Fields!ValorA.Value, Fields!ValorB.Value)

' Tratar divisão por zero
=IIF(Fields!Denominador.Value = 0, 0, Fields!Numerador.Value / Fields!Denominador.Value)

' Converter para número
=CDbl(Fields!ValorTexto.Value)    ' para Double
=CInt(Fields!IdTexto.Value)       ' para Integer
=CDec(Fields!ValorTexto.Value)    ' para Decimal
```

---

## 8. Funções de Agregação (usadas em totais e grupos)

```vb
' Soma
=Sum(Fields!ValorTotal.Value)
=Sum(Fields!ValorTotal.Value, "DS_Pedidos")    ' de um dataset específico

' Contagem
=Count(Fields!PedidoID.Value)
=CountDistinct(Fields!ClienteID.Value)         ' conta valores únicos

' Média
=Avg(Fields!ValorTotal.Value)

' Máximo e mínimo
=Max(Fields!ValorTotal.Value)
=Min(Fields!DataPedido.Value)

' Primeiro e último valor do grupo
=First(Fields!NomeCompleto.Value)
=Last(Fields!NomeCompleto.Value)

' Número da linha no grupo
=RowNumber(Nothing)              ' número da linha no relatório inteiro
=RowNumber("NomeGrupo")          ' número dentro do grupo

' Percentual sobre total
=Fields!ValorTotal.Value / Sum(Fields!ValorTotal.Value) * 100

' Percentual formatado
=Format(Fields!ValorTotal.Value / Sum(Fields!ValorTotal.Value), "P1")
```

---

## 9. Expressões de Cor e Formatação Condicional

```vb
' Cor de fundo por status
=Switch(
    Fields!Status.Value = 1, "LightBlue",    ' Aberto
    Fields!Status.Value = 2, "LightGreen",   ' Faturado
    Fields!Status.Value = 3, "LightCoral",   ' Cancelado
    True,                    "White"
)

' Cor de texto por valor
=IIF(Fields!ValorTotal.Value < 0, "Red", "Black")

' Cor de linha alternada (zebra)
=IIF(RowNumber(Nothing) Mod 2 = 0, "WhiteSmoke", "White")

' Negrito condicional
=IIF(Fields!Status.Value = 3, "Bold", "Normal")

' Visibilidade condicional (ocultar/mostrar linha)
=IIF(Fields!Ativo.Value = False, True, False)
' True = oculto, False = visível
```

---

## 10. Expressões Comuns no Cabeçalho e Rodapé

```vb
' Número de página
="Página " & Globals!PageNumber & " de " & Globals!TotalPages

' Data de emissão
="Emitido em: " & Format(Globals!ExecutionTime, "dd/MM/yyyy HH:mm")

' Nome do relatório
=Globals!ReportName

' Parâmetros usados na impressão
="Período: " & Format(Parameters!DataInicio.Value,"dd/MM/yyyy") &
 " a " & Format(Parameters!DataFim.Value,"dd/MM/yyyy")

' Usuário que gerou
=User!UserID
```

---

## 11. Armadilhas Comuns

### NULL em expressões

```vb
' ❌ Pode gerar erro se o campo for NULL
=Fields!Observacao.Value & " - detalhes"

' ✅ Tratar NULL antes
=IIF(IsNothing(Fields!Observacao.Value), "", Fields!Observacao.Value) & " - detalhes"

' ✅ Usando função personalizada com IIF
=IIF(IsNothing(Fields!Valor.Value), 0, Fields!Valor.Value)
```

### Divisão por zero

```vb
' ❌ Erro quando denominador é 0
=Fields!Vendas.Value / Fields!Meta.Value

' ✅ Proteger
=IIF(Fields!Meta.Value = 0, 0, Fields!Vendas.Value / Fields!Meta.Value)
```

### Tipos incompatíveis

```vb
' ❌ Somar string com número
=Fields!ValorTexto.Value + 10

' ✅ Converter explicitamente
=CDbl(Fields!ValorTexto.Value) + 10
```

### Aspas dentro de strings

```vb
' Para colocar aspas dentro de uma string, use duas aspas seguidas
="O campo ""Nome"" é obrigatório"
' Resultado: O campo "Nome" é obrigatório
```
