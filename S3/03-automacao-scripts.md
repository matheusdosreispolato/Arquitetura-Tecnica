# Amazon S3 — Automação e Scripts PowerShell

> Scripts reutilizáveis para operações automatizadas no S3 sem acesso ao console AWS.

---

## 1. Limpeza Automática por Retenção

```powershell
# ============================================================
# Deletar objetos no S3 mais antigos que X dias
# ============================================================
param(
    [string]$Bucket        = "meu-bucket",
    [string]$Prefix        = "backups/",
    [int]   $DiasRetencao  = 30,
    [switch]$DryRun         # Só mostra, não deleta
)

Import-Module AWS.Tools.S3 -ErrorAction Stop

$limite  = (Get-Date).AddDays(-$DiasRetencao)
$objetos = Get-S3Object -BucketName $Bucket -Prefix $Prefix |
           Where-Object { $_.LastModified -lt $limite }

if (-not $objetos) {
    Write-Host "Nenhum objeto mais antigo que $DiasRetencao dias encontrado." -ForegroundColor Yellow
    return
}

Write-Host "Encontrados $($objetos.Count) objeto(s) para remover:" -ForegroundColor Cyan

$totalMB = 0
foreach ($obj in $objetos) {
    $sizeMB   = [math]::Round($obj.Size / 1MB, 2)
    $totalMB += $sizeMB
    Write-Host "  [$($obj.LastModified.ToString('yyyy-MM-dd'))] $($obj.Key)  ($sizeMB MB)"

    if (-not $DryRun) {
        Remove-S3Object -BucketName $Bucket -Key $obj.Key -Force
    } else {
        Write-Host "  → DryRun: não deletado." -ForegroundColor DarkGray
    }
}

Write-Host "`nTotal: $($objetos.Count) objeto(s) | $([math]::Round($totalMB,1)) MB liberados." -ForegroundColor Green

# Uso:
# .\limpar-s3.ps1 -Bucket meu-bucket -Prefix backups/ -DiasRetencao 30 -DryRun
# .\limpar-s3.ps1 -Bucket meu-bucket -Prefix backups/ -DiasRetencao 30
```

---

## 2. Inventário e Relatório do Bucket

```powershell
# ============================================================
# Gerar relatório de conteúdo do bucket em CSV
# ============================================================
param(
    [string]$Bucket      = "meu-bucket",
    [string]$Prefix      = "",
    [string]$Saida       = "C:\Temp\inventario_s3.csv"
)

Import-Module AWS.Tools.S3

Write-Host "Buscando objetos em s3://$Bucket/$Prefix ..." -ForegroundColor Cyan

$objetos = Get-S3Object -BucketName $Bucket -Prefix $Prefix

$relatorio = $objetos | Select-Object `
    Key,
    @{N='TamanhoMB';  E={[math]::Round($_.Size/1MB, 4)}},
    LastModified,
    StorageClass,
    @{N='Pasta'; E={ ($_.Key -split '/')[0..($_.Key.Split('/').Count-2)] -join '/' }}

$relatorio | Export-Csv -Path $Saida -NoTypeInformation -Encoding UTF8

$totalGB = [math]::Round(($objetos | Measure-Object -Property Size -Sum).Sum / 1GB, 2)
Write-Host "Total: $($objetos.Count) objetos | $totalGB GB"
Write-Host "Relatorio salvo em: $Saida" -ForegroundColor Green
```

---

## 3. Sync Bidirecional (Backup Local ↔ S3)

```powershell
# ============================================================
# Sincronizar pasta local com S3 (upload de novos/modificados)
# ============================================================
param(
    [string]$PastaLocal = "C:\Dados\Backups",
    [string]$Bucket     = "meu-bucket",
    [string]$Prefix     = "sync/backups",
    [switch]$Delete      # Remove objetos no S3 que não existem mais localmente
)

$s3Path = "s3://$Bucket/$Prefix/"
Write-Host "Sincronizando $PastaLocal → $s3Path" -ForegroundColor Cyan

$args = @(
    "s3", "sync",
    $PastaLocal, $s3Path,
    "--exclude", "*.tmp",
    "--exclude", "*.log",
    "--storage-class", "STANDARD_IA"
)

if ($Delete) { $args += "--delete" }

& aws @args

Write-Host "Sincronizacao concluida." -ForegroundColor Green
```

---

## 4. Pipeline Completo — Backup RDS → S3 → Limpeza

```powershell
# ============================================================
# Pipeline: Backup RDS → S3 → Deletar antigos
# Estrategia: FULL domingo, DIFFERENTIAL demais dias
# ============================================================
param(
    [string]$SqlEndpoint  = "meu-rds.xxxx.sa-east-1.rds.amazonaws.com,1433",
    [string]$SqlUser      = "admin",
    [string]$SqlPassword  = "SuaSenha123!",
    [string]$Banco        = "MeuBanco",
    [string]$S3Bucket     = "meu-bucket",
    [string]$S3Prefix     = "backups",
    [int]   $RetencaoDias = 30
)

# ---- Configuração ----
$timestamp  = Get-Date -Format "yyyyMMdd_HHmmss"
$dia        = (Get-Date).DayOfWeek
$tipo       = if ($dia -eq 'Sunday') { 'FULL' } else { 'DIFFERENTIAL' }
$s3ARN      = "arn:aws:s3:::$S3Bucket/$S3Prefix/${Banco}_${tipo}_${timestamp}.bak"

function Write-Step($msg) {
    Write-Host "[$(Get-Date -f HH:mm:ss)] $msg" -ForegroundColor Cyan
}
function Write-OK($msg) {
    Write-Host "[$(Get-Date -f HH:mm:ss)] OK: $msg" -ForegroundColor Green
}
function Write-Err($msg) {
    Write-Host "[$(Get-Date -f HH:mm:ss)] ERRO: $msg" -ForegroundColor Red
}

Write-Host "=============================================" -ForegroundColor Yellow
Write-Host " PIPELINE BACKUP RDS"
Write-Host " Banco  : $Banco"
Write-Host " Tipo   : $tipo"
Write-Host " Destino: $s3ARN"
Write-Host "=============================================" -ForegroundColor Yellow

# ---- ETAPA 1: Iniciar Backup ----
Write-Step "Iniciando backup $tipo..."

$queryBackup = @"
EXEC msdb.dbo.rds_backup_database
    @source_db_name       = '$Banco',
    @s3_arn_to_backup_to  = '$s3ARN',
    @type                 = '$tipo',
    @overwrite_s3_backup_file = 1,
    @backup_compression   = 1;
"@

try {
    Invoke-Sqlcmd -ServerInstance $SqlEndpoint -Username $SqlUser -Password $SqlPassword `
                  -Query $queryBackup -QueryTimeout 60
    Write-OK "Backup iniciado."
} catch {
    Write-Err "Falha ao iniciar backup: $_"; exit 1
}

# ---- ETAPA 2: Monitorar Progresso ----
Write-Step "Monitorando progresso..."

$maxTentativas = 120
$tentativa     = 0
$status        = ""
$queryStatus   = "SELECT TOP 1 lifecycle, [% complete], task_info FROM msdb.dbo.rds_fn_task_status('$Banco',0) WHERE task_type='BACKUP' ORDER BY created_at DESC"

do {
    Start-Sleep -Seconds 30
    $tentativa++
    $r      = Invoke-Sqlcmd -ServerInstance $SqlEndpoint -Username $SqlUser -Password $SqlPassword -Query $queryStatus
    $status = $r.lifecycle
    Write-Host "  Status: $status | $($r.'% complete')%"
} while ($status -notin @('SUCCESS','ERROR') -and $tentativa -lt $maxTentativas)

if ($status -ne 'SUCCESS') {
    Write-Err "Backup falhou. Info: $($r.task_info)"; exit 1
}
Write-OK "Backup concluido: $s3ARN"

# ---- ETAPA 3: Limpeza de Antigos ----
Write-Step "Limpando backups com mais de $RetencaoDias dias..."

Import-Module AWS.Tools.S3 -ErrorAction SilentlyContinue

$limite  = (Get-Date).AddDays(-$RetencaoDias)
$antigos = Get-S3Object -BucketName $S3Bucket -Prefix "$S3Prefix/" |
           Where-Object { $_.LastModified -lt $limite }

if ($antigos) {
    $antigos | ForEach-Object {
        Remove-S3Object -BucketName $S3Bucket -Key $_.Key -Force
        Write-Host "  Deletado: $($_.Key)"
    }
    Write-OK "$($antigos.Count) objeto(s) antigo(s) removido(s)."
} else {
    Write-Host "  Nenhum objeto para remover."
}

Write-Host "=============================================" -ForegroundColor Yellow
Write-Host " PIPELINE CONCLUIDO COM SUCESSO"
Write-Host "=============================================" -ForegroundColor Yellow
```

---

## 5. Verificar Integridade do Backup no S3

```powershell
# ============================================================
# Verificar se um backup no S3 é válido (tamanho mínimo)
# ============================================================
param(
    [string]$Bucket         = "meu-bucket",
    [string]$Prefix         = "backups/",
    [int]   $TamanhoMinimoMB = 10   # Backup suspeito se menor que X MB
)

Import-Module AWS.Tools.S3

$objetos = Get-S3Object -BucketName $Bucket -Prefix $Prefix |
           Where-Object { $_.Key -like "*.bak" }

Write-Host "Verificando $($objetos.Count) backup(s):" -ForegroundColor Cyan

$alertas = 0
foreach ($obj in $objetos | Sort-Object LastModified -Descending) {
    $sizeMB = [math]::Round($obj.Size / 1MB, 2)
    $status = if ($sizeMB -lt $TamanhoMinimoMB) { "ALERTA"; $alertas++ } else { "OK" }
    $cor    = if ($status -eq "ALERTA") { "Red" } else { "Green" }
    Write-Host "  [$status] $($obj.Key) | $sizeMB MB | $($obj.LastModified.ToString('yyyy-MM-dd HH:mm'))" -ForegroundColor $cor
}

if ($alertas -gt 0) {
    Write-Host "`n$alertas backup(s) com tamanho suspeito!" -ForegroundColor Red
} else {
    Write-Host "`nTodos os backups parecem OK." -ForegroundColor Green
}
```

---

## 6. Copiar Backups Entre Buckets (Disaster Recovery)

```powershell
# ============================================================
# Copiar backups de um bucket para outro (DR / outra região)
# ============================================================
param(
    [string]$BucketOrigem  = "meu-bucket-sa-east-1",
    [string]$BucketDestino = "meu-bucket-us-east-1",
    [string]$Prefix        = "backups/",
    [int]   $DiasRecentes  = 7   # Somente backups dos últimos X dias
)

Import-Module AWS.Tools.S3

$limite  = (Get-Date).AddDays(-$DiasRecentes)
$objetos = Get-S3Object -BucketName $BucketOrigem -Prefix $Prefix |
           Where-Object { $_.LastModified -ge $limite }

Write-Host "Copiando $($objetos.Count) objeto(s) de $BucketOrigem para $BucketDestino..." -ForegroundColor Cyan

foreach ($obj in $objetos) {
    Copy-S3Object `
        -BucketName     $BucketOrigem `
        -Key            $obj.Key `
        -DestinationBucket $BucketDestino `
        -DestinationKey    $obj.Key

    Write-Host "  Copiado: $($obj.Key)"
}

Write-Host "Replicacao concluida." -ForegroundColor Green
```

---

## 7. Script de Diagnóstico do Ambiente S3

```powershell
# ============================================================
# Diagnóstico: verifica credenciais, acesso ao bucket e conectividade
# ============================================================
param(
    [string]$Bucket    = "meu-bucket",
    [string]$Prefix    = "backups/"
)

Write-Host "============ DIAGNOSTICO S3 ============" -ForegroundColor Yellow

# 1. Identidade
Write-Host "`n[1] Identidade AWS:" -ForegroundColor Cyan
try {
    $id = aws sts get-caller-identity | ConvertFrom-Json
    Write-Host "  Account: $($id.Account)"
    Write-Host "  ARN:     $($id.Arn)" -ForegroundColor Green
} catch {
    Write-Host "  ERRO: Credenciais inválidas ou não configuradas." -ForegroundColor Red
}

# 2. Listagem do bucket
Write-Host "`n[2] Acesso ao bucket s3://$Bucket/$Prefix :" -ForegroundColor Cyan
try {
    $lista = aws s3 ls "s3://$Bucket/$Prefix" 2>&1
    if ($LASTEXITCODE -eq 0) {
        Write-Host "  Acesso OK. Objetos encontrados: $($lista.Count)" -ForegroundColor Green
    } else {
        Write-Host "  ERRO: $lista" -ForegroundColor Red
    }
} catch {
    Write-Host "  ERRO: $_" -ForegroundColor Red
}

# 3. Tamanho total
Write-Host "`n[3] Tamanho total no prefixo:" -ForegroundColor Cyan
aws s3 ls "s3://$Bucket/$Prefix" --recursive --summarize |
    Select-String "Total" | ForEach-Object { Write-Host "  $_" }

Write-Host "`n============ FIM DO DIAGNOSTICO ============" -ForegroundColor Yellow
```
