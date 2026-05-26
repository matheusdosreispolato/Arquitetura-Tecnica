# Estrutura de Projetos Python

> Um projeto Python bem estruturado é previsível: qualquer pessoa que entre no repositório sabe imediatamente onde está cada coisa. Este documento define o padrão a seguir.

---

## Estrutura Padrão

```
meu_projeto/
│
├── src/                        ← código-fonte principal
│   └── meu_projeto/
│       ├── __init__.py
│       ├── main.py             ← ponto de entrada
│       ├── config.py           ← configurações e variáveis de ambiente
│       ├── models/             ← classes de domínio / schemas
│       │   └── __init__.py
│       ├── services/           ← lógica de negócio
│       │   └── __init__.py
│       └── utils/              ← funções auxiliares reutilizáveis
│           └── __init__.py
│
├── tests/                      ← testes automatizados
│   ├── __init__.py
│   ├── conftest.py             ← fixtures compartilhadas do pytest
│   ├── unit/                   ← testes unitários
│   └── integration/            ← testes de integração
│
├── docs/                       ← documentação adicional (opcional)
├── scripts/                    ← scripts avulsos de manutenção ou migração
│
├── .env.example                ← variáveis de ambiente de exemplo (sem valores reais)
├── .gitignore
├── pyproject.toml              ← configuração central do projeto
├── requirements.txt            ← dependências (gerado do pyproject.toml)
└── README.md
```

> **Por que `src/`?** Evita que o Python importe o pacote local em vez do instalado, o que causa bugs sutis em testes e CI. É o padrão recomendado pela Python Packaging Authority.

---

## Ambientes Virtuais

Sempre isole as dependências do projeto em um ambiente virtual. Nunca instale pacotes diretamente no Python global do sistema.

### venv (padrão — sem instalação adicional)

```bash
# Criar o ambiente virtual
python -m venv .venv

# Ativar
source .venv/bin/activate        # Linux / macOS
.venv\Scripts\activate           # Windows (PowerShell)
.venv\Scripts\activate.bat       # Windows (CMD)

# Desativar
deactivate
```

### pyenv (gerenciar múltiplas versões do Python)

```bash
# Instalar uma versão específica
pyenv install 3.12.0

# Definir a versão para este projeto
pyenv local 3.12.0

# Verificar
python --version   # → Python 3.12.0
```

### Boas práticas com ambientes virtuais

- Sempre adicione `.venv/` ao `.gitignore`
- Nunca commite o conteúdo do `.venv/`
- Documente a versão mínima do Python no `pyproject.toml`
- Use `python -m pip` (não apenas `pip`) para garantir que está instalando no ambiente correto

---

## pyproject.toml — Configuração Central

O `pyproject.toml` é o arquivo único que centraliza toda a configuração do projeto: metadados, dependências, ferramentas de linting, formatação e testes.

```toml
[build-system]
requires      = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name        = "meu-projeto"
version     = "0.1.0"
description = "Descrição curta do projeto"
readme      = "README.md"
requires-python = ">=3.11"

# Dependências de produção
dependencies = [
    "requests>=2.31",
    "pydantic>=2.0",
    "python-dotenv>=1.0",
]

[project.optional-dependencies]
# Dependências de desenvolvimento (não vão para produção)
dev = [
    "pytest>=7.4",
    "pytest-cov>=4.1",
    "ruff>=0.1",
    "black>=23.0",
    "mypy>=1.5",
]

[tool.setuptools.packages.find]
where = ["src"]

# ── Configuração do Ruff (linter) ────────────────────────────────────────────
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B"]
# E/F = erros básicos | I = imports | N = nomenclatura | UP = upgrade syntax | B = bugbear

# ── Configuração do Black (formatador) ───────────────────────────────────────
[tool.black]
line-length    = 100
target-version = ["py311"]

# ── Configuração do mypy (tipagem) ───────────────────────────────────────────
[tool.mypy]
python_version         = "3.11"
strict                 = true
ignore_missing_imports = true

# ── Configuração do pytest ───────────────────────────────────────────────────
[tool.pytest.ini_options]
testpaths    = ["tests"]
addopts      = "-v --cov=src --cov-report=term-missing"
```

---

## Gerenciamento de Dependências

### Instalação

```bash
# Instalar dependências de produção
pip install -e .

# Instalar com dependências de desenvolvimento
pip install -e ".[dev]"
```

### Gerar requirements.txt (para deploy ou Docker)

```bash
pip freeze > requirements.txt

# Ou usando pip-compile (mais preciso, lista apenas o necessário)
pip install pip-tools
pip-compile pyproject.toml
```

### Atualizar dependências

```bash
pip list --outdated          # ver o que está desatualizado
pip install --upgrade pacote
```

---

## Variáveis de Ambiente

Nunca coloque credenciais, URLs de banco de dados ou chaves de API no código. Use variáveis de ambiente.

### .env.example (commitar — sem valores reais)

```ini
# Banco de dados
DB_HOST=localhost
DB_PORT=5432
DB_NAME=nome_do_banco
DB_USER=usuario
DB_PASSWORD=

# API
API_KEY=
API_BASE_URL=https://api.exemplo.com

# Ambiente
ENVIRONMENT=development   # development | staging | production
LOG_LEVEL=INFO
```

### .env (NÃO commitar — valores reais)

```ini
DB_HOST=meu-servidor.exemplo.com
DB_PORT=1433
DB_NAME=MeuBanco
DB_USER=app_pipeline
DB_PASSWORD=SenhaSegura123!

API_KEY=sk-...
ENVIRONMENT=production
LOG_LEVEL=WARNING
```

### config.py — Leitura centralizada das variáveis

```python
# src/meu_projeto/config.py
from pydantic_settings import BaseSettings
from functools import lru_cache


class Settings(BaseSettings):
    # Banco de dados
    db_host: str
    db_port: int = 1433
    db_name: str
    db_user: str
    db_password: str

    # API
    api_key: str
    api_base_url: str = "https://api.exemplo.com"

    # Ambiente
    environment: str = "development"
    log_level: str = "INFO"

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"


@lru_cache
def get_settings() -> Settings:
    """Retorna as configurações do projeto (cached)."""
    return Settings()
```

### Uso no código

```python
from meu_projeto.config import get_settings

settings = get_settings()
print(settings.db_host)     # meu-servidor.exemplo.com
print(settings.environment) # production
```

---

## .gitignore padrão para Python

```gitignore
# Ambiente virtual
.venv/
venv/
env/

# Python
__pycache__/
*.py[cod]
*.pyo
*.pyd
*.so
*.egg
*.egg-info/
dist/
build/

# Variáveis de ambiente (NUNCA commitar)
.env
.env.local
.env.*.local

# Testes e cobertura
.pytest_cache/
.coverage
coverage.xml
htmlcov/

# Ferramentas
.ruff_cache/
.mypy_cache/
.DS_Store

# IDEs
.vscode/
.idea/
*.swp
```

---

## Estruturas por Tipo de Projeto

### Projeto de Script / Automação

```
automacao_relatorio/
├── src/automacao_relatorio/
│   ├── __init__.py
│   ├── main.py          ← executa o pipeline
│   ├── extrator.py      ← busca os dados
│   ├── transformador.py ← processa os dados
│   └── exportador.py    ← gera o arquivo de saída
├── tests/
├── .env.example
└── pyproject.toml
```

### Projeto de API (FastAPI)

```
minha_api/
├── src/minha_api/
│   ├── __init__.py
│   ├── main.py          ← app FastAPI
│   ├── routers/         ← endpoints por domínio
│   ├── schemas/         ← Pydantic models (request/response)
│   ├── services/        ← lógica de negócio
│   └── database/        ← conexão e modelos ORM
├── tests/
├── Dockerfile
└── pyproject.toml
```

### Projeto de IA / LLM

```
agente_dados/
├── src/agente_dados/
│   ├── __init__.py
│   ├── main.py
│   ├── prompts/         ← arquivos .txt ou .jinja com prompts
│   ├── tools/           ← ferramentas disponíveis para o agente
│   ├── memory/          ← implementação de memória
│   └── evals/           ← scripts de avaliação
├── tests/
├── .env.example
└── pyproject.toml
```

> Para projetos de IA, consulte também `IA/01-estrutura-projeto-ia.md`.
