# Usando Claude Code e Codex em Projetos

> Claude Code (Anthropic) e Codex/ChatGPT (OpenAI) são agentes de IA que operam diretamente no seu repositório. Para que funcionem bem, eles precisam entender o projeto — e isso se faz com documentação estratégica: `AGENTS.md`, `README.md` nas subpastas e arquivos de contexto.

---

## O que são Claude Code e Codex

| Ferramenta | O que é | Como opera |
|---|---|---|
| **Claude Code** | CLI da Anthropic que roda no terminal | Lê arquivos, escreve código, executa comandos — dentro do seu repositório |
| **OpenAI Codex** (agente) | Agente da OpenAI no ChatGPT/API | Sandbox isolado com acesso ao código via upload ou integração |

Ambos funcionam melhor quando encontram documentação clara sobre **o que o projeto faz**, **como está organizado** e **quais convenções seguir**. Sem isso, o agente lê o código às cegas e toma decisões que podem não se encaixar no padrão do projeto.

---

## AGENTS.md — O Arquivo de Briefing

O `AGENTS.md` é lido automaticamente pelo Claude Code antes de qualquer tarefa. É o "briefing do projeto" — o que qualquer colaborador (humano ou IA) precisa saber para trabalhar corretamente.

### O que colocar no AGENTS.md

```markdown
# AGENTS.md

## O que é este projeto
[Uma ou duas frases explicando o propósito]

## Estrutura de pastas
[Árvore de diretórios com descrição de cada pasta]

## Como rodar localmente
[Comandos mínimos para instalar dependências e executar]

## Convenções obrigatórias
[Padrões de código, nomenclatura, estrutura de arquivos]

## O que NÃO fazer
[Restrições importantes: não mockar banco, não usar print, etc.]

## Como adicionar conteúdo novo
[Passo a passo para criar um novo módulo, endpoint, arquivo]
```

### Exemplo real (projeto FastAPI)

```markdown
# AGENTS.md

## O que é este projeto
API REST de gestão de pedidos em FastAPI + PostgreSQL.
Autenticação via JWT. Deploy em Docker.

## Estrutura
src/minha_api/
├── routers/      ← endpoints HTTP (sem lógica de negócio)
├── services/     ← lógica de negócio (sem SQL direto)
├── repositories/ ← todo acesso ao banco (só aqui tem SQL)
├── schemas/      ← Pydantic models de entrada e saída
└── models/       ← SQLAlchemy ORM models

## Rodar localmente
docker-compose up -d
pip install -e ".[dev]"
uvicorn minha_api.main:app --reload

## Convenções
- Nomenclatura: snake_case para funções/variáveis, PascalCase para classes
- Nunca colocar lógica de negócio no router
- Nunca colocar queries SQL fora do repository
- Type hints obrigatórios em todas as funções públicas
- Docstrings no padrão Google

## Nunca fazer
- print() em código de produção — usar logging
- Credenciais no código — usar .env
- Importar o banco diretamente no service — usar repository

## Adicionar novo endpoint
1. Criar schema em schemas/<dominio>.py
2. Criar repository em repositories/<dominio>_repository.py
3. Criar service em services/<dominio>_service.py
4. Criar router em routers/<dominio>.py
5. Registrar em main.py com app.include_router(...)
6. Criar testes em tests/integration/test_<dominio>_api.py
```

---

## README.md nas Subpastas

Cada pasta deve ter um `README.md` que explica o que ela contém. Isso serve tanto para desenvolvedores humanos quanto para o agente navegar sem precisar ler todos os arquivos.

### Por que isso importa para o agente

Quando você pede "crie um novo endpoint de clientes", o agente precisa saber:
- Onde estão os outros endpoints (para seguir o mesmo padrão)
- Quais arquivos já existem
- Como o router existente está estruturado

Com README.md em cada pasta, o agente encontra isso em segundos sem precisar abrir arquivo por arquivo.

### Template de README.md por pasta

```markdown
# [Nome da Pasta] — [Descrição em uma linha]

> [Para que serve esta pasta — uma frase]

## Arquivos

| Arquivo | Responsabilidade |
|---|---|
| `pedido_service.py` | Lógica de negócio: criação, cancelamento, cálculo de desconto |
| `cliente_service.py` | Lógica de negócio: cadastro, busca, validação de CPF |

## Convenções desta pasta

- Todo service recebe o repository via `__init__` (injeção de dependência)
- Métodos públicos retornam schemas Pydantic, nunca ORM models
- Lançar exceções de domínio (nunca HTTPException — isso fica no router)

## Adicionar novo service

1. Criar arquivo `<dominio>_service.py`
2. Injetar o repository correspondente no `__init__`
3. Criar testes em `tests/unit/test_<dominio>_service.py`
```

---

## Estrutura de Projeto Pensada para Agentes

Um projeto legível por agentes tem as seguintes características:

### 1. Convenções explícitas e documentadas

```
# ❌ Convenção implícita (o agente não sabe)
"Todo mundo sabe que services não devem ter SQL..."

# ✅ Convenção no AGENTS.md
"## Regra obrigatória
Queries SQL só existem dentro de repositories/.
Se um service precisar de dado do banco, cria ou usa um método no repository."
```

### 2. Exemplos de referência apontados explicitamente

```markdown
## Referência
Para criar um novo endpoint, siga o padrão de `routers/pedidos.py`.
Para criar um novo service, siga `services/pedido_service.py`.
```

### 3. Comandos de verificação documentados

```markdown
## Verificação antes de commitar
ruff check .        # linting
black --check .     # formatação
mypy src/           # tipos
pytest              # testes
```

### 4. Contexto de negócio mínimo

O agente gera código melhor quando entende o domínio:

```markdown
## Domínio
Sistema de e-commerce B2B. Clientes são empresas (não pessoas físicas).
Pedidos passam pelos status: RASCUNHO → PENDENTE → APROVADO → ENVIADO → ENTREGUE.
Um pedido só pode ser cancelado se estiver em RASCUNHO ou PENDENTE.
```

---

## Como Instruir o Agente por Tarefa

Além do `AGENTS.md` global, você pode dar contexto específico para cada tarefa:

### Prompt de tarefa bem escrito

```
Crie o endpoint POST /clientes seguindo o padrão de POST /pedidos em routers/pedidos.py.

O schema de entrada deve ter: nome (str), cnpj (str), email (str).
Validar que o CNPJ tem 14 dígitos numéricos.
O service deve verificar se já existe cliente com o mesmo CNPJ antes de criar.

Arquivos a criar:
- schemas/cliente.py
- repositories/cliente_repository.py
- services/cliente_service.py
- routers/clientes.py (registrar em main.py)
- tests/integration/test_clientes_api.py
```

### O que torna um prompt de tarefa eficaz

| Elemento | Por quê ajuda |
|---|---|
| Referência a arquivo existente ("siga o padrão de X") | Agente copia estrutura em vez de inventar |
| Lista de arquivos a criar | Agente sabe o escopo da tarefa |
| Regras de negócio explícitas | Agente implementa corretamente na primeira vez |
| Critérios de validação | Agente escreve o teste correto |

---

## Reduzindo Tokens na Interação com Agentes

Claude Code e Codex leem o repositório a cada tarefa. Quanto maior o projeto, mais tokens são consumidos só para "entender o contexto".

**Estratégias para economizar:**

| Estratégia | Impacto |
|---|---|
| AGENTS.md enxuto e objetivo | Menos tokens gastos no briefing inicial |
| README.md em subpastas (vs. tudo no raiz) | Agente carrega apenas o contexto da pasta relevante |
| Nomes de arquivos descritivos | Agente entende sem precisar abrir o arquivo |
| Funções curtas e bem nomeadas | Agente entende sem ler a implementação inteira |
| `.claudeignore` / `.gitignore` | Excluir pastas irrelevantes (node_modules, .venv, dist) |

### .claudeignore (Claude Code)

Equivalente ao `.gitignore` para o Claude Code — exclui arquivos da leitura do agente:

```
.venv/
__pycache__/
*.pyc
.mypy_cache/
.ruff_cache/
dist/
build/
*.egg-info/
notebooks/
evals/results/
```

---

## Fluxo de Trabalho com Claude Code

```
1. AGENTS.md e README.md nas pastas já existem
        ↓
2. Abrir terminal na raiz do projeto
        ↓
3. Executar: claude
        ↓
4. Descrever a tarefa com contexto suficiente
   (mencionar arquivos de referência, regras de negócio, arquivos a criar)
        ↓
5. Revisar o código gerado antes de aceitar
        ↓
6. Rodar verificações: ruff, mypy, pytest
        ↓
7. Commitar
```

> Claude Code lê automaticamente `AGENTS.md`, `CLAUDE.md` e `README.md` da raiz ao iniciar. Subpastas são lidas sob demanda quando o agente navega até elas.

---

## Relacionado

- `IA/02-engenharia-prompts.md` — como escrever prompts eficazes (tokens, técnicas, templates)
- `IA/01-estrutura-projeto-ia.md` — estrutura de projeto quando o produto *é* um sistema de IA
- `Agents/01-conceitos.md` — diferença entre agente e chamada simples à API
