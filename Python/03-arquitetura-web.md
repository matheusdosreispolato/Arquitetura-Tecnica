# Arquitetura de Projetos Web em Python

> Projetos web precisam de uma estrutura que separe claramente as responsabilidades: o que recebe a requisição, o que executa a lógica de negócio e o que acessa dados. Este documento cobre os padrões para APIs e aplicações web com Python.

---

## Camadas de uma Aplicação Web

Toda aplicação web bem estruturada segue uma separação em camadas. Cada camada tem uma responsabilidade única e só se comunica com as camadas adjacentes:

```
Cliente (browser, app, outro serviço)
        ↓
┌─────────────────────────────────┐
│  Routers / Controllers          │  ← recebe a requisição, valida entrada
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│  Services / Use Cases           │  ← lógica de negócio
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│  Repositories / DAL             │  ← acesso a dados (banco, API, arquivo)
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│  Banco de Dados / APIs externas │
└─────────────────────────────────┘
```

| Camada | Responsabilidade | O que NÃO deve ter |
|---|---|---|
| Router/Controller | Receber requisição, validar schema, chamar service, retornar resposta | Lógica de negócio, queries SQL |
| Service | Orquestrar a lógica de negócio, aplicar regras, chamar repositórios | Detalhes de HTTP, queries diretas |
| Repository | Toda comunicação com banco de dados ou APIs externas | Lógica de negócio |
| Model/Schema | Definição das estruturas de dados (Pydantic, ORM) | Lógica de negócio |

---

## Estrutura de Projeto FastAPI

FastAPI é o padrão atual para APIs Python de alta performance:

```
minha_api/
│
├── src/
│   └── minha_api/
│       ├── __init__.py
│       ├── main.py                  ← criação do app FastAPI, registro de routers
│       ├── config.py                ← settings via pydantic_settings
│       ├── dependencies.py          ← dependências reutilizáveis (get_db, get_current_user)
│       │
│       ├── routers/                 ← endpoints agrupados por domínio
│       │   ├── __init__.py
│       │   ├── pedidos.py           ← GET/POST/PUT/DELETE /pedidos
│       │   ├── clientes.py
│       │   └── auth.py
│       │
│       ├── schemas/                 ← Pydantic models de request/response
│       │   ├── __init__.py
│       │   ├── pedido.py            ← PedidoCreate, PedidoResponse, PedidoUpdate
│       │   └── cliente.py
│       │
│       ├── services/                ← lógica de negócio
│       │   ├── __init__.py
│       │   ├── pedido_service.py
│       │   └── cliente_service.py
│       │
│       ├── repositories/            ← acesso a dados
│       │   ├── __init__.py
│       │   ├── pedido_repository.py
│       │   └── cliente_repository.py
│       │
│       ├── models/                  ← modelos ORM (SQLAlchemy)
│       │   ├── __init__.py
│       │   ├── base.py              ← Base declarativa do SQLAlchemy
│       │   ├── pedido.py
│       │   └── cliente.py
│       │
│       └── utils/                   ← helpers genéricos
│           ├── __init__.py
│           └── formatters.py
│
├── tests/
│   ├── conftest.py                  ← fixtures: TestClient, banco de teste
│   ├── unit/
│   │   ├── test_pedido_service.py
│   │   └── test_schemas.py
│   └── integration/
│       └── test_pedidos_api.py      ← testa os endpoints HTTP reais
│
├── alembic/                         ← migrações de banco de dados
│   ├── versions/
│   └── env.py
│
├── Dockerfile
├── docker-compose.yml
├── .env.example
└── pyproject.toml
```

---

## Exemplo Completo — Camada por Camada

### Schema (entrada e saída)

```python
# src/minha_api/schemas/pedido.py
from pydantic import BaseModel, field_validator
from decimal import Decimal
from datetime import datetime
from typing import Optional


class PedidoCreate(BaseModel):
    """Dados necessários para criar um pedido."""
    cliente_id: int
    produto_id: int
    quantidade: int
    observacao: Optional[str] = None

    @field_validator("quantidade")
    @classmethod
    def quantidade_positiva(cls, v: int) -> int:
        if v <= 0:
            raise ValueError("Quantidade deve ser maior que zero")
        return v


class PedidoResponse(BaseModel):
    """Dados retornados após criar ou consultar um pedido."""
    pedido_id: int
    cliente_id: int
    produto_id: int
    quantidade: int
    valor_total: Decimal
    status: str
    criado_em: datetime

    model_config = {"from_attributes": True}  # permite criar a partir de ORM model
```

### Router (HTTP)

```python
# src/minha_api/routers/pedidos.py
from fastapi import APIRouter, Depends, HTTPException, status
from minha_api.schemas.pedido import PedidoCreate, PedidoResponse
from minha_api.services.pedido_service import PedidoService
from minha_api.dependencies import get_pedido_service

router = APIRouter(prefix="/pedidos", tags=["Pedidos"])


@router.post("/", response_model=PedidoResponse, status_code=status.HTTP_201_CREATED)
async def criar_pedido(
    dados: PedidoCreate,
    service: PedidoService = Depends(get_pedido_service),
) -> PedidoResponse:
    """Cria um novo pedido."""
    return await service.criar(dados)


@router.get("/{pedido_id}", response_model=PedidoResponse)
async def buscar_pedido(
    pedido_id: int,
    service: PedidoService = Depends(get_pedido_service),
) -> PedidoResponse:
    """Retorna um pedido pelo ID."""
    pedido = await service.buscar_por_id(pedido_id)
    if pedido is None:
        raise HTTPException(status_code=404, detail="Pedido não encontrado")
    return pedido
```

### Service (lógica de negócio)

```python
# src/minha_api/services/pedido_service.py
import logging
from minha_api.schemas.pedido import PedidoCreate, PedidoResponse
from minha_api.repositories.pedido_repository import PedidoRepository

logger = logging.getLogger(__name__)


class PedidoService:
    def __init__(self, repository: PedidoRepository) -> None:
        self._repo = repository

    async def criar(self, dados: PedidoCreate) -> PedidoResponse:
        """Valida regras de negócio e cria o pedido."""
        logger.info("Criando pedido para cliente %d", dados.cliente_id)

        # Regra de negócio: verificar estoque
        estoque = await self._repo.buscar_estoque(dados.produto_id)
        if estoque < dados.quantidade:
            raise ValueError(f"Estoque insuficiente: {estoque} unidades disponíveis")

        pedido = await self._repo.criar(dados)
        logger.info("Pedido criado: %d", pedido.pedido_id)
        return pedido

    async def buscar_por_id(self, pedido_id: int) -> PedidoResponse | None:
        return await self._repo.buscar_por_id(pedido_id)
```

### Repository (acesso a dados)

```python
# src/minha_api/repositories/pedido_repository.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from minha_api.models.pedido import PedidoModel
from minha_api.schemas.pedido import PedidoCreate, PedidoResponse


class PedidoRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def criar(self, dados: PedidoCreate) -> PedidoResponse:
        pedido = PedidoModel(**dados.model_dump())
        self._session.add(pedido)
        await self._session.commit()
        await self._session.refresh(pedido)
        return PedidoResponse.model_validate(pedido)

    async def buscar_por_id(self, pedido_id: int) -> PedidoResponse | None:
        result = await self._session.execute(
            select(PedidoModel).where(PedidoModel.pedido_id == pedido_id)
        )
        pedido = result.scalar_one_or_none()
        return PedidoResponse.model_validate(pedido) if pedido else None

    async def buscar_estoque(self, produto_id: int) -> int:
        # consulta à tabela de estoque
        ...
```

### main.py (ponto de entrada)

```python
# src/minha_api/main.py
from fastapi import FastAPI
from minha_api.routers import pedidos, clientes, auth

app = FastAPI(
    title="Minha API",
    version="1.0.0",
    description="API de gestão de pedidos",
)

# Registrar routers
app.include_router(pedidos.router)
app.include_router(clientes.router)
app.include_router(auth.router)


@app.get("/health")
async def health_check() -> dict:
    return {"status": "ok"}
```

---

## Padrões de Nomenclatura para Aplicações Web

### URLs e Endpoints

```
# Padrão REST: substantivo no plural, sem verbos
GET    /pedidos           ← listar
GET    /pedidos/{id}      ← buscar um
POST   /pedidos           ← criar
PUT    /pedidos/{id}      ← substituir completamente
PATCH  /pedidos/{id}      ← atualizar parcialmente
DELETE /pedidos/{id}      ← deletar

# Sub-recursos
GET    /clientes/{id}/pedidos   ← pedidos de um cliente específico
POST   /pedidos/{id}/cancelar   ← ação que não é CRUD (verbo é aceitável aqui)
```

### Respostas HTTP

| Situação | Status code |
|---|---|
| Recurso criado com sucesso | `201 Created` |
| Sucesso sem corpo de resposta | `204 No Content` |
| Recurso não encontrado | `404 Not Found` |
| Dados de entrada inválidos | `422 Unprocessable Entity` |
| Sem permissão | `403 Forbidden` |
| Não autenticado | `401 Unauthorized` |
| Erro interno do servidor | `500 Internal Server Error` |

---

## Autenticação com JWT

```python
# src/minha_api/dependencies.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt

security = HTTPBearer()


def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
) -> dict:
    try:
        payload = jwt.decode(
            credentials.credentials,
            key=get_settings().jwt_secret,
            algorithms=["HS256"],
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expirado")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Token inválido")
```

---

## Testes de API

```python
# tests/integration/test_pedidos_api.py
import pytest
from httpx import AsyncClient
from minha_api.main import app


@pytest.mark.asyncio
async def test_criar_pedido_valido():
    async with AsyncClient(app=app, base_url="http://test") as client:
        resposta = await client.post("/pedidos", json={
            "cliente_id": 1,
            "produto_id": 10,
            "quantidade": 2,
        })
    assert resposta.status_code == 201
    dados = resposta.json()
    assert dados["status"] == "PENDENTE"
    assert dados["quantidade"] == 2


@pytest.mark.asyncio
async def test_buscar_pedido_inexistente():
    async with AsyncClient(app=app, base_url="http://test") as client:
        resposta = await client.get("/pedidos/999999")
    assert resposta.status_code == 404
```

---

## Docker para Desenvolvimento

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY pyproject.toml .
RUN pip install -e ".[dev]"
COPY src/ src/

CMD ["uvicorn", "minha_api.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./src:/app/src   # hot reload em desenvolvimento
    env_file: .env
    depends_on:
      - banco

  banco:
    image: postgres:16
    environment:
      POSTGRES_DB: minha_api
      POSTGRES_USER: app
      POSTGRES_PASSWORD: senha_dev
    ports:
      - "5432:5432"
```

---

## Estrutura por Tipo de Projeto Web

### API simples (sem ORM)

```
api_simples/
├── src/api_simples/
│   ├── main.py          ← FastAPI app + routers inline (ok para projetos pequenos)
│   ├── config.py
│   ├── database.py      ← conexão direta com pyodbc/asyncpg
│   └── schemas.py       ← todos os Pydantic models num só arquivo
```

### API completa (com ORM + autenticação)

Usar a estrutura completa mostrada acima com `routers/`, `services/`, `repositories/`, `models/` e `schemas/` separados.

### BFF (Backend for Frontend)

```
bff/
├── src/bff/
│   ├── routers/         ← endpoints que o frontend consome
│   ├── clients/         ← HTTP clients para os microserviços internos
│   └── aggregators/     ← combina dados de múltiplos serviços para uma tela
```

---

## Relacionado

- `Python/01-estrutura-projeto.md` — fundamentos de estrutura de projeto Python
- `Python/02-boas-praticas.md` — tipagem, docstrings, testes, logging
- `IA/01-estrutura-projeto-ia.md` — estrutura para projetos que incluem um LLM
