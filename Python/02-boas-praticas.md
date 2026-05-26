# Boas Práticas em Python

> Escrever código que funciona é o mínimo. Escrever código que qualquer pessoa da equipe consegue ler, manter e evoluir é o objetivo. Este documento cobre as práticas que fazem essa diferença.

---

## Tipagem Estática

Python é dinamicamente tipado, mas **anotações de tipo** tornam o código muito mais legível e permitem que ferramentas como o `mypy` detectem bugs antes da execução.

```python
# ❌ Sem tipagem — o que é `dados`? O que retorna?
def processar(dados, modo):
    ...

# ✅ Com tipagem — auto-documentado e verificável
from typing import Literal

def processar(dados: list[dict], modo: Literal["incremental", "full"]) -> int:
    """Processa os dados e retorna o total de registros afetados."""
    ...
```

### Tipos mais usados

```python
from typing import Optional, Any
from collections.abc import Callable, Generator

# Tipos básicos
nome: str = "João"
idade: int = 30
ativo: bool = True
preco: float = 9.99

# Coleções
ids: list[int] = [1, 2, 3]
config: dict[str, Any] = {"host": "localhost", "port": 5432}
opcoes: tuple[str, ...] = ("A", "B", "C")
unicos: set[str] = {"bronze", "silver", "gold"}

# Opcional (pode ser None)
usuario: Optional[str] = None        # equivale a: str | None
usuario: str | None = None           # sintaxe moderna (Python 3.10+)

# Callable
callback: Callable[[int, str], bool] = lambda n, s: True

# Retorno de função que não retorna nada
def registrar(msg: str) -> None:
    print(msg)
```

### Pydantic para validação de dados

```python
from pydantic import BaseModel, field_validator
from datetime import date
from decimal import Decimal


class Pedido(BaseModel):
    pedido_id: str
    cliente_id: str
    data_pedido: date
    valor_total: Decimal
    status: str

    @field_validator("valor_total")
    @classmethod
    def valor_deve_ser_positivo(cls, v: Decimal) -> Decimal:
        if v <= 0:
            raise ValueError("Valor total deve ser maior que zero")
        return v

    @field_validator("status")
    @classmethod
    def status_valido(cls, v: str) -> str:
        validos = {"PENDENTE", "PAGO", "CANCELADO"}
        if v.upper() not in validos:
            raise ValueError(f"Status inválido. Use: {validos}")
        return v.upper()


# Uso — valida automaticamente ao instanciar
pedido = Pedido(
    pedido_id="P-1234",
    cliente_id="C-789",
    data_pedido="2024-06-15",
    valor_total="150.00",
    status="pago",  # normalizado para "PAGO"
)
```

---

## Docstrings

Docstrings documentam **o que** a função faz, **quais parâmetros** recebe e **o que retorna** — não como ela funciona internamente (isso é comentário).

### Padrão Google (recomendado)

```python
def carregar_dados(
    caminho: str,
    encoding: str = "utf-8",
    separador: str = ",",
) -> list[dict]:
    """Lê um arquivo CSV e retorna os dados como lista de dicionários.

    Args:
        caminho: Caminho absoluto ou relativo para o arquivo CSV.
        encoding: Encoding do arquivo. Padrão: utf-8.
        separador: Caractere separador de colunas. Padrão: vírgula.

    Returns:
        Lista de dicionários onde cada item representa uma linha do CSV.

    Raises:
        FileNotFoundError: Se o arquivo não existir no caminho informado.
        ValueError: Se o arquivo estiver vazio ou mal formatado.

    Example:
        >>> dados = carregar_dados("pedidos.csv")
        >>> print(len(dados))
        1500
    """
    ...
```

### Quando escrever docstring

- **Sempre:** funções públicas, classes, módulos
- **Opcional:** funções privadas simples (prefixo `_`)
- **Nunca:** código óbvio que se autodescreve

---

## Logging

Nunca use `print()` em código de produção. Use o módulo `logging` — permite controlar o nível de saída, formatar mensagens e enviar para arquivos ou sistemas externos.

```python
# src/meu_projeto/logging_config.py
import logging
import sys
from meu_projeto.config import get_settings


def configurar_logging() -> None:
    """Configura o logging da aplicação."""
    settings = get_settings()

    logging.basicConfig(
        level=getattr(logging, settings.log_level.upper(), logging.INFO),
        format="%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
        handlers=[
            logging.StreamHandler(sys.stdout),
            # logging.FileHandler("app.log"),  # descomente para gravar em arquivo
        ],
    )
```

```python
# Uso em qualquer módulo
import logging

logger = logging.getLogger(__name__)  # nome do módulo como identificador

def processar_bronze(dados: list[dict]) -> int:
    logger.info("Iniciando processamento Bronze: %d registros", len(dados))

    try:
        # ... processamento ...
        logger.info("Bronze concluído: %d inseridos", inseridos)
        return inseridos
    except Exception as e:
        logger.error("Falha no processamento Bronze: %s", e, exc_info=True)
        raise
```

### Níveis de log

| Nível | Quando usar |
|---|---|
| `DEBUG` | Informações detalhadas para diagnóstico em desenvolvimento |
| `INFO` | Fluxo normal da aplicação (início/fim de etapas) |
| `WARNING` | Algo inesperado mas não crítico (dado faltando, fallback usado) |
| `ERROR` | Falha em uma operação (com `exc_info=True` para stack trace) |
| `CRITICAL` | Falha que impede a aplicação de continuar |

---

## Linting e Formatação

### Ruff (linter — substitui Flake8, isort, pyupgrade)

```bash
# Verificar problemas
ruff check .

# Corrigir automaticamente o que for possível
ruff check --fix .

# Verificar um arquivo específico
ruff check src/meu_projeto/main.py
```

### Black (formatador automático)

```bash
# Formatar todos os arquivos
black .

# Verificar sem modificar (para CI)
black --check .
```

### mypy (verificação de tipos)

```bash
# Verificar tipos
mypy src/

# Verificar um arquivo específico
mypy src/meu_projeto/services/pedido_service.py
```

### Automatizar com pre-commit

```yaml
# .pre-commit-config.yaml — roda antes de cada git commit
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.5.1
    hooks:
      - id: mypy
        additional_dependencies: [pydantic]
```

```bash
# Instalar pre-commit
pip install pre-commit
pre-commit install   # ativa os hooks no repositório
```

---

## Testes com pytest

```
tests/
├── conftest.py          ← fixtures compartilhadas
├── unit/
│   ├── test_pedido.py
│   └── test_utils.py
└── integration/
    └── test_pipeline.py
```

### Exemplo de teste unitário

```python
# tests/unit/test_pedido.py
import pytest
from decimal import Decimal
from meu_projeto.models.pedido import Pedido


def test_pedido_valido():
    pedido = Pedido(
        pedido_id="P-001",
        cliente_id="C-100",
        data_pedido="2024-06-15",
        valor_total=Decimal("150.00"),
        status="PAGO",
    )
    assert pedido.status == "PAGO"
    assert pedido.valor_total == Decimal("150.00")


def test_pedido_valor_zero_levanta_erro():
    with pytest.raises(ValueError, match="maior que zero"):
        Pedido(
            pedido_id="P-002",
            cliente_id="C-100",
            data_pedido="2024-06-15",
            valor_total=Decimal("0.00"),
            status="PENDENTE",
        )


def test_status_normalizado_para_maiusculo():
    pedido = Pedido(
        pedido_id="P-003",
        cliente_id="C-100",
        data_pedido="2024-06-15",
        valor_total=Decimal("50.00"),
        status="pago",   # minúsculo
    )
    assert pedido.status == "PAGO"  # normalizado
```

### Fixtures com conftest.py

```python
# tests/conftest.py
import pytest
from decimal import Decimal
from meu_projeto.models.pedido import Pedido


@pytest.fixture
def pedido_valido() -> Pedido:
    """Fixture reutilizável: pedido com dados válidos."""
    return Pedido(
        pedido_id="P-FIXTURE",
        cliente_id="C-FIXTURE",
        data_pedido="2024-01-01",
        valor_total=Decimal("100.00"),
        status="PENDENTE",
    )


@pytest.fixture
def lista_pedidos(pedido_valido: Pedido) -> list[Pedido]:
    return [pedido_valido] * 5
```

### Rodar testes

```bash
# Todos os testes
pytest

# Com cobertura
pytest --cov=src --cov-report=term-missing

# Apenas uma pasta
pytest tests/unit/

# Apenas um arquivo
pytest tests/unit/test_pedido.py

# Apenas um teste específico
pytest tests/unit/test_pedido.py::test_pedido_valido

# Mostrar output de print (útil em debug)
pytest -s
```

---

## Tratamento de Erros

```python
# ❌ Capturar Exception genérica esconde bugs
try:
    resultado = processar(dados)
except Exception:
    pass  # nunca faça isso

# ✅ Capturar erros específicos e tratar adequadamente
class ErroProcessamento(Exception):
    """Erro específico do domínio de processamento."""
    def __init__(self, mensagem: str, dados_problematicos: dict | None = None):
        super().__init__(mensagem)
        self.dados_problematicos = dados_problematicos


def processar(dados: list[dict]) -> int:
    try:
        return _executar_processamento(dados)
    except ValueError as e:
        logger.error("Dados inválidos: %s", e)
        raise ErroProcessamento(f"Falha na validação: {e}") from e
    except ConnectionError as e:
        logger.error("Falha de conexão: %s", e, exc_info=True)
        raise  # relançar erros de infraestrutura sem envolver
```

---

## Convenções de Nomenclatura

Python tem convenções bem definidas pela [PEP 8](https://peps.python.org/pep-0008/). Seguir essas convenções torna o código reconhecível por qualquer desenvolvedor Python, independente do projeto.

### Resumo Rápido

| Identificador | Convenção | Exemplo |
|---|---|---|
| Variável | `snake_case` | `total_pedidos`, `preco_unitario` |
| Função | `snake_case` | `calcular_desconto()`, `buscar_cliente()` |
| Método | `snake_case` | `self.processar()`, `self.to_dict()` |
| Parâmetro | `snake_case` | `def processar(dados: list, modo: str)` |
| Constante | `UPPER_SNAKE_CASE` | `MAX_TENTATIVAS`, `URL_BASE` |
| Classe | `PascalCase` | `PedidoService`, `ClienteRepository` |
| Módulo/arquivo | `snake_case` | `pedido_service.py`, `date_utils.py` |
| Pacote/pasta | `snake_case` (sem hífens) | `meu_projeto/`, `utils/` |
| Variável privada | `_snake_case` (prefixo `_`) | `_cache`, `_conexao` |
| Método privado | `_snake_case` | `def _validar_cpf(self)` |
| Variável "muito privada" (name mangling) | `__snake_case` | `self.__token` |

### Variáveis

```python
# ✅ Nomes descritivos — o leitor sabe o que é sem contexto
total_vendas_mes = 0
preco_com_desconto = preco_base * (1 - percentual_desconto)
lista_clientes_ativos: list[Cliente] = []
data_ultima_carga: datetime = datetime.now()

# ❌ Nomes genéricos ou abreviações crípticas
t = 0
p = preco * (1 - d)
lst = []
dlc = datetime.now()
```

**Exceções aceitáveis para nomes curtos:**
```python
for i, item in enumerate(pedidos):   # i é convencional em loops
    ...

df = pd.DataFrame(dados)             # df é universal em pandas
x, y = coordenadas                   # ok para desestruturação óbvia
```

### Funções e Métodos

Funções fazem coisas — use **verbo + substantivo**:

```python
# ✅ Verbo + substantivo claro
def calcular_total(itens: list[Item]) -> Decimal: ...
def buscar_cliente_por_id(cliente_id: int) -> Cliente | None: ...
def enviar_email_confirmacao(destinatario: str, pedido: Pedido) -> None: ...
def validar_cpf(cpf: str) -> bool: ...

# ❌ Sem verbo, nome ambíguo
def total(itens): ...
def cliente(id): ...
def email(d, p): ...
```

**Prefixos convencionais:**

| Prefixo | Uso | Exemplo |
|---|---|---|
| `get_` | Retorna algo (sem efeito colateral) | `get_preco()`, `get_settings()` |
| `set_` | Define um valor | `set_status()` |
| `buscar_` / `fetch_` | Busca em banco ou API | `buscar_pedido()` |
| `criar_` / `create_` | Cria novo recurso | `criar_usuario()` |
| `processar_` | Executa transformação | `processar_bronze()` |
| `validar_` / `is_` | Verifica condição | `validar_cpf()`, `is_ativo()` |
| `calcular_` | Computa valor | `calcular_desconto()` |
| `enviar_` | Envia dado externo | `enviar_notificacao()` |

### Classes

Use `PascalCase`. O nome deve ser um substantivo que descreve o que a classe representa:

```python
# ✅
class PedidoService: ...        # serviço de domínio
class ClienteRepository: ...    # repositório de acesso a dados
class ConfiguracaoEmail: ...    # configuração tipada
class ErroProcessamento(Exception): ...  # exceção de domínio

# ❌
class pedido_service: ...       # snake_case não é Python
class Processar: ...            # verbo, não substantivo
class Manager: ...              # genérico demais
```

**Sufixos por responsabilidade:**

| Sufixo | O que indica | Exemplo |
|---|---|---|
| `Service` | Lógica de negócio | `PedidoService` |
| `Repository` | Acesso a dados | `ClienteRepository` |
| `Router` / `Controller` | Camada HTTP | `PedidoRouter` |
| `Schema` / `Model` | Estrutura de dados | `PedidoSchema` |
| `Config` / `Settings` | Configuração | `AppSettings` |
| `Error` / `Exception` | Exceção de domínio | `PedidoInvalidoError` |
| `Handler` | Tratador de evento | `WebhookHandler` |
| `Client` | Integração externa | `CorreiosClient` |

### Constantes

Defina no topo do módulo ou em um arquivo `constants.py`:

```python
# src/meu_projeto/constants.py
MAX_TENTATIVAS_API = 3
TIMEOUT_SEGUNDOS = 30
STATUS_PENDENTE = "PENDENTE"
STATUS_PAGO = "PAGO"
STATUS_CANCELADO = "CANCELADO"

STATUSES_VALIDOS = frozenset({STATUS_PENDENTE, STATUS_PAGO, STATUS_CANCELADO})
```

### O que Evitar

```python
# ❌ Tipo no nome (redudante com anotações de tipo)
lista_de_pedidos: list[Pedido] = []   # a anotação já diz que é lista
str_nome: str = "João"                # o tipo está na anotação

# ✅
pedidos: list[Pedido] = []
nome: str = "João"


# ❌ Prefixo húngaro (legado de outras linguagens)
strNome = "João"
intIdade = 30
boolAtivo = True

# ✅
nome = "João"
idade = 30
ativo = True


# ❌ Número sequencial sem significado
pedido1 = ...
pedido2 = ...

# ✅ Use listas ou dicionários
pedidos = [pedido_novo, pedido_antigo]


# ❌ Abreviações que economizam 3 caracteres mas custam clareza
calc_desc_perc = ...
proc_ord = ...

# ✅
calcular_percentual_desconto = ...
processar_ordem = ...
```

---

## Checklist de Qualidade

```
Estrutura:
[ ] Código em src/, testes em tests/, nenhuma mistura
[ ] pyproject.toml como configuração central
[ ] .env.example commitado, .env no .gitignore
[ ] README.md com instruções de como rodar o projeto

Código:
[ ] Todas as funções públicas têm anotações de tipo
[ ] Todas as funções públicas têm docstring
[ ] Nenhum print() em código de produção — usar logging
[ ] Nenhuma credencial hardcoded — usar variáveis de ambiente
[ ] Erros específicos tratados, nunca Exception genérica silenciosa

Qualidade:
[ ] ruff check . passa sem erros
[ ] black --check . passa sem diferenças
[ ] mypy src/ passa sem erros
[ ] pytest passa com cobertura > 80%
```
