# S3 — Amazon S3 via CLI e PowerShell

> Operações no Amazon S3 sem acesso ao console AWS. Tudo via **AWS CLI** e módulo **AWS.Tools.S3** para PowerShell.

---

## Estrutura desta pasta

```
S3/
├── README.md               ← este arquivo
├── 01-configuracao.md      ← Instalar e configurar AWS CLI e módulo PowerShell
├── 02-operacoes-basicas.md ← Listar, upload, download, copiar, deletar, URL pré-assinada
└── 03-automacao-scripts.md ← Scripts prontos: limpeza, inventário, pipeline de backup, DR
```

---

## Documentos

| Arquivo | Conteúdo |
|---|---|
| [01 — Configuração](./01-configuracao.md) | Instalar AWS CLI via `winget`, `aws configure`, perfis nomeados, módulo PowerShell `AWS.Tools.S3`, credenciais via variável de ambiente, testar acesso ao bucket |
| [02 — Operações Básicas](./02-operacoes-basicas.md) | Listar buckets e objetos, upload/download de arquivos e pastas, Storage Classes (STANDARD, STANDARD_IA, GLACIER), copiar/mover/renomear, deletar com `--dryrun`, metadados, URL pré-assinada, tabela de referência rápida |
| [03 — Automação e Scripts](./03-automacao-scripts.md) | Limpeza automática por retenção (X dias), inventário em CSV, sync bidirecional, pipeline completo Backup RDS→S3→Limpeza, verificar integridade de backups, replicação entre buckets (DR), script de diagnóstico |

---

## Referência Rápida — AWS CLI

```powershell
# Listar
aws s3 ls s3://meu-bucket/pasta/

# Upload
aws s3 cp arquivo.bak s3://meu-bucket/backups/

# Download
aws s3 cp s3://meu-bucket/backups/arquivo.bak ./

# Sync pasta → S3
aws s3 sync ./local s3://meu-bucket/pasta/

# Deletar (com simulação)
aws s3 rm s3://meu-bucket/backups/ --recursive --dryrun

# URL temporária (1 hora)
aws s3 presign s3://meu-bucket/arquivo.bak --expires-in 3600
```

---

## Pré-requisitos

| Ferramenta | Instalação |
|---|---|
| **AWS CLI** | `winget install Amazon.AWSCLI` |
| **AWS.Tools.S3** | `Install-Module AWS.Tools.S3` |
| **Credenciais** | `aws configure` (Access Key + Secret Key + Região) |
