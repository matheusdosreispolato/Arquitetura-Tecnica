# Usuários e Permissões no AWS RDS para SQL Server

> Existem **dois níveis distintos** de gerenciamento de usuários no RDS. Entender essa separação é essencial para não confundir o que é feito na infraestrutura AWS com o que é feito dentro do banco de dados.

---

## Os dois níveis

```
┌─────────────────────────────────────────────────────────┐
│  NÍVEL 1 — RDS / AWS                                    │
│  Quem gerencia: AWS CLI / PowerShell AWS SDK            │
│  O que controla: usuário master, IAM auth,              │
│                  senha do admin, acesso à instância     │
├─────────────────────────────────────────────────────────┤
│  NÍVEL 2 — SQL Server                                   │
│  Quem gerencia: T-SQL (sqlcmd / SSMS)                   │
│  O que controla: logins SQL, usuários de banco,         │
│                  roles, permissões por objeto/schema    │
└─────────────────────────────────────────────────────────┘
```

---

## 1. Usuário Master (admin) — Nível RDS

O usuário master é criado **no momento do provisionamento da instância RDS**. Ele não pode ser criado via T-SQL após o fato — é gerenciado pela AWS.

### ☁️ Nível RDS — AWS CLI

```powershell
# Verificar qual é o usuário master atual da instância
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].MasterUsername" `
    --output text

# Alterar a senha do usuário master (sem acesso ao console)
aws rds modify-db-instance `
    --db-instance-identifier minha-instancia-rds `
    --master-user-password "NovaSenha@Segura2024!" `
    --apply-immediately

# Aguardar a instância voltar ao status "available"
aws rds wait db-instance-available `
    --db-instance-identifier minha-instancia-rds

# Verificar status da instância após a modificação
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].{Status:DBInstanceStatus, MasterUser:MasterUsername}" `
    --output table
```

### 🗄️ Nível SQL Server — T-SQL

```sql
-- O usuário master aparece como login SQL no servidor
SELECT name, type_desc, is_disabled, default_database_name
FROM sys.server_principals
WHERE type = 'S'
ORDER BY name;

-- Verificar permissões do usuário atual (master/admin)
SELECT * FROM fn_my_permissions(NULL, 'SERVER');
SELECT * FROM fn_my_permissions(NULL, 'DATABASE');

-- O usuário master pode alterar sua própria senha via T-SQL também
ALTER LOGIN [admin] WITH PASSWORD = 'NovaSenha@Segura2024!';
```

> **Importante:** O usuário master no RDS tem poderes equivalentes a `sysadmin`, mas não pode:
> - Adicionar outros usuários à role `sysadmin`
> - Acessar arquivos do sistema operacional
> - Usar `xp_cmdshell`

---

## 2. Autenticação IAM — Nível RDS

O RDS permite que usuários se conectem usando credenciais IAM da AWS, sem senha SQL fixa.

### ☁️ Nível RDS — AWS CLI

```powershell
# Habilitar autenticação IAM na instância
aws rds modify-db-instance `
    --db-instance-identifier minha-instancia-rds `
    --enable-iam-database-authentication `
    --apply-immediately

# Verificar se IAM auth está habilitado
aws rds describe-db-instances `
    --db-instance-identifier minha-instancia-rds `
    --query "DBInstances[0].IAMDatabaseAuthenticationEnabled" `
    --output text

# Gerar token de autenticação IAM (válido por 15 minutos)
aws rds generate-db-auth-token `
    --hostname meu-rds.xxxx.sa-east-1.rds.amazonaws.com `
    --port 1433 `
    --region sa-east-1 `
    --username app_iam_user
```

### 🗄️ Nível SQL Server — T-SQL

```sql
-- Criar login SQL associado à autenticação IAM
-- (o login deve existir no SQL Server para que o token IAM funcione)
CREATE LOGIN app_iam_user WITH PASSWORD = 'SenhaTemporaria@1!';
GO

USE MeuBanco;
CREATE USER app_iam_user FOR LOGIN app_iam_user;
ALTER ROLE db_datareader ADD MEMBER app_iam_user;
GO
```

---

## 3. Logins SQL Server — Nível SQL

Logins comuns criados e gerenciados diretamente no SQL Server via T-SQL. **Este nível é independente da AWS** — é SQL Server puro.

### 🗄️ Nível SQL Server — T-SQL

```sql
-- Criar login SQL
CREATE LOGIN app_vendas   WITH PASSWORD = 'App@Vendas2024!';
CREATE LOGIN svc_erp      WITH PASSWORD = 'Svc@Erp2024!';
CREATE LOGIN ro_relatorio WITH PASSWORD = 'Ro@Rel2024!';
GO

-- Listar todos os logins SQL
SELECT
    name,
    type_desc,
    is_disabled,
    default_database_name,
    create_date,
    modify_date
FROM sys.server_principals
WHERE type = 'S'        -- S = SQL Login
ORDER BY name;

-- Alterar senha de um login
ALTER LOGIN app_vendas WITH PASSWORD = 'App@NovaS3nha2024!';

-- Desabilitar / Habilitar login
ALTER LOGIN app_vendas DISABLE;
ALTER LOGIN app_vendas ENABLE;

-- Remover login
DROP LOGIN IF EXISTS app_vendas;
```

> **Diferença do MSSQL padrão:** No RDS, **logins Windows (Active Directory local) não são suportados** via T-SQL. Para AD, configure via AWS Directory Service (requer ajuste na instância RDS).

---

## 4. Usuários de Banco de Dados — Nível SQL

Um login existe no servidor, mas precisa de um **usuário** para acessar um banco específico.

```
Login (nível servidor) → mapeado para → Usuário (nível banco)
```

### 🗄️ Nível SQL Server — T-SQL

```sql
-- Criar usuário para um login existente
USE MeuBanco;
GO

CREATE USER app_vendas   FOR LOGIN app_vendas;
CREATE USER svc_erp      FOR LOGIN svc_erp;
CREATE USER ro_relatorio FOR LOGIN ro_relatorio;
GO

-- Listar usuários do banco atual
SELECT
    name,
    type_desc,
    default_schema_name,
    authentication_type_desc,
    create_date
FROM sys.database_principals
WHERE type IN ('S', 'U')
  AND name NOT IN ('dbo','guest','INFORMATION_SCHEMA','sys')
ORDER BY name;

-- Remover usuário do banco (não remove o login do servidor)
DROP USER IF EXISTS app_vendas;
```

---

## 5. Roles e Permissões — Nível SQL

### 🗄️ Roles fixas do banco

```sql
-- Leitura em todas as tabelas
ALTER ROLE db_datareader ADD MEMBER ro_relatorio;

-- Leitura + escrita em todas as tabelas
ALTER ROLE db_datareader ADD MEMBER app_vendas;
ALTER ROLE db_datawriter ADD MEMBER app_vendas;

-- Controle total no banco (cuidado)
ALTER ROLE db_owner ADD MEMBER dba_suporte;

-- Remover de role
ALTER ROLE db_datawriter REMOVE MEMBER app_vendas;

-- Ver membros de todas as roles
SELECT
    rp.name  AS Role,
    dp.name  AS Membro,
    dp.type_desc
FROM sys.database_role_members drm
JOIN sys.database_principals rp ON rp.principal_id = drm.role_principal_id
JOIN sys.database_principals dp ON dp.principal_id = drm.member_principal_id
ORDER BY rp.name, dp.name;
```

### 🗄️ Permissões granulares por objeto

```sql
-- Por tabela
GRANT SELECT, INSERT, UPDATE ON vendas.Pedido TO app_vendas;
DENY  DELETE                  ON vendas.Pedido TO app_vendas;

-- Por schema inteiro
GRANT SELECT ON SCHEMA::vendas TO ro_relatorio;

-- Por procedure
GRANT EXECUTE ON vendas.usp_CancelarPedido TO app_vendas;

-- Revogar permissão
REVOKE DELETE ON vendas.Pedido FROM app_vendas;

-- Ver permissões de um usuário específico
SELECT
    dp.name             AS Usuario,
    ISNULL(o.name,'')   AS Objeto,
    p.permission_name,
    p.state_desc        AS Acao    -- GRANT ou DENY
FROM sys.database_permissions p
JOIN sys.database_principals dp ON dp.principal_id = p.grantee_principal_id
LEFT JOIN sys.objects o          ON o.object_id = p.major_id
WHERE dp.name = 'app_vendas'
ORDER BY p.permission_name;
```

### 🗄️ Criar roles customizadas

```sql
-- Role de leitura no schema vendas
CREATE ROLE role_vendas_readonly;
GRANT SELECT ON SCHEMA::vendas TO role_vendas_readonly;
ALTER ROLE role_vendas_readonly ADD MEMBER ro_relatorio;

-- Role de execução de procedures
CREATE ROLE role_vendas_execute;
GRANT EXECUTE ON SCHEMA::vendas TO role_vendas_execute;
ALTER ROLE role_vendas_execute ADD MEMBER app_vendas;
ALTER ROLE role_vendas_execute ADD MEMBER svc_erp;
```

---

## 6. Perfis Prontos para Uso

### 🗄️ Nível SQL Server — T-SQL

```sql
-- ===== USUÁRIO DE APLICAÇÃO =====
CREATE LOGIN app_portal WITH PASSWORD = 'App@Portal2024!';
GO
USE MeuBanco;
CREATE USER app_portal FOR LOGIN app_portal;
ALTER ROLE db_datareader  ADD MEMBER app_portal;
ALTER ROLE db_datawriter  ADD MEMBER app_portal;
GRANT EXECUTE ON SCHEMA::vendas TO app_portal;
GO

-- ===== USUÁRIO SOMENTE LEITURA =====
CREATE LOGIN ro_bi WITH PASSWORD = 'Ro@BI2024!';
GO
USE MeuBanco;
CREATE USER ro_bi FOR LOGIN ro_bi;
ALTER ROLE db_datareader ADD MEMBER ro_bi;
GO

-- ===== USUÁRIO DE SERVIÇO / INTEGRAÇÃO =====
CREATE LOGIN svc_integracao WITH PASSWORD = 'Svc@Int2024!';
GO
USE MeuBanco;
CREATE USER svc_integracao FOR LOGIN svc_integracao;
GRANT SELECT, INSERT, UPDATE ON SCHEMA::integracao TO svc_integracao;
GRANT EXECUTE              ON SCHEMA::integracao TO svc_integracao;
GO

-- ===== DBA DE SUPORTE (sem sysadmin) =====
CREATE LOGIN dba_suporte WITH PASSWORD = 'Dba@Sup2024!';
GO
USE MeuBanco;
CREATE USER dba_suporte FOR LOGIN dba_suporte;
ALTER ROLE db_owner ADD MEMBER dba_suporte;
GO
```

---

## 7. Monitorar Sessões Ativas

### 🗄️ Nível SQL Server — T-SQL

```sql
-- Sessões ativas por login
SELECT
    s.session_id,
    s.login_name,
    s.host_name,
    s.program_name,
    s.status,
    s.login_time,
    r.blocking_session_id AS BloqueadoPor,
    t.text                AS QueryAtual
FROM sys.dm_exec_sessions s
LEFT JOIN sys.dm_exec_requests r ON r.session_id = s.session_id
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE s.is_user_process = 1
ORDER BY s.login_name;

-- Encerrar sessão específica
KILL 55;   -- substitua pelo session_id
```

### ☁️ Nível RDS — AWS CLI

```powershell
# Ver eventos recentes da instância (logins, erros de autenticação)
aws rds describe-events `
    --source-identifier minha-instancia-rds `
    --source-type db-instance `
    --duration 60 `
    --output table

# Reiniciar instância (força desconexão de todas as sessões)
aws rds reboot-db-instance `
    --db-instance-identifier minha-instancia-rds

# Aguardar instância voltar online
aws rds wait db-instance-available `
    --db-instance-identifier minha-instancia-rds
```

---

## 8. Resumo — Onde fazer cada ação

| Ação | Onde fazer | Ferramenta |
|---|---|---|
| Criar usuário master | Provisionamento da instância | Console / CLI (uma vez) |
| Alterar senha do master | Nível RDS | `aws rds modify-db-instance` |
| Habilitar IAM auth | Nível RDS | `aws rds modify-db-instance` |
| Criar login SQL | Nível SQL Server | T-SQL: `CREATE LOGIN` |
| Criar usuário de banco | Nível SQL Server | T-SQL: `CREATE USER` |
| Atribuir role | Nível SQL Server | T-SQL: `ALTER ROLE` |
| Dar permissão por objeto | Nível SQL Server | T-SQL: `GRANT / DENY` |
| Ver sessões ativas | Nível SQL Server | T-SQL: `sys.dm_exec_sessions` |
| Ver eventos de autenticação | Nível RDS | `aws rds describe-events` |
| Reiniciar instância | Nível RDS | `aws rds reboot-db-instance` |
