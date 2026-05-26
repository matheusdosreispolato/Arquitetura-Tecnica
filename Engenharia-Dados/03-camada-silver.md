# Camada Silver — Limpeza e Transformação

> A Silver é a camada onde os dados brutos do Bronze se tornam **confiáveis e utilizáveis**. Aqui os tipos são corrigidos, as duplicatas são removidas, os formatos são padronizados e os registros inválidos são isolados com rastreabilidade.

---

## O papel da Silver

Se a Bronze é o depósito de matéria-prima bruta, a Silver é a linha de montagem que processa essa matéria-prima e entrega peças prontas para uso. Qualquer analista que quiser trabalhar com dados limpos e confiáveis deve olhar para a Silver — nunca diretamente para a Bronze.

---

## Princípios

| Princípio | Descrição |
|---|---|
| **Sempre parte da Bronze** | A Silver nunca busca dados diretamente da fonte — sempre da Bronze |
| **Sem agregação** | A granularidade é a mesma da Bronze — um registro transformado, não resumido |
| **Rastreabilidade** | Cada registro Silver sabe de qual registro Bronze ele veio |
| **Reprocessável** | A Silver pode ser apagada e recriada integralmente a partir da Bronze |
| **Transparente nos erros** | Registros rejeitados são guardados com o motivo da rejeição |

---

## O que a Silver faz com cada registro

```
Bronze (dado bruto)
        ↓
   [Validação]
        ↓
   Válido? ──Não──→ Tabela de Rejeições (com motivo)
        │
       Sim
        ↓
   [Transformação]
   - Converter tipos (texto → data, texto → número)
   - Padronizar formatos (datas, moeda, texto)
   - Normalizar valores (maiúsculas/minúsculas, remover espaços)
        ↓
   [Deduplicação]
   - Se o mesmo PedidoID já existe, manter apenas o mais recente
        ↓
Silver (dado limpo e confiável)
```

---

## Dimensões de Qualidade Aplicadas na Silver

| Dimensão | O que verifica | Ação ao falhar |
|---|---|---|
| **Completude** | Campos obrigatórios preenchidos | Rejeitar com motivo "campo X nulo" |
| **Validade de tipo** | Data é realmente uma data? Número é realmente um número? | Rejeitar com motivo "tipo inválido no campo X" |
| **Validade de negócio** | Valor total > 0? Data não é futura? | Rejeitar com motivo "regra de negócio violada" |
| **Unicidade** | PedidoID aparece só uma vez? | Manter o registro mais recente, descartar os outros |

---

## Tabela de Rejeições

Todo registro que não passa na validação vai para uma tabela de rejeições — nunca é silenciosamente descartado.

```
silver.Pedido_Rejeitado
├── bronze_id        → referência ao registro Bronze original
├── motivo_rejeicao  → descrição legível: "Data de pedido inválida"
├── campo_problema   → qual campo causou a rejeição: "dt_pedido_raw"
├── valor_problema   → qual era o valor problemático: "31/02/2024"
└── dt_rejeicao      → quando foi rejeitado
```

A tabela de rejeições é fundamental para:
- Reportar problemas de qualidade para a equipe da fonte
- Acompanhar se a taxa de rejeição aumenta ao longo do tempo
- Reprocessar registros rejeitados após correção na fonte

---

## Merge Incremental (Upsert)

Quando a fonte envia atualizações de registros já existentes (ex.: status de um pedido que mudou de "Pendente" para "Pago"), a Silver precisa atualizar o registro existente em vez de inserir uma duplicata.

Esse padrão é chamado de **upsert** (update + insert):
- Se o registro **já existe**: atualizar com os novos valores
- Se o registro **não existe**: inserir como novo

O campo de controle `versao` registra quantas vezes aquele registro foi atualizado — útil para auditoria.

---

## Reprocessamento

A Silver pode ser completamente recriada a partir da Bronze quando necessário:

1. Apagar todos os registros da Silver (e da tabela de rejeições)
2. Resetar o flag `is_processado` na Bronze para `0` (indicando "não processado")
3. Reexecutar o pipeline de Silver

Isso é possível porque a Bronze nunca é deletada. O reprocessamento é necessário quando:
- Uma regra de negócio muda (ex.: um status que era considerado válido agora não é)
- Um bug na lógica de limpeza é corrigido
- Uma nova fonte de dados entra e precisa ser integrada retroativamente

---

## Para implementação específica

- **SQL Server (MSSQL):** o comando `MERGE` implementa o upsert nativamente. Ver `MSSQL/01-basico-dml.md`
- **Python / PySpark:** use `DataFrame.merge()` ou `MERGE` via Spark SQL
- **dbt:** use `incremental` materializations com `unique_key`
- **Agendamento:** consulte `Engenharia-Dados/06-airflow.md` para orquestrar Bronze → Silver como Tasks dependentes
