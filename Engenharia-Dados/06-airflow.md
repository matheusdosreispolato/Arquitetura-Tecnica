# Apache Airflow — Orquestração de Pipelines de Dados

> O Apache Airflow é uma plataforma open-source para **criar, agendar e monitorar workflows** como código Python. No contexto da arquitetura Medallion, o Airflow é o responsável por garantir que os dados fluam de Bronze para Silver para Gold na ordem certa, no horário certo, e com tratamento de falhas.

---

## Para quem nunca usou: a analogia da linha de produção

Imagine uma fábrica com uma linha de produção. Cada estação executa uma etapa: cortar, montar, pintar, embalar. A ordem importa — você não pode pintar antes de montar. Se a estação de montagem falha, a de pintura espera até a montagem ser refeita.

O Airflow é o **gerente da linha de produção dos dados**:
- Define a ordem das etapas
- Garante que cada etapa só comece quando a anterior terminar com sucesso
- Agenda quando a linha de produção deve rodar (diariamente, de hora em hora, etc.)
- Alerta quando algo dá errado
- Permite reexecutar apenas a etapa que falhou, sem reiniciar do zero

---

## Conceitos Fundamentais

| Conceito | O que é |
|---|---|
| **DAG** | *Directed Acyclic Graph* — o fluxo completo de tarefas com suas dependências e agendamento. O "pipeline" do Airflow. |
| **Task** | Cada etapa dentro de uma DAG. Pode ser um script Python, um comando SQL, uma chamada de API, etc. |
| **Operator** | O tipo de trabalho que uma Task executa. Define como a tarefa roda (ex: Python, Bash, SQL, HTTP). |
| **Dependency** | A relação entre tasks — qual deve terminar antes da próxima começar. |
| **Schedule** | Quando a DAG deve ser executada automaticamente (ex: todo dia às 02:00). |
| **DAG Run** | Uma execução específica de uma DAG (ex: a execução do dia 15/06). |
| **Hook** | Conexão reutilizável com um sistema externo (banco de dados, API, storage). |
| **Connection** | Credenciais armazenadas com segurança no Airflow e referenciadas pelo nome nas DAGs. |
| **Variable** | Configuração dinâmica armazenada no Airflow (ex: nome do bucket, modo de execução). |
| **Sensor** | Tipo especial de Task que espera uma condição ser verdadeira antes de prosseguir. |
| **XCom** | Mecanismo para Tasks trocarem pequenos dados entre si dentro de uma DAG Run. |

---

## Airflow e a Arquitetura Medallion

O Airflow se encaixa naturalmente como o orquestrador do pipeline Medallion. Cada camada vira um conjunto de Tasks dentro de uma DAG, com dependências entre elas.

```
DAG: pipeline_dados_diario (roda todo dia às 02:00)
│
├── [1] ingerir_bronze          → captura dados da fonte e persiste na camada Bronze
│        ↓
├── [2] processar_silver        → limpa, valida e persiste na camada Silver
│        ↓
├── [3] carregar_gold           → agrega e persiste na camada Gold
│        ↓
└── [4] verificar_qualidade     → valida contagens e métricas esperadas
```

Se `processar_silver` falhar, o Airflow:
- Interrompe o fluxo (não executa Gold)
- Registra o erro com log completo
- Notifica por email / Slack (se configurado)
- Permite reexecutar só `processar_silver` após a correção — Bronze intacto, sem reprocessar do zero

---

## Estrutura de uma DAG

Uma DAG é um arquivo Python que descreve o fluxo. Abaixo está a estrutura mínima:

```python
# dags/pipeline_medallion.py

from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator

# ── Configurações padrão de cada Task ────────────────────────────────────────
default_args = {
    "owner"           : "engenharia_dados",   # responsável
    "retries"         : 2,                    # tentar 2 vezes antes de falhar
    "retry_delay"     : timedelta(minutes=5), # aguardar 5min entre tentativas
    "email_on_failure": ["alertas@empresa.com"],
}

# ── Definição da DAG ─────────────────────────────────────────────────────────
with DAG(
    dag_id            = "pipeline_medallion_diario",
    description       = "Fluxo Bronze → Silver → Gold — Dados de Vendas",
    default_args      = default_args,
    schedule_interval = "0 2 * * *",   # cron: todo dia às 02:00
    start_date        = datetime(2024, 1, 1),
    catchup           = False,         # não reprocessar execuções passadas
    max_active_runs   = 1,             # evitar execuções paralelas da mesma DAG
    tags              = ["medallion", "vendas"],
) as dag:

    # ── Funções Python que cada Task vai executar ─────────────────────────────
    def ingerir_bronze(**context):
        """Busca dados na fonte e persiste na camada Bronze."""
        data_ref = context["ds"]  # data da execução, ex: "2024-06-15"
        print(f"Ingerindo dados de {data_ref} para o Bronze...")
        # aqui entra a lógica de ingestão específica do seu sistema

    def processar_silver(**context):
        """Lê do Bronze, limpa e valida, persiste na Silver."""
        print("Processando Bronze → Silver...")
        # limpeza, validação, deduplicação

    def carregar_gold(**context):
        """Lê da Silver, agrega e persiste na Gold."""
        print("Carregando Silver → Gold...")
        # agregações, dimensões, fatos

    def verificar_qualidade(**context):
        """Valida que os dados chegaram corretamente na Gold."""
        print("Verificando qualidade dos dados...")
        # contagens, SLAs de qualidade

    # ── Criação das Tasks ─────────────────────────────────────────────────────
    t1 = PythonOperator(task_id="ingerir_bronze",     python_callable=ingerir_bronze)
    t2 = PythonOperator(task_id="processar_silver",   python_callable=processar_silver)
    t3 = PythonOperator(task_id="carregar_gold",      python_callable=carregar_gold)
    t4 = PythonOperator(task_id="verificar_qualidade",python_callable=verificar_qualidade)

    # ── Dependências (ordem de execução) ─────────────────────────────────────
    t1 >> t2 >> t3 >> t4
    # lê-se: t1 deve terminar antes de t2, t2 antes de t3, etc.
```

---

## Operadores Mais Usados

| Operator | Quando usar |
|---|---|
| `PythonOperator` | Executar qualquer função Python |
| `BashOperator` | Executar comandos de terminal |
| `SqlOperator` / variantes | Executar queries em bancos de dados |
| `HttpOperator` | Chamar APIs REST |
| `EmailOperator` | Enviar notificações por email |
| `BranchPythonOperator` | Decidir qual caminho seguir baseado em uma condição |
| `S3KeySensor` | Aguardar um arquivo aparecer no S3 |
| `FileSensor` | Aguardar um arquivo aparecer no sistema de arquivos |
| `ExternalTaskSensor` | Aguardar outra DAG ou Task terminar |
| `TriggerDagRunOperator` | Disparar outra DAG como parte do fluxo |
| `EmptyOperator` | Marcador sem ação — útil para agrupar dependências |

---

## Padrões de Dependência no Medallion

### Sequencial simples (mais comum)
```python
bronze >> silver >> gold >> validacao
```

### Com ramificação condicional
```python
# Se não houver dados novos, pular o processamento
from airflow.operators.python import BranchPythonOperator

def checar_dados(**ctx):
    tem_dados = verificar_se_ha_dados_novos()
    return "processar_silver" if tem_dados else "sem_dados_novos"

branch = BranchPythonOperator(task_id="checar_dados", python_callable=checar_dados)
sem_dados = EmptyOperator(task_id="sem_dados_novos")

bronze >> branch >> [processar_silver, sem_dados]
processar_silver >> gold
```

### Com sensor — aguardar arquivo antes de processar
```python
from airflow.sensors.filesystem import FileSensor

aguardar = FileSensor(
    task_id       = "aguardar_arquivo",
    filepath      = "/dados/entrada/pedidos_{{ ds_nodash }}.csv",
    poke_interval = 300,   # verificar a cada 5 minutos
    timeout       = 7200,  # desistir após 2 horas
    mode          = "reschedule",  # libera o worker enquanto espera
)

aguardar >> ingerir_bronze >> processar_silver >> carregar_gold
```

### Múltiplas fontes em paralelo, convergindo na Silver
```python
# Ingerir de dois sistemas ao mesmo tempo
bronze_erp = PythonOperator(task_id="bronze_erp",    ...)
bronze_api = PythonOperator(task_id="bronze_api",    ...)
silver     = PythonOperator(task_id="processar_silver", ...)

[bronze_erp, bronze_api] >> silver >> gold
# bronze_erp e bronze_api rodam em paralelo
# silver só começa quando AMBOS terminarem
```

---

## Agendamento (Schedule)

O Airflow usa expressões **cron** para o agendamento:

```
┌─── minuto    (0–59)
│ ┌─── hora      (0–23)
│ │ ┌─── dia do mês (1–31)
│ │ │ ┌─── mês        (1–12)
│ │ │ │ ┌─── dia da semana (0–7, 0 e 7 = domingo)
│ │ │ │ │
* * * * *
```

| Expressão | Significado |
|---|---|
| `"0 2 * * *"` | Todo dia às 02:00 |
| `"0 * * * *"` | A cada hora (no minuto 0) |
| `"0 2 * * 1"` | Toda segunda-feira às 02:00 |
| `"0 2 1 * *"` | Todo dia 1 do mês às 02:00 |
| `"*/15 * * * *"` | A cada 15 minutos |
| `"@daily"` | Todo dia à meia-noite |
| `"@hourly"` | Toda hora |
| `None` | Sem agendamento — execução somente manual |

---

## Idempotência no Airflow

Uma DAG é **idempotente** quando executá-la duas vezes para a mesma data produz o mesmo resultado que executá-la uma vez. Isso é essencial para poder reprocessar sem medo.

**Boas práticas para idempotência:**
- Usar a data de execução (`{{ ds }}`) como referência, não `GETDATE()` / `NOW()`
- Deletar ou sobrescrever os dados do período antes de inserir novos (não acumular duplicatas)
- Garantir que cada Task possa ser reexecutada isoladamente sem depender do estado de execuções anteriores

```python
def carregar_gold(**context):
    data_ref = context["ds"]  # "2024-06-15" — fixo para esta execução
    # deleta os dados deste dia antes de recarregar
    deletar_gold_para_data(data_ref)
    # recarrega do zero para esta data
    carregar_gold_para_data(data_ref)
```

---

## Variáveis e Connections

Configurações sensíveis (senhas, endpoints) e dinâmicas (nomes de ambientes, flags) ficam armazenadas no Airflow, não no código.

```python
from airflow.models import Variable

# Lendo variáveis (configuradas em Admin → Variables na UI)
ambiente      = Variable.get("ambiente",       default_var="producao")
batch_size    = Variable.get("batch_size",     default_var="5000")
notificar_em  = Variable.get("email_alertas",  default_var="dba@empresa.com")

# Lendo como JSON
config = Variable.get("config_pipeline", deserialize_json=True)
# Exemplo de valor: {"camadas_ativas": ["bronze", "silver", "gold"], "modo": "incremental"}
```

As **Connections** armazenam credenciais de sistemas externos (bancos, APIs, storage) e são referenciadas pelo `conn_id` nas Tasks — sem senhas no código.

---

## Monitoramento e Alertas

O Airflow oferece uma interface web com:
- **Grafo da DAG:** visualização das Tasks e dependências
- **Grid view:** histórico de execuções com status por Task por data
- **Logs:** log completo de cada Task por execução
- **Gantt:** tempo de execução por Task

Para alertas automáticos, configure callbacks na DAG:

```python
def ao_falhar(context):
    """Chamado automaticamente quando uma Task falha."""
    dag_id  = context["dag"].dag_id
    task_id = context["task_instance"].task_id
    data    = context["execution_date"]
    # enviar para Slack, Teams, PagerDuty, email, etc.
    print(f"FALHA: {dag_id} / {task_id} em {data}")

default_args = {
    "on_failure_callback": ao_falhar,
    # ...
}
```

---

## Backfill — Reprocessar Datas Passadas

Quando uma regra muda ou um bug é corrigido, é possível reprocessar datas históricas:

```bash
# Reprocessar um intervalo de datas
airflow dags backfill \
    --start-date 2024-06-01 \
    --end-date   2024-06-30 \
    pipeline_medallion_diario

# Reprocessar apenas uma data específica
airflow dags backfill \
    --start-date 2024-06-15 \
    --end-date   2024-06-15 \
    pipeline_medallion_diario
```

O backfill só funciona bem quando as DAGs são idempotentes — por isso esse princípio é tão importante.

---

## Boas Práticas

| Prática | Por quê |
|---|---|
| **`catchup = False`** | Evita que o Airflow execute todas as datas passadas ao ativar uma DAG nova |
| **`max_active_runs = 1`** | Evita que duas execuções da mesma DAG rodem ao mesmo tempo e causem conflito |
| **Credenciais em Connections** | Nunca colocar senhas no código Python |
| **Configurações em Variables** | Permite mudar parâmetros sem alterar o código |
| **Tasks atômicas e idempotentes** | Cada Task pode ser reexecutada sem efeito colateral |
| **Nomear Tasks com verbos** | `ingerir_bronze`, `processar_silver` — mais legível no grafo |
| **Testar localmente antes de subir** | `python dags/meu_pipeline.py` valida a sintaxe antes de ir para produção |
| **`mode="reschedule"` em Sensors** | Libera o worker enquanto espera — não trava um slot desnecessariamente |
| **Usar `{{ ds }}` em vez de `NOW()`** | Garante que a data de referência seja sempre a data da execução, não a atual |

---

## Relação com Outras Ferramentas

| Ferramenta | Relação com Airflow |
|---|---|
| **dbt** | Airflow pode chamar transformações dbt como Tasks dentro do pipeline Medallion |
| **Spark / PySpark** | Airflow pode submeter jobs Spark para processar volumes muito grandes |
| **Great Expectations** | Airflow pode executar suítes de qualidade após cada camada |
| **OpenLineage / Marquez** | Integração com Airflow para capturar linhagem automaticamente por execução |
| **Kubernetes** | Airflow pode executar Tasks em pods isolados via KubernetesPodOperator |
