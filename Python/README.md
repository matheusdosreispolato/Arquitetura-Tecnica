# Python — Padronização de Projetos

> Guia de boas práticas para estruturar e organizar projetos Python de forma profissional, mantível e colaborativa.

---

## Estrutura desta pasta

```
Python/
├── README.md                 ← este arquivo
├── 01-estrutura-projeto.md   ← estrutura de pastas, ambientes virtuais, pyproject.toml
├── 02-boas-praticas.md       ← tipagem, docstrings, linting, formatação, testes, nomenclatura
└── 03-arquitetura-web.md     ← FastAPI, camadas (router/service/repository), schemas, Docker
```

---

## Documentos

| Arquivo | Conteúdo |
|---|---|
| [01 — Estrutura de Projeto](./01-estrutura-projeto.md) | Layout de pastas, ambientes virtuais (venv/pyenv), gerenciamento de dependências (pyproject.toml, requirements.txt), configuração inicial |
| [02 — Boas Práticas](./02-boas-praticas.md) | Tipagem estática, docstrings, linting (Ruff/Flake8), formatação (Black), testes (pytest), logging, convenções de nomenclatura (variáveis, funções, classes, constantes) |
| [03 — Arquitetura Web](./03-arquitetura-web.md) | Projeto FastAPI com camadas (Router → Service → Repository), schemas Pydantic, autenticação JWT, testes de API, Docker |

---

## Referência Rápida

```bash
# Criar projeto novo
mkdir meu_projeto && cd meu_projeto
python -m venv .venv
source .venv/bin/activate        # Linux/macOS
.venv\Scripts\activate           # Windows

# Instalar dependências
pip install -r requirements.txt

# Rodar testes
pytest

# Checar qualidade do código
ruff check .
```
