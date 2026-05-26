# AGENTS.md — Guia para IA

> Este arquivo orienta qualquer agente de IA (Claude, Copilot, GPT, etc.) sobre o propósito deste repositório, suas convenções e como contribuir corretamente com novos conteúdos.

---

## O que é este projeto

Este repositório contém **documentação técnica operacional** de um ambiente com:

- **AWS RDS para SQL Server** (MSSQL) — sem acesso ao console AWS
- **Amazon S3** — gerenciado exclusivamente via CLI e PowerShell
- **Report Builder / SSRS** — relatórios `.rdl`

O usuário acessa tudo via **client SQL externo** (SSMS, Azure Data Studio, sqlcmd) e **PowerShell**. Não há acesso ao painel web da AWS.

---

## Estrutura de Pastas

```
Arquitetura Tecnica/
├── AGENTS.md                        ← este arquivo (leia primeiro)
├── README.md                        ← índice geral navegável
│
├── MSSQL/                           ← tudo sobre SQL Server
│   ├── README.md                    ← índice da pasta MSSQL
│   ├── 01-basico-dml.md             ← SELECT, INSERT, UPDATE, DELETE
│   ├── 02-ddl-tabelas-indices.md    ← CREATE TABLE, índices, constraints
│   ├── 03-programabilidade.md       ← Procedures, Functions, Triggers
│   ├── 04-avancado-performance.md   ← CTEs, Window Functions, performance
│   ├── 09-padronizacao-nomenclaturas.md  ← padrões de nome de objetos
│   ├── 10-joins-views-procedures.md      ← conceitos para leigos e técnicos
│   └── RDS/                         ← específico do AWS RDS
│       ├── README.md                ← índice da pasta RDS
│       ├── 01-conexao-configuracao.md
│       ├── 02-usuarios-permissoes.md
│       ├── 03-backup-restore.md
│       └── 04-limitacoes-diferencas.md
│
├── RDL/                             ← Report Builder / SSRS
│   ├── README.md
│   ├── 01-introducao-estrutura.md
│   ├── 02-expressoes-sintaxe.md
│   ├── 03-datasets-parametros.md
│   └── 04-formatacao-grupos-agregacoes.md
│
└── S3/                              ← Amazon S3
    ├── README.md
    ├── 01-configuracao.md
    ├── 02-operacoes-basicas.md
    └── 03-automacao-scripts.md
```

---

## Convenções Obrigatórias

### Formato dos arquivos

- Todos os documentos são `.md` (Markdown)
- Código SQL em blocos ` ```sql `
- Código PowerShell em blocos ` ```powershell `
- Código XML (RDL) em blocos ` ```xml `
- Código VB.NET (expressões RDL) em blocos ` ```vb `

### Marcadores de ambiente (use em todo conteúdo MSSQL)

```
🖥️ MSSQL Padrão   → funciona em qualquer SQL Server
☁️ AWS RDS         → específico ou diferente no RDS
(sem marcador)     → funciona igual nos dois ambientes
```

Quando um comando for igual nos dois ambientes, **não use marcador**. Quando houver diferença, mostre os dois blocos lado a lado com os marcadores.

### Nomenclatura de arquivos

- Formato: `NN-nome-descritivo.md` (número de 2 dígitos + hífen + nome em kebab-case)
- Novos arquivos T-SQL gerais entram em `MSSQL/` com numeração contínua
- Novos arquivos RDS entram em `MSSQL/RDS/` com numeração contínua
- Não use underline `_` nem espaços em nomes de arquivo

### Cada pasta DEVE ter um `README.md`

Todo diretório precisa de um `README.md` com:
1. Descrição do que a pasta contém
2. Estrutura de arquivos (árvore)
3. Tabela de links para cada arquivo com descrição de uma linha

---

## Contexto do Ambiente

Informe-se sobre estas restrições antes de sugerir qualquer solução:

| Restrição | Detalhe |
|---|---|
| Sem console AWS | O usuário não acessa o painel web da AWS. Toda operação é via T-SQL, sqlcmd ou PowerShell |
| Sem `xp_cmdshell` | Não disponível no RDS. Alternativa: PowerShell externo |
| Sem `BACKUP TO DISK` | Não disponível no RDS. Use `msdb.dbo.rds_backup_database` → S3 |
| Sem logins Windows | Não disponível via T-SQL no RDS. Alternativa: AWS Directory Service |
| Sem SSMS no servidor | O acesso é sempre remoto, via client externo |
| Sem acesso ao SO | O RDS não expõe sistema de arquivos nem permite comandos de OS |

---

## Como Adicionar Conteúdo Novo

### 1. Novo tópico T-SQL geral

Crie em `MSSQL/` com o próximo número disponível:

```
MSSQL/11-novo-topico.md
```

Atualize `MSSQL/README.md` adicionando uma linha na tabela de documentos.

### 2. Novo tópico exclusivo do RDS

Crie em `MSSQL/RDS/` com o próximo número disponível:

```
MSSQL/RDS/05-novo-topico-rds.md
```

Atualize `MSSQL/RDS/README.md`.

### 3. Novo tópico de Report Builder

Crie em `RDL/` com o próximo número:

```
RDL/05-novo-topico-rdl.md
```

Atualize `RDL/README.md`.

### 4. Novo tópico S3

Crie em `S3/` com o próximo número:

```
S3/04-novo-topico-s3.md
```

Atualize `S3/README.md`.

### 5. Sempre atualize o README raiz

Após criar qualquer arquivo novo, adicione uma linha em `README.md` na seção correspondente.

---

## Tom e Estilo de Escrita

- **Público-alvo misto:** o conteúdo deve ser compreensível para DBAs experientes E para analistas/desenvolvedores com menos experiência em banco
- **Para leigos:** use analogias do mundo real antes de mergulhar na parte técnica (veja como está feito em `10-joins-views-procedures.md`)
- **Para técnicos:** sempre inclua exemplos de código completos e funcionais
- **Português:** todo o conteúdo é em português do Brasil
- **Sem jargão desnecessário:** explique siglas na primeira ocorrência
- **Exemplos reais:** use nomes de tabelas/colunas do domínio de negócio (`vendas.Pedido`, `dbo.Cliente`) — nunca `tabela1`, `col_a`

---

## Padrões de Nomenclatura SQL (resumo)

Documentados em detalhes em `MSSQL/09-padronizacao-nomenclaturas.md`. Resumo:

| Objeto | Padrão | Exemplo |
|---|---|---|
| Schema | lowercase | `vendas`, `financeiro` |
| Tabela | PascalCase singular | `vendas.Pedido` |
| Coluna PK | `<Tabela>ID` | `PedidoID` |
| Coluna FK | `<TabelaRef>ID` | `ClienteID` |
| View | `vw_<Descricao>` | `vw_PedidoCompleto` |
| Procedure | `usp_<Verbo><Entidade>` | `usp_CancelarPedido` |
| Function | `fn_<Descricao>` | `fn_FormatarCPF` |
| Trigger | `trg_<Tabela>_<Evento>` | `trg_Pedido_Auditoria` |
| Login app | `app_<sistema>` | `app_vendas` |
| Role | `role_<dominio>_<nivel>` | `role_vendas_readonly` |

---

## Verificação antes de publicar

Antes de finalizar qualquer documento novo, verifique:

- [ ] O arquivo tem um título `# Título` na primeira linha
- [ ] Todo código está em blocos de código com linguagem especificada
- [ ] Exemplos usam nomes de objetos no padrão definido acima
- [ ] Diferenças entre MSSQL padrão e RDS estão marcadas com os ícones corretos
- [ ] O README da pasta foi atualizado com o novo arquivo
- [ ] O README raiz foi atualizado se necessário
- [ ] Não há credenciais reais, senhas ou endpoints de produção nos exemplos
