# Arquitetura Medallion — Conceitos e Fundamentos

> O padrão Medallion é uma forma de organizar dados em camadas progressivas de qualidade — do bruto ao refinado. É agnóstico de tecnologia: funciona em SQL Server, Databricks, BigQuery, Snowflake ou qualquer banco de dados.

---

## Para quem nunca ouviu falar: a analogia do ouro

Imagine uma mineradora. O processo começa com pedra bruta retirada do solo — suja, misturada, sem nenhum tratamento. Essa pedra passa por etapas de refinamento: primeiro é lavada e classificada, depois é processada para separar o mineral, e finalmente se transforma em ouro puro, pronto para ser usado.

A arquitetura Medallion funciona da mesma forma com dados:

```
Bronze  = pedra bruta retirada da mina     → dados como vieram da fonte, sem alteração
Silver  = mineral lavado e classificado    → dados limpos, validados, sem duplicatas
Gold    = ouro refinado, pronto para usar  → dados agregados, modelados para análise
```

Cada camada tem uma responsabilidade única. Você nunca joga fora a pedra bruta — ela é a prova de que tudo o que está nas camadas seguintes é derivado de uma fonte real.

---

## Os Três Níveis em Detalhe

### 🥉 Bronze — Raw (Bruto)

**O que é:** cópia exata dos dados como chegaram da fonte, sem nenhuma transformação.

**Regras:**
- Nunca alterar ou deletar registros nessa camada
- Sempre adicionar colunas de auditoria: quando foi carregado, de qual fonte, qual versão
- Guardar até dados com erro — o erro também é informação
- Estrutura pode refletir exatamente a estrutura da fonte de origem

**Exemplos do que entra aqui:**
- Arquivo CSV importado linha por linha, incluindo linhas com formato errado
- Resposta de uma API salva como JSON em uma coluna NVARCHAR
- Tabela de outro sistema copiada integralmente sem filtros

```sql
-- Exemplo de registro Bronze: dados brutos de pedido com coluna de auditoria
SELECT
    bronze_id,
    dados_origem_json,      -- dado original, sem transformar
    fonte_sistema,          -- de qual sistema veio
    dt_carga,               -- quando foi carregado
    hash_registro,          -- hash do conteúdo para detectar mudanças
    arquivo_origem          -- nome do arquivo/batch de origem
FROM bronze.Pedido_Raw;
```

---

### 🥈 Silver — Cleaned (Limpo)

**O que é:** dados do Bronze que foram validados, limpos e padronizados. Prontos para análise, mas ainda no nível de registro individual (não agregado).

**Regras:**
- Aplicar regras de qualidade: remover duplicatas, validar tipos, tratar nulos
- Padronizar formatos: datas no mesmo fuso, textos em maiúscula/minúscula consistente, CPF sem pontuação
- Não agregar dados — a granularidade é a mesma do Bronze
- Registrar o motivo de rejeição dos registros que falharam na validação

**Exemplos:**
- Pedidos com data válida, cliente identificado e valor positivo
- Clientes deduplicados por CPF, com endereço padronizado
- Transações com moeda convertida para a moeda base

```sql
-- Exemplo de registro Silver: pedido limpo e validado
SELECT
    PedidoID,
    ClienteID,
    CAST(dt_pedido AS DATE)             AS DataPedido,
    UPPER(TRIM(status))                 AS Status,
    ROUND(valor_total, 2)               AS ValorTotal,
    dt_processamento_silver             -- quando passou pela camada Silver
FROM silver.Pedido;
```

---

### 🥇 Gold — Business (Analítico)

**O que é:** dados do Silver modelados para responder perguntas de negócio. Podem estar agregados, sumarizados ou organizados em dimensões e fatos.

**Regras:**
- Otimizados para leitura e análise — não para escrita transacional
- Podem ser views, tabelas materializadas, cubos ou tabelas agregadas
- Nomes e colunas devem usar vocabulário do negócio, não da tecnologia
- São os dados que alimentam dashboards, relatórios e aplicações

**Exemplos:**
- Total de pedidos por mês, categoria e região
- Receita acumulada por cliente no último trimestre
- Rank dos produtos mais vendidos por loja

```sql
-- Exemplo de registro Gold: vendas mensais por categoria
SELECT
    AnoMes,
    Categoria,
    TotalPedidos,
    ReceitaTotal,
    TicketMedio,
    ClientesUnicos
FROM gold.vw_VendasMensaisPorCategoria
WHERE AnoMes >= '2024-01';
```

---

## Por que usar esse padrão?

| Problema sem Medallion | Como o Medallion resolve |
|---|---|
| Dado transformado diretamente da fonte — impossível rastrear erros | Bronze preserva o original, erros são rastreáveis |
| Pipeline quebra e perde dados já processados | Cada camada é reprocessável a partir da anterior |
| Relatório mostra número diferente do sistema | Gold vem de Silver validado, origem auditável |
| Mudança de regra de negócio exige reprocessar tudo | Reprocessa Silver e Gold sem tocar o Bronze |
| Analistas consultam tabelas operacionais e travam o sistema | Gold é separado do sistema transacional |

---

## Quando aplicar

**Aplique quando:**
- Dados chegam de múltiplas fontes com formatos diferentes
- É necessário rastrear a origem de um número em um relatório
- A equipe precisa de dados históricos confiáveis
- Há processos de ETL que precisam ser reexecutados sem risco

**Não é necessário quando:**
- O volume de dados é pequeno e vem de uma única fonte confiável
- O objetivo é apenas expor dados de um sistema operacional sem transformação
- A estrutura do sistema de origem já é o modelo analítico final

---

## Estrutura de Schemas Recomendada

```sql
-- Criar os schemas para cada camada
CREATE SCHEMA bronze;   -- dados brutos da fonte
CREATE SCHEMA silver;   -- dados limpos e validados
CREATE SCHEMA gold;     -- dados analíticos e agregados
CREATE SCHEMA pipeline; -- tabelas de controle de execução (logs, marcadores)
GO
```

---

## Fluxo Completo

```
┌────────────┐    ┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│  Fontes    │    │    BRONZE      │    │    SILVER      │    │     GOLD       │
│            │    │                │    │                │    │                │
│ - Sistemas │───▶│ Cópia exata    │───▶│ Limpo          │───▶│ Agregado       │
│ - Arquivos │    │ Sem alteração  │    │ Validado       │    │ Modelado       │
│ - APIs     │    │ Com auditoria  │    │ Sem duplicata  │    │ Para consumo   │
└────────────┘    └────────────────┘    └────────────────┘    └────────────────┘
                       ↓ sempre               ↓ reprocessável        ↓ otimizado
                    preservado              a partir do Bronze     para leitura
```

---

## Nomenclatura Padrão

| Objeto | Padrão | Exemplo |
|---|---|---|
| Schema das camadas | `bronze`, `silver`, `gold`, `pipeline` | `bronze.Pedido_Raw` |
| Tabela Bronze | `<Entidade>_Raw` | `bronze.Pedido_Raw` |
| Tabela Silver | `<Entidade>` (sem sufixo) | `silver.Pedido` |
| Tabela Gold | `<Entidade>` ou `fato_<Entidade>` / `dim_<Entidade>` | `gold.fato_Venda`, `gold.dim_Cliente` |
| View Gold | `vw_<Descricao>` | `gold.vw_VendasMensais` |
| Tabela de controle | `pipeline.Log_Execucao` | `pipeline.Log_Execucao` |
| Procedure de carga | `usp_<Camada>_Carregar<Entidade>` | `usp_Bronze_CarregarPedido` |
