# Amazon S3 — Operações Básicas

> Todos os comandos via AWS CLI e módulo PowerShell. Sem acesso ao console AWS.

---

## 1. Listar Buckets e Objetos

```powershell
# ===== AWS CLI =====

# Listar todos os buckets
aws s3 ls

# Listar conteúdo de um bucket (raiz)
aws s3 ls s3://meu-bucket/

# Listar pasta específica
aws s3 ls s3://meu-bucket/backups/

# Listar recursivamente (todas as subpastas)
aws s3 ls s3://meu-bucket/ --recursive

# Listar com tamanho legível (KB, MB, GB)
aws s3 ls s3://meu-bucket/backups/ --recursive --human-readable

# Listar com sumário de tamanho total
aws s3 ls s3://meu-bucket/ --recursive --summarize

# Ordenar por data via PowerShell
aws s3 ls s3://meu-bucket/backups/ --recursive |
    Sort-Object { ($_ -split '\s+')[0..1] -join ' ' }

# ===== AWS PowerShell =====
Import-Module AWS.Tools.S3

Get-S3Bucket

# Listar objetos de uma pasta
Get-S3Object -BucketName "meu-bucket" -Prefix "backups/"

# Listar com detalhes formatados
Get-S3Object -BucketName "meu-bucket" -Prefix "backups/" |
    Select-Object Key,
                  @{N='TamanhoMB'; E={[math]::Round($_.Size/1MB,2)}},
                  LastModified,
                  StorageClass |
    Sort-Object LastModified -Descending |
    Format-Table -AutoSize
```

---

## 2. Upload de Arquivos

```powershell
# ===== AWS CLI =====

# Arquivo único
aws s3 cp "C:\Temp\backup.bak" s3://meu-bucket/backups/backup.bak

# Com Storage Class mais barato (menos acessado)
aws s3 cp "C:\Temp\backup.bak" s3://meu-bucket/backups/backup.bak `
    --storage-class STANDARD_IA

# Pasta inteira (sync)
aws s3 sync "C:\Temp\Backups\" s3://meu-bucket/backups/

# Sync excluindo extensões
aws s3 sync "C:\Temp\Backups\" s3://meu-bucket/backups/ `
    --exclude "*.tmp" --exclude "*.log"

# Com metadata personalizado
aws s3 cp "C:\Temp\backup.bak" s3://meu-bucket/backups/backup.bak `
    --metadata "ambiente=producao,banco=MeuBanco,data=$(Get-Date -f yyyyMMdd)"

# ===== AWS PowerShell =====

# Arquivo único
Write-S3Object `
    -BucketName  "meu-bucket" `
    -Key         "backups/backup.bak" `
    -File        "C:\Temp\backup.bak" `
    -StorageClass "STANDARD_IA"

# Pasta inteira
Write-S3Object `
    -BucketName "meu-bucket" `
    -KeyPrefix  "backups/" `
    -Folder     "C:\Temp\Backups"
```

### Storage Classes disponíveis

| Classe | Uso recomendado | Custo |
|---|---|---|
| `STANDARD` | Dados acessados frequentemente | Alto |
| `STANDARD_IA` | Acessados < 1x/mês | Médio |
| `GLACIER` | Arquivo, acessado raramente | Baixo |
| `DEEP_ARCHIVE` | Retenção longa (>7 anos) | Muito baixo |

---

## 3. Download de Arquivos

```powershell
# ===== AWS CLI =====

# Arquivo único
aws s3 cp s3://meu-bucket/backups/backup.bak "C:\Temp\backup.bak"

# Pasta inteira
aws s3 sync s3://meu-bucket/backups/ "C:\Temp\Backups\"

# Filtro por extensão
aws s3 cp s3://meu-bucket/backups/ "C:\Temp\" `
    --recursive --exclude "*" --include "*.bak"

# Sem barra de progresso (útil em scripts)
aws s3 cp s3://meu-bucket/backups/backup.bak "C:\Temp\backup.bak" --no-progress

# ===== AWS PowerShell =====

Read-S3Object `
    -BucketName "meu-bucket" `
    -Key        "backups/backup.bak" `
    -File       "C:\Temp\backup.bak"

# Pasta inteira
Read-S3Object `
    -BucketName "meu-bucket" `
    -KeyPrefix  "backups/" `
    -Folder     "C:\Temp\Backups"
```

---

## 4. Copiar e Mover Objetos

```powershell
# ===== AWS CLI =====

# Copiar dentro do mesmo bucket
aws s3 cp s3://meu-bucket/backups/backup.bak `
          s3://meu-bucket/archive/backup.bak

# Mover (copia + deleta origem)
aws s3 mv s3://meu-bucket/backups/backup.bak `
          s3://meu-bucket/archive/backup.bak

# Copiar para outro bucket
aws s3 cp s3://bucket-origem/backup.bak `
          s3://bucket-destino/backup.bak

# Sync entre buckets
aws s3 sync s3://bucket-origem/backups/ `
            s3://bucket-destino/backups/

# Renomear = mv com novo nome
aws s3 mv s3://meu-bucket/backups/old.bak `
          s3://meu-bucket/backups/new.bak

# ===== AWS PowerShell =====
Copy-S3Object `
    -BucketName    "meu-bucket" `
    -Key           "backups/backup.bak" `
    -DestinationKey "archive/backup.bak"
```

---

## 5. Deletar Objetos

```powershell
# ===== AWS CLI =====

# Arquivo único
aws s3 rm s3://meu-bucket/backups/backup.bak

# Pasta inteira
aws s3 rm s3://meu-bucket/backups/ --recursive

# Com filtro
aws s3 rm s3://meu-bucket/backups/ `
    --recursive --exclude "*" --include "*.tmp"

# Simulação (--dryrun — só mostra o que seria deletado)
aws s3 rm s3://meu-bucket/backups/ --recursive --dryrun

# ===== AWS PowerShell =====

Remove-S3Object -BucketName "meu-bucket" -Key "backups/backup.bak" -Force

# Deletar múltiplos
$objetos = Get-S3Object -BucketName "meu-bucket" -Prefix "backups/old/"
$objetos | ForEach-Object {
    Remove-S3Object -BucketName "meu-bucket" -Key $_.Key -Force
    Write-Host "Deletado: $($_.Key)"
}
```

---

## 6. Metadados e Propriedades

```powershell
# ===== AWS CLI =====

# Metadados de um objeto
aws s3api head-object --bucket meu-bucket --key backups/backup.bak

# Listar versões (se versionamento habilitado)
aws s3api list-object-versions --bucket meu-bucket --prefix backups/backup.bak

# ===== AWS PowerShell =====
Get-S3ObjectMetadata -BucketName "meu-bucket" -Key "backups/backup.bak"

# Verificar se objeto existe
$obj = Get-S3Object -BucketName "meu-bucket" -Key "backups/backup.bak" -ErrorAction SilentlyContinue
if ($obj) { "Existe: $($obj.Key) | $([math]::Round($obj.Size/1MB,1)) MB" }
else      { "Não encontrado." }
```

---

## 7. URL Pré-assinada (Acesso temporário sem credenciais)

```powershell
# ===== AWS CLI =====

# URL válida por 1 hora
aws s3 presign s3://meu-bucket/backups/backup.bak --expires-in 3600

# URL válida por 24 horas
aws s3 presign s3://meu-bucket/backups/backup.bak --expires-in 86400

# ===== AWS PowerShell =====
Get-S3PreSignedURL `
    -BucketName "meu-bucket" `
    -Key        "backups/backup.bak" `
    -Expires    (Get-Date).AddHours(24)
```

---

## 8. Referência Rápida — Comandos AWS CLI

| Operação | Comando |
|---|---|
| Listar buckets | `aws s3 ls` |
| Listar objetos | `aws s3 ls s3://bucket/pasta/` |
| Listar recursivo | `aws s3 ls s3://bucket/ --recursive` |
| Upload arquivo | `aws s3 cp arquivo s3://bucket/pasta/` |
| Download arquivo | `aws s3 cp s3://bucket/pasta/arquivo ./` |
| Sync pasta → S3 | `aws s3 sync ./local s3://bucket/pasta/` |
| Sync S3 → pasta | `aws s3 sync s3://bucket/pasta/ ./local` |
| Copiar objeto | `aws s3 cp s3://b/a.bak s3://b/b.bak` |
| Mover objeto | `aws s3 mv s3://b/a.bak s3://b/b.bak` |
| Deletar objeto | `aws s3 rm s3://bucket/pasta/arquivo` |
| Deletar pasta | `aws s3 rm s3://bucket/pasta/ --recursive` |
| URL pré-assinada | `aws s3 presign s3://b/obj --expires-in 3600` |
| Metadados | `aws s3api head-object --bucket b --key k` |
| Simular deleção | `aws s3 rm s3://bucket/pasta/ --recursive --dryrun` |
