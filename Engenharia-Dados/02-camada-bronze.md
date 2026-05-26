# Camada Bronze — Ingestão de Dados Brutos

> A Bronze é a **primeira camada de aterrissagem dos dados**. Tudo que chega de qualquer fonte entra aqui primeiro, exatamente como veio — sem transformação, sem filtro, sem limpeza.

---

## A regra de ouro da Bronze

**Nunca altere. Nunca delete.**

O dado bruto é o único registro fiel da realidade no momento em que aconteceu. Se houver um erro nas camadas seguintes, você sempre pode voltar à Bronze e reprocessar. Se você deletar ou modificar a Bronze, perde essa garantia para sempre.

---

## Princípios

| Princípio | Descrição |
|---|---|
| **Imutabilidade** | Registros inseridos nunca são alterados ou deletados |
| **Fidelidade** | O dado é copiado exatamente como chegou da fonte |
| **Rastreabilidade** | Toda linha sabe de onde veio, quando chegou e de qual sistema |
| **Tolerância a erro** | Mesmo registros com formato errado são armazenados — o erro é informação |
| **Completude** | Nenhum dado é descartado antes de chegar à Bronze |

---

## O que a Bronze armazena

Além dos campos de negócio vindos da fonte, toda tabela Bronze deve conter **metadados de auditoria**:

| Campo de Auditoria | Propósito |
|---|---|
| `fonte_sistema` | Identifica de qual sistema o dado veio (ERP, API, arquivo CSV, etc.) |
| `arquivo_origem` | Nome do arquivo ou identificador do batch de origem |
| `dt_carga` | Quando o dado foi inserido na Bronze |
| `hash_registro` | Impressão digital do conteúdo — permite detectar duplicatas sem comparar campo a campo |
| `is_processado` | Flag que indica se a Silver já processou este registro |

---

## Estrutura típica de uma tabela Bronze

```
bronze.Pedido_Raw
├── bronze_id          (chave técnica gerada pela Bronze — nunca da fonte)
├── fonte_sistema      (auditoria)
├── arquivo_origem     (auditoria)
├── dt_carga           (auditoria)
├── hash_registro      (auditoria)
├── is_processado      (auditoria)
│
├── pedido_id_origem   → texto bruto, como veio
├── cliente_id_origem  → texto bruto, como veio
├── dt_pedido_raw      → texto bruto (pode ser "15/06/2024" ou "2024-06-15" — a Bronze não se importa)
├── status_raw         → texto bruto
└── valor_total_raw    → texto bruto (pode ter vírgula, ponto, símbolo de moeda)
```

> **Por que tudo como texto?** Porque a Bronze não sabe se o campo `dt_pedido` virá como `DATE`, `DATETIME`, `VARCHAR` ou com fuso horário. Guardar como texto garante que nenhum dado seja perdido por incompatibilidade de tipo — a conversão é trabalho da Silver.

---

## Staging: a antessala da Bronze

Na maioria dos pipelines, os dados não vão direto da fonte para a Bronze. Eles passam por uma tabela de **staging** primeiro.

```
Fonte → Staging (temporária) → Bronze (permanente)
```

| Staging | Bronze |
|---|---|
| Truncada antes de cada carga | Nunca truncada |
| Sem histórico | Histórico completo |
| Onde os dados "pousam" primeiro | Onde os dados são registrados para sempre |
| Pode ter qualquer estrutura da fonte | Tem estrutura padronizada com campos de auditoria |

O processo é:
1. Truncar a staging
2. Carregar os novos dados na staging (via arquivo, API, replicação, etc.)
3. Mover da staging para a Bronze, adicionando os metadados de auditoria

---

## Controle de Duplicatas com Hash

Para evitar inserir o mesmo dado duas vezes (por exemplo, se o pipeline rodar duas vezes para a mesma data), a Bronze usa um **hash do conteúdo** do registro.

O hash é calculado concatenando os campos-chave do registro e gerando uma impressão digital única. Antes de inserir, verifica-se se aquele hash já existe na Bronze. Se existir, o registro é ignorado.

```
hash("pedido_id=1234|cliente_id=789|data=2024-06-15|valor=150.00")
→ "a3f8c2d1..." (64 caracteres)
```

Essa abordagem é mais eficiente do que comparar campo a campo, especialmente para registros com muitas colunas.

---

## Monitoramento da Bronze

Indicadores importantes para acompanhar:

- **Volume por fonte e data:** quantos registros chegaram de cada sistema por dia
- **Backlog:** quantos registros Bronze ainda não foram processados pela Silver
- **Taxa de chegada:** o volume está dentro do esperado ou houve queda/pico?
- **Última carga:** quando foi a última vez que cada fonte enviou dados?

---

## Para implementação específica

A Bronze é um conceito agnóstico de tecnologia. Para ver como implementá-la no ambiente deste projeto:

- **SQL Server (MSSQL):** consulte `MSSQL/03-programabilidade.md` para procedures de carga e `MSSQL/02-ddl-tabelas-indices.md` para estrutura de tabelas
- **Ingestão de arquivos via PowerShell:** consulte `S3/03-automacao-scripts.md`
- **Agendamento da carga:** consulte `Engenharia-Dados/06-airflow.md`
