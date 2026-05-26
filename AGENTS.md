# AGENTS.md — Guia para IA

> Este arquivo orienta qualquer agente de IA (Claude Code, Copilot, GPT, etc.) sobre o propósito deste repositório, suas convenções e como contribuir corretamente com novos conteúdos.
> Leia este arquivo antes de qualquer tarefa. Ele é o briefing completo do projeto.

---

## O que é este projeto

Repositório de **documentação técnica operacional** para um time de engenharia que trabalha com:

| Área | Tecnologia | Acesso |
|---|---|---|
| Banco de dados | AWS RDS SQL Server | T-SQL, sqlcmd, SSMS (sem console AWS) |
| Armazenamento | Amazon S3 | AWS CLI, PowerShell |
| Relatórios | SSRS / Report Builder | Arquivos `.rdl` |
| Engenharia de dados | Arquitetura Medallion (Bronze/Silver/Gold) | Airflow, dbt, SQL |
| Projetos Python | APIs, scripts, pipelines | FastAPI, pytest, ruff |
| Projetos de IA | LLMs, prompts, evals | Anthropic, OpenAI |
| Agentes | Automação com LLMs | LangGraph, CrewAI, AutoGen |

**Restrição crítica:** o usuário **não tem acesso ao painel web da AWS**. Toda operação de infraestrutura é feita via AWS CLI ou PowerShell. Nunca sugira "acesse o console da AWS".

---

## Estrutura Completa de Pastas

```
Arquitetura Tecnica/
│
├── AGENTS.md                              ← este arquivo (leia primeiro)
├── README.md                              ← índice geral navegável
│
├── MSSQL/                                 ← SQL Server — T-SQL e objetos de banco
│   ├── README.md
│   ├── 01-basico-dml.md                   ← SELECT, INSERT, UPDATE, DELETE
│   ├── 02-ddl-tabelas-indices.md          ← CREATE TABLE, índices, constraints
│   ├── 03-programabilidade.md             ← Procedures, Functions, Triggers
│   ├── 04-avancado-performance.md         ← CTEs, Window Functions, performance
│   ├── 09-padronizacao-nomenclaturas.md   ← padrões de nome de objetos SQL
│   ├── 10-joins-views-procedures.md       ← conceitos para leigos e técnicos
│   └── RDS/                               ← específico do AWS RDS
│       ├── README.md
│       ├── 01-conexao-configuracao.md     ← conexão, Parameter Group, Option Group
│       ├── 02-usuarios-permissoes.md      ← logins, roles, permissões (RDS vs SQL Server)
│       ├── 03-backup-restore.md           ← snapshots (AWS CLI) e .bak no S3 (T-SQL)
│       └── 04-limitacoes-diferencas.md    ← o que não existe no RDS e as alternativas
│
├── Engenharia-Dados/                      ← Arquitetura Medallion + orquestração + governança
│   ├── README.md
│   ├── 01-conceitos-medallion.md          ← fundamentos, analogias, schemas, nomenclatura
│   ├── 02-camada-bronze.md                ← ingestão raw, hash, staging, imutabilidade
│   ├── 03-camada-silver.md                ← limpeza, validação, MERGE, tabela de rejeições
│   ├── 04-camada-gold.md                  ← dimensões, fatos, views, agregações
│   ├── 05-pipeline-orquestracao.md        ← idempotência, marca d'água, modos de execução
│   ├── 06-airflow.md                      ← DAGs, operators, sensors, backfill, boas práticas
│   └── 07-governanca-dados.md             ← catálogo, linhagem, qualidade, acesso, retenção
│
├── RDL/                                   ← Report Builder / SSRS
│   ├── README.md
│   ├── 01-introducao-estrutura.md
│   ├── 02-expressoes-sintaxe.md
│   ├── 03-datasets-parametros.md
│   └── 04-formatacao-grupos-agregacoes.md
│
├── S3/                                    ← Amazon S3 (via AWS CLI e PowerShell)
│   ├── README.md
│   ├── 01-configuracao.md
│   ├── 02-operacoes-basicas.md
│   └── 03-automacao-scripts.md
│
├── Python/                                ← Padronização de projetos Python
│   ├── README.md
│   ├── 01-estrutura-projeto.md            ← src/ layout, venv, pyproject.toml, .env
│   ├── 02-boas-praticas.md                ← tipagem, docstrings, linting, testes, nomenclatura
│   └── 03-arquitetura-web.md              ← FastAPI, camadas (Router/Service/Repository), Docker
│
├── IA/                                    ← Projetos com LLMs
│   ├── README.md
│   ├── 01-estrutura-projeto-ia.md         ← pastas prompts/, tools/, memory/, evals/
│   ├── 02-engenharia-prompts.md           ← definições, técnicas, economia de tokens, templates
│   ├── 03-integracao-api.md               ← SDKs Anthropic/OpenAI, retries, streaming, custo
│   ├── 04-evals.md                        ← avaliação de qualidade: datasets, métricas
│   └── 05-claude-code-codex.md            ← usar Claude Code/Codex com AGENTS.md e README.md
│
└── Agents/                                ← Agentes de IA
    ├── README.md
    ├── 01-conceitos.md                    ← ReAct, tools, memória, multi-agente, limitações
    ├── 02-frameworks.md                   ← LangChain, LangGraph, CrewAI, AutoGen
    └── 03-agents-dados.md                 ← text-to-SQL, análise, diagnóstico de pipeline
```

---

## Princípio de Organização: Uma Pasta = Um Tema

Cada pasta tem um tema exclusivo. **Não misture conteúdo entre pastas.**

| Pasta | Contém | Não contém |
|---|---|---|
| `MSSQL/` | T-SQL, objetos de banco, SQL Server | Python, Airflow, conceitos genéricos |
| `MSSQL/RDS/` | Diferenças e operações específicas do AWS RDS | T-SQL genérico (fica em `MSSQL/`) |
| `Engenharia-Dados/` | Conceitos agnósticos de Medallion, Airflow, governança | SQL específico (referencia `MSSQL/`) |
| `Python/` | Estrutura, boas práticas, arquitetura web Python | Conteúdo de IA ou agentes |
| `IA/` | Prompts, APIs LLM, evals, uso de Claude Code | Código de agentes com frameworks |
| `Agents/` | Frameworks e padrões de agentes | Conceitos básicos de LLM (ficam em `IA/`) |

Quando um documento de `Engenharia-Dados/` precisar de exemplo SQL, coloque uma referência cruzada:
```
> Para implementação em SQL Server, consulte `MSSQL/03-programabilidade.md`.
```

---

## Marcadores de Ambiente (MSSQL)

Use estes marcadores em todo conteúdo da pasta `MSSQL/`:

```
🖥️ MSSQL Padrão       → funciona em qualquer SQL Server (on-premises, VM, Docker)
☁️ Nível RDS           → operações de infraestrutura via AWS CLI (aws rds, aws ec2)
🗄️ Nível SQL Server   → operações dentro do banco via T-SQL no RDS
(sem marcador)         → comportamento idêntico nos dois ambientes
```

### Quando usar cada marcador em `MSSQL/RDS/`

| Operação | Nível correto |
|---|---|
| Snapshot, backup automático, retenção | ☁️ AWS CLI |
| Parameter Group, Option Group, reiniciar | ☁️ AWS CLI |
| Multi-AZ, failover, Security Group | ☁️ AWS CLI |
| Criar logins, usuários, roles | 🗄️ T-SQL |
| Backup/restore de banco individual | 🗄️ T-SQL (`rds_backup_database`) |
| Monitorar sessões, DMVs, query plans | 🗄️ T-SQL |
| SQL Agent jobs | 🗄️ T-SQL |

---

## Convenções de Código

### SQL Server

Documentado em detalhes em `MSSQL/09-padronizacao-nomenclaturas.md`. Resumo:

| Objeto | Padrão | Exemplo |
|---|---|---|
| Schema | `lowercase` | `vendas`, `financeiro` |
| Tabela | `PascalCase` singular | `vendas.Pedido` |
| Coluna PK | `<Tabela>ID` | `PedidoID` |
| Coluna FK | `<TabelaRef>ID` | `ClienteID` |
| View | `vw_<Descricao>` | `vw_PedidoCompleto` |
| Procedure | `usp_<Verbo><Entidade>` | `usp_CancelarPedido` |
| Function | `fn_<Descricao>` | `fn_FormatarCPF` |
| Trigger | `trg_<Tabela>_<Evento>` | `trg_Pedido_Auditoria` |
| Login app | `app_<sistema>` | `app_vendas` |
| Role | `role_<dominio>_<nivel>` | `role_vendas_readonly` |

### Engenharia de Dados (Medallion)

Documentado em `Engenharia-Dados/01-conceitos-medallion.md`:

| Objeto | Padrão | Exemplo |
|---|---|---|
| Schema | `bronze`, `silver`, `gold`, `pipeline`, `governanca` | |
| Procedure de carga | `usp_<Camada>_<Acao><Entidade>` | `usp_Bronze_IngerirPedido` |
| DAG Airflow | `pipeline_<dominio>_<frequencia>` | `pipeline_vendas_diario` |

### Python

Documentado em `Python/02-boas-praticas.md`:

| Identificador | Convenção | Exemplo |
|---|---|---|
| Variável / função | `snake_case` | `total_pedidos`, `calcular_desconto()` |
| Classe | `PascalCase` | `PedidoService` |
| Constante | `UPPER_SNAKE_CASE` | `MAX_TENTATIVAS` |
| Arquivo | `snake_case.py` | `pedido_service.py` |
| Privado | prefixo `_` | `_cache`, `_validar()` |

---

## Restrições do Ambiente

| Restrição | Detalhe |
|---|---|
| Sem console AWS | Toda operação via T-SQL, sqlcmd ou PowerShell/AWS CLI |
| Sem `xp_cmdshell` | Não disponível no RDS. Alternativa: PowerShell externo |
| Sem `BACKUP TO DISK` | Não disponível no RDS. Use `msdb.dbo.rds_backup_database` → S3 |
| Sem logins Windows | Não disponível via T-SQL no RDS. Alternativa: AWS Directory Service |
| Sem acesso ao SO | O RDS não expõe o sistema de arquivos |

---

## Como Adicionar Conteúdo Novo

### Novo arquivo em qualquer pasta

1. Nomear como `NN-nome-descritivo.md` (número de 2 dígitos + kebab-case)
2. Primeira linha: `# Título do Documento`
3. Todo código em bloco com linguagem especificada (` ```python `, ` ```sql `, etc.)
4. Atualizar o `README.md` da pasta com o novo arquivo
5. Atualizar o `README.md` raiz se for uma pasta nova

### Novo arquivo MSSQL geral

```
MSSQL/11-novo-topico.md   ← numeração contínua
```
Atualizar `MSSQL/README.md`.

### Novo arquivo RDS

```
MSSQL/RDS/05-novo-topico.md
```
Atualizar `MSSQL/RDS/README.md`. Dividir em `### ☁️ Nível RDS` e `### 🗄️ Nível SQL Server`.

### Novo arquivo Engenharia de Dados

```
Engenharia-Dados/08-novo-topico.md
```
Conteúdo deve ser **agnóstico de tecnologia**. Para código SQL específico, referenciar `MSSQL/`.

### Novo arquivo Python / IA / Agents

Seguir a numeração contínua da pasta. Atualizar o `README.md` da pasta.

---

## Tom e Estilo de Escrita

- **Idioma:** português do Brasil em todo o conteúdo
- **Público-alvo:** engenheiros e analistas com nível variado — use analogias antes de mergulhar na técnica
- **Exemplos:** nomes de domínio reais (`vendas.Pedido`, `PedidoService`) — nunca `tabela1`, `classe_a`
- **Sem jargão não explicado:** defina siglas na primeira ocorrência
- **Cada pasta tem seu tema:** não misture SQL em `IA/`, não misture Python em `MSSQL/`

---

## Verificação antes de Finalizar

- [ ] O arquivo começa com `# Título`
- [ ] Todo código tem linguagem no bloco de código
- [ ] Nomes de objetos seguem o padrão da pasta
- [ ] O `README.md` da pasta foi atualizado
- [ ] Se há diferença MSSQL padrão vs RDS, os marcadores estão corretos
- [ ] Não há credenciais, senhas ou endpoints de produção
- [ ] Conteúdo da pasta não invade tema de outra pasta (use referências cruzadas)
