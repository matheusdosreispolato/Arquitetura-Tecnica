# IA — Projetos de Inteligência Artificial

> Guia para estruturar, desenvolver e avaliar projetos que utilizam modelos de linguagem (LLMs). Cobre desde a organização do código até engenharia de prompts, integração com APIs e avaliação de qualidade.

---

## Estrutura desta pasta

```
IA/
├── README.md                    ← este arquivo
├── 01-estrutura-projeto-ia.md   ← organização de pastas, versionamento de prompts, configuração
├── 02-engenharia-prompts.md     ← definições, técnicas, economia de tokens, templates prontos
├── 03-integracao-api.md         ← SDKs Anthropic/OpenAI, streaming, retries, custo
├── 04-evals.md                  ← avaliação de qualidade: datasets, métricas, evals automatizados
└── 05-claude-code-codex.md      ← usar Claude Code/Codex no projeto via AGENTS.md e README.md
```

---

## Documentos

| Arquivo | Conteúdo |
|---|---|
| [01 — Estrutura de Projeto IA](./01-estrutura-projeto-ia.md) | Layout de pastas, onde guardar prompts, ferramentas, memória e avaliações |
| [02 — Engenharia de Prompts](./02-engenharia-prompts.md) | Definições, técnicas (few-shot, CoT, roles), economia de tokens, templates de system prompt |
| [03 — Integração com APIs LLM](./03-integracao-api.md) | SDKs Anthropic/OpenAI, streaming, retries com backoff, async, tool use, controle de custo |
| [04 — Evals](./04-evals.md) | Tipos de avaliação (exact match, model-graded), datasets, métricas, quando rodar evals |
| [05 — Claude Code e Codex](./05-claude-code-codex.md) | Como estruturar AGENTS.md e README.md para que agentes de código trabalhem bem no projeto |

---

## Princípio central

> **O modelo é um componente, não o produto.** Um projeto de IA bem estruturado é igual a qualquer outro projeto de software: organizado, testável, versionado e observável. O que muda é que parte da lógica está nos prompts — e prompts precisam ser tratados com o mesmo cuidado que código.

---

## Referência Rápida

```bash
# Instalar SDKs
pip install anthropic openai

# Variáveis de ambiente (nunca hardcodar chaves)
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."

# Rodar evals
python -m pytest evals/
```

---

## Relacionado

- `Python/` — estrutura e boas práticas de projetos Python (base para qualquer projeto de IA)
- `Agents/` — uso de agentes: ReAct, frameworks (LangChain, LangGraph, CrewAI, AutoGen)
