# Governança de Dados

> Governança de dados é o conjunto de **políticas, processos, papéis e responsabilidades** que garantem que os dados de uma organização sejam confiáveis, seguros, rastreáveis e usados de forma adequada. Não é uma ferramenta — é uma prática organizacional.

---

## Para quem nunca ouviu falar: a analogia da biblioteca

Uma biblioteca sem governança seria um caos: livros sem catalogação, sem saber quem pegou emprestado, sem regras de acesso para seções restritas, sem prazo de devolução, e sem ninguém responsável por manter o acervo organizado.

Governança de dados é exatamente o trabalho de ser o **bibliotecário dos dados da organização**:

| Biblioteca | Governança de Dados |
|---|---|
| Catálogo de livros | Catálogo de dados — o que existe e do que trata |
| Classificação por seção | Classificação por sensibilidade e domínio |
| Registro de empréstimos | Linhagem — quem criou, transformou e consumiu |
| Regras de acesso a seções restritas | Controle de acesso por perfil |
| Prazo de devolução / descarte | Política de retenção |
| Responsável pelo acervo | Data Owner e Data Steward |

---

## Por que governança importa?

Sem governança, organizações frequentemente enfrentam:

- **Número diferente em dois relatórios** para a mesma métrica — sem saber qual está certo
- **Dado sensível exposto** para quem não deveria ter acesso
- **Impossibilidade de rastrear** de onde veio um número em um dashboard
- **Dados duplicados ou conflitantes** vindos de sistemas diferentes
- **Perda de histórico** por falta de política de retenção
- **Bloqueios em auditoria** por falta de documentação de origem dos dados

---

## Os Seis Pilares

### 1. 📚 Catálogo de Dados
O inventário de todos os ativos de dados da organização. O catálogo responde:
- Quais tabelas/datasets existem?
- O que cada um representa?
- Quem é o responsável?
- Com que frequência é atualizado?
- Qual é a fonte de origem?

Um bom catálogo permite que qualquer pessoa da organização encontre os dados que precisa **sem precisar perguntar para a engenharia**.

---

### 2. 🔗 Linhagem de Dados
O rastreamento do caminho percorrido por cada dado — da fonte bruta até o consumo final.

```
Sistema de Origem
      ↓  ingestão
  Camada Bronze (dado bruto)
      ↓  limpeza e validação
  Camada Silver (dado limpo)
      ↓  agregação e modelagem
  Camada Gold (dado analítico)
      ↓  consumo
  Dashboard / Relatório / Aplicação
```

A linhagem permite responder: *"esse número do relatório de vendas — de onde ele veio exatamente?"*

**Granularidades da linhagem:**

| Nível | Descrição | Exemplo |
|---|---|---|
| **Dataset** | Qual tabela originou qual tabela | `bronze.Pedido_Raw` → `silver.Pedido` |
| **Coluna** | Qual campo originou qual campo | `valor_total_raw` → `ValorTotal` após limpeza |
| **Registro** | Qual registro individual originou qual | ID 1234 no ERP → linha 5678 no Gold |

---

### 3. ✅ Qualidade de Dados
Garantia de que os dados atendem a padrões mínimos de confiabilidade. Qualidade é medida por dimensões:

| Dimensão | Pergunta que responde | Exemplo de regra |
|---|---|---|
| **Completude** | Todos os campos obrigatórios estão preenchidos? | `ClienteID` não pode ser nulo |
| **Precisão** | O dado reflete a realidade? | Data de pedido não pode ser futura |
| **Consistência** | O mesmo dado é igual em fontes diferentes? | Status "CANCELADO" não pode ter valor > 0 |
| **Unicidade** | Existem duplicatas onde não deveria haver? | Um PedidoID deve aparecer só uma vez |
| **Pontualidade** | O dado chegou dentro do prazo esperado? | Pipeline deve terminar até 03:00 |
| **Validade** | O formato está correto? | CEP com 8 dígitos, CPF com 11 |

Cada equipe deve definir **SLAs de qualidade** por camada e ser notificada quando violados.

---

### 4. 🔒 Classificação e Segurança
Todo dado deve ter um nível de sensibilidade atribuído, que determina quem pode acessá-lo e como.

| Classificação | Descrição | Exemplos |
|---|---|---|
| **Público** | Pode ser compartilhado sem restrições | Estatísticas publicadas, dados abertos |
| **Interno** | Uso dentro da organização | Dados de negócio sem informação pessoal |
| **Confidencial** | Acesso restrito a times específicos | Metas financeiras, dados estratégicos |
| **Restrito** | Acesso mínimo, sempre auditado | CPF, dados bancários, saúde, senhas |

A classificação deve ser atribuída por **tabela** e por **coluna** (uma tabela pode ser Interno, mas ter colunas Restritas).

---

### 5. 🗓️ Retenção de Dados
Define por quanto tempo cada tipo de dado deve ser mantido e o que acontece ao final do prazo.

| Ação | Quando usar |
|---|---|
| **Arquivar** | Dado não é mais consultado ativamente mas precisa ser preservado (fiscal, auditoria) |
| **Anonimizar** | Dado pessoal que precisa ser desidentificado mas mantido para análise agregada |
| **Deletar** | Dado sem valor após o prazo ou por exigência regulatória (LGPD, GDPR) |

A política de retenção deve considerar:
- Requisitos legais (LGPD, Receita Federal, BACEN, etc.)
- Valor analítico ao longo do tempo
- Custo de armazenamento

---

### 6. 👤 Responsabilidade (Ownership)
Cada dado precisa de um dono — sem dono, ninguém é responsável pela qualidade ou acesso.

| Papel | Responsabilidade |
|---|---|
| **Data Owner** | Define o que o dado significa, aprova acessos, define classificação. Geralmente gestor de negócio. |
| **Data Steward** | Mantém o catálogo atualizado, valida definições, resolve conflitos entre fontes. |
| **Data Engineer** | Constrói e mantém os pipelines, garante a linhagem técnica, implementa regras de qualidade. |
| **Data Analyst** | Consome os dados, reporta problemas de qualidade, valida resultados com o negócio. |
| **DBA / Infra** | Gerencia acessos físicos, executa políticas de retenção, monitora segurança e auditoria. |

---

## Governança na Arquitetura Medallion

Cada camada tem responsabilidades distintas de governança:

### 🥉 Bronze
- **Catálogo:** documentar a fonte de origem, sistema de origem, frequência de ingestão
- **Linhagem:** registrar qual sistema alimentou qual tabela Bronze
- **Qualidade:** não há regras de negócio aqui — apenas garantia de que o dado chegou
- **Acesso:** restrito ao time de Engenharia de Dados
- **Retenção:** depende do contrato com a fonte; típico 1–5 anos

### 🥈 Silver
- **Catálogo:** documentar todas as colunas, seus tipos, significado e restrições
- **Linhagem:** registrar quais transformações foram aplicadas (limpeza, deduplicação)
- **Qualidade:** aplicar e monitorar as dimensões de qualidade (completude, unicidade, validade)
- **Acesso:** analistas de dados e engenheiros autorizados
- **Retenção:** dado limpo tem valor analítico longo; típico 5–10 anos

### 🥇 Gold
- **Catálogo:** documentar as métricas de negócio, suas fórmulas e interpretação correta
- **Linhagem:** documentar como cada métrica Gold foi derivada da Silver
- **Qualidade:** SLAs de disponibilidade (estar pronto até hora X), consistência entre métricas
- **Acesso:** usuários de BI, gestores, aplicações analíticas
- **Retenção:** tabelas de resumo podem ser recriadas, mas snapshots históricos devem ser preservados

---

## Processos de Governança

Governança não acontece por acidente — requer processos recorrentes:

| Processo | Frequência | Responsável |
|---|---|---|
| Revisar e atualizar o catálogo | A cada novo pipeline ou mudança de schema | Data Steward + Data Engineer |
| Executar verificações de qualidade | A cada carga de dados | Automatizado pelo pipeline |
| Revisar permissões de acesso | Trimestral ou em mudanças de equipe | Data Owner + DBA |
| Auditar acessos a dados sensíveis | Mensal ou sob demanda | DBA / Segurança |
| Aplicar política de retenção | Conforme vencimento | Automatizado + aprovação do Data Owner |
| Resolver conflitos de definição | Sob demanda | Data Steward + áreas envolvidas |

---

## Maturidade em Governança

A governança não precisa ser implementada de uma vez. Existe uma progressão natural:

```
Nível 1 — Reativo
  Problemas de qualidade são descobertos pelos usuários finais.
  Não há catálogo formal. Acesso é gerenciado manualmente.

Nível 2 — Definido
  Existe um catálogo, mesmo que parcial.
  Responsáveis por domínios definidos.
  Regras de qualidade documentadas (mesmo que não automatizadas).

Nível 3 — Gerenciado
  Verificações de qualidade automatizadas no pipeline.
  Linhagem técnica documentada e rastreável.
  Controle de acesso baseado em roles.

Nível 4 — Otimizado
  Qualidade monitorada com alertas automáticos e SLAs.
  Catálogo integrado com ferramentas de BI.
  Governança como parte do processo de entrega de dados.
```

A maioria das organizações começa no Nível 1 e evolui gradualmente. **Começar simples é melhor que não começar.**

---

## Checklist por Camada

```
BRONZE:
[ ] Fonte de origem documentada (sistema, API, arquivo)
[ ] Responsável pelo dado de origem identificado
[ ] Frequência e janela de ingestão definidas
[ ] Política de retenção atribuída
[ ] Acesso restrito ao time de engenharia

SILVER:
[ ] Todas as colunas documentadas no catálogo
[ ] Colunas com dados pessoais identificadas e classificadas como Restrito
[ ] Regras de qualidade definidas para campos críticos
[ ] Linhagem registrada (de onde cada coluna Silver veio na Bronze)
[ ] Critérios de rejeição de registro documentados

GOLD:
[ ] Métricas de negócio com definição formal (fórmula + contexto)
[ ] Responsável de negócio (Data Owner) definido por domínio
[ ] SLA de disponibilidade documentado e monitorado
[ ] Linhagem registrada (como cada métrica Gold foi derivada da Silver)
[ ] Usuários e perfis com acesso mapeados

GERAL:
[ ] Existe uma pessoa responsável pela governança (Data Steward ou equivalente)?
[ ] Os usuários finais sabem onde encontrar os dados no catálogo?
[ ] Há um processo para reportar problemas de qualidade?
[ ] Existe um processo de aprovação para acesso a dados sensíveis?
```

---

## Ferramentas de Mercado (Referência)

> As ferramentas ajudam a implementar a governança, mas não a substituem. Governança é processo e cultura, não software.

| Categoria | Exemplos de ferramentas |
|---|---|
| **Catálogo de dados** | Apache Atlas, DataHub, Alation, Collibra, Google Dataplex |
| **Linhagem** | OpenLineage, Marquez, DataHub, dbt (parcial) |
| **Qualidade de dados** | Great Expectations, dbt tests, Soda Core, Monte Carlo |
| **Controle de acesso** | Apache Ranger, AWS Lake Formation, Unity Catalog (Databricks) |
| **Auditoria** | Ferramentas nativas dos bancos de dados, AWS CloudTrail |
| **Orquestração + Lineage** | Apache Airflow (com plugins OpenLineage) |
