# Pipeline e Orquestração

> Um pipeline de engenharia de dados é a sequência ordenada de etapas que move os dados da fonte até o consumo final. Este documento cobre os **princípios e padrões** de um pipeline Medallion bem construído — independente de tecnologia.

> Para a ferramenta de orquestração, consulte `Engenharia-Dados/06-airflow.md`.

---

## O que é um pipeline de dados

Um pipeline conecta e ordena as etapas de processamento de dados:

```
Fonte ──→ Bronze ──→ Silver ──→ Gold ──→ Consumidores
```

Cada seta representa uma **transformação** executada por uma etapa do pipeline. O pipeline garante que:
- As etapas rodem na ordem certa
- Uma etapa só comece quando a anterior terminar com sucesso
- Falhas sejam detectadas, registradas e reportadas
- O pipeline possa ser reexecutado após uma falha sem inconsistência

---

## Princípios de um bom pipeline

| Princípio | O que significa | Por que importa |
|---|---|---|
| **Idempotente** | Executar duas vezes produz o mesmo resultado que executar uma | Permite reprocessamento seguro sem medo de duplicatas |
| **Rastreável** | Cada execução tem log de início, fim, status e volume | Facilita diagnóstico e auditoria |
| **Recuperável** | Em caso de falha, é possível reexecutar sem inconsistência | Reduz tempo de resolução de incidentes |
| **Modular** | Cada camada é independente e reexecutável isoladamente | Facilita manutenção e evolução |
| **Observável** | É possível saber o status sem precisar olhar os dados diretamente | Permite monitoramento automatizado |

---

## Idempotência — o princípio mais importante

Um pipeline idempotente não tem medo de ser reexecutado. Se rodar duas vezes para o mesmo dia, o resultado é o mesmo que rodar uma vez.

**Como garantir idempotência por camada:**

| Camada | Estratégia |
|---|---|
| Bronze | Verificar hash antes de inserir — não inserir se o registro já existe |
| Silver | Usar upsert (MERGE) — atualizar se existe, inserir se não existe |
| Gold | Deletar os dados do período antes de recarregar — nunca acumular |

**O inimigo da idempotência** é usar `NOW()` / `GETDATE()` como referência de data em vez da data de execução do pipeline. Se o pipeline usa a data atual, reexecutá-lo no dia seguinte produz resultado diferente.

---

## Marca d'Água (Watermark)

A marca d'água é o mecanismo que registra **até onde o pipeline já processou**, permitindo cargas incrementais eficientes.

```
Primeira execução:  processa IDs 1 a 1.000   → salva marca d'água = 1.000
Segunda execução:   processa IDs 1.001 a 1.500 → salva marca d'água = 1.500
Terceira execução:  processa IDs 1.501 a 2.000 → salva marca d'água = 2.000
```

Sem marca d'água, o pipeline precisaria processar tudo do início a cada execução. Com ela, processa apenas o que é novo.

A marca d'água pode ser baseada em:
- **ID sequencial:** `WHERE id > :ultima_marca`
- **Data/hora:** `WHERE updated_at > :ultima_carga`
- **Número de arquivo:** `WHERE arquivo_id > :ultimo_arquivo`

---

## Tabelas de Controle do Pipeline

Todo pipeline robusto tem tabelas de controle para rastrear execuções:

**Log de Execuções**
```
pipeline.Log_Execucao
├── LogID
├── NomePipeline    → "CarregarPedido_Bronze"
├── Camada          → "Bronze" | "Silver" | "Gold"
├── DtInicio
├── DtFim
├── Status          → "EM_ANDAMENTO" | "SUCESSO" | "ERRO"
├── TotalRegistros  → quantos registros foram processados
└── Mensagem        → detalhes de erro, se houver
```

**Marca d'Água**
```
pipeline.MarcaDAgua
├── NomePipeline    → identifica qual pipeline
├── UltimaCarga     → quando foi a última execução bem-sucedida
└── UltimoID        → último ID ou timestamp processado
```

---

## Modos de Execução

Um pipeline bem projetado suporta dois modos:

| Modo | Quando usar | O que faz |
|---|---|---|
| **Incremental** | Execuções regulares (diárias, horárias) | Processa apenas os dados novos desde a última carga |
| **Full Reload** | Após mudança de regra de negócio ou correção de bug | Apaga e recria as camadas do zero a partir da Bronze |

---

## Checklist de Validação Pós-Carga

Após cada execução do pipeline, verificar:

```
[ ] Bronze: volume do dia está dentro do esperado?
[ ] Bronze: não há registros pendentes de processamento (is_processado = 0)?
[ ] Silver: taxa de rejeição está dentro do limite aceitável?
[ ] Silver: não há duplicatas por PedidoID?
[ ] Gold: totais batem com a Silver?
[ ] Gold: data mais recente na Gold é a data de execução?
[ ] Pipeline: duração total dentro do SLA?
[ ] Pipeline: nenhum erro registrado no log de execução?
```

---

## Estratégia de Reprocessamento

| Situação | Ação |
|---|---|
| Dado da fonte chegou errado e foi corrigido | Reinserir no Bronze → reprocessar Silver e Gold |
| Regra de negócio mudou na Silver | Full Reload: apagar Silver, resetar Bronze, reprocessar |
| Tabela Gold desatualizada | Rodar apenas a carga Gold, sem tocar Bronze e Silver |
| Pipeline travou no meio | Verificar log para saber em qual camada parou → reexecutar a partir dali |
| Nova coluna adicionada na fonte | Alterar tabela Bronze, reprocessar Silver e Gold |

---

## Agendamento

O pipeline deve ter um **horário definido e monitorado**. Algumas estratégias:

| Frequência | Casos de uso típicos |
|---|---|
| **Diário (madrugada)** | Relatórios com dados do dia anterior, dashboards de gestão |
| **Por hora** | Dados operacionais que precisam de atualização frequente |
| **Em tempo real (streaming)** | Alertas, monitoramento, sistemas transacionais críticos |
| **Sob demanda** | Reprocessamentos, cargas históricas, migrações |

Para implementar o agendamento, consulte `Engenharia-Dados/06-airflow.md`.

---

## Para implementação específica

- **SQL Server (MSSQL):** SQL Agent para agendamento nativo, procedures para cada etapa — ver `MSSQL/RDS/03-backup-restore.md` (padrão de monitoramento de tasks) e `MSSQL/03-programabilidade.md`
- **Apache Airflow:** orquestração avançada com DAGs Python, retentativas, monitoramento — ver `Engenharia-Dados/06-airflow.md`
- **Ferramentas alternativas:** Azure Data Factory, AWS Glue, dbt Cloud, Prefect, Dagster seguem os mesmos princípios com interfaces diferentes
