# Camada Gold — Modelos Analíticos e Consumo

> A Gold é a camada que **responde às perguntas do negócio**. Os dados chegam limpos da Silver e são organizados para análise: agregados, modelados em dimensões e fatos, ou expostos como views com vocabulário de negócio.

---

## O papel da Gold

Se a Silver é o estoque de peças prontas, a Gold é a vitrine — organizada, rotulada com termos que o cliente entende, e montada para responder perguntas específicas.

Quem consome a Gold são **analistas, ferramentas de BI, relatórios e aplicações**. Eles não precisam saber como os dados foram limpos ou de onde vieram — só precisam que estejam corretos e bem organizados.

---

## Princípios

| Princípio | Descrição |
|---|---|
| **Vocabulário de negócio** | Colunas e tabelas com nomes que o negócio entende, não nomes técnicos |
| **Sempre parte da Silver** | A Gold nunca lê diretamente da Bronze |
| **Otimizada para leitura** | Índices, pré-agregações e modelos pensados para consulta, não para escrita |
| **Reprocessável** | A Gold pode ser recriada a partir da Silver sem perda de informação |
| **Sem lógica de limpeza** | Qualquer transformação de qualidade pertence à Silver |

---

## Dois modelos de organização

### Modelo Dimensional (Estrela)
Indicado quando os dados vão alimentar ferramentas de BI com análises multidimensionais — "vendas por produto, por região, por mês".

```
            dim_Tempo
                │
dim_Cliente ────┼──── fato_Venda
                │
           dim_Produto
```

- **Tabelas de dimensão** (`dim_`): descrevem o *quem*, *onde*, *quando*, *o quê* — Cliente, Produto, Tempo, Loja
- **Tabelas de fato** (`fato_`): armazenam os eventos com métricas numéricas — Venda, Transação, Acesso

### Tabelas Agregadas por Tema
Indicado quando o objetivo é responder perguntas específicas de forma rápida, sem um modelo dimensional completo.

```
gold.VendasMensais       → total por mês
gold.RankingProdutos     → produtos mais vendidos
gold.ClientesAtivos      → clientes com compra nos últimos 90 dias
```

Os dois modelos podem coexistir na mesma camada Gold.

---

## Estrutura típica — Modelo Dimensional

**Dimensão Tempo**
```
gold.dim_Tempo
├── TempoID       (chave: formato YYYYMMDD, ex: 20240615)
├── Data
├── Ano
├── Trimestre
├── Mes / NomeMes
├── Semana
├── DiaSemana / NomeDiaSemana
├── IsFinDeSemana
└── IsFeriado
```

**Dimensão Cliente**
```
gold.dim_Cliente
├── ClienteKey    (chave técnica da dimensão)
├── ClienteID     (chave natural — ID do sistema de origem)
├── Nome
├── Segmento
├── UF / Cidade
├── DataPrimeiraCompra
└── is_ativo
```

**Tabela Fato**
```
gold.fato_Venda
├── VendaKey       (chave técnica)
├── TempoKey       → referência para dim_Tempo
├── ClienteKey     → referência para dim_Cliente
├── PedidoID       (chave degenerada — não tem dimensão própria)
├── Quantidade
├── ValorUnitario
├── ValorTotal
├── Desconto
└── ValorLiquido   (calculado: ValorTotal - Desconto)
```

---

## Views de Consumo

Views na Gold encapsulam a lógica de análise e expõem apenas o que o consumidor precisa, com nomes de negócio:

| View | Responde |
|---|---|
| `gold.vw_VendasMensais` | Quanto vendemos por mês? |
| `gold.vw_RankingClientes` | Quais clientes geraram mais receita? |
| `gold.vw_ClientesInativos` | Quais clientes não compram há mais de 90 dias? |
| `gold.vw_TicketMedioPorCategoria` | Qual o ticket médio por categoria de produto? |

---

## Tabelas Materializadas

Quando uma view é muito consultada ou lenta para recalcular em tempo real, os resultados podem ser materializados (gravados em tabela física) e atualizados periodicamente pelo pipeline.

```
Execução do pipeline
        ↓
usp_Gold_AtualizarVendasMensais
        ↓
Trunca gold.VendasMensais
        ↓
Recarrega com dados atualizados da Silver
        ↓
Disponível para consulta imediata
```

**Quando materializar:**
- A query Silver é pesada e roda muitas vezes por dia
- O consumidor precisa de resposta em milissegundos
- O dado não precisa ser em tempo real — uma atualização por hora ou por dia é suficiente

---

## Consultas analíticas típicas

Com a Gold bem modelada, perguntas de negócio se tornam simples:

```
"Qual foi o crescimento de receita mês a mês?"
→ Consultar gold.VendasMensais, calcular variação percentual entre meses

"Quem são os top 10 clientes do ano?"
→ Consultar gold.vw_RankingClientes, filtrar por ano, ordenar por receita

"Qual a receita acumulada no ano (YTD)?"
→ Somar ReceitaLiquida em gold.VendasMensais onde Ano = ano atual
```

---

## Para implementação específica

- **SQL Server (MSSQL):** ver `MSSQL/04-avancado-performance.md` para Window Functions e CTEs usados em análises Gold, e `MSSQL/02-ddl-tabelas-indices.md` para criação de índices em tabelas fato
- **dbt:** Gold é ideal para modelos `table` ou `view` no dbt com materialização configurável
- **Power BI / SSRS:** conectar diretamente nas views e tabelas Gold — ver `RDL/` para relatórios SSRS
- **Agendamento:** consulte `Engenharia-Dados/06-airflow.md` para orquestrar Silver → Gold como Tasks dependentes
