# Amazon S3 — Configuração e Credenciais

> Configuração do ambiente para operar o S3 via linha de comando, sem acesso ao console AWS.

---

## 1. Opções de Acesso

| Ferramenta | Linguagem | Quando usar |
|---|---|---|
| **AWS CLI** | Shell / PowerShell | Operações simples, scripts de CI/CD |
| **AWS.Tools.S3** (PowerShell) | PowerShell | Automações complexas, integração com scripts existentes |
| **boto3** (Python) | Python | Aplicações, lambdas, scripts de dados |
| **AWS SDK .NET** | C# | Aplicações .NET integradas ao RDS |

---

## 2. AWS CLI — Instalação e Configuração

```powershell
# Instalar via winget (Windows)
winget install Amazon.AWSCLI

# Verificar instalação
aws --version
# aws-cli/2.x.x Python/3.x Windows/10

# Configurar credenciais (interativo)
aws configure
# AWS Access Key ID [None]:     AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]:   sa-east-1
# Default output format [None]: json

# Configurar perfil adicional (múltiplas contas/ambientes)
aws configure --profile homologacao
aws configure --profile producao

# Usar perfil específico em um comando
aws s3 ls --profile producao

# Definir perfil padrão para a sessão do PowerShell
$env:AWS_PROFILE = "producao"

# Verificar identidade configurada
aws sts get-caller-identity
```

### Onde ficam as credenciais

```
C:\Users\<usuario>\.aws\credentials   → Access Key / Secret Key
C:\Users\<usuario>\.aws\config        → Região, output, perfis
```

### Formato do arquivo credentials

```ini
[default]
aws_access_key_id     = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[producao]
aws_access_key_id     = AKIA...PRODUCAO
aws_secret_access_key = ...

[homologacao]
aws_access_key_id     = AKIA...HOMO
aws_secret_access_key = ...
```

---

## 3. Módulo AWS PowerShell — Instalação

```powershell
# Instalar módulo específico do S3 (mais leve)
Install-Module -Name AWS.Tools.S3 -Force -AllowClobber

# Ou instalar o módulo completo (todos os serviços AWS)
Install-Module -Name AWSPowerShell -Force -AllowClobber

# Verificar instalação
Get-Module -ListAvailable AWS*

# Importar na sessão
Import-Module AWS.Tools.S3

# Configurar credenciais via PowerShell (salva em ~/.aws/credentials)
Set-AWSCredential `
    -AccessKey "AKIAIOSFODNN7EXAMPLE" `
    -SecretKey "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY" `
    -StoreAs "producao"

# Definir região padrão
Set-DefaultAWSRegion -Region "sa-east-1"

# Usar perfil em um comando
Get-S3Bucket -ProfileName "producao"

# Verificar identidade
(Get-STSCallerIdentity).Arn
```

---

## 4. Credenciais via Variáveis de Ambiente (CI/CD / Scripts automatizados)

```powershell
# Definir via variáveis de ambiente (não salva em arquivo)
$env:AWS_ACCESS_KEY_ID     = "AKIAIOSFODNN7EXAMPLE"
$env:AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
$env:AWS_DEFAULT_REGION    = "sa-east-1"

# Verificar
aws sts get-caller-identity
```

---

## 5. Testar Acesso ao Bucket

```powershell
# Listar buckets disponíveis
aws s3 ls

# Verificar se um bucket específico está acessível
aws s3 ls s3://meu-bucket/

# Via PowerShell
Get-S3Bucket | Select-Object BucketName, CreationDate

# Verificar permissões no bucket
aws s3api get-bucket-acl --bucket meu-bucket
```

---

## 6. Regiões Comuns AWS (Brasil)

| Região | Código |
|---|---|
| São Paulo | `sa-east-1` |
| Leste dos EUA (N. Virginia) | `us-east-1` |
| Leste dos EUA (Ohio) | `us-east-2` |
| Oeste dos EUA (Oregon) | `us-west-2` |
| Europa (Irlanda) | `eu-west-1` |

```powershell
# Definir região para o comando (sobrescreve o default)
aws s3 ls s3://meu-bucket/ --region us-east-1
```
