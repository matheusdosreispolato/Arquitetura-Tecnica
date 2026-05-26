# Engenharia de Dados — Arquitetura Medallion

> Documentação sobre o padrão **Camada Medallion** (Bronze → Silver → Gold) para construção de pipelines de dados robustos, rastreáveis e evolutivos. Os conceitos são agnósticos de plataforma; os exemplos de código usam T-SQL e PowerShell.

---

## O que é a Arquitetura Medallion

A arquitetura medallion organiza os dados em três camadas progressivas de qualidade, cada uma com um propósito claro:

```
Fontes          Bronze          Silver          Gold
─────────  →  ──────────  →  ──────────  →  ──────────
Sistemas        Raw / Bruto     Limpo /         Agregado /
de origem       Sem alteração   Validado        Pronto p/ consumo
```

---

## Estrutura desta pasta

```
Engenharia-Dados/
├── README.md                   ← este arquivo
├── 01-conceitos-medallion.md   ← O que é, por que usar, analogias, quando aplicar
├── 02-camada-bronze.md         ← Ingestão raw, schemas, auditoria, versionamento
├── 03-camada-silver.md         ← Limpeza, transformação, qualidade de dados
├── 04-camada-gold.md           ← Agregações, modelos analíticos, consumo
└── 05-pipeline-orquestracao.md ← Pipelines, idempotência, agendamento, logging
```

---

## Documentos

| Arquivo | Conteúdo |
|---|---|
| [01 — Conceitos e Arquitetura](./01-conceitos-medallion.md) | O que é o padrão medallion, por que usar, diferença entre as camadas, quando aplicar, analogias para leigos |
| [02 — Camada Bronze](./02-camada-bronze.md) | Ingestão de dados brutos, schema de auditoria, controle de duplicatas, versionamento de registros |
| [03 — Camada Silver](./03-camada-silver.md) | Limpeza, padronização, validação de qualidade, transformações, merge incremental |
| [04 — Camada Gold](./04-camada-gold.md) | Modelos analíticos, agregações, dimensões e fatos, views de consumo, tabelas para relatórios |
| [05 — Pipeline e Orquestração](./05-pipeline-orquestracao.md) | Estrutura de pipeline, idempotência, logging de execução, tratamento de erros, agendamento |
| [06 — Apache Airflow](./06-airflow.md) | DAGs, operators MsSQL/S3/Python, sensors, connections, variáveis, templates Jinja, backfill, boas práticas |
| [07 — Governança de Dados](./07-governanca-dados.md) | Catálogo de dados, linhagem, classificação, controle de acesso, qualidade com SLAs, auditoria, retenção |

---

## Referência Rápida

| Camada | Schema padrão | Propósito | Quem consome |
|---|---|---|---|
| Bronze | `bronze` | Dados brutos, imutáveis, com metadados de carga | Engenheiros de dados |
| Silver | `silver` | Dados limpos, validados, sem duplicatas | Analistas, engenheiros |
| Gold | `gold` | Dados agregados e modelados para análise | BI, relatórios, aplicações |
| Pipeline | `pipeline` | Controle de execução, logs, marca d'água | Engenheiros de dados |
| Governança | `governanca` | Catálogo, linhagem, qualidade, auditoria | Todos os papéis |
